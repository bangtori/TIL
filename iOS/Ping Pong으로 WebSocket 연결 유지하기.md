일자 : 2025.04.05.월<br>
**📌 TIL: Ping Pong으로 WebSocket 연결 유지하기**

## 🛠 문제 상황

채팅(소켓)에서 1분 정도 연결 동안 아무런 데이터 전달이 없으면 연결이 끊어지는 문제가 발생했습니다. 서버에서는 1분 동안 아무런 동작이 없으면 연결을 끊도록 설정되어 있었습니다. <br>
따라서 채팅방에서 일정시간동안 아무 동작이 없다 메시지를 보내면 해당 메시지가 전달되지 않는 문제가 발생했습니다.

따라서 계속해서 소켓 연결을 유지할 수 있도록 해야했습니다.

## ✅ 해결 방법

따라서 소켓의 기본동작 중 하나인 Ping, Pong 메시지를 활용하여 해결했습니다.

현재 서버에서 Ping 메시지를 따로 보내지 않기에,
클라이언트 측에서 주기적으로 Ping 메시지를 보내고,
그에 대한 Pong 메시지를 받아 연결을 유지하는 방식으로 해결했습니다.

- `connectionStatusSubject` 옵저버를 사용해 연결 상태를 추적하고, 연결 상태에 따라 Ping 타이머를 시작하거나 중지하는 방식으로 구현했습니다.
- `pingTimer`를 사용하여 일정 시간 간격으로 Ping 메시지를 서버에 전송합니다.
- `sendPing()` 메서드를 통해 Ping 메시지를 전송하며, 연결이 끊어졌을 경우 타이머를 종료합니다.
- `didReceive(event:)` 메서드를 통해 서버에서 전송한 Pong 메시지를 처리합니다. 따로 메시지를 받았을 때 처리할 내용은 없어서 제대로 Ping 메시지가 전달되었는지 확인하는 용도로 사용했습니다.

```swift
private var connectionStatusSubject = BehaviorSubject<Bool>(value: false)
private var pingTimer: Timer?

init() throws {
    // ... 기타 init 코드

    connectionStatusSubject
        .observe(on: MainScheduler.asyncInstance)
        .subscribe(onNext: { [weak self] isConnected in
            if isConnected {
                self?.startPingTimer()
            } else {
                self?.stopPingTimer()
            }
        })
        .disposed(by: disposeBag)
    print("✅ [DEBUG] SocketService 초기화 완료")
}

private func startPingTimer() {
    pingTimer?.invalidate() // 기존 타이머가 있으면 종료
    guard let isConnected = try? connectionStatusSubject.value(), isConnected else {
        print("⚠️ WebSocket 연결이 끊어졌습니다. Ping 타이머 시작을 중지합니다.")
        return
    }
    pingTimer = Timer.scheduledTimer(timeInterval: 30.0, target: self, selector: #selector(sendPing), userInfo: nil, repeats: true)
    print("✅ Ping 타이머 시작됨")
}

@objc private func sendPing() {
    guard let isConnected = try? connectionStatusSubject.value(), isConnected else {
        print("⚠️ WebSocket 연결이 끊어졌습니다. Ping 메시지 전송을 중지합니다.")
        pingTimer?.invalidate()
        return
    }

    socket?.write(ping: Data())
    print("📩 Ping 메시지 전송")
}

private func stopPingTimer() {
    pingTimer?.invalidate()
    print("❌ Ping 타이머 중지됨")
}
```

## 🎯 배운 내용 정리

**Ping Pong**은 네트워크 연결을 유지하기 위한 기법으로, 서버와 클라이언트가 서로 주고받는 메시지입니다.<br>
이를 통해 연결이 살아있는지 확인하고, 일정 시간동안 데이터가 없으면 연결을 끊는 서버 측의 설정을 우회할 수 있습니다.

- **Starscream**에서 핑 메시지를 보내는 방법은 `socket?.write(ping: Data())`로, 클라이언트에서 직접 핑 메시지를 보내고, 서버에서 해당 메시지에 대해 pong 응답을 보내도록 처리됩니다.

- **`didReceive(event:)`** 메서드를 통해 서버에서 보내는 다양한 이벤트를 처리할 수 있는데 이때 Ping 메시지와 Pong 메시지를 받아 처리할 수 있습니다

## 💡 추가로 깨달은 점

소켓 통신은 클라이언트와 서버가 **지속적으로 연결된 상태**를 유지하는 방식입니다.

일정 시간 동안 아무런 메시지가 오가지 않으면 연결을 끊는 이유는 **연결 상태의 신뢰성을 확보**하기 위해서입니다.

소켓은 **상태 기반** 연결을 유지하지만, 이 연결이 **실제로 활성화된 상태인지** 확인할 방법이 없기 때문에 주기적으로 데이터를 교환해야 합니다.<br>
그렇지 않으면 네트워크 상의 중단, 불안정성, 혹은 오류로 인해 연결이 끊어졌는지 확인할 수 없습니다.

따라서 일정 시간동안 이벤트가 오가지 않았을 경우 연결이 끊어졌을 수 있기에 서버에서 해당 소켓의 연결을 종료합니다.

**Ping Pong의 역할**
그래서 Ping Pong 기법은 **연결 유지를 위한 신호**로, 클라이언트와 서버가 서로 주고받는 메시지를 통해 연결이 **활성 상태인지 확인**하는 방법입니다.
서버나 클라이언트는 일정 시간마다 **Ping 메시지**를 주고받으며, **Pong 메시지**를 받았을 때 연결이 정상임을 확인합니다. 이를 통해 네트워크가 정상적으로 작동하고, 데이터가 제대로 전달되고 있음을 알 수 있습니다.

즉 다른 메시지를 보내지않더라도 Ping Pong 을 통해 네트워크 소켓 연결에 문제가 없음을 인증하여 신뢰성을 유지합니다.

따라서 **서버에서는 자원 낭비를 줄이고, 불필요한 연결을 종료하여 시스템의 성능을 유지하기 위해 무제한적인 연결을 할 수 없었기에 클라이언트와 서버는 Ping Pong 을 통해 연결 상태를 확인**해야했습니다.

## 📌 마무리

이번 문제 해결을 통해 소켓 통신의 특성과 그에 따른 여러 문제들을 좀 더 실감할 수 있었습니다.

진행하면서 **Ping Pong** 방식으로 소켓 연결을 유지하는 방법을 배웠지만, **지속된 연결이 자원 사용에 미치는 영향**에 대해서 많이 고민하게 되었습니다.<br>
연결이 계속 유지되다 보니 **자원 소모가 크다는 점**에서 효율적인 방식으로 개선할 필요성을 느꼈습니다. 처음에는 단순히 연결을 유지하는 데 집중했지만, 클라이언트와 서버가 서로 주고받는 데이터나 연결 상태를 어떻게 **효율적으로 관리**할 수 있을지에 대한 고민이 많이 들었습니다.

특히, **연결을 끊지 않고 지속하는 것**만으로는 충분하지 않다는 점을 깨달았고, 어떻게 하면 자원을 낭비하지 않으면서도 안정적인 연결을 유지할 수 있을지에 대한 공부가 필요하다는 생각이 들었습니다. 앞으로는 **자원 효율성**을 중시하면서도 **연결 유지**의 최적화 방법에 대해 더 깊이 공부해보고, 이를 프로젝트에 어떻게 적용할 수 있을지 고민해보려고 합니다.
