일자 : 2025.04.14.월<br>
**📌 TIL: FileManager 기반 캐싱을 통해 홈 초기 로딩 속도 3배 개선한 경험**

## 🛠 문제 상황

앱을 출시한 이후, 홈 화면에서 게시글 리스트가 늦게 로딩되는 현상을 발견하였습니다.
특히 **네트워크가 불안정한 환경에서는 빈 리스트 화면이 먼저 보여지고, 몇 초 뒤에야 게시글이 나타나는 문제**가 발생하였습니다.

이러한 구조는 사용자에게 **앱이 멈춘 것 같은 인상**을 줄 수 있으며, **데이터가 없는 줄 착각하는 사용자 경험을 유발할 수 있다**고 판단하였습니다.
또한, 앱 전체적으로 **API 호출 횟수를 최대한 줄일 수 있는 방법**에 대해서도 함께 고민하게 되었습니다.

## ✅ 해결 방법

- FileManager 기반의 JSON 캐시를 도입하여, **게시글 리스트를 디스크에 저장**하도록 하였습니다.
- 현재 게시글 리스트에 우선적으로 적용했지만, 이후 확장성을 위해 CacheType enum과 CacheService 유틸 클래스를 별도로 설계하여, **타입 안정성과 확장성, 유지보수성을 확보**하였습니다.

### CacheService 코드

```swift
import Foundation

enum CacheType {
    case postList
    var fileName: String {
        switch self {
        case .postList:
            return "PostListCache.json"
        }
    }

    var maxAge: TimeInterval {
        switch self {
        case .postList:
            return 60 * 15 // 15분
        }
    }
}

final class CacheService {
    enum CacheError: Error {
        case cacheDirectoryNotFound
    }
    static let shared = try? CacheService()
    private let cacheDirectory: URL

    init() throws {
        guard let directory = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask).first else {
            throw CacheError.cacheDirectoryNotFound
        }
        self.cacheDirectory = directory
    }

    /// 저장
    func save<T: Codable>(_ object: T, to type: CacheType) {
        let fileURL = cacheDirectory.appendingPathComponent(type.fileName)
        do {
            let data = try JSONEncoder().encode(object)
            try data.write(to: fileURL)
        } catch {
            print("❌ 캐시 저장 실패: \(error)")
        }
    }

    /// 로드
    func load<T: Codable>(from type: CacheType) -> T? {
        let fileURL = cacheDirectory.appendingPathComponent(type.fileName)
        do {
            let data = try Data(contentsOf: fileURL)
            let object = try JSONDecoder().decode(T.self, from: data)
            return object
        } catch {
            print("❌ 캐시 로드 실패: \(error)")
            return nil
        }
    }

    /// 캐시 유효성 검사 (초 단위로 만료 시간 지정)
    func isCacheValid(for type: CacheType) -> Bool {
        let fileURL = cacheDirectory.appendingPathComponent(type.fileName)
        guard let attributes = try? FileManager.default.attributesOfItem(atPath: fileURL.path),
              let modificationDate = attributes[.modificationDate] as? Date else {
            return false
        }

        let age = Date().timeIntervalSince(modificationDate)
        return age < type.maxAge
    }

    /// 삭제
    func clear(type: CacheType) {
        let fileURL = cacheDirectory.appendingPathComponent(type.fileName)
        try? FileManager.default.removeItem(at: fileURL)
    }
}
```

### 게시물 리스트 캐싱 정책

- 앱 실행시에는 캐시가 있으면 즉시 화면에 렌더링하고, 없을 시에는 API 호출결과를 렌더링하도록하였습니다.
- 캐시가 있을 경우 우선적으로 캐시 내용을 화면에 렌더링 후 `isCacheValid`를 통해 유효성 검사를 하여 마지막 업데이트 시간이 지정한 캐시 만료 시간보다 오래됐을 경우 API를 호출하여 화면 데이터 및 캐시 내용을 업데이트 합니다.
- 사용자가 직접 위에서 아래로 스와이프하여 새로고침을 하였을 경우는 무조건 API를 새로 호출하고, 캐시를 업데이트합니다.
- 0페이지 외에 이후 페이지들은 따로 캐싱하지 않았습니다.

즉 내용을 표로 정리해보면 다음과 같습니다:

| **상황**                          | **캐시 존재 여부**     | **동작**                                  | **비고**                    |
| --------------------------------- | ---------------------- | ----------------------------------------- | --------------------------- |
| 앱 실행 시                        | ✅ 있음                | 캐시 우선 렌더링 → 만료 시 API 호출       | isCacheValid 기준으로 판단  |
| 앱 실행 시                        | ⚠️ 있음<br>(캐시 만료) | 캐시 먼저 렌더링 → 이후 API 호출하여 갱신 | 사용자 체감 속도 향상       |
| 앱 실행 시                        | ❌ 없음                | 바로 API 호출                             | 캐시 없이 데이터 로딩       |
| 사용자 새로고침 (Pull to Refresh) | 관계 없음              | 무조건 API 호출 → 캐시 갱신               | 항상 최신 데이터 확보       |
| 페이징 (0페이지 제외)             | 관계 없음              | API 호출                                  | 이후 페이지는 캐싱하지 않음 |

## 🎯 성능테스트 결과

성능 테스트는 다음과 같이 진행했습니다.

### 측정 방식

_ReactorKit을 사용했기에 뷰모델을 Reactor 기준으로 설명합니다._

- 앱 재빌드와 로그아웃 후 로그인 두 가지 시나리오에서 각각 3회씩 측정하였습니다.
- Date()를 이용한 시간 측정
  - Reactor(ViewModel) 초기화 시 startDate 저장
  - reduce 에서 posts 를 재설정할 시 첫 재설정 (현재 posts가 비어있고, 재설정을 한다면) 시작시간과 현재시간의 차이 계산

```swift
    func reduce(state: State, mutation: Mutation) -> State {
        var newState = state
        switch mutation {
        case .loadPost(let posts, let append):
            if currentState.posts.isEmpty {
                let elapsed = Date().timeIntervalSince(startTime)
                print("⏱️ 초기 API 게시글 로드 완료 (소요 시간: \(elapsed)초)")
            }
            ...
```

### 측정 결과 요약

| **시나리오**      | **캐싱 미적용 평균** | **캐싱 적용 평균** | **개선 폭**        |
| ----------------- | -------------------- | ------------------ | ------------------ |
| 재빌드 후 홈 진입 | 0.16초               | 0.14초             | 약 1.1배           |
| 로그아웃 → 로그인 | 0.07초               | **0.023초**        | **약 3배 향상** ✅ |

## 📌 마무리

캐시를 통해 **빈 화면 없이 게시글을 빠르게 보여주는 경험을 제공**할 수 있었고, 특히 **네트워크 상태가 좋지 않은 환경에서 캐시의 효과가 더욱 크게 체감**되었습니다.
처음에는 네트워크 호출을 줄이기 위해 캐시를 적용하려했지만, **사용자에게 빠르게 보여주는 것**을 통해 사용자 경험을 크게 향상시킬 수 있는 좋은 방식임을 깨달았습니다.

또한 아직 앱의 실사용자 수가 적기 때문에, 현재 단계에서는 API 호출량 감소에 따른 정량적인 효과를 명확하게 테스트하거나 확인하기는 어려운 상황입니다.

그러나 이번 캐싱 구조는 **사용자 수가 증가했을 때 API 호출 수를 획기적으로 줄이고, 서버 부하를 크게 완화할 수 있는 기반**이 될 것이라 기대하고 있습니다.

예를 들어, 일간 활성 사용자(DAU)가 10,000명이고 그 중 70%가 하루 평균 3회 홈 화면에 진입한다고 가정하면,캐싱이 적용되지 않은 경우 약 **21,000회의 API 호출**이 발생하게 됩니다.

하지만 캐싱을 통해 절반만 캐시로 처리된다고 해도 **하루 약 10,500회의 API 호출을 줄일 수 있어**,
이는 **트래픽 비용 절감과 서버 안정성 확보에 있어 상당한 이점을 제공할 수 있는 구조**입니다.

앞으로는 게시글 상세, 찜 목록 등 **다른 뷰에도 상황에 맞는 캐싱 전략을 적용해볼 계획**입니다.
