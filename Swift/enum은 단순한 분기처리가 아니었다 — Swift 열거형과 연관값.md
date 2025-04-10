**📌 TIL: enum은 단순한 분기처리가 아니었다 — Swift 열거형과 연관값**

## 🛠  탐구 계기

ReactorKit을 사용하면서 Action, Mutation 열거형(enum) 내부에서 `case inputChanged(String)` 처럼 괄호 안에 값이 들어가는 형태를 자주 보게 되었습니다.

처음에는 단순한 문법이라고 생각했지만, “왜 case에 괄호가 붙을까?”라는 의문이 생겼고, 이를 계기로 Swift의 **연관값(Associated Value)** 개념을 알게 되었습니다.

## ✅ 내용 정리

리액터킷의 형식 기반으로 정리하였으며, 열거형과 연관값의 내용이므로 리액터킷의 자세한 코드나 설명은 생략합니다.

### 열거형(enum)의 기본 형태

Swift에서 열거형은 다음과 같은 형태로 자주 사용됩니다.

```swift
enum Direction {
    case up
    case down
    case left
    case right
}
```

이처럼 상태를 나열하여 조건 분기 등에 활용할 수 있습니다. 하지만 이는 열거형의 가장 기본적인 사용 방식일 뿐입니다.

###  연관값(Associated Values)이란?

Swift에서는 열거형의 각 case가 **값(value)을 가질 수 있는** 기능이 있습니다. 이를 **연관값**이라고 합니다.

연관값을 사용하면 enum 하나로 상태뿐 아니라 그에 필요한 데이터를 함께 전달할 수 있습니다.

리액터킷의 코드로 예시를 들면:

```swift
enum Action {
    case login(id: String, password: String)
    case logout
}
```

이처럼 **case 옆에 원하는 타입을 튜플 형태로 명시하여** 선언할 수 있습니다.

해당 열거형을 생성하여 사용하는 방법은 다음과 같습니다:

```swift
let action = Action.login(id: "test", password: "1234")
```

그리고 리액터킷에서는 각각의 Action과 Mutation을 처리하는 mutate, reduce 함수에서는 다음과 같이 switch문을 사용하여 값을 처리합니다.

```swift
switch action {
case .login(let id, let password):
    print("로그인 시도 - ID: \(id), PW: \(password)")
case .logout:
    print("로그아웃")
}
```

기존 열거형은 상태만 구분할 수 있어, 값을 직접 전달하기 어려웠습니다.  
하지만 연관값을 사용하면 상황에 따라 다양한 데이터를 함께 전달할 수 있어 이 단점을 보완할 수 있습니다.

따라서 리액터 킷에서도 Action이나 Mutation에 원하는 값을 전달하면서 흐름을 이어갈 수 있었습니다.

## 💡 추가로 깨달은 점

연관값 위의 예시처럼 let 키워드를 통해 값을 상수에 바인딩해서 사용할 수 있지만 직접 값을 비교할 수도 있습니다.
예를 들면 아래와 같습니다:

```swift
enum Response {
    case statusCode(Int)
}

let response: Response = .statusCode(404)

switch response {
case .statusCode(200):
    print("정상 응답입니다.")
case .statusCode(let code) where code >= 400:
    print("에러 응답: \(code)")
default:
    print("기타 응답")
}
```

1. `case .statusCode(200)`  
      → 연관값이 정확히 200일 때 해당 case가 실행됩니다.
2. `case .statusCode(let code) where code >= 400`  
      → 연관값을 `code`라는 상수에 바인딩하고, `where` 절을 통해 추가 조건을 걸 수 있습니다.

- 또한 let 이 아니고 var인 변수로도 바인딩할 수 있습니다.

**<주의점>**<br>
연관값을 사용할 때 주의할 점은 연관값과 열거형의 원시값(raw value)를 같이 사용할 수 없다는 점입니다.

## 📌 마무리

아무렇지 않게 작성하던`case .something()`코드 속 괄호가 단순한 괄호가 아니라 **의미 있는 값을 담는 구조**라는 점이 인상 깊었습니다.

익숙하다고 생각한 코드 속에도 아직 모르는 개념이 숨어 있다는 것을 느꼈고,<br>
앞으로는 자주 보던 문법이라도 가볍게 넘기지 않고, "왜 이렇게 쓰는 걸까?"라는 시선을 가지며 더 깊이 있는 개발자가 되고자 합니다.
