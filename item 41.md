# 아이템 41. hashCode의 규약을 지켜라

오버라이드할 수 있는 Any의 메서드로는 hashCode가 있습니다. 일단 hashCode가 왜 필요한지부터 살펴보겠습니다. hashCode 함수는 수많은 컬렉션과 알고리즘에 사용되는 자료 구조인 해시 테이블을 구축할 때 사용됩니다.

### 해시 테이블

해시 테이블의 개념은 컴퓨터 사이언스에서 매우 많이 사용됩니다.

- 데이터베이스
- 인터넷 프로토콜
- 여러 언어의 표준 라이브러리 컬렉션

코틀린/JVM에 있는 기본 세트<small>(LinkedHashSet)</small>과 기본 맵<small>(LinkedHashMap)</small>도 이를 사용합니다. 코틀린은 해시 코드를 만들 때 hashCode함수를 사용합니다.

- 일반적으로 hashCode 함수가 Int를 리턴하므로, 32비트 부호 있는 정수만큼의 버킷이 만들어집니다. 즉, 4294967296개의 버킷이 만들어집니다. 한두 개의 요소만 포함할 세트로는 이는 너무 큰 크기입니다. 따라서 기본적으로 숫자를 더 작게 만드는 변환을 사용하다가, 필요한 경우 변환 방법을 바꿔서 해시 테이블을 크게 만들고, 요소를 재배치합니다.

### 가변성과 관련된 문제

요소가 추가될 때만 해시 코드를 계산합니다. 요소가 변경되어도 해시 코드는 계산되지 않으며, 버킷 재배치도 이루어지지 않습니다. 그래서 기본적인 LinkedHashSet와 'LinkedHashMap의 키'는 한 번 추가한 요소를 변경할 수 없습니다.

```kotlin
data class FullName(
    var name: String,
    var surname: String
)

val person = FullName("Kamiyu", "Bidan")
val s = mutableSetOf<FullName>()
s.add(person)
person.surname = "Ray"
print(person) // FullName(name=Kamiyu, surname=Ray)
print(person in s) // false
print(s.first() == person) // true
```

이러한 문제는 `아이템 1: 가변성을 제한하라`에서 간단하게 다루었습니다. 그래서 해시 등의 'mutable 프로퍼티로 요소를 조합하는 자료 구조'에서는 mutable 객체가 사용되지 않습니다. 따라서 세트와 맵의 키로 mutable 요소를 사용하면 안 되며, 사용하더라도 요소를 변경해서는 안 됩니다. 이러한 이유로 immutable 객체를 많이 사용합니다.

### hashCode의 규약

hashCode는 명확한 규약이 있습니다. 코틀린 1.3.11을 기준으로 공식적인 규약을 정리해 보면, 다음과 같습니다.

- 어떤 객체를 변경하지 않았다면(equals에서 비교에 사용된 정보가 수정되지 않는 이상), hashCode는 여러 번 호출해도 그 결과가 항상 같아야 합니다.
- equals 메서드의 실행 결과로 두 객체가 같다고 나오면, hashCode 메서드의 호출 결과도 같다고 나와야 합니다.

첫 번째 요구 사항은 일관성 유지를 위해서 hashCode가 필요하다는 것입니다. 두 번째 요구 사항은 많은 개발자가 자주 잊어버리는 것들 중 하나이므로 강조되어야 합니다. hashCode는 equals와 같이 일관성 있는 동작을 해야 합니다. 즉, 같은 요소는 반드시 같은 해시 코드를 가져야 한다는 의미입니다. 그렇지 않으면 컬렉션 내부 요소가 들어 있는지 제대로 확인하지 못하는 문제가 발생할 수 있습니다.

```kotlin
data class FullName(
    var name: String,
    var surname: String
) {
    override fun equals(other: Any?): Boolean =
        other is FullName
            && other.name == name
            && other.surname == surname
}

val s = mutableSetOf<FullName>()
s.add(FullName("Kamiyu", "Bidan"))
val p = FullName("Kamiyu", "Bidan")
print(p in s) // false
print(p == s.first()) // true
```

그래서 코틀린은 equals 구현을 오버라이드할 때, hashCode도 함께 오버라이드하는 것을 추천합니다.

필수 요구 사항은 아니지만 제대로 사용하려면 지켜야 하는 요구 사항이 있습니다. 바로 hashCode는 최대한 요소를 넓게 퍼뜨려야 한다는 것입니다. 다른 요소라면 최대한 다른 해시 값을 갖는 것이 좋습니다.

많은 요소가 같은 버킷에 배치되는 경우를 생각해 봅시다. 해시 테이블을 쓸 이유 자체가 없어질 것입니다. 극단적인 예로 hashCode가 항상 동일한 숫자를 리턴하는 경우를 살펴봅시다. 이렇게 구현하면, 요소를 항상 같은 버킷에 배치할 것입니다. 이렇게 구현한다고 규약을 위반하는 것은 아니지만, 쓸모가 없어질 것입니다. hashCode가 항상 같은 값을 리턴한다면, 해시 테이블을 사용할 필요가 없습니다.

다음 코드를 살펴봅시다. Terrible은 hashCode가 항상 0을 리턴하는 코드입니다. 성능을 확인할 수 있게 equals 내부에 사용된 횟수를 계산하는 카운터를 추가했습니다. 실행 결과를 보면, Proper와 Terrible의 equals 메서드 실행 횟수가 크게 차이 나는 것을 볼 수 있습니다.

### hashCode 구현하기

일반적으로 data 한정자를 붙이면, 코틀린이 알아서 적당한 equals와 hashCode를 정의해 주므로 이를 직접 정의할 일은 거의 없습니다. 다만 equals를 따로 정의했다면, 반드시 hashCode도 함께 정의해 줘야 합니다. equals를 따로 정의하지 않았다면, 정당한 이유가 없는 이상 hashCode를 따로 정의하지 않는 것이 좋습니다. equals로 같은 요소라고 판정되는 요소는 hashCode가 반드시 같은 값을 리턴해야 합니다.

hashCode는 기본적으로는 equals에서 비교에 사용되는 프로퍼티를 기반ㄷ으로 해시 코드를 만들어야 합니다. 해시 코드를 어떻게 만들어 낼까요? 일반적으로 모든 해시 코드의 값을 더합니다. 더하는 과정마다 이전까지의 결과에 31을 곱한 뒤 더해 줍니다. 물론 31일 필요는 없지만, 관례적으로 31을 많이 사용합니다. data 한정자를 붙일 때도 이렇게 구현됩니다.

#### 일반적인 equals와 hashCode의 구현 예

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

   override fun hashCode(): Int {
       var result = millis.hashCode()
       result = result * 31 + timeZone.hashCode()
       return result
   }
}
```

이때 유용한 함수로는 코틀린/JVM의 Objects.hashCode가 있습니다. 이 함수는 해시를 계산해 줍니다.

```kotlin
override fun hashCode(): Int =
   Objects.hash(timeZone, millis)
```

코틀린 stdlib에는 이러한 함수가 따로 없습니다. 따라서 다른 플랫폼에서는 다음과 같은 함수를 구현해서 사용해야 합니다.

```kotlin
override fun hashCode(): Int =
   hashCodeFrom(timeZone, millis)

inline fun hashCodeOf(vararg values: Any?) =
   values.fold(0) { acc, value ->
       (acc * 31) + value.hashCode()
   }
```

코틀린 stdlib이 이러한 함수를 기본적으로 제공하지 않는 이유는 사실 hashCode를 우리가 직접 구현할 일이 거의 없기 때문입니다. 예를 들어 위의 DateTime 클래스는 equals와 hashCode를 직접 구현하지 않아도, 다음과 같이 data 한정자를 붙이기만 하면 됩니다.

```kotlin
data class DateTime2(
   private var millis: Long = 0L,
   private var timeZone: TimeZone? = null
) {
   private var asStringCache = ""
   private var changed = false
}
```

hashCode를 구현할 때 가장 중요한 규칙은 '언제나 equals와 일관된 결과가 나와야 한다'입니다. 같은 객체라면 언제나 같은 값을 리턴하게 만들어 주세요.

