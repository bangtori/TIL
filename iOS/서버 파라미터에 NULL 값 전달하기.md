일자 : 2025.01.13.월

### 문제 상황
서버에서 임시 저장 로직 구현 시 특정 파라미터에 값이 없을 경우 **null 값을 전달해야 한다는 요청**을 받았습니다.

하지만 `[String: Any]` 딕셔너리로 파라미터를 전달하고 있었으며, **Swift에서는 `[String: Any]` 딕셔너리에 nil을 직접 추가할 수 없습니다.**

예를 들어, 아래와 같은 파라미터를 서버에 전달해야 한다고 가정해 봅시다:

``` json
{
  "username": "example",
  "profilePicture": null
}
```

여기서 profilePicture는 null로 명시적으로 보내야 한다면, Swift에서 다음과 같이 작성하면 에러가 발생합니다:

``` swift
var parameters: [String: Any] = [:]
parameters["username"] = "example"
parameters["profilePicture"] = nil // ❌ 컴파일 에러!
```

nil **대신 아예 파라미터를 생략하여 요청**하면 서버에서 이를 처리하는 로직을 작성해 주었기 때문에 기능적으로는 문제가 없었습니다. 하지만 **클라이언트에서 “값이 없다(null)“는 의도를 명확히 전달하기 위해**, null 값을 보내는 방식을 선택하기로 했습니다.

## 해결방법: NSNull을 활용하기

Swift의 NSNull은 JSON이나 Objective-C에서 null을 표현하기 위한 객체입니다.

**NSNull을 사용하면 `[String: Any]`에서도 null 값을 안전하게 추가할 수 있습니다.**

``` swift
var parameters: [String: Any] = [:]

// 정상적으로 null 값 추가
parameters["username"] = "example"
parameters["profilePicture"] = NSNull()

print(parameters)
// 출력: ["username": "example", "profilePicture": <null>]
```

#### JSON 변환

위에서 작성한 딕셔너리를 JSON 데이터로 변환하면, NSNull은 올바르게 JSON의 null로 변환됩니다.

``` swift
let parameters: [String: Any] = [
    "username": "example",
    "profilePicture": NSNull()
]

if let jsonData = try? JSONSerialization.data(withJSONObject: parameters, options: .prettyPrinted),
   let jsonString = String(data: jsonData, encoding: .utf8) {
    print(jsonString)
    // 출력:
    // {
    //   "username": "example",
    //   "profilePicture": null
    // }
}
```

### nil과 NSNull의 차이점

• nil: Swift에서 값이 없음을 나타냅니다.

• NSNull: JSON 또는 딕셔너리에서 null을 표현하기 위해 사용됩니다.

#### 왜 null을 명시적으로 보냈을까?

1. **서버와 클라이언트 간 명확한 의사소통**:
- null 값은 선택하지 않았다는 **명확한 의도**를 전달합니다.
- 파라미터를 생략하면 “기본값을 사용하라”는 뜻인지, “값이 없다(null)“는 뜻인지 서버 입장에서 혼란을 줄 수 있습니다.

2. **데이터 초기화 요청**:
- 특정 필드(예: profilePicture)를 초기화하거나 삭제하는 의도를 서버에 전달할 수 있습니다.

3. **API 설계와 일관성 유지**:
- 모든 필드를 명시적으로 관리하며, 필요한 경우 값을 null로 설정하는 로직을 클라이언트에서 명확히 처리할 수 있습니다.

### 마무리
이번 문제를 해결하면서 클라이언트와 서버 간 데이터 전송에서 "명확한 의도"가 얼마나 중요한지 깨달았습니다. 처음에는 파라미터를 생략하는 것이 더 간단하다고 생각했지만, 상황에 따라 `null`을 명시적으로 전달해야 서버가 의도를 더 잘 파악할 수 있다는 점이 인상 깊었습니다.  

또한, Swift에서 `NSNull`을 활용하여 JSON 통신에서 `null` 값을 처리하는 방법을 알게 되어, 앞으로 비슷한 상황에서 더 유연하게 대처할 수 있을 것 같습니다. 특히, API 설계 문서를 읽을 때 의도를 더 깊이 이해하고 서버 개발자와 협력해야겠다는 다짐도 하게 되었습니다.
