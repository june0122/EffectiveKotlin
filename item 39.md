# 아이템 39. 태그 클래스보다는 클래스 계층을 사용하라

큰 규모의 프로젝트에서는 상수 '모드'를 가진 클래스를 꽤 많이 볼 수 있습니다. 이러한 상수 모드를 태그<small>(tag)</small>라고 부르며, 태그를 포함한 클래스를 태그 클래스<small>(tagged class)</small>라고 부릅니다. 그런데 태그 클래스는 다양한 문제를 내포하고 있습니다. 이러한 문제는 서로 다른 책임을 한 클래스에 태그로 구분해서 넣는다는 것에서 시작합니다. 예를 들어 다음 코드를 살펴보면, 테스트에 사용되는 클래스로서 어떤 값이 기준에 만족하는지 확인하기 위해 사용되는 클래스를 볼 수 있습니다. 이 예제는 실제 큰 규모의 프로젝트에서 일부를 발췌하고, 조금 간단하게 만든 것입니다.

```kotlin
class ValueMatcher<T> private constructor(
    private val value: T? = null,
    private val matcher: Matcher
) {
    enum class Matcher {
        EQUAL,
        NOT_EQUAL,
        LIST_EMPTY,
        LIST_NOT_EMPTY
    }
    
    fun match(value: T?) = when (matcher) {
        Matcher.EQUAL -> value == this.value
        Matcher.NOT_EQUAL -> value != this.value
        Matcher.LIST_EMPTY -> value is List<*> && value.isEmpty()
        Matcher.LIST_NOT_EMPTY -> value is List<*> && value.isNotEmpty()
    }
    
    companion object {
        fun <T> equal(value: T) = ValueMatcher(value = value, matcher = Matcher.EQUAL)
        fun <T> notEqual(value: T) = ValueMatcher(value = value, matcher = Matcher.NOT_EQUAL)
        fun <T> emptyList() = ValueMatcher<T>(matcher = Matcher.LIST_EMPTY)
        fun <T> notEmptyList() = ValueMatcher<T>(matcher = Matcher.LIST_NOT_EMPTY)
    }
}
```

이러한 접근 방법에는 굉장히 많은 단점이 있습니다.

- 한 클래스에 여러 모드를 처리하기 위한 상용구<small>(boilerplate)</small>가 추가됩니다.
- 여러 목적으로 사용해야 하므로 프로퍼티가 일관적이지 않게 사용될 수 있으며, 더 많은 프로퍼티가 필요합니다. 예를 들어 위의 예제에서 value는 모드가 LIST_EMPTY 또는 LIST_NOT_EMPTY일 때 아예 사용되지도 않습니다.
- 요소가 여러 목적을 가지고, 요소를 여러 방법으로 설정할 수 있는 경우에는 상태의 일관성과 정확성을 지키기 어렵습니다.
- 팩토리 메서드를 사용해야 하는 경우가 많습니다. 그렇지 않으면 객체가 제대로 생성되었는지 확인하는 것 자체가 굉장히 어렵습니다.

코틀린은 그래서 일반적으로 태그 클래스보다 sealed 클래스를 많이 사용합니다. 한 클래스에 여러 모드를 만드는 방법 대신에, 각각의 모드를 여러 클래스로 만들고 타입 시스템과 다형성을 활용하는 것입니다. 그리고 이러한 클래스에는 sealed 한정자를 붙여서 서브클래스 정의를 제한합니다. 구현 방법은 다음과 같습니다.

```kotlin
sealed class ValueMatcher<T> {
    abstract fun match(value: T): Boolean
    
    class Equal<T>(val value: T): ValueMatcher<T>() {
        override fun match(value: T): Boolean = value == this.value
    }
    
    class NotEqual<T>(val value: T): ValueMatcher<T>() {
        override fun match(value: T): Boolean = value != this.value
    }
    
    class EmptyList<T> : ValueMatcher<T>() {
        override fun match(value: T): Boolean = value is List<*> && value.isEmpty()
    }
    
    class NotEmptyList<T> : ValueMatcher<T>() {
        override fun match(value: T): Boolean = value is List<*> && value.isNotEmpty()
    }
}
```

이렇게 구현하면 책임이 분산되므로 훨씬 깔끔합니다. 각각의 객체들은 자신에게 필요한 데이터만 있으며, 적절한 파라미터만 갖습니다. 이와 같은 계층을 사용하면, 태그 클래스의 단점을 모두 할 수 있습니다.

### sealed 한정자

반드시 sealed 한정자를 사용해야 하는 것은 아닙니다. 대신 abstract 한정자를 사용할 수도 있지만, sealed 한정자는 외부 파일에서 서브클래스를 만드는 행위 자체를 모두 제한합니다. 외부에서 추가적인 서브클래스를 만들 수 없으므로, 타입이 추가되지 않을 거라는 게 보장됩니다. 따라서 when을 사용할 때 else 브랜치를 따로 만들 필요가 없습니다. 이러한 장점을 이용해서 새로운 기능을 쉽게 추가할 수 있으며, when 구문에서 이를 처리하는 것을 잊어버리지 않을 수도 있습니다.

when은 모드를 구분해서 다른 처리를 만들 때 굉장히 편리합니다. 예를 들어 어떤 처리를 각각의 서브클래스에 구현할 필요 없이, when을 활용하는 확장 함수로 정의하면 한번에 구현할 수 잇습니다. 다음 코드는 reversed라는 확장 함수를 하나만 정의해서, 클래스의 종류에 따라서 서로 다른 처리를 하게 만듭니다.

```kotlin
fun <T> ValueMatcher<T>.reversed(): ValueMatcher<T> = when (this) {
    is ValueMatcher.EmptyList -> ValueMatcher.NotEmptyList()
    is ValueMatcher.NotEmptyList -> ValueMatcher.EmptyList()
    is ValueMatcher.Equal -> ValueMatcher.NotEqual(value)
    is ValueMatcher.NotEqual -> ValueMatcher.Equal(value)
}
```

반면 abstract 키워드를 사용하면, 다른 개발자가 새로운 인스턴스를 만들어서 사용할 수도 있습니다. 이러한 경우에는 함수를 abstract로 선언하고, 서브클래스 내부에 구현해야 합니다. when을 사용하면, 프로젝트 외부에서 새로운 클래스가 추가될 때 함수가 제대로 동작하지 않을 수 있기 때문입니다.

sealed 한정자를 사용하면, 확장 함수를 사용해서 클래스에 새로운 함수를 추가하거나, 클래스의 다양한 변경을 쉽게 처리할 수 있습니다. abstract 클래스는 계층에 새로운 클래스를 추가할 수 있는 여지를 남깁니다. 클래스의 서브클래스를 제어하려면, sealed 한정자를 사용해야 합니다. abstract는 상속과 관련된 설계를 할 때 사용합니다.

### 태그 클래스와 상태 패턴의 차이

태그 클래스와 상태 패턴<small>(state pattern)</small>을 혼동하면 안 됩니다. 상태 패턴은 객체의 내부 상태가 변화할 때, 객체의 동작이 변하는 소프트웨어 디자인 패턴입니다. 상태 패턴은 프론트엔드 컨트롤러<small>(controller)</small>, 프레젠터<small>(presenter)</small>, 뷰모델<small>(view-model)</small>을 설계할 때 많이 사용됩니다<small>(각각 MVC, MVP, MVVM 아키텍처에서)</small>.

예를 들어 아침 운동을 위한 애플리케이션을 만들 때, 각각의 운동 전에 준비 시간이 있습니다. 또한 운동을 모두 하고 나면 완료했다는 화면을 출력합니다.

상태 패턴을 사용한다면, 서로 다른 상태를 나타내는 클래스 계층 구조를 만들게 됩니다. 그리고 현재 상태를 나타내기 위한 읽고 쓸 수 있는 프로퍼티도 만들게 됩니다.

```kotlin
sealed class WorkoutState

class PrepareState(val exercise: Exercise): WorkoutState()

class ExerciseState(val exercise: Exercise): WorkoutState()

object DoneState: WorkoutState()

fun List<Exercise>.toStates(): List<WorkoutState> = flatMap { exercise ->
    listOf(PrepareState(exercise), ExerciseState(exercise))
} + DoneState

class WorkoutPresenter(/*...*/) {
    private var state: WorkoutState = states.first()
}
```

여기에서 차이점은 다음과 같습니다.

- 상태는 더 많은 책임을 가진 클래스입니다.
- 상태는 변경할 수 있습니다.

구체 상태<small>(concreate state)</small>는 객체를 활용해서 표현하는 것이 일반적이며, 태그 클래스보다는 sealed 클래스 계층으로 만듭니다. 또한 이를 immutable 객체로 만들고, 변경해야 할 때마다 state 프로퍼티를 변경하게 만듭니다. 그리고 뷰에서 이러한 state의 변화를 관찰<small>(observe)</small>합니다.

```kotlin
private var state: WorkoutState by Delegates.observable(states.first()) { _, _, - ->
    updateView()
}
```

### 정리

코틀린에서는 태그 클래스보다 타입 계층을 사용하는 것이 좋습니다. 그리고 일반적으로 이러한 타입 계층을 만들 때는 sealed 클래스를 사용합니다. 이는 상태 패턴과는 다릅니다. 타입 계층과 상태 패턴은 실질적으로 함께 사용하는 협력 관계라고 할 수 있습니다. 하나의 뷰를 가지는 경우보다는 여러 개의 상태로 구분할 수 있는 뷰를 가질 때 많이 활용합니다.