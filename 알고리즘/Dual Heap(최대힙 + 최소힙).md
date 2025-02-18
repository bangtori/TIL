일자 : 2025.02.18.화

**📌 TIL: Dual Heap (최대힙 + 최소힙)**

## 🛠 문제 상황

[프로그래머스\_Lv3.이중우선순위큐](https://school.programmers.co.kr/learn/courses/30/lessons/42628) <br>
프로그래머스에서 해당 문제를 풀기위해서는 최댓값과 최솟값을 모두 O(1)로 조회하면서도 O(log N)으로 삽입/삭제가 가능한 자료구조가 필요했습니다.

기존의 단일 우선순위 큐(Heap)만으로는 한쪽 방향의 정렬만 유지할 수 있어, 최댓값과 최솟값을 동시에 효율적으로 관리할 수 없었습니다.

## ✅ 해결 방법

최대 힙(Max Heap)과 최소 힙(Min Heap)을 동시에 유지하는 **Dual Heap**을 구현했습니다. 이중 우선순위 큐는 두 개의 힙을 사용하여 최댓값과 최솟값을 빠르게 찾을 수 있도록 합니다.

#### 📌 핵심 아이디어

1. **최소 힙(minHeap)**: 최솟값을 빠르게 조회하기 위한 Heap
2. **최대 힙(maxHeap)**: 최댓값을 빠르게 조회하기 위한 Heap
3. **동기화(syncHeap)**: 삭제된 요소가 남아 있지 않도록 동기화하는 과정 필요
4. **딕셔너리(dict) 활용**: 값의 개수를 저장하여 동기화에 사용

즉 딕셔너리에서 값의 개수가 0이라면 힙에서 삭제된 값을 의미합니다.

1. 삽입 연산

   최대 최소힙에 모두 insert 후 딕셔너리의 insert value 값을 +1 을 해줍니다.

2. sync 연산

   heap을 입력받아 힙의 값을 peek 하면서 만약 유효하지 않은 값 = 딕셔너리의 값이 0인값이라면 삭제해줍니다.
   이미 다른 힙에서 삭제됐음을 의미하기 때문입니다.

3. 삭제 연산

   remove 연산을 진행하기 전 루트에 유효하지않은 값이 있을 수 있으므로, 동기화 작업을 진행해줍니다.
   그 후 최댓값, 최솟값인지에 따라 해당 힙의 remove작업을 진행합니다.

## 🎯 코드

```swift
import Foundation

/// 힙 (Heap)
struct Heap<T: Comparable> {
    private var elements: [T] = []
    private let sort: (T, T) -> Bool

    init(sort: @escaping (T, T) -> Bool) {
        self.sort = sort
    }

    var isEmpty: Bool { return elements.isEmpty }
    var count: Int { return elements.count }

    func peek() -> T? {
        return elements.first
    }

    mutating func insert(_ value: T) {
        elements.append(value)
        siftUp(from: elements.count - 1)
    }

    mutating func remove() -> T? {
        guard !elements.isEmpty else { return nil }
        elements.swapAt(0, elements.count - 1)
        let removed = elements.removeLast()
        siftDown(from: 0)
        return removed
    }

    private mutating func siftUp(from index: Int) {
        var child = index
        var parent = (child - 1) / 2
        while child > 0 && sort(elements[child], elements[parent]) {
            elements.swapAt(child, parent)
            child = parent
            parent = (child - 1) / 2
        }
    }

    private mutating func siftDown(from index: Int) {
        var parent = index
        while true {
            let left = 2 * parent + 1
            let right = 2 * parent + 2
            var candidate = parent

            if left < elements.count && sort(elements[left], elements[candidate]) {
                candidate = left
            }
            if right < elements.count && sort(elements[right], elements[candidate]) {
                candidate = right
            }
            if candidate == parent { return }
            elements.swapAt(parent, candidate)
            parent = candidate
        }
    }
}

struct DualHeap {
    private var minHeap = Heap<Int>(sort: <)
    private var maxHeap = Heap<Int>(sort: >)
    private var syncDict = [Int: Int]()

    mutating func insert(_ value: Int) {
        minHeap.insert(value)
        maxHeap.insert(value)
        syncDict[value, default: 0] += 1
    }
    mutating func removeMin() {
        minHeap = syncHeap(minHeap)
        if let value = minHeap.remove() {
            syncDict[value, default: 0] -= 1
        }
    }
    mutating func removeMax() {
        maxHeap = syncHeap(maxHeap)
        if let value = maxHeap.remove() {
            syncDict[value, default: 0] -= 1
        }
    }

    mutating func getMin() -> Int? {
        minHeap = syncHeap(minHeap)
        return minHeap.peek()
    }
    mutating func getMax() -> Int? {
        maxHeap = syncHeap(maxHeap)
        return maxHeap.peek()
    }

    private func syncHeap(_ heap: Heap<Int>) -> Heap<Int> {
        var newHeap = heap
        while let value = newHeap.peek(), (syncDict[value] ?? 0) <= 0 {
            _ = newHeap.remove()  // 힙에서 유효하지 않은 값 제거
        }
        return newHeap
    }
}
```

## 💡 추가로 깨달은 점

1. Swift의 구조체에서의 메모리 접근 제약

처음 코드 작성 시 `syncHeap()`의 파라미터를 `inout` 으로 힙을 전달 받아 직접 수정하고자 하였지만 `Overlapping accesses to 'self', but modification requires exclusive access; consider copying to a local variable` 다음과 같은 오류가 발생했습니다.<br>
Swift의 구조체에서는 **동일한 인스턴스의 일부를 동시에 수정할 수 없기 때문에**, `syncHeap()`을 `inout`이 아닌 값 복사 방식으로 변경해야 했습니다.<br>
이는 Swift의 메모리 관리 원칙(Exclusive Access to Memory)에 따른 것이며, 동기화 과정에서 `inout`을 사용할 경우 충돌이 발생할 수 있기 때문에 오류가 발생했습니다.

2. sort 방식과 비교

`syncHeap()`을 활용하면 정렬된 배열을 `sort()`를 통해 계속 반대로 정렬하는 것보다 **O(log N)** 의 시간복잡도로 최댓값과 최솟값을 유지할 수 있습니다.<br>
반면, `sort()`를 매번 호출하면 **O(N log N)** 이 걸리므로, `syncHeap()`을 사용하는 방식이 더 효율적입니다.

## 📌 마무리

이번 문제 풀이를 통해 Swift에서 **최대 힙 + 최소 힙을 활용하는 이중 우선순위 큐(Dual Heap)를 효율적으로 구현하는 방법을 익혔습니다**.
또한, Swift의 구조체에서의 메모리 접근 제약을 이해하고, 이를 우회하는 방법을 알 수 있었습니다.
