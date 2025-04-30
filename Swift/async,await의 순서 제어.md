**📌 TIL: async await의 비동기 순서 제어**

## 🛠  탐구 계기

프로젝트를 진행하던 중, **비동기 작업 간의 순서를 보장해야 하는** 상황이 발생했습니다.  
예를 들어, 소켓 통신 과정에서는 `connect`를 먼저 호출하고 성공 시 `CONNECT` 프레임을 전송한 후, 그 다음에 `SUBSCRIBE` 프레임을 보내 구독을 진행해야 했습니다.

물론 RxSwift를 사용하고 있는 프로젝트였기 때문에 Rx를 이용해 순서 제어가 가능했지만,<br>
해당 **소켓 서비스 클래스는 Rx 방식이 아닌 일반 방식**으로 작성되고 있었기에 전체 구조를 Rx로 수정하기보다  **Swift의 async/await을 적용하여** 보다 자연스럽고 명확하게 비동기 흐름을 제어해보기로 결정했습니다.

---

## ✅ 내용 정리

_이 글에서는 `async/await` 사용법에 초점을 맞추며, 동기와 비동기에 대한 일반적인 설명은 생략합니다._

기존에는 보통 GCD와 completion handler를 이용해 비동기 코드를 작성했습니다.<br>
하지만 이러한 방식은 **콜백 중첩으로 인한 가독성 저하**와 **컨텍스트 스위칭 비용** 등 여러 단점을 가지고 있었습니다.

이러한 문제를 해결하기 위해 Swift Concurrency가 등장했고, <br>
그 핵심 문법인 `async`와 `await`을 통해 **비동기 작업을 마치 동기처럼 깔끔하게** 표현할 수 있게 되었습니다.

---

### async 란?

- **비동기 함수(비동기 작업)를 정의**할 때 사용하는 키워드입니다.

```swift
func fetchData() async -> String {
    // 비동기 작업 수행
    return "결과"
}
```

### await 란?

- async 함수 내부에서 **다른 비동기 함수의 결과를 기다릴 때** 사용하는 키워드입니다.
- await을 붙이면 해당 비동기 작업이 **완료될 때까지 “해당 줄에서 기다립니다”**, 즉 코드 실행을 일시 중단합니다.
- 이 기다림은 백그라운드에서 이루어지며, 메인 스레드를 블로킹하지 않습니다.
- **정리하자면** async는 비동기 함수임을 나타내고, await은 실제로 기다리는 역할을 합니다.

```swift
func setup() async {
	await fetchData1()
	await fetchData2()
}


Task {
    let data = await fetchData()
    print(data)
}
```

### 그렇다면 어떻게 순서를 제어하는걸까?

비동기 작업은 본래 **작업 완료 순서가 예측되지 않는 비동시적 흐름**입니다.<br>
하지만 await을 사용하면 다음과 같은 일이 일어납니다:

1. await someAsyncTask() 를 만나면 → Swift는 해당 작업이 완료될 때까지 **그 줄에서 대기**
2. 완료되면 → **결과를 받아 다음 줄로 이동**
3. 따라서 코드 흐름은 **동기 코드처럼 순차적으로 실행**

즉, await은 비동기 작업의 결과를 **기다리게 함으로써 실행 순서를 보장**합니다.

**⚠️ 주의할 점**<br>
이때 await이 기다리는 단위는 **스레드가 아닌 Task 단위**입니다.<br>
즉, **같은 Task 안에서 이전 작업의 완료를 기다리는 구조**입니다.<br>
메인 스레드는 블로킹되지 않으며, 다른 Task나 UI 작업 등은 계속 진행됩니다.

여기서 말하는 **같은 Task**란,

- 하나의 async 함수 블록 안이거나,
- 하나의 Task {} 블록 안을 의미합니다.

예시 :

```swift
import Foundation
import PlaygroundSupport

PlaygroundPage.current.needsIndefiniteExecution = true  // 플레이그라운드 꺼지지 않도록 하는 설정

func connectSocket() async throws {
    print("1. 소켓 연결 시도 중...")
    try await Task.sleep(nanoseconds: 1_000_000_000) 
    print("2. 소켓 연결 성공!")
}

func sendConnectFrame() async throws {
    print("3. CONNECT 프레임 전송")
    try await Task.sleep(nanoseconds: 500_000_000)
}


func sendSubscribeFrame() async throws {
    print("4. SUBSCRIBE 프레임 전송")
    try await Task.sleep(nanoseconds: 500_000_000)

}


func setup() async throws {
    try await connectSocket()
    try await sendConnectFrame()
    try await sendSubscribeFrame()
    print("✅ 순서대로 완료!")
}

Task {
    try await setup()
}

print("Task 바깥의 출력")
```

출력

```plain
1. 소켓 연결 시도 중...
Task 바깥의 출력
2. 소켓 연결 성공!
3. CONNECT 프레임 전송
4. SUBSCRIBE 프레임 전송
✅ 순서대로 완료!
```

- Task 블록 안에서 실행된 비동기 함수들은 await으로 인해 순차적으로 실행됩니다.
- 반면 Task 바깥에 있는 print()는 Task의 실행과 무관하게 **즉시 실행**됩니다.

## 💡 추가로 깨달은 점

비동기 작업을 병렬로 실행하려면 Task를 여러 개 만들 수도 있지만, <br>
`async let` 을 사용하는 방법이 있음을 알게되었습니다.

```swift
import Foundation

func fetchImage() async -> String {
    print("이미지 다운로드 시작")
    try? await Task.sleep(nanoseconds: 1_000_000_000)
    return "🖼 이미지 완료"
}

func fetchText() async -> String {
    print("텍스트 다운로드 시작")
    try? await Task.sleep(nanoseconds: 1_000_000_000)
    return "📄 텍스트 완료"
}

func loadData() async {
    let start = DispatchTime.now() // ⏱ 시작 시각

    async let image = fetchImage()
    async let text = fetchText()

    let (img, txt) = await (image, text)

    let end = DispatchTime.now()   // ⏱ 끝나는 시각

    let elapsed = Double(end.uptimeNanoseconds - start.uptimeNanoseconds) / 1_000_000_000

    print(img, txt)
    print("⏱ 전체 걸린 시간: \(String(format: "%.2f", elapsed))초")
}
```

출력

```plain
이미지 다운로드 시작
텍스트 다운로드 시작
🖼 이미지 완료 📄 텍스트 완료
⏱ 전체 걸린 시간: 1.07초
```

출력시간은 약간의 오차가 있을 수 있지만, 두 작업이 병렬적으로 실행되어 약 1초 동안 실행된 것을 확인할 수 있습니다.

만약 loadData 함수를 다음과 같이 바꾼다면:

```swift
func loadData() async {

    let start = DispatchTime.now() // ⏱ 시작 시각

    let image = await fetchImage()
    let text = await fetchText()
   
    let end = DispatchTime.now()   // ⏱ 끝나는 시각
    let elapsed = Double(end.uptimeNanoseconds - start.uptimeNanoseconds) / 1_000_000_000

    print(image, text)
    print("⏱ 전체 걸린 시간: \(String(format: "%.2f", elapsed))초")
}
```

```plain
이미지 다운로드 시작
텍스트 다운로드 시작
🖼 이미지 완료 📄 텍스트 완료
⏱ 전체 걸린 시간: 2.13초
```

이미지 다운로드 시작, 텍스트 다운로드 시작 이 동시에 출력되던 이전 코드와 달리 1초 텀을 가지고 출력되며 전체 걸린 시각도 2초대로 늘어남을 확인할 수 있었습니다.

### 마무리 요약

| **개념**  | **설명**                                             |
| --------- | ---------------------------------------------------- |
| async     | 비동기 함수를 정의할 때 사용                         |
| await     | 비동기 작업이 끝날 때까지 기다리고, 순서 제어        |
| Task      | 비동기 흐름을 실행할 수 있는 컨텍스트                |
| async let | 여러 비동기 작업을 병렬로 실행할 수 있는 간편한 방식 |
| 순서 제어 | 같은 Task 안에서 await을 사용하면 순차적 실행 보장   |
| 병렬 실행 | async let, 여러 Task로 가능하며, 시간 단축 효과 있음 |

## 📌 마무리

async/await 에 대해서는 그동안 얕게 알고 있었지만, 이번에 직접 **비동기 작업의 순서를 제어**해보면서 기 코드의 흐름을 어떻게 제어하고, 병렬성과 순차성을 어떻게 구분해야 하는지\*\*에 대해 깊이 있게 이해할 수 있었습니다.

특히 실제 앱 개발 중 순서 보장이 필요한 부분에 적용했을 때,
**코드의 가독성과 안정성이 눈에 띄게 향상됨을 체감**할 수 있었고,
기존의 복잡한 completion handler 방식보다 훨씬 직관적인 흐름으로 코드를 구성할 수 있다는 점이 인상 깊었습니다.

또한, 이번 학습을 통해 새롭게 async let이라는 개념을 알게 되었고,
이를 활용하면 성능 병목이 발생할 수 있는 작업들도 **효율적으로 병렬 처리**할 수 있을 것이라 기대됩니다.

앞으로는 기존의 GCD나 completion handler 방식보다
async/await 기반의 코드를 **우선적으로 고려하고 적극적으로 활용**해봐야겠다고 생각했습니다.
