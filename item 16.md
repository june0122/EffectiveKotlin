# 아이템 16. 프로퍼티는 동작이 아니라 상태를 나타내야 한다

코틀린의 프로퍼티는 자바의 필드와 비슷해 보이지만, 사실 서로 완전히 다른 개념입니다.

#### 코틀린의 프로퍼티

```kotlin
var name: String? = null
```

#### 자바의 필드

```java
String name = null;
```

둘 다 데이터를 저장한다는 점은 같지만, 프로퍼티에는 더 많은 기능이 있습니다. 일단 기본적으로 프로퍼티는 사용자 정의 세터와 게터를 가질 수 있습니다.

```kotlin
var name: String? = null
    get() = field?.toUpperCase()
    set(value) {
        if (!value.isNullOrBlack()) {
            field = value
        }
    }
```

이 코드에서 `field`라는 식별자를 확인할 수 있습니다. 이는 프로퍼티의 데이터를 저장해두는 백킹 필드<small>(backing field)</small>에 대한 레퍼런스입니다. 이러한 백킹 필드는 세터와 게터의 디폴트 구현에 사용되므로, 따로 만들지 않아도 디폴트로 생성됩니다. 참고로 `val`을 사용해서 읽기 전용 프로퍼티를 만들 때는 field가 만들어지지 않습니다.

```kotlin
val fullName: String
    get() = "$name $surname"
```

`var`을 사용해서 만든 읽고 쓸 수 있는 프로퍼티는 게터와 세터를 정의할 수 있습니다. 이러한 프로퍼티를 <b>파생 프로퍼티<small>(derived property)</small></b>라고 부르며, 자주 사용됩니다.

이처럼 코틀린의 모든 프로퍼티는 디폴트로 캡슐화되어 있습니다. 예를 들어 자바 표준 라이브러리 `Date`를 활용해 객체에 날짜를 저장해서 많이 활용한 상황을 가정해 봅시다. 그런데 프로젝트를 진행하는 중에 직렬화 문제 등으로 객체를 더 이상 이러한 타입으로 저장할 수 없게 되었는데, 이미 프로젝트 전체에서 이 프로퍼티를 많이 참조하고 있다면 어떻게 해야 할까요? 코틀린은 데이터를 `millis`라는 별도의 프로퍼티로 옮기고. 이를 활용해서 `date` 프로퍼티에 데이터를 저장하지 않고, 랩<small>(wrap)</small>/언랩<small>(unwrap)</small>하도록 코드를 변경하기만 하면 됩니다.

```kotlin
var date: Date
    get() = Date(millis)
    set(value) {
        millis = value.time
    }
```

프로퍼티는 필드가 필요 없습니다. 오히려 프로퍼티는 개념적으로 접근자<small>(val의 경우 게터, var의 경우 게터와 세터)</small>를 나타냅니다. 따라서 코틀린은 인터페이스에도 프로퍼티를 정의할 수 있는 것입니다.

```kotlin
interface Person {
    val name: String
}
```

이렇게 코드를 작성하면, 이는 게터를 가질 거라는 것을 나타냅니다. 따라서 다음과 같이 오버라이드할 수 있습니다.

```kotlin
open class Supercomputer {
    open val theAnswer: Long = 42
}

class AppleComputer : Supercomputer() {
    override val theAnswer: Long = 1_800_275_2273
}
```

마찬가지의 이유로 프로퍼티를 위임할 수도 있습니다.

```kotlin
val db: Database by lazy { connectToDb() }
```

프로퍼티 위임<small>(property delegation)</small>은 `아이템 21: 일반적인 프로퍼티 패턴은 프로퍼티 위임으로 만들어라`에서 자세히 설명됩니다. 프로퍼티는 본질적으로 함수이므로, 확장 프로퍼티를 만들 수도 있습니다.

```kotlin
val Context.preferences: SharedPreferences
	get() = PreferenceManager.getDefaultSharedPreferences(this)

val Context.inflater: LayoutInflater
	get() = getSystemService(Context.LAYOUT_INFLATER_SERVICE) as LayoutInflater

val Context.notificationManager: NotificationManager
    get() = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
```

코드에서 확인할 수 있는 것처럼 프로퍼티는 필드가 아니라 접근자를 나타냅니다. 이처럼 프로퍼티를 함수 대신 사용할 수도 있지만, 그렇다고 완전히 대체해서 사용하는 것은 좋지 않습니다. 예를 들어 **프로퍼티로 다음과 같이 알고리즘의 동작을 나타내는 것은 좋지 않습니다.**

```kotlin
// 이렇게 하지 마세요!!
val Tree<Int>.sum: Int
    get() = when (this) {
        is Leaf -> value
        is Node -> left.sum + right.sum
    }
```

여기에서 `sum` 프로퍼티는 모든 요소를 반복 처리하므로, 알고리즘의 동작을 나타낸다고 할 수 있습니다. 이런 프로퍼티는 여러 가지 오해를 불러일으킬 수 있습니다. 큰 컬렉션의 경우 답을 찾을 때, 많은 계산량이 필요합니다. 하지만 관습적으로 이런 게터에 그런 계산량이 필요하다고 예상하지는 않습니다. 따라서 이러한 처리는 프로퍼티가 아니라 함수로 구현해야 합니다.

```kotlin
fun Tree<Int>.sum(): Int = when (this) {
    is Leaf -> value
    is Node -> left.sum() + right.sum()
}
```

**원칙적으로 프로퍼티는 상태를 나타내거나 설정하기 위한 목적으로만 사용하는 것이 좋고, 다른 로직 등을 포함하지 않아야 합니다.** 어떤 것을 프로퍼티로 해야 하는지 판단할 수 있는 간단한 질문이 있습니다. "이 프로퍼티를 함수로 정의할 경우, 접두사로 *get* 또는 *set*을 붙일 것인가?" 만약 아니라면, 이를 프로퍼티로 만드는 것은 좋지 않습니다.

### 프로퍼티 대신 함수를 사용하는 것이 좋은 경우

- 연산 비용이 높거나, 복잡도가 <i>O(1)</i>보다 큰 경우: 관습적으로 프로퍼티를 사용할 때 연산 비용이 많이 필요하다고 생각하지 않습니다. 연산 비용이 많이 들어간다면, 함수를 사용하는 것이 좋습니다. 그래야 사용자가 연산 비용을 예측하기 쉽고, 이를 기반으로 캐싱 등을 고려할 수 있기 때문입니다.
- 비즈니스 로직<small>(애플리케이션의 동작)</small>을 포함하는 경우: 관습적으로 코드를 읽을 때 프로퍼티가 로깅, 리스너 통지, 바인드된 요소 변경과 같은 단순한 동작 이상을 할 거라고 기대하지 않습니다.
- 결정적이지 않은 경우: 같은 동작을 연속적으로 두 번 했는데 다른 값이 나올 수 있다면, 함수를 사용하는 것이 좋습니다.
- 변환의 경우: 변환은 관습적으로 `Int.toDouble()`과 같은 변환 함수로 이루어집니다. 따라서 이러한 변환을 프로퍼티로 만들면, 오해를 불러 일으킬 수 있습니다.
- 게터에서 프로퍼티의 상태 변경이 일어나야 하는 경우: 관습적으로 게터에서 프로퍼티의 상태 변화를 일으킨다고 생각하지는 않습니다. 따라서 게터에서 프로퍼티의 상태 변화를 일으킨다면, 함수를 사용하는 것이 좋습니다.

예를 들어 요소의 합계를 계산하려면, 모든 요소를 더하는 반복 처리가 필요합니다<small>(어떤 처리가 실질적으로 이루어지므로, 상태가 아니라 동작입니다)</small>. 선형 복잡도를 가지므로, 이는 프로퍼티가 아니라 함수로 정의하는 것이 좋습니다.

표준 라이브러리에서도 다음과 같이 함수로 정의되어 있습니다.

```kotlin
val s = (1..100).sum()
```

반대로 상태를 추출/설정할 때는 프로퍼티를 사용해야 합니다. 특별한 이유가 없다면 함수를 사용하면 안 됩니다.

```kotlin
// 이렇게 하지 마세요!!
class UserIncorrect {
    private var name: String = ""

    fun getName() = name

    fun setName(name: String) {
        this.name = name
    }
}

class UserCorrect {
    var name: String = ""
}
```

많은 사람은 경험적으로, **프로퍼티는 상태 집합을 나타내고, 함수는 행동을 나타낸다**고 생각합니다.