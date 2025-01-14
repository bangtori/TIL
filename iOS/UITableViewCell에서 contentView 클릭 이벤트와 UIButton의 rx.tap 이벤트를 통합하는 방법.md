일자 : 2025.01.14.화

## 문제 상황
CheckButton을 포함한 UITableViewCell에서 다음과 같은 개선 요구사항이 있었습니다:
1. 현재는 **체크 버튼**을 눌러야만 셀 아이템이 선택되는 구조.
2. 체크 버튼의 크기가 작아 사용자 입장에서 누르기 불편하며, <b>체크 버튼 외의 영역(title 등)</b>을 클릭해도 셀 아이템이 선택되도록 개선이 필요.

이에 따라 **Cell contentView를 클릭해도 checkButton을 클릭한 것과 동일한 로직**이 실행되도록 연결하는 작업을 진행했습니다.

---
#### 기존 문제
- 기존에는 contentView의 클릭 이벤트와 checkButton.rx.tap 이벤트를 별도로 관리했기 때문에 코드 중복이 발생했습니다.
- 또한, 선택된 Cell을 직접 찾아야 하는 로직이 추가되어 비효율적이고 복잡했습니다.
- 두 이벤트를 동일하게 처리하는 일관된 방식이 없어 유지보수성도 떨어졌습니다.

---
#### 해결 방향
이 문제를 해결하기 위해, **contentView와 checkButton의 클릭 이벤트를 하나의 스트림으로 통합하여 관리**하는 방식을 떠올렸습니다. 이를 통해 중복 코드를 제거하고, 두 이벤트를 동일한 로직으로 처리할 수 있도록 구현을 개선하고자 했습니다.
<br>
<br>
## 해결방법
contentView와 checkButton의 클릭 이벤트를 **하나의 이벤트 스트림**으로 통합하여 관리하기 위해, **RxSwift의 PublishSubject**를 활용했습니다.

#### 왜 PublishSubject를 선택했는가?
- **초기값이 필요하지 않음**: contentView와 checkButton의 클릭 이벤트는 사용자 액션이 발생한 시점에만 처리되며, 초기값이 필요하지 않습니다.
- **새로운 이벤트만 전달**: 이전에 발생한 클릭 이벤트는 중요하지 않으므로, **현재 구독 중인 구독자**에게만 새로운 이벤트를 전달하는 PublishSubject가 적합합니다.
- **다수의 구독자 지원**: 클릭 이벤트를 여러 구독자가 동시에 처리할 수 있도록 지원합니다.
---
####  PublishSubject를 사용한 이벤트 통합
- RxSwift의 PublishSubject를 활용하여 contentView와 checkButton의 이벤트를 하나의 스트림으로 통합했습니다.
- UITableViewCell 내부에서 UITapGestureRecognizer를 사용해 contentView의 탭 이벤트를 감지하고, 이를 checkButton.rx.tap과 동일한 스트림(tapSubject)에 바인딩했습니다.

```swift
class TableViewCell: UITableViewCell {
    let tapSubject = PublishSubject<Void>() // 이벤트 통합 스트림
    private let tapGesture = UITapGestureRecognizer()

    private let checkButton: UIButton = {
        let button = UIButton()
		// 버튼 설정
        return button
    }()

  

    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        // 기타 init코드
        setupTapGesture()
    }

  // ... 그 외 코드들

    private func setupTapGesture() {
        contentView.addGestureRecognizer(tapGesture)
        // contentView 클릭 이벤트를 tapSubject로 전달
        tapGesture.rx.event
            .map { _ in }
            .bind(to: tapSubject)
            .disposed(by: disposeBag)
  
        // checkButton 클릭 이벤트를 tapSubject로 전달
        checkButton.rx.tap
            .bind(to: tapSubject)
            .disposed(by: disposeBag)
    }
}
```

즉 이 과정을 통해 ContentView의 클릭 이벤트, checkButton의 클릭 이벤트 모두 tapSubject로 전달이 되어 tapSubject 하나로 두 개의 이벤트를 관리할 수 있게 되었습니다.

#### TableView에서 tapSubject 처리
셀의 tapSubject를 테이블 뷰에서 구독하여, **체크 상태를 관리하는 로직**을 통일했습니다. tapSubject로부터 이벤트가 발생하면 선택된 셀을 갱신하고, UI를 업데이트할 수 있도록 하였습니다.
```swift

    .bind(to: tableView.rx.items(cellIdentifier: "TableViewCell", cellType: TableViewCell.self)) { [weak self] row, team, cell in
        guard let self = self else { return }
        // 기타 셀 초기화 코드
        // tapSubject 구독
        cell.tapSubject
            .subscribe(onNext: {
                // checkButton 클릭 시 실행되던 로직
            })
            .disposed(by: cell.disposeBag)
    }
    .disposed(by: disposeBag)
```


#### 적용 결과
1. PublishSubject**로 클릭 이벤트 통합**:
- contentView와 checkButton의 이벤트를 동일하게 처리.

2. **중복 코드 제거**:
- 두 이벤트를 별도로 관리하던 기존 코드 구조를 개선하여, 동일한 로직을 재사용할 수 있도록 구현.

3. **사용자 경험 개선**:
- 체크 버튼 외의 영역을 클릭해도 동일한 동작을 수행하도록 설정.

## 마무리
기존에는 RxCocoa의 기본 이벤트를 사용하여 1:1로 이벤트를 관리했지만, **Subject를 활용한 이벤트 통합 패턴**을 통해 가독성과 유지보수성이 높은 코드를 작성할 수 있었습니다.

앞으로도 이 패턴을 다른 프로젝트나 UI 개선 작업에 적극 활용할 계획이며, 다양한 Subject(BehaviorSubject, ReplaySubject 등)를 상황에 맞게 사용하여 코드의 효율성을 더욱 높이고자 합니다.