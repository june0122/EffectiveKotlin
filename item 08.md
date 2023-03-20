# 아이템 8. 적절하게 null을 처리하라

null은 '값이 부족하다<small>(lack of value)</small>'는 것을 나타냅니다. 프로퍼티가 null이라는 것은 **값이 설정되지 않았거나, 제거**되었다는 것을 나타냅니다.

함수가 null을 리턴한다는 것은 여러 의미를 가질 수 있습니다. 예시로,

- `String.toIntOrNull()`은 String을 Int로 적절하게 변환할 수 없는 경우 null을 리턴
- `Iterable<T>.firstOrNull(() -> Boolean)`은 주어진 조건에 맞는 요소가 없을 경우 null을 리턴

이처럼 null은 최대한 명확한 의미를 갖는 것이 좋습니다. 이는 nullable 값을 처리해야 하기 때문인데, 이를 처리하는 사람은 API를 사용하는 개발자입니다. 

```kotlin
val printer: Printer? = getPrinter()
printer.print() // 컴파일 오류

printer.print() // 안전 호출
if (printer != null) printer.print() // 스마트 캐스팅
printer!!.print() // not-null assertion
```

기본적으로 nullable 타입은 세 가지 방법으로 처리합니다.

1. `?.`, 스마트 캐스팅, Elvis 연산자 등을 활용해서 안전하게 처리
2. 오류를 throw한다.
3. 함수 또는 프로퍼티를 리팩토링해서 nullable 타입이 나오지 않게 한다.

## null을 안전하게 처리하기

null을 안전하게 처리하는 방법 중 널리 사용되는 방법으로는 안전 호출<small>(safe call)</small>과 스마트 캐스팅<small>(smart casting)</small>이 있습니다.

```kotlin
printer?.print() // 안전 호출
if (printer != null) printer.print() // 스마트 캐스팅
```

두 가지 모두 printer가 null이 아닐 때 print 함수를 호출합니다. 애플리케이션 사용자의 관점에서 가장 안전한 방법이며, 사실 개발자에게도 정말 편리하여 nullable 값을 처리할 때 이 방법을 가장 많이 활용합니다.

코틀린은 nullable 변수와 관련된 처리를 굉장히 광범위하게 지원합니다. 대표적으로 인기 있는 다른 방법은 Elvis 연산자를 사용하는 것입니다. Elvis 연산자는 오른쪽에 return 또는 throw을 포함한 모든 표현식이 허용됩니다.

> return과 throw 모두 `Noting`<small>(모든 타입의 서브타입)</small>을 리턴하게 설계되어서 가능

```kotlin
val printerName1 = printer?.name ?: "Unnamed"
val printerName2 = printer?.name ?: return
val printerName3 = printer?.name ?:
    throw Error("Printer must be named")
```

많은 객체가 nullable과 관련된 처리를 지원합니다.
- 예를 들어 컬렉션 처리를 할 때 무언가 없다는 것을 나타낼 때는 null이 아닌 빈 컬렉션을 사용하는 것이 일반적입니다.
- 따라서 `Collection<T>?.orEmpty()` 확장 함수를 사용하면 nullable이 아닌 `List<T>`를 리턴받습니다.

스마트 캐스팅은 코틀린의 규약 기능<small>(contracts feature)</small>을 지원합니다. 이 기능을 사용하면 다음 코드처럼 스마트 캐스팅 가능합니다.

```kotlin
println("What is your name?")
val name = readLine()
if (!name.isNullOrBlank) {
    println("Hello ${name.toUpperCase()}")
}

val news: List<News>? = getNews()
if (!news.isNullOrEmpty()) {
    news.forEach { notifyUser(it) }
}
```

## 오류 throw하기

이전에 살펴보았던 코드에서는 printer가 null일 때, 이를 개발자에게 알리지 않고 코드가 그대로 진행됩니다. 하지만 printer가 null이 되리라 예상하지 못했다면, print 메서드가 호출되지 않아서 이상할 것이며, 개발자가 오류를 찾기 어렵게 만듭니다. 따라서 다른 개발자가 어떤 코드를 보고 선입견처럼 '당연히 그럴 것이다'라고 생각하게 되는 부분이 있고, 그 부분에서 문제가 발생할 경우에는 개발자에게 오류를 강제로 발생시켜 주는 것이 좋습니다. 오류를 강제로 발생시킬 때는 `thorw`, `!!`, `requireNotNull`, `checkNotNull` 등을 활용합니다.

```kotlin
fun process(user: User) {
    requireNotNull(user.name)
	val context = checkNotNull(context)
	val networkService =
		getNetworkService(context) ?:
		throw NoInternetConnection()
	networkService.getData { data, userData ->
		show(data!!, userData!!)
	}
}
```

## not-null assertion(!!)과 관련된 문제

nullable을 처리하는 가장 간단한 방법은 not-null assertion인 `!!`을 사용하는 것이지만, `!!`을 사용하면 자바에서 nullable을 처리할 때 발생할 수 있는 문제가 똑같이 발생합니다. 어떤 대상이 null이 아니라고 생각하고 다루면 NPE 예외가 발생합니다.

<b>`!!`은 사용하기 쉽지만 좋은 해결 방법은 아닙니다.</b>
- 예외가 발생할 때, 어떤 설명도 없는 제네릭 예외<small>(generic exception)</small>가 발생
- 코드가 짧고 너무 사용하기 쉽다 보니 남용하게 되는 문제

`!!`은 **타입은 nullable이지만, null이 나오지 않는 다는 것이 거의 확실한 상황에서 많이 사용**됩니다. 하지만 현재 확실하다고, 미래에 확실한 것은 아닙니다.

nullability<small>(널일 수 있는지)</small>와 관련된 정보는 숨겨져 있으므로, 굉장히 쉽게 놓칠 수 있습니다. 변수와 비슷합니다. 변수를 일단 선언하고, 이후에 사용하기 전에 값을 할당해서 사용하기로 하고, 다음과 같은 코드를 작성했다고 해 봅시다. 이처럼 변수를 null로 설정하고, 이후에 `!!` 연산자를 사용하는 방법은 좋은 방법이 아닙니다.

```kotlin
class UserControllerTest {

    private var dao: UserDao? = null
    private var controller: UserController? = null

    @BeforeEach
    fun init() {
        dao = mockk()
        controller = UserController(dao!!)
    }

    @Test
    fun test() {
        controller!!.doSomething()
    }
}
```

이렇게 코드를 작성하면 이후에 프로퍼티를 계속해서 언팩<small>(unpack)</small>해야 하므로 사용하기 귀찮습니다. 또한 해당 프로퍼티가 실제로 이후에 의미 있는 null 값을 가질 가능성 자체를 차단해 버립니다. 이러한 코드를 작성하는 올바른 방법은 `lateinit` 또는 `Delegates.notNull`을 사용하는 것입니다.

미래에 코드가 어떻게 변화할지는 아무도 알 수 없습니다. `!!` 연산자를 사용하거나 명시적으로 예외를 발생시키는 형태로 설계하면, 미래의 어느 시점에서 해당 코드가 오류를 발생시킬 수 있다는 것을 염두에 둬야 합니다. 예외는 예상하지 못한 잘못된 부분을 알려 주기 위해서 발생하는 것입니다<small>(`아이템 7: 결과 부족이 발생할 경우 null과 Failure를 사용하라`)</small>. 하지만 명시적 오류눈 제네릭 NPE보다는 훨씬 더 많은 정보를 제공해줄 수 있으므로 `!!` 연산자를 사용하는 것보다는 훨씬 좋습니다.

`!!` 연산자가 의미 있는 경우는 굉장히 드뭅니다. 일반적으로 nullability가 제대로 표현되지 않는 라이브러리를 사용할 때 정도에만 사용해야 합니다. 코틀린을 대상으로 설계된 API를 활용한다면 `!!` 연산자를 사용하는 것을 이상하게 생각해야 합니다.

`!!` 연산자를 보면 반드시 조심하고, 무언가가 잘못되어 있을 가능성을 생각합시다.

## 의미 없는 nullability 피하기

nullability는 어떻게든 적절하게 처리해야 하므로, 추가 비용이 발생합니다. 따라서 필요한 경우가 아니라면, nullability 자체를 피하는 것이 좋습니다. null은 중요한 메시지를 전달하는 데 사용될 수 있습니다. 따라서 다른 개발자가 보기에 의미가 없을 때는 null을 사용하지 않는 것이 좋습니다. 만약 이유 없이 null을 사용했다면, 다른 개발자들이 코드를 작성할 때, 위험한 `!!` 연산자를 사용하게 되고, 의미 없이 코드를 더럽히는 예외 처리를 해야 할 것입니다.

nullability를 피할 때 사용할 수 있는 몇 가지 방법을 소개하겠습니다.

- 클래스에서 nullability에 따라 여러 함수를 만들어서 제공하기
  - 대표적인 예로 `List<T>`의 get과 getOrNull 함수가 있습니다.
- 어떤 값이 클래스 생성 이후에 확실하게 설정된다는 보장이 있다면 laitinit 프로퍼티와 notNull 델리게이트를 사용
- 빈 컬렉션 대신 null을 리턴하지 않기
  - `List<Int>?`와 `Set<String?>`과 같은 컬렉션을 빈 컬렉션으로 둘 때와 null로 둘 때는 의미가 완전히 다릅니다. null은 컬렉션 자체가 없다는 것을 나타냅니다.
  - 요소가 부족하다는 것을 나타내려면 빈 컬렉션을 사용하세요.
- nullable enum과 None enum 값은 완전히 다른 의미입니다.
  - null enum은 별도로 처리해야 하지만, None enum 정의에 없으므로 필요한 경우에 사용하는 쪽에서 추가해서 활용할 수 있다는 의미입니다.

## lateinit 프로퍼티와 notNull 델리게이트

클래스가 클래스 생성 중에 초기화할 수 없는 프로퍼티를 가지는 것은 드문 일은 아니지만 분명 존재하는 일입니다. 이러한 프로퍼티는 사용 전에 반드시 초기화해서 사용해야 합니다.

#### 다른 함수들보다도 먼처 호출되는 함수에서 프로퍼티가 설정되는 JUnit의 `@BeforeEach`

```kotlin
class UserControllerTest {

    private var dao: UserDao? = null
    private var controller: UserController? = null

    @BeforeEach
    fun init() {
        dao = mockk()
        controller = UserController(dao!!)
    }

    @Test
    fun test() {
        controller!!.doSomething()
    }
}
```

프로퍼티를 사용할 때마다 nullable에서 null이 아닌 것으로 타입 변환하는 것은 바람직하지 않습니다. 이러한 값은 테스트 전에 설정될 것이라는 것이 명확하므로, 의미 없는 코드가 사용된다고 할 수 있습니다. 이러한 코드에 대한 바람직한 해결책은 나중에 속성을 초기화할 수 있는, lateinit 한정자를 사용하는 것입니다. lateinit 한정자는 프로퍼티가 이후에 설정될 것임을 명시하는 한정자입니다.

```kotlin
class UserControllerTest {

    private lateinit var dao: UserDao
    private lateinit var controller: UserController

    @BeforeEach
    fun init() {
        dao = mockk()
        controller = UserController(dao)
    }

    @Test
    fun test() {
        controller.doSomething()
    }
}
```

물론 lateinit를 사용할 경우에도 비용이 발생합니다. 만약 초기화 전에 값을 사용하려고 하면 예외가 발생합니다.
- 사용하는 것에 걱정할 필요 없이, 처음 사용학기 전에 반드시 초기화가 되어 있을 경우에만 lateinit을 붙이면 됩니다.
- 만약 그런 값이 사용되어 예외가 발생한다면, 그 사실을 알아야 하므로 예외가 발생하는 것은 오히려 좋은 일입니다.

#### lateinit과 nullable의 비교

- `!!` 연산자로 언팩<small>(unpack)</small>하지 않아도 됨
- 이후에 어떤 의미를 나타내기 위해서 null을 사용하고 싶을 때, nullable로 만들 수도 있음
- 프로퍼티가 초기화된 이후에는 초기화되지 않은 상태로 돌아갈 수 없음

lateinit은 프로퍼티를 처음 사용하기 전에 반드시 초기화될 거라고 예상되는 상황에 활용합니다. 이러한 상황으로는 라이프 사이클<small>(lifecycle)</small>을 갖는 클래스처럼 메서드 호출에 명확한 순서가 있을 경우가 있습니다.

- 안드로이드: Activity의 onCreate
- iOS: UIViewController의 viewDidAppear
- 리액트: React.Component의 componentDidMount

```kotlin
class DoctorActivity: Activity() {
    private var doctorId: Int by Delegates.notNull()
    private var fromNotification: Boolean by Delegates.notNull()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        doctorId = intent.extras.getInt(DOCTOR_ID_ARG)
        fromNotification = intent.extras.getBoolean(FROM_NOTIFICATION_ARG)
    }
}
```

위의 코드처럼 onCreate 때 초기화하는 프로퍼티는 지연 초기화하는 형태로 다음과 같이 프로퍼티 위임<small>(property delegation)</small>을 사용할 수도 있습니다.

```kotlin
class DoctorActivity: Activity() {
    private var doctorId: Int by arg(DOCTOR_ID_ARG)
    private var fromNotification: Boolean by arg(FROM_NOTIFICATION_ARG)
}
```

프로퍼티 위임을 사용하는 패턴은 `아이템 21: 일반적인 프로퍼티 패턴은 프로퍼티 위임으로 만들어라`에서 자세하게 다룰 예정입니다.

프로퍼티 위임을 사용하면, nullability로 발생하는 여러 가지 문제를 안전하게 처리할 수 있습니다.