# 아이템 42. compareTo의 규약을 지켜라

`compareTo` 메서드는 Any 클래스에 있는 메서드가 아닙니다. 이는 수학적인 부등식으로 변환되는 연산자입니다.

```kotlin
obj1 > obj2 // obj1.compareTo(obj2) > 0 으로 변환
obj1 < obj2 // obj1.compareTo(obj2) < 0 으로 변환
obj1 >= obj2 // obj1.compareTo(obj2) >= 0 으로 변환
obj1 <= obj2 // obj1.compareTo(obj2) <= 0 으로 변환
```

참고로 `compareTo` 메서드는 `Comparable<T>` 인터페이스에도 들어 있습니다. 어떤 객체가 이 인터페이스를 구현하고 있거나 compareTo라는 연산자 메서드를 갖고 있다는 의미는 해당 객체가 어떤 순서를 갖고 있으므로, 비교할 수 있다는 것입니다. `compareTo`는 다음과 같이 동작해야 합니다.

- 비대칭적 동작 : `a >= b`이고 `b >= a`라면, `a == b`여야 합니다. 즉, 비교와 동등성 비교에 어떠한 관계가 있어야 하며, 서로 일관성이 있어야 합니다.
- 연속적 동작 : `a >= b`이고 `b >= c`라면, `a >= c`여야 합니다. 마찬가지로 `a > b`이고 `b > c`라면, `a > c`여야 합니다. 이러한 동작을 하지 못하면, 요소 정렬이 무한 반복에 빠질 수 있습니다.
- 코넥스적 동작: 두 요소는 어떤 확실한 관계를 갖고 있어야 합니다. 즉, `a >= b` 또는 `b >= a` 중에 적어도 하나는 항상 true여야 합니다. 두 요소 사이에 관계가 없으면, 퀵 정렬과 삽입 정렬 등의 고전적인 정렬 알고리즘을 사용할 수 없습니다. 대신 위상 정렬<small>(topological sort)</small>과 같은 정렬 알고리즘만 사용할 수 있습니다.

### compareTo를 따로 정의해야 할까?

코틀린에서 `compareTo`를 따로 정의해야 하는 상황은 거의 없습니다. 일반적으로 어떤 프로퍼티 하나를 기반으로 순서를 지정하는 것으로 충분하기 때문입니다. 예를 들어 `sortedBy`를 사용하면, 원하는 키로 컬렉션을 정렬할 수 있습니다. 다음 코드는 surname 프로퍼티를 기반으로 정렬하는 예입니다.

```kotlin
class User(val name: String, val surname: String)
val names = listOf<User>( /* ... */ )

val sorted = names.sortedBy { it.surname }
```

여러 프로퍼티를 기반으로 정렬해야 한다면 어떻게 해야 할까요? 그럴 때는 `sortedWith` 함수를 사용하면 됩니다. 이 함수는 다음과 같이 사용합니다. `compareBy`를 활용해서 비교기<small>(comparator)</small>를 만들어서 사용합니다. 다음 코드는 surname으로 정렬을 하고, 만약 surname이 같은 경우에는 name까지 비교해서 정렬합니다.

```kotlin
val sorted = names.sortedWith(compareBy({ it.surname }, { it.name }))
```

물론 User가 `Comparable<User>`를 구현하는 형태로 만들 수도 있습니다. 이럴 때는 순서를 어떻게 해야 할까요? 특정 프로퍼티를 기반으로 정렬하게 하면 됩니다. 만약 비교에 대한 절대적인 기준이 없다면, 아예 비교하지 못하게 만드는 것도 좋습니다.

문자열은 알파벳과 숫자 등의 순서가 있습니다. 따라서 내부적으로 `Comparable<String>`을 구현하고 있습니다. 텍스트는 일반적으로 알파벳과 숫자 순서로 정렬해야 하는 경우가 많으므로 굉장히 유용합니다. 하지만 단점도 있습니다. 예를 들어 직관적이지 않은 부등호 기호를 기반으로 두 문자열을 비교하는 코드를 작성할 수 있습니다. 두 문자열이 부등식으로 비교된 코드를 보면, 이해하는 데 약간의 시간이 걸립니다.

#### 이렇게 하지 마세요

```kotlin
print("Kotlin" > "Java") // true
```

자연스러운 순서를 갖는 객체들이 있습니다. 예를 들어 측정 단위, 날짜, 시간 등이 모두 자연스러운 순서를 갖습니다. 객체가 자연스러운 순서인지 확실하지 않다면, 비교기<small>(comparator)</small>를 사용하는 것이 좋습니다. 이를 자주 사용한다면, 클래스에 companion 객체로 만들어 두는 것도 좋습니다.

```kotlin
class User(val name: String, val surname: String) {
    // ...

    companion object {
        val DISPLAY_ORDER = compareBy(User::surname, User::name)
    }
}

val sorted = names.sortedWith(User.DISPLAY_ORDER)
```

### compareTo 구현하기

`compareTo`를 구현할 때 유용하게 활용할 수 있는 톱레벨 함수가 있습니다. 두 값을 단순하게 비교하기만 한다면, `compareValues` 함수를 다음과 같이 활용할 수 있습니다.

```kotlin
class User(
    val name: String,
    val surname: String
): Comparable<User> {
    override fun compareTo(other: User): Int = compareValues(surname, other.surname) 
}
```

더 많은 값을 비교하거나, 선택기<small>(selector)</small>를 활용해서 비교하고 싶다면, 다음과 같이 `compareValuesBy`를 사용합니다.

```kotlin
class User(
    val name: String,
    val surname: String
): Comparable<User> {
    override fun compareTo(other: User): Int =
        compareValuesBy(this, other, { it.surname }, { it.name }) 
}
```

이 함수는 비교기를 만들 때 도움이 됩니다. 특별한 논리를 구현해야 하는 경우에는 이 함수가 다음 값을 리턴해야 한다는 것을 기억하세요.

- 0 : 리시버와 other가 같은 경우
- 양수 : 리시버가 other보다 큰 경우
- 음수 : 리시버가 other보다 작은 경우

이를 구현한 뒤에는 이 함수가 비대칭적 동작, 연속적 동작, 코넥스적 동작을 하는지 확인하세요.