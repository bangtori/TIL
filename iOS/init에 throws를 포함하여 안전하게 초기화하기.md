일자 : 2025.02.03.월<br>
 📌 **TIL: init에서 throws를 사용하여 WebSocket 서비스 초기화하기**

## 🛠 문제 상황
WebSocket 서비스를 구현할 때, **서버 URL을 번들에서 가져와 초기화**하는 과정이 필요했다.
서버 URL은 연결을 위해 필수적이므로 init 과정에서 초기화하도록 의도하였다.
하지만 번들에서 URL을 가져오는 과정에서 다음과 같은 문제가 발생할 가능성이 있었다.

- 번들 내 plist 또는 설정 파일에서 URL이 **누락**되었을 경우
- 가져온 URL이 **유효한 형식이 아닐 경우**
- URL을 가져오는 과정에서 **예기치 못한 오류**가 발생할 경우

따라서init 과정에서 URL 유효성을 검사하고 문제가 발생할 경우 적절한 핸들링을 하도록 처리하여야 했다.

## ✅ 해결 방법
WebSocket 서비스 클래스를 만들 때, **초기화 과정에서 URL 유효성 검사를 수행**하고,
**문제가 발생하면 throws를 활용해 예외를 발생**시키도록 구현했다.

```swift 
import Foundation

enum WebSocketError: LocalizedErrorWithCode {
	// LocalizedErrorWithCode 는 LocalizedError 프로토콜을 준수하는 커스텀 에러 
    case invalidURL
    // ...웹 소켓 통신 시 발생할 수 있는 여러 에러
}
final class SocketService {
    static var shared: SocketService?
    private let serverURL: URL
    // .. 생략
    init() throws {
	    guard let urlString = Bundle.main.socketURL, let url = URL(string: urlString) else {
		    throw SocketError.invalidURL
		    }
		    
        self.serverURL = url
        let request = URLRequest(url: serverURL)
        self.socket = WebSocket(request: request)
        self.socket?.delegate = self
    }
    // .. 생략
}
```

이제 WebSocketService를 사용할 때 try를 통해 오류를 처리할 수 있도록 만들었다.

```swift
do {
	SocketService.shared = try SocketService()
	print("✅ SocketService 인스턴스 생성 성공"
	SocketService.shared?.connect()  // WebSocket 연결
} catch {
	print("❌ SocketService 초기화 실패: \(error)")
}
```
## 🎯 init에서 throws를 사용한 이점 
1. **객체의 잘못된 생성 방지**
   URL이 유효하지 않거나 누락되었을 경우 **객체가 생성되지 않도록 막아**, 이후 발생할 수 있는 예외적인 상황을 사전에 방지함.

2. **초기화 과정에서 에러 감지 가능**
   일반적인 init에서는 오류를 반환할 방법이 없지만, throws를 사용하면 **초기화 단계에서 바로 문제를 감지**하고 대응할 수 있음.

3. **오류 원인 분리 및 명확한 예외 처리 가능**
   WebSocketError.missingURL, WebSocketError.invalidURL과 같이 다양한 오류 케이스를 분리하여 **문제가 발생한 원인을 쉽게 파악**할 수 있음.

4. **서비스 사용 시 안정성 향상**
   try를 통해 WebSocket을 사용할 때 반드시 오류를 체크하도록 강제됨.
   예기치 않은 크래시를 방지하고, **안전한 서비스 동작을 보장**할 수 있음.


## 💡 추가로 깨달은 점
**📌 1. 기본 라이브러리에서 init에 throws가 적용된 사례**

이번 경험을 통해 **Swift 기본 라이브러리에서도 객체 생성 시** throws**를 활용하는 패턴을 쉽게 찾아볼 수 있다는 점**을 알게 되었다.
예를 들어, FileManager, JSONDecoder, URL 등 여러 기본 클래스의 init에서도 try를 사용해야 하는데, 이는 초기화 과정에서 실패 가능성을 염두에 둔 설계 방식이라는 걸 다시금 체감할 수 있었다.

```swift
do {
    let data = try Data(contentsOf: URL(string: "https://example.com")!) // URL이 잘못되었으면 throws
    let json = try JSONSerialization.jsonObject(with: data, options: []) // JSON 파싱 오류 발생 가능
} catch {
    print("오류 발생: \(error)")
}
```

Swift에서도 **throws를 사용해 안전한 객체 생성을 유도하는 것이 일반적임을 확인할 수 있었다.**

**📌 2. 유효하지 않은 초기화 값을 사용한 핸들링과 throws의 비교**

기존에는 초기화 시 기본적으로 유효하지 않은 값(예: -1) 할당 혹은 옵셔널 변수로 선언 후, 이후에 이를 검사하는 방식이 많이 사용했다.

```swift
class User {
    let age: Int
    
    init(age: Int) {
        self.age = (1...100).contains(age) ? age : -1  // 잘못된 값이면 -1 할당
    }
    func checkValidation() {
        if age == -1 {
            print("잘못된 나이 값입니다.")
        }
    }
}
```

**이 방식의 문제점**
- 초기화는 됐지만, 이후에 따로 checkValidation()을 호출해야 오류를 감지할 수 있음. 여러 메서드에서 범용적으로 age를 사용되는 경우 모든 메서드에서 checkValidation()을 호출하여 검증하는 과정을 포함해야함
- age == -1이라는 매직 넘버를 사용하여 오류를 표현하는 것은 코드 가독성이 떨어짐
- 객체가 “잘못된 상태”로 생성될 수 있음 (나이는 1~100이어야 하는데 -1이 들어가 있음)

**옵셔널 파라미터로 초기화하는 방식도 비슷한 맥락의 문제를 가지고 있음**
- 파라미터 사용 시점에 checkValidation처럼 언랩핑 과정으로 nil 체크 후 사용할 수 있어 불필요한 코드가 중복됨
- 특히 값이 무조건 있어야 하는 파라미터의 경우 옵셔널은 유효하지 않은 값임. 즉 유효하지 않은 값으로 기본값을 할당하는 방식과 똑같음

대신, throws를 활용하면 **잘못된 값으로 객체가 생성되지 않도록 강제할 수 있다.**

```swift
enum UserError: Error {
    case invalidAge
}

class User {
    let age: Int
    
    init(age: Int) throws {
        guard (1...100).contains(age) else {
            throw UserError.invalidAge
        }
        self.age = age
    }
}

do {
    let user = try User(age: -1)  // 오류 발생, 객체 생성 자체가 안됨
} catch {
    print("사용자 생성 실패: \(error)")
}
```

✅ **이 방식의 장점**
- 잘못된 데이터로 객체가 생성되지 않아 불필요하고 중복되는 검증 과정을 줄일 수 있음 
- 명확한 에러 메시지를 제공하여 디버깅이 쉬움
- 불필요한 기본값(-1) 할당 없이 안전한 코드 작성 가능
## 📌 마무리
이번 경험을 통해 **초기화 과정에서 실패 가능성이 있는 경우 throws를 사용하면 코드의 안정성이 높아진다는 점을 다시 한 번 깨달았다.**

- Swift의 기본 라이브러리에서도 throws를 사용하는 패턴이 많다는 것을 확인했다.
- 기존처럼 무의미한 초기화 값을 넣고 사후 검증하는 방식보다는, 초기화 과정에서 throws를 통해 오류를 처리하는 것이 훨씬 깔끔하고 안전한 코드 작성 방법이라는 것을 알게 되었다.
- 
앞으로는 **객체 생성 시 유효성 검사가 필요한 경우 throws를 적극적으로 활용**해야겠다고 생각했다
