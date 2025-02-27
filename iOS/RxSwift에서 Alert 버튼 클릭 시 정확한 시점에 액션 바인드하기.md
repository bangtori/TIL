일자 : 2025.01.13.월

**📌 TIL: RxSwift에서 Alert 버튼 클릭 시 정확한 시점에 액션 바인드하기**

## 🛠 문제 상황

RxSwift를 활용하여 `exitButton` 클릭 시 **"채팅방을 나갈까요?"** 라는 Alert을 띄우고, "나가기" 버튼을 눌렀을 때만 `exitRoom` 액션을 실행하도록 구현하려고 했다.

하지만 기존 방식에서는 `map` 내부에서 `showCMAlert`을 호출했기 때문에 **Alert의 버튼이 눌리기도 전에 값이 반환되는 문제**가 발생했다.

```swift
exitButton.rx.tap
    .throttle(.seconds(3), latest: false, scheduler: MainScheduler.instance)
    .withUnretained(self)
    .map { vc, _ -> Bool in
        var flag = false
        vc.showCMAlert(titleText: "채팅방을 나갈까요?", importantButtonText: "나가기", commonButtonText: "취소") {
            flag = true
        } commonAction: {
            flag = false
        }
        return flag // ⚠️ 아직 버튼을 누르지 않았는데 flag를 반환함
    }
    .filter { $0 } // true일 때만 실행
    .map { ChatRoomReactor.Action.exitRoom }
```

위 코드에서는 `showCMAlert`의 버튼이 눌리기 전에 `flag`가 반환되므로, 항상 `false`가 되어 의도한 동작을 하지 않았다

## ✅ 해결 방법

이 문제를 해결하기 위해 `flatMapLatest`를 사용했다.

`flatMapLatest`는 **새로운 Observable이 생성될 때 기존 Observable을 해제하고, 가장 최근의 Observable의 값만 구독하는 연산자**다. <br>
이를 활용하면 **Alert이 완료될 때까지 기다렸다가, 사용자의 선택 결과를 기반으로 스트림을 이어서 처리할 수 있다.**

### ✅ 개선된 코드 (flatMapLatest 활용)

```swift
exitButton.rx.tap
    .throttle(.seconds(3), latest: false, scheduler: MainScheduler.instance)
    .withUnretained(self)
    .flatMapLatest { vc, _ -> Single<Bool> in
        return Single.create { single in
            vc.showCMAlert(titleText: "채팅방을 나갈까요?", importantButtonText: "나가기", commonButtonText: "취소") {
                single(.success(true))  // "나가기" 선택 시 true 방출
            } commonAction: {
                single(.success(false)) // "취소" 선택 시 false 방출
            }
            return Disposables.create()
        }
    }
    .filter { $0 } // true일 때만 실행
    .map { ChatRoomReactor.Action.exitRoom }
```

## 🎯 배운 내용 정리

### 왜 문제가 발생했을까?

- `map`은 입력을 변환하는 용도로만 사용되며, **비동기 이벤트를 기다리는 용도로 적합하지 않음**.
- `map`은 **즉시 실행되므로** 비동기적인 UI 이벤트를 기다리지 못하고, `flag`가 초기값 `false`를 그대로 반환함.
- `showCMAlert`이 **비동기적으로 실행**되기 때문에, `flag` 값이 변경되기 전에 반환되는 문제가 발생함.

### 해결책이 왜 동작하는가?

- `flatMapLatest`를 사용하면 **이전 스트림을 해제하고 새로운 Alert의 결과를 기다릴 수 있음**.
- 하지만 여기서는 Alert이 뜨는 동안 새로운 `exitButton.rx.tap` 이벤트가 발생할 가능성이 적기 때문에, `flatMapLatest`가 꼭 필요한 상황은 아니지만, 비동기 작업에서 최신 값만 유지하는 좋은 패턴이다.
- `Single.create`를 활용하여 Alert이 끝날 때까지 기다렸다가, 사용자의 선택이 완료된 후에 값을 방출할 수 있음.

## 💡 추가로 깨달은 점

### `Single` 활용하기

- `Single.create`를 사용하면 **비동기 작업이 한 번만 실행되고 완료되도록 보장**할 수 있음.
- `Single`은 `Observable`과 달리 **한 번만 값을 방출하고 자동으로 종료되므로**, Alert과 같이 한번의 이벤트만 발생할 때 적합함.
- `Observable`을 사용할 경우 `.take(1)` 등의 연산을 추가해야 하지만, `Single`을 사용하면 필요 없음.

## 📌 마무리

map을 통해 입력을 변환할 수 있다는 사실은 잘 알고 활용하였지만 즉시 실행된다는 사실을 놓치고 있었다. <br>
즉 중간에 이벤트가 필요할 때는 map을 사용하지 않고 옵저버를 활용해야한다는 것을 알게 되었다.

또한 `Single`이 **한 번만 실행되는 비동기 작업을 처리하는 데 유용한 도구**라는 점도 배울 수 있었다. <br>
앞으로는 불필요하게 계속해서 유지하지 않는 이벤트의 경우 `Observable`을 사용하지 않고 `Single`을 사용해야겠다.
