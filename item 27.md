# 아이템 27. 변화로부터 코드를 보호하려면 추상화를 사용하라

추상화를 통해 변화로부터 코드를 보호하는 행위가 어떤 자유를 가져오는지 세 가지 실제 사례를 살펴보고, 여러 추상화의 균형을 맞추는 방법에 대해서 알아봅시다.

### 상수

일단 가장 간단한 추상화인 상수<small>(constant value)</small>부터 알아봅시다. 리터럴은 아무것도 설명하지 않습니다. 따라서 코드에서 반복적으로 등장할 때 문제가 됩니다. 이러한 리터럴을 상수 프로퍼티로 변경하면 해당 값에 의미있는 이름을 붙일 수 있으며, 상수의 값을 변경해야 할 때 훨씬 쉽게 변경할 수 있습니다. 비밀번호 유효성을 검사하는 간단한 예를 살펴봅시다.

```kotlin
fun isPasswordVaild(text: String): Boolean {
    if (text.length < 7) return false
    // ...
}
```

여기서 숫자 7은 아마도 '비밀번호의 최소 길이'를 나타내겠지만, 이해하는 데 시간이 걸립니다. 상수로 빼낸다면 훨씬 쉽게 이해할 수 있을 것입니다.

```kotlin
const val MIN_PASSWORD_LENGTH = 7

fun isPasswordVaild(text: String): Boolean {
    if (text.length < MIN_PASSWORD_LENGTH) return false
    // ...
}
```

이렇게 하면 '비밀번호의 최소 길이'를 변경하기도 쉽습니다. 함수의 내부 로직을 전혀 이해하지 못해도, 상수의 값만 변경하면 됩니다. 그래서 두 번 이상 사용되는 값은 이렇게 상수로 추출하는 것이 좋습니다. 예를 들어 데이터베이스에 동시에 연결할 수있는 최대 스레드 수를 다음과 같이 정의했다고 합시다.

```kotlin
val MAX_THREADS = 10
```

일단 이렇게 추출하면 변경이 필요할 때 쉽게 변경할 수 있습니다. 이러한 숫자가 프로젝트 전체에 퍼져 있다면 변경하기 정말 힘들 것입니다.

상수로 추출하면
- 이름을 붙일 수 있고
- 나중에 해당 값을 쉽게 변경할 수 있습니다.

### 함수

애플리케이션을 개발하고 있는데, 사용자에게 토스트 메시지를 자주 출력해야 하는 상황이 발생했다고 합시다. 기본적으로 다음과 같은 코드를 사용해서 토스트 메시지를 출력합니다.

```kotlin
Toast.makeText(this, message, Toast.LENTH_LONG).show()
```

이렇게 많이 사용되는 알고리즘은 다음과 같이 간단한 확장 함수로 만들어서 사용할 수 있습니다.

```kotlin
fun Context.toast(
    message: String,
    duration: Int = Toast.LENTH_LONG
) {
    Toast.makeText(this, message, duration).show()
}

// 사용
context.toast(message)

// 액티비티 또는 컨텍스트의 서브 클래스에서 사용할 경ㅇ우
toast(message)
```

이렇게 일반적인 알고리즘을 추출하면, 토스트를 출력하는 코드를 항상 기억해두지 않아도 괜찮습니다. 또한 이후에 토스트를 출력하는 방법이 변경된다 할지라도, 확장 함수 부분만 수정하면 되므로 유지보수성이 향상됩니다.

만약 토스트가 아니라 스낵바라는 다른 형태의 방식으로 출력해야 한다면 어떻게 해야 할까요? 다음과 같이 스낵바를 출력하는 확장 함수를 만들고, 기존의 `Context.toast()`를 `Context.snackbar()`로 한꺼번에 수정하면 됩니다.

```kotlin
fun Context.snackbar(
    message: String,
    duration: Int = Toast.LENTH_LONG
) {
    // ...
}
```

하지만 이런 해결 방법은 좋지 않습니다. 내부적으로만 사용하더라도 함수의 이름을 직접 바꾸는 것은 위험할 수 있습니다<small>(아이템 28: API 안정성을 확인하라)</small>. 다른 모듈이 이 함수에 의존하고 있다면, 다른 모듈에 큰 문제가 발생할 것입니다. 또한 함수의 이름은 한꺼번에 바꾸기 쉽지만, 파라미터는 한꺼번에 바꾸기가 쉽지 않으므로, 메시지의 지속시간을 나타내기 위한 `Toast.LENTH_LONG`이 계속 사용되고 있다는 문제도 있습니다. 스낵바를 출력하는 행위가 토스트의 필드에 영향을 받는 것은 좋지 않습니다. 다른 한편으로 스낵바의 enum으로 모든 것을 변경하는 것도 문제를 발생시킬 수 있습니다.

```kotlin
fun Context.snackbar(
    message: String,
    duration: Int = Snackbar.LENTH_LONG
) {
    // ...
}
```

메시지의 출력 방법이 바뀔 수 있다는 것을 알고 있다면, 이때부터 중요한 것은 메시지의 출력 방법이 아니라, 사용자에게 메시지를 출력하고 싶다는 의도 자체입니다. 따라서 메시지를 출력하는 더 추상적인 방법이 필요합니다. 토스트 출력을 토스트라는 개념과 무관한 showMessage라는 높은 레벨의 함수로 옮겨봅시다.

```kotlin
fun Context.showMessage(
    message: String,
    duration: MessageLength = MessageLength.LONG
) {
    val toastDuration = when (duration) {
        SHORT -> Length.LENGTH_SHORT
        LONG -> Length.LENGTH_LONG
    }
    Toast.makeText(this, message, toastDuration).show()
}

enum class MessageLength { SHORT, LONG }
```

가장 큰 변화는 이름입니다. 일부 개발자는 이름 변경은 그냥 레이블을 붙이는 방식의 변화이므로, 큰 차이가 없다고 생각하기도 합니다. 하지만 이러한 관점은 사실 컴파일러의 관점에서만 유효합니다. 사람의 관점에서는 이름이 바뀌면 큰 변화가 일어난 것입니다. 함수는 추상화를 표현하는 수단이며, 함수 시그니처는 이 함수가 어떤 추상화를 표현하고 있는지 알려 줍니다. 따라서 의미있는 이름은 굉장히 중요합니다.

함수는 매우 단순한 추상화지만, 제한이 많습니다. 예를 들어 함수는 상태를 유지하지 않습니다. 또한 함수 시그니처를 변경하면 프로그램 전체에 큰 영향을 줄 수 있습니다. 구현을 추상화할 수 있는 더 강력한 방법으로는 클래스가 있습니다.

### 클래스

그럼 이전의 메시지 출력을 클래스로 추상화해 봅시다.

```kotlin
class MessageDisplay(val context: Context) {

    fun show(
        message: String,
        duration: MessageLength = MessageLength.Long
    ) {
        val toastDuration = when (duration) {
            SHORT -> Length.LENGTH_SHORT
            LONG -> Length.LENGTH_LONG
        }
        Toast.makeText(this, message, toastDuration).show()
    }
}

enum class MessageLength { SHORT, LONG }

// 사용
val messageDisplay = MessageDisplay(context)
messageDisplay.show("Message")
```

클래스가 함수보다 더 강력한 이유는 상태를 가질 수 있으며, 많은 함수를 가질 수 있다는 점 때문입니다<small>(클래스 멤버 함수를 메서드라고 부릅니다)</small>. 현재 위의 코드에서 클래스의 상태인 `context`는 기본 생성자를 통해 주입됩니다. 의존성 주입 프레임워크를 사용하면, 클래스 생성을 위임할 수도 있습니다.

```kotlin
@Inject lateinit var messageDisplay: MessageDisplay
```

또한 mock 객체를 활용해서 해당 클래스에 의존하는 다른 클래스의 기능을 테스트할 수 있습니다.

```kotlin
val messageDisplay: MessageDisplay = mockk()
```

게다가 메시지를 출력하는 더 다양한 종류의 메서드를 만들 수도 있습니다.

```kotlin
messageDisplay.setChristmasMode(true)
```

이처럼 클래스는 훨씬 더 많은 자유를 보장해 줍니다. 하지만 여전히 한계가 있습니다. 예를 들어 클래스가 `final`이라면, 해당 클래스 타입 아래에 어떤 구현이 있는지 알 수 있습니다. `open` 클래스를 활용하면 조금은 더 자유를 얻을 수 있습니다. `open` 클래스는 서브클래스를 대신 제공할 수 있기 때문입니다. 더 많은 자유를 얻으려면, 더 추상적이게 만들면 됩니다. 바로 인터페이스 뒤에 클래스를 숨기는 방법입니다.

### 인터페이스

코틀린 표준 라이브러리를 읽어보면, 거의 모든 것이 인터페이스로 표현된다는 것을 확인할 수 있을 것입니다.

예를 들어,
- listOf 함수는 List를 리턴합니다. 여기서 List는 인터페이스입니다. listOf는 팩토리 메서드라고 할 수 있습니다<small>(아이템 33: 생성자 대신 팩토리 함수를 사용하라)</small>.
- 컬렉션 처리 함수는 Iterable 또는 Collection의 확장 함수로써, List, Map 등을 리턴합니다. 이것들은 모두 인터페이스입니다.
- 프로퍼티 위임은 ReadOnlyProperty 또는 ReadWriteProperty 뒤에 숨겨집니다. 이것들도 모두 인터페이스입니다. 실질적인 클래스는 일반적으로 private입니다. 함수 lazy는 Lazy 인터페이스를 리턴합니다.

라이브러리를 만드는 사람은 내부 클래스의 가시성을 제한하고, 인터페이스를 통해 이를 노출하는 코드를 많이 사용합니다. 이렇게 하면 사용자가 클래스를 직접 사용하지 못하므로, 라이브러리를 만드는 사람은 인터페이스만 유지한다면, 별도의 걱정 없이 자신이 우너하는 형태로 그 구현을 변경할 수 있습니다. 즉, 인터페이스 뒤에 객체를 숨김으로써 실질적인 구현을 추상화하고, 사용자가 추상화된 것에만 의존하게 만들 수 있는 것입니다. 즉, 결합<small>(coupling)</small>을 줄일 수 있는 것입니다.

코틀린이 클래스가 아니라 인터페이스를 리턴하는 데에는 이외에도 여러 이유가 있습니다.