# 아이템 38. 연산 또는 액션을 전달할 때는 인터페이스 대신 함수 타입을 사용하라

대부분의 프로그래밍 언어에는 함수 타입이라는 개념이 없기에, 연산 또는 액션을 전달할 때 메서드가 하나만 있는 인터페이스를 활용하고, 이러한 인터페이스를 SAM<small>(Single-Abstract Method)</small>이라고 부릅니다.

#### 뷰를 클릭했을 때 발생하는 정보를 전달하는 SAM

```kotlin
interface OnClick {
    fun clicked(view: View)
}
```

함수가 SAM을 받는다면, 이러한 인터페이스를 구현한 객체를 전달받는다는 의미입니다.

```kotlin
fun setOnClickListener(listener: OnClick) {
    // ...
} 

setOnClickListener(object: OnClick {
    override fun clicked(view: View) {
        // ...
    }
})
```

이런 코드를 함수 타입을 사용하는 코드로 변경하면, 더 많은 자유를 얻을 수 있습니다.

```kotlin
fun setOnClickListener(listener: (View) -> Unit) {
    // ...
}
```

예를 들어 다음과 같은 방법으로 파라미터를 전달할 수 있습니다.

- 람다 표현식 또는 익명 함수로 전달

```kotlin
setOnClickListener { /*...*/ }
setOnClickListener(fun(view) { /*...*/ })
```

- 함수 레퍼런스 또는 제한된 함수 레퍼런스로 전달

```kotlin
setOnClickListener(::println)
setOnClickListener(this:showUsers)
```

- 선언된 함수 타입을 구현한 객체로 전달

```kotlin
class ClickListener: (View) -> Unit {
    override fun invoke(view: View) {
        // ...
    }
}

setOnClickListener(ClickListener())
```

이러한 방법들은 굉장히 광범위하게 사용됩니다. 참고로, SAM의 장점은 '그 아규먼트에 이름이 붙어 있는 것'이라고 말하는 사람도 있습니다. 하지만 타입 별칭<small>(type alias)</small>를 사용하면, 함수 타입도 이름을 붙일 수 있습니다.

```kotlin
typealias OnClick = (View) -> Unit
```

파라미터도 이름을 가질 수 있습니다. 이름을 붙이면, IDE의 지원을 받을 수 있다는 굉장히 큰 장점이 있습니다.

```kotlin
fun setOnClickListener(listener: OnClick) { /*...*/ }
typealias OnClick = (view:View) -> Unit
```

람다 표현식을 사용할 때는 아규먼트 분해<small>(destructure argument)</small>도 사용할 수 있습니다. 이것도 SAM보다 함수 타입을 사용하는 것이 훨씬 더 좋은 이유입니다.

여러 옵저버를 설정할 때, 이 장점을 확인할 수 있습니다. 고전적인 자바는 다음과 같이 인터페이스를 기반으로 구현했습니다.

```kotlin
class CalendarView {
    var listener: Listener? = null

    interface Listener {
        fun onDateClicked(date: Date)
        fun onPageChanged(date: Date)
    }
}
```

개인적으로 필자는 이는 게으름의 결과라고 생각합니다. API를 소비하는 사용자의 관점에서는 함수 타입을 따로따로 갖는 것이 훨씬 이용하기 쉽습니다.

```kotlin
class CalendarView {
    var onDateClicked: ((date: Date) -> Unit)? = null
    var onPageChanged: ((date: Date) -> Unit)? = null
}
```

이렇게 onDateClicked와 onPageChanged를 한꺼번에 묶지 않으면, 각각의 것을 독립적으로 변경할 수 있다는 장점이 생깁니다.

인터페이스를 사용해야 하는 특별한 이유가 없다면, 함수 타입을 활용하는게 좋습니다. 함수 타입은 다양한 지원을 받을 수 있으며, 코틀린 개발자들 사이에서 이미 널리 사용되고 있습니다.

### 언제 SAM을 사용해야 할까?

딱 한 가지 경우에는 SAM을 사용하는 것이 좋습니다. 코틀린이 아닌 다른 언어에서 사용할 클래스를 설계할 때입니다. 자바에서는 인터페이스가 더 명확합니다. 함수 타입으로 만들어진 클래스는 자바에서 타입 별칭과 IDE의 지원 등을 제대로 받을 수 없습니다. 마지막으로 다른 언어(자바 등)에서 코틀린의 함수 타입을 사용하려면, Unit을 명시적으로 리턴하는 함수가 필요합니다.

```kotlin
class CalendarView {
    var onDateClicked: ((date: Date) -> Unit)? = null
    var onPageChanged: OnDateClicked? = null
}

interface OnDateClicked {
    fun onClick(date: Date)
}

// 자바
CalendarView c = new CalendarView();
c.setOnDateClicked(date -> Unit.INSTANCE);
c.setOnPageChanged(date -> {});
```

자바에서 사용하기 위한 API를 설계할 때는 함수 타입보다 SAM을 사용하는 것이 합리적입니다. 하지만 이외의 경우에는 함수 타입을 사용하는 것이 좋습니다.