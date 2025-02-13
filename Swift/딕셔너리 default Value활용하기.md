일자: 2025.02.13목

**📌 TIL: Swift Dictionary의 default 활용하기**

## 🛠 문제 상황

Swift의 Dictionary에서 특정 키의 값을 업데이트할 때, 해당 키가 존재하지 않으면 nil을 반환한다.

알고리즘 문제를 풀면서 시간복잡도의 최적화를 위해 딕셔너리를 사용하는 일이 자주 있었고,
특히 특정 값을 카운팅하는 상황에서 if let을 사용하거나, 미리 초기값을 설정하는 방식이 필요했지만, 코드가 길어지는 문제가 있었다.

예를 들어, 특정 단어의 등장 횟수를 저장하는 딕셔너리를 만들고 싶을 때, 기존 방식은 다음과 같았다:

```swift
var wordCount = [String: Int]()

if let count = wordCount["hello"] {
    wordCount["hello"] = count + 1
} else {
    wordCount["hello"] = 1
}
```

위 코드처럼 if let을 사용하면 키 존재 여부를 직접 확인해야 하고, 값이 없을 경우 수동으로 초기화해야 하는 번거로움이 있었다

## ✅ 해결 방법

Swift에서는 Dictionary의 **default 파라미터**를 활용하면 키가 없을 때 기본값을 자동으로 설정하고, 간결하게 값을 수정할 수 있다.<br>

**🎯 default를 활용한 개선된 코드**

```swift
var wordCount = [String: Int]()

wordCount["hello", default: 0] += 1
wordCount["world", default: 0] += 1
wordCount["hello", default: 0] += 1

print(wordCount)  // ["hello": 2, "world": 1]
```

**키가 없으면 default 값을 사용하여 자동으로 초기화**한 후 값을 증가시키므로 if let 없이도 간결하게 처리할 수 있다.

## 💡 추가로 깨달은 점

배열을 저장하는 딕셔너리에서도 유용하게 사용할 수 있다.

```swift
var categoryItems = [String: [String]]()
categoryItems["fruits", default: []].append("apple")
```

"fruits" 키가 없으면 빈 배열([])을 기본값으로 설정 후 append 수행.

## 📌 마무리

1. default 활용을 통해 Dictionary 값을 더 간결하고 안전하게 관리할 수 있다는 점을 배웠다.
2. 앞으로 문제를 풀면서 **데이터를 카운트하거나, 리스트를 관리할 때** 더 적극적으로 사용해야겠다.
