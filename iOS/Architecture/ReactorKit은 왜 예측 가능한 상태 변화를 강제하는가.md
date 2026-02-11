**📌 TIL: ReactorKit은 왜 예측 가능한 상태 변화를 강제하는가 — Flux와 Side Effect 관점에서**

## 🛠 탐구 계기

iOS 앱을 설계하면서 자연스럽게 ReactorKit 기반의 단방향 아키텍처를 사용해왔습니다.
그 이유를 “테스트 용이성” 때문이라고 알고 있었지만, 사실상 암기 수준의 이해에 머물러 있었습니다.

- 왜 단방향이어야 테스트가 쉬운가?
- 왜 상태 변화 지점을 한 곳으로 제한해야 하는가?
- Side Effect란 정확히 무엇이고 왜 분리해야 하는가?

이 질문에 대한 구조적 답을 찾기 위해,</br>
ReactorKit이 채택한 Flux 개념과 Side Effect 관점에서 정리해보았습니다.

---

## ✅ 내용 정리

### 1. ReactorKit이란 무엇인가

ReactorKit은 UI의 상태(State)와 사용자 동작(Action)을 Reactor라는 중간 계층을 통해 관리하는 단방향 아키텍처 프레임워크입니다.

구조는 다음과 같습니다:

<img src="/image/ReactorKit_DataFlow.png" height=300/>

```
View → Action → Reactor → State → View
```

View

- 사용자 입력을 Action으로 변환하여 Reactor에 전달합니다.
- Reactor가 방출하는 State를 구독하여 UI를 갱신합니다.

Reactor

- 비즈니스 로직의 중심입니다.
- Action을 받아 상태를 어떻게 변경할지 결정합니다.

이 구조는 Flux 아키텍처를 기반으로 설계되었습니다.

### Flux와 단방향 데이터 흐름

Flux는 MVC의 복잡성을 해결하기 위해 등장했습니다.

기존 MVC에서는 Model과 View가 양방향으로 참조되며, 규모가 커질수록 데이터 흐름이 추적 불가능해지는 문제가 발생했습니다.

예를 들어:

- Model 변경 → View 변경
- View 내부 로직 → 또 다른 Model 변경
- 연쇄적 상태 변경 발생

이러한 구조는 예측을 어렵게 만듭니다.

Flux의 핵심 철학은 다음과 같습니다.

> 데이터는 반드시 한 방향으로만 흐른다.

구조는 다음과 같습니다:

```
Action → Dispatcher → Store → View
```

ReactorKit에서는 이를 다음과 같이 대응시킵니다.
| Flux | ReactorKit |
|:----:|:----------:|
|Action|Action|
|Dispatcher / Store|Reactor|
|View|View|

-> 즉, 상태는 오직 Reactor에서만 변경됩니다.

### ReactorKit이 단방향을 “강제”하는 이유

단방향은 단순한 패턴이 아니라,예측 가능성을 확보하기 위한 설계 전략입니다.

Reactor 내부의 상태 변화 흐름은 다음과 같습니다:

```
Action → mutate() → Mutation → reduce() → State
```

이 흐름을 벗어나 상태를 직접 수정할 수 없습니다.

이로 인해 다음이 보장됩니다:

1. 예측 가능성 (Predictability)</br>
   현재 상태는 어떤 Action이 들어왔는지 추적하면 반드시 설명 가능합니다.

2. 단일 상태 변경 지점 (Single Source of Truth)</br>
   상태를 변경하는 위치가 Reactor로 제한됩니다.
   디버깅 범위가 극단적으로 좁아집니다.

3. 테스트 가능성 (Testability)</br>
   View 없이 Action을 주입해 State 결과만 검증할 수 있습니다.

### Side Effect란 무엇인가

Side Effect란,
함수가 외부 상태를 변경하거나 외부 세계와 상호작용하는 부수 효과를 의미합니다.

예를 들면:

- 네트워크 요청
- 데이터베이스 저장
- 파일 I/O
- 타이머
- 로깅

Side Effect가 많아질수록 동일한 입력에 대해 결과가 달라질 수 있습니다.

예시:

```swift
var counter = 0

func impureAdd(a: Int, b: Int) -> Int {
    counter += 1
    return a + b + counter
}
```

같은 입력이라도 호출 순서에 따라 결과가 달라집니다. 이는 예측 불가능성을 유발합니다.
ReactorKit에서 이 사이드 이팩트를 허용하는 구간은 `mutate()` 뿐 입니다.

#### ReactorKit에서 Side Effect를 mutate()로 제한하는 이유

- mutate() → 외부 세계와 상호작용
- reduce() → 순수 함수처럼 동작 (State 계산 전용)

이 분리를 통해 다음을 확보합니다.

- 상태 계산은 항상 예측 가능
- 네트워크 실패 여부와 관계없이 상태 로직은 순수하게 유지
- 테스트 시 Side Effect를 Mock 처리 가능

즉, **핵심 로직을 “순수 영역”으로 보호**하기 위함입니다.

### 마무리 요약

- ReactorKit은 Flux 기반의 **단방향 데이터 흐름**을 채택합니다.
- **상태 변경은 오직 Reactor 내부에서만** 발생합니다.
- **Side Effect는 mutate() 단계로 제한**됩니다.
- 이를 통해 **예측 가능성, 디버깅 용이성, 테스트 독립성**을 확보합니다.

## 📌 마무리

이번 정리를 통해,ReactorKit이 단순히 “테스트가 쉬운 패턴”이 아니라</br>
**복잡한 앱에서도 상태를 통제 가능하게 만들기 위한 설계 전략**임을 이해할 수 있었습니다.

특히 Side Effect를 어디에 배치하느냐가 아키텍처의 예측 가능성을 결정한다는 점이 인상 깊었습니다.

앞으로 아키텍처를 설계할 때에는
단순히 패턴을 선택하는 것이 아니라,

- 상태 변경 지점이 명확한가
- Side Effect가 분리되어 있는가
- 동일한 입력에 대해 동일한 결과가 보장되는가

를 기준으로 판단하고자 합니다.
