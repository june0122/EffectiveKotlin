# 아이템 6. 사용자 정의 오류보다는 표준 오류를 사용하라

`require`, `check`, `assert` 함수를 사용하면 대부분의 코틀린 오류를 처리할 수 있습니다. 하지만 이외에도 예측하지 못한 상황을 나타내야 하는 경우가 있습니다.

#### JSON 형식을 파싱하는 라이브러리의 구현

> 기본적으로 입력된 JSON 파일의 형식에 문제가 있다면, *JSONParsingException* 등을 발생시키는 것이 좋을 것입니다.

```kotlin
inline fun <reified T> String.readObject(): T {
    //...
    if (incorrectSign) {
        throw JsonParsingException()
    }
    //...
    return result
}
```

표준 라이브러리에는 이를 나타내는 적절한 오류가 없으므로, 사용자 정의 오류를 사용했습니다. 하지만 가능하다면, 직접 오류를 정의하는 것보다는 최대한 표준 라이브러리의 오류를 사용하는 것이 좋습니다. 표준 라이브러리의 오류는 많은 개발자가 알고 있으므로, 이를 재사용하는 것이 좋습니다. 잘 만들어진 규약을 가진 널리 알려진 요소를 재사용하면, 다른 사람들이 API를 더 쉽게 배우고 이해할 수 있습니다. 일반적으로 사용되는 예외를 몇 가지 정리해보면 다음과 같습니다.

- `IllegalArgumentException`과 `IllegalStateException`: ***아이템 5: 예외를 활용해 코드에 제한을 걸어라***에서 다루었던 `require`와 `check`를 사용해 throw 할 수 있는 예외입니다.
- `IndexOutOfBoundsException`: 인덱스 파라미터의 값이 범위를 벗어났다는 것을 나타냅니다. 일반적으로 컬렉션 또는 배열과 함께 사용합니다. 예를 들어 `ArrayList.get(Int)`를 사용할 때 throw됩니다.
- `ConcurrentModificationException`: 동시 수정<small>(concurrent modification)</small>을 금지했는데, 발생해 버렸다는 것을 나타냅니다.
- `UnsupportedOperationException`: 사용자가 사용하려고 했던 메서드가 현재 객체에서는 사용할 수 없다는 것을 나타냅니다. 기본적으로는 사용할 수 없는 메서드는 클래스에 없는 것이 좋습니다.
  - 이렇게 구현해 버리면 인터페이스 분리 원칙을 위반하게 됩니다. 인터페이스 분리 원칙은 클라이언트가 자신이 사용하지 않는 메서드에 의존하면 안 된다는 원칙입낟.
  - 하지만 클래스 내부에 사용할 수 없는 메서드를 일부러 두는 경우도 있습니다. 예를 들어 `listOf`에는 사용하면 무조건적으로 오류가 발생하는 메서드들이 있습니다. 이는 '`listOf`로 만들어진 컬렉션은 immutable이므로 조작할 수 없습니다'를 명시적으로 알려 주기 위한 목적으로 사용됩니다.
- NoSuchElementException: 사용자가 사용하려고 했던 요소가 존재하지 않음을 나타냅니다. 예를 들어 내부에 없는 Iterable에 대해 next를 호출할 때 발생합니다.