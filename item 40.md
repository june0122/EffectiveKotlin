# 아이템 40. equals의 규약을 지켜라

코틀린의 Any에는 다음과 같이 잘 설정된 규약들을 가진 메서드들이 있습니다.

- equals
- hashCode
- toString

이러한 메서드들의 규약은 주석과 문서에 잘 설명되어 있습니다. `아이템 32: 추상화 규약을 지켜라`에서 설명했던 것처럼, Any 클래스를 상속받는 모든 매서드는 이러한 규약을 잘 지켜주는 것이 좋습니다. 이 메서드들은 자바 때부터 정의되어 있던 메서드라서 코틀린에서 중요한 위치를 차지하고 있으며, 수많은 객체와 함수들이 이 규약에 의존하고 있습니다. 따라서 규약을 위반하면, 일부 객체 또는 기능이 제대로 동작하지 않을 수도 있습니다. 중요한 내용이므로, 이번 아이템과 다음 아이템에서는 이러한 내용에 대해서 자세하게 알아보겠습니다. 그럼 equals부터 살펴봅시다.

### 동등성

코틀린에는 두 가지 종류의 동등성<small>(equality)</small>이 있습니다.

- 구조적 동등성<small>(structural equality)</small>: equals 메서드와 이를 기반으로 만들어진 `==` 연산자(`!=`포함)로 확인하는 동등성입니다. a가 nullable이 아니라면 `a == b`는 `a.equals(b)`로 변환되고, a가 nullable이라면 `a?.equals(b) ?: (b === null)`로 변환됩니다.
- 레퍼런스적 동등성<small>(referential equality)</small>: `===` 연산사(`!==` 포함)로 확인하는 동등성입니다. 두 피연산자가 같은 객체를 가리키면 true를 리턴합니다. equals는 모든 클래스의 슈퍼클래스인 Any에 구현되어 있으므로, 모든 객체에서 사용할 수 있습니다. 다만 연산자를 사용해서 다른 타입의 두 객체를 비교하는 것은 허용되지 않습니다.
  - `"".equals(1)`은 가능하지만, `"" == 1`은 불가능

```kotlin
open class Animal
class Book
Animal() == Book() // 오류: Animal과 Book에는 == 연산자를 사용할 수 없습니다.
Animal() === Book() // 오류: Animal과 Book에는 === 연산자를 사용할 수 없습니다.
```

물론 다음과 같이 타입을 비교하거나, 둘이 상속 관계를 갖는 경우에는 비교할 수 있습니다.

```kotlin
class Cat: Animal()
Animal() == Cat() // Cat은 Animal의 서브클래스이기 때문에 가능
Animal() === Cat() // Cat은 Animal의 서브클래스이기 때문에 가능
```

다른 타입의 두 객체를 비교하는 것은 큰 의미가 없으므로, 이렇게 구현되어 있습니다. 이는 이후에 equals와 관련된 규약을 다룰 때 다시 설명하겠습니다.

### equals가 필요한 이유

Any 클래스에 구현되어 있는 equals 메서드는 디폴트로 `===`처럼 두 인스턴스가 완전히 같은 객체인지를 비교합니다. 이 모든 객체는 디폴트로 유일한 객체라는 것을 의미합니다.

```kotlin
class Name(val name: String)
val name1 = Name("Marcin")
val name2 = Name("Marcin")
val name1Ref = name1

name1 == name1 // true
name1 == name2 // false
name1 == name1Ref // true

name1 === name1 // true
name1 === name2 // false
name1 === name1Ref // true
```

이러한 동작은 데이터베이스 연결, 리포지토리, 스레드 등의 활동 요소<small>(active element)</small>를 활용할 때 굉장히 유용합니다. 하지만 동등성을 약간 다른 형태로 표현해야 하는 객체가 있습니다. 예를 들어 두 객체가 기본 생성자의 프로퍼티가 같다면, 같은 객체로 보는 형태가 있을 수 있습니다. data 한정자를 붙여서 데이터 클래스로 정의하면, 자동으로 이와 같은 동등성으로 동작합니다.

```kotlin
data class Name(val name: String, val surname: String)
val name1 = Name("Marcin", "Moskala")
val name2 = Name("Marcin", "Moskala")
val name3 = Name("Maja", "Moskala")

name1 == name1 // true
name1 == name2 // ture, 데이터가 같다
name1 == name3 // true

name1 === name1 // true
name1 === name2 // false
name1 === name3 // false
```

데이터 클래스는 내부에 어떤 값을 갖고 있는지가 중요하므로, 이와 같이 동작하는 것이 좋습니다. 그래서 일반적으로 데이터 모델을 표현할 때는 data 한정자를 붙입니다.

데이터 클래스의 동등성은 모든 프로퍼티가 아니라 일부 프로퍼티와 비교해야 할 때도 유용합니다. 간단한 예로, 다음과 같은 날짜와 시간을 표현하는 객체를 살펴봅시다. 이 객체는 동등성 확인 때 검사되지 않는 `asStringCache`와 `changed`라는 프로퍼티를 갖습니다. 일반적으로 캐시를 위한 객체는 캐시에 영향을 주지 않는 프로퍼티가 복사되지 않는 것이 좋습니다. 그래서 equals 내부에서 `asStringCache`와 `changed`를 비교하지 않는 것입니다.

```kotlin
class DateTime(
    private var millis: Long = 0L,
    private var timeZone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false

    override fun equals(other: Any?): Boolean =
        other is DateTime &&
                other.millis == millis &&
                other.timeZone == timeZone

    // ...
}
```

다음과 같은 data 한정자를 사용해도 같은 결과를 낼 수 있습니다.

```kotlin
data class DateTime(
    private var millis: Long = 0L,
    private var timeZone: TimeZone? = null
) {
    private var asStringCache = ""
    private var changed = false
    // ...
}
```

참고로 이렇게 코드를 작성한 경우, 기본 생성자에 선언되지 않은 프로퍼티는 `copy`로 복사되지 않습니다. 기본 생성자에 선언되지 않는 프로퍼티까지 복사하는 것은 굉장히 의미 없는 일이므로, 기본 생성자에 선언되지 않은 것을 복사하지 않는 동작은 올바른 동작이라고 할 수 있습니다.

이렇게 data 한정자를 기반으로 동등성의 동작을 조작할 수 있으므로, 일반적으로 코틀린에서는 equals를 직접 구현할 필요가 없습니다. 다만 상황에 따라서 equals를 직접 구현해야 하는 경우가 있을 수도 있습니다.

또한 일부 프로퍼티만 같은지 확인해야 하는 경우 등이 있을 수 있습니다. 예를 들어 다음 코트들 살펴봅시다. 다음 User 클래스는 id만 같으면 같은 객체라고 판단합니다.

```kotlin
class User(
    val id: Int,
    val name: String,
    val surname: String
) {
    override fun equals(other: Any?): Boolean =
        other is User && other.id == id

    override fun hashCode(): Int = id
}
```

equals를 직접 구현해야 하는 경우를 정리해 보면, 다음과 같습니다.

- 기본적으로 제공되는 동작과 다른 동작을 해야 하는 경우
- 일부 프로퍼티만으로 비교해야 하는 경우
- data 한정자를 붙이는 것을 원하지 않거나, 비교해야 하는 프로퍼티가 기본 생성자에 없는 경우

### equals 구현하기

특별한 이유가 없는 이상, 직접 equals를 구현하는 것은 좋지 않습니다. 기본적으로 제공되는 것을 그대로 쓰거나, 데이터 클래스로 만들어서 사용하는 것이 좋습니다. 그래도 직접 구현해야 한다면, 반사적, 대칭적, 연속적, 일관적 동작을 하는지 꼭 확인하세요. 그리고 이러한 클래스는 final로 만드는 것이 좋습니다. 만약 상속을 한다면, 서브클래스에서 equals가 작동하는 방식을 변경하면 안 된다는 것을 기억하세요. 상속을 지원하면서도 완벽한 사용자 정의 equals 함수를 만드는 것은 거의 불가능에 가깝습니다. 참고로 데이터 클래스는 언제나 final입니다.