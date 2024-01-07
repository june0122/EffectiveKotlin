# 아이템 49. 하나 이상의 처리 단계를 가진 경우에는 시퀀스를 사용하라

Iterable과 Sequence는 정의가 거의 동일하므로 많은 사람들이 둘의 차이를 잊어버립니다.

```kotlin
interface Iterable<out T> {
    operator fun iterator(): Iterator<T>
}

interface Sequence<out T> {
    operator fun iterator(): Iterator<T>
}
```

하지만 이 둘은 완전히 다른 목적으로 설계되어 다른 형태로 동작합니다.
- 무엇보다 Sequence는 lazy하게 처리되므로, 시퀀스 처리 함수를 사용하면 데코레이터 패턴으로 꾸며진 새로운 시퀀스가 리턴됩니다. 최종적인 계산은 toList 또는 count 등의 최종 연산이 이루어질 때 수행됩니다.
- Iterable은 처리 함수를 사용할 때마다 연산이 이루어져 List가 만들어집니다.

```kotlin
public inline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    return filterTo(ArrayList<T>(), predicate)
}

public fun <T> Sequence<T>.filter(
    predicate: (T) -> Boolean
): Sequence<T> {
    return FilteringSequence(this, true, predicate)
}
```

- 컬렉션 처리 연산은 호출할 때 연산이 이루어짐
- 시퀀스 처리 함수는 최종 연산이 이루어지기 전까지는 각 단계에서 연산이 일어나지 않음
  - 예로 시퀀스 처리 함수 filter는 중간 연산이기에 어떠한 연산 처리도 하지 않고, 기존의 시퀀스를 필터링하는 데코레이터만 설치합니다. 실질적인 필터링 처리는 toList 등과 같은 최종 연산을 할 때 이루어집니다.

<p align = 'center'>
<img width = '500' src = 'https://github.com/june0122/TIL/assets/39554623/44b29142-6ebe-4a51-bc81-7de0fd6bacd4'>
</p>

```kotlin
val list = listOf(1, 2, 3)
val listFiltered = list
    .filter { print("f$it "); it % 2 == 1 }
// f1 f2 f3
println(listFiltered) // [1, 3]

val seq = sequenceOf(1, 2, 3)
val filtered = seq.filter { print("f$it "); it % 2 == 1 }
println(filtered)  // FilteringSequence@...

val asList = filtered.toList()
// f1 f2 f3
println(asList) // [1, 3]
```

시퀀스 지연 처리의 장점들은 아래와 같고, 각각의 장점을 자세히 살펴봅시다.

- 자연스러운 처리 순서를 유지합니다.
- 최소한만 연산합니다.
- 무한 시퀀스 형태로 사용할 수 있습니다.
- 각각의 단계에서 컬렉션을 만들어 내지 않습니다.

### 순서의 중요성

이터러블 처리와 시퀀스 처리는 연산의 순서가 달라지면, 다른 결과과 나옵니다.
- 시퀀스 처리: 요소 하나하나에 지정한 연산을 한꺼번에 적용함(element-by-element order 또는 lazy order)
- 이터러블: 요소 전체를 대상으로 연산을 차근차근 적용해나감(steb-by-step order 또는 eager order)

```kotlin
listOf(1, 2, 3)
    .filter { print("F$it, "); it % 2 == 1 }
    .map { print("M$it, "); it * 2 }
    .forEach { print("E$it, ") }
// Prints: F1, F2, F3, M1, M3, E2, E6,

sequenceOf(1, 2, 3)
    .filter { print("F$it, "); it % 2 == 1 }
    .map { print("M$it, "); it * 2 }
    .forEach { print("E$it, ") }
// Prints: F1, M1, E2, F2, F3, M3, E6,
```

<p align = 'center'>
<img width = '500' src = 'https://github.com/june0122/TIL/assets/39554623/9179a79f-02cf-4282-a0d6-5cafe19bb3bf'>
</p>

컬렉션 처리 함수를 사용하지 않고, 고전적인 반복문과 조건문을 활용해서 다음과 같은 코드를 구현한다면, 이는 시퀀스 처리인 element-by-element order와 같습니다.

```kotlin
for (e in listOf(1, 2, 3)) {
    print("F$e, ")
    if (e % 2 == 1) {
        print("M$e, ")
        val mapped = e * 2
        print("E$mapped, ")
    }
}
// Prints: F1, M1, E2, F2, F3, M3, E6,
```

따라서 시퀀스 처리에 사용되는 element-by-element order가 훨씬 자연스러운 처리라 할 수 있으며, 시퀀스 처리는 기본적인 반복문과 조건문을 사용하는 코드와 같으므로, 아마도 조만간 낮은 레벨 컴파일러 최적화가 처리를 더 빠르게 만들어 줄 수도 있을 것입니다.

### 최소 연산

컬렉션에 어떠한 처리를 적용하고, 앞의 요소 10개만 필요한 상황은 굉장히 자주 접할 수 있는 상황입니다. 이터러블 처리는 기본적으로 중간 연산이라는 개념이 없기에, 원하는 처리를 컬렉션 전체에 적용한 뒤, 앞의 요소 10개를 사용해야 합니다. 하지만 시퀀스는 중간 연산의 개념이 있으므로, 앞의 요소 10개에만 원하는 처리를 적용할 수 있습니다.

<p align = 'center'>
<img width = '500' src = 'https://github.com/june0122/TIL/assets/39554623/dd870a1e-55d1-485e-b956-a14df6e13d35'>
</p>

#### find를 사용한 이터러블과 시퀀스의 비교 예시

```kotlin
(1..10)
    .filter { print("F$it, "); it % 2 == 1 }
    .map { print("M$it, "); it * 2 }
    .find { it > 5 }
// Prints: F1, F2, F3, F4, F5, F6, F7, F8, F9, F10,
// M1, M3, M5, M7, M9,

(1..10).asSequence()
    .filter { print("F$it, "); it % 2 == 1 }
    .map { print("M$it, "); it * 2 }
    .find { it > 5 }
// Prints: F1, M1, F2, F3, M3,
```

이러한 이유로 **중간 처리 단계를 모든 요소에 적용할 필요가 없는 경우에는 시퀀스를 사용하는 것이 좋습니다.**

위의 코드에서 find처럼 처리를 적용하고 싶은 요소를 선택하는 연산으로는 *first, find, take, any, all, none, indexOf*가 있습니다.

## 정리

컬렉션과 시퀀스는 같은 처리 메서드를 지원하며, 사용하는 형태가 거의 비슷합니다. 일반적으로 데이터를 컬렉션에 저장하므로, 시퀀스 처리를 하려면 시퀀스로 변환하는 과정이 필요합니다. 또한 최종적으로 컬렉션 결과를 원하는 경우가 많으므로, 시퀀스를 다시 컬렉션으로 변환하는 과정도 필요합니다. 이것이 시퀀스 처리의 단점이라고 할 수 있습니다. 하지만 시퀀스는 lazy하게 처리됩니다. 이로 인해서 다음과 같은 장점이 발생합니다.

- 자연스러운 처리 순서를 유지합니다.
- 최소한만 연산합니다.
- 무한 시퀀스 형태로 사용할 수 있습니다.
- 각각의 단계에서 컬렉션을 만들어 내지 않습니다.

결과적으로 무거운 객체나 규모가 큰 컬렉션을 여러 단계에 걸쳐서 처리할 때는 시퀀스를 사용하는 것이 좋습니다. 또한 시퀀스 처리는 'Kotlin Sequence Debugger' 플러그인을 통해 처리 단계를 시각적으로 확인할 수 있습니다. 상황에 따라서 시퀀스 처리를 활용하면 큰 성능 향상이 있을 수 있습니다.