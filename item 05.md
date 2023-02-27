# 아이템 5. 예외를 활용해 코드에 제한을 걸어라

확실하게 어떤 형태로 동작해야 하는 코드가 있다면, 예외를 활용해 제한을 걸어주는 것이 좋습니다. 코틀린에서는 코드의 동작에 제한을 걸 때 다음과 같은 방법을 사용할 수 있습니다.

- `require` 블록: 아규먼트를 제한
- `check` 블록: 상태와 관련된 동작을 제한
- `assert` 블록: 어떤 것이 true인지 확인<small>(테스트 모드에서만 작동)</small>
- return 또는 throw와 함께 활용하는 `Elvis 연산자`

다음은 이러한 메커니즘을 사용하는 간단한 예입니다.

#### `Stack<T>`의 일부

```kotlin
fun pop(num: Int = 1): List<T> {
    require(num <= size) {
        "Cannot remove more elements than current size"
    }
    check(isOpen) { "Cannot pop from closed stack" }
    val ret = collection.take(num)
    collection = collection.drop(num)
    assert(ret.size == num)
    return ret
}
```

이렇게 제한을 걸어 주면 다양한 장점이 발생합니다.

- 제한을 걸면 문서를 읽지 않은 개발자도 문제를 확인할 수 있습니다.
- 문제가 있을 경우에 함수가 예상하지 못한 동작을 하지 않고 예외를 throw 합니다. 예상하지 못한 동작을 하는 것은 예외를 throw하는 것보다 굉장히 위험하며, 상태를 관리하는 것이 굉장히 힘듭니다. 이러한 제한으로 인해서 문제를 놓치지 않을 수 있고, 코드가 더 안정적으로 작동하게 됩니다.
- 코드가 어느 정도 자체적으로 검사됩니다. 따라서 이와 관련된 단위 테스트를 줄일 수 있습니다.
- 스마트 캐스트 기능을 활용할 수 있게 되므로, 캐스트<small>(타입 변화)</small>를 적게 할 수 있습니다.

## 아규먼트

함수를 정의할 때 타입 시스템을 활용해서 아규먼트<small>(argument)</small>에 제한을 거는 코드를 많이 사용합니다. 몇 가지 예를 살펴봅시다.

- 숫자를 아규먼트로 받아서 팩토리얼을 계산한다면 숫자는 양의 정수여야 합니다.
- 좌표들을 아규먼트로 받아서 클러스터를 찾을 때는 비어 있지 않은 좌표 목록이 필요합니다.
- 사용자로부터 이메일 주소를 입력받을 때는 값이 입력되어 있는지, 그리고 이메일 형식이 올바른지 확인해야 합니다.

일반적으로 이러한 제한을 걸 때는 `require` 함수를 사용합니다. `require` 함수를는 제한을 확인하고, 제한을 만족하지 못할 경우 예외를 throw합니다.

```kotlin
fun factorial(n: Int): Long {
    require(n >= 0)
    return if (n <= 1) 1 else factorial(n - 1) * n
}

fun findClusters(points: List<Point>): List<Cluster> {
    require(points.isNotEmpty())
    //...
}

fun sendEmail(user: User, message: String) {
    requireNotNull(user.email)
    require(isValidEmail(user.email))
    //...
}
```

이와 같은 형태의 입력 유효성 검사 코드는 함수의 가장 앞부분에 배치되므로, 읽는 사람도 쉽게 확인할 수 있습니다<small>(물론 코드를 사용하는 모든 사람이 실제 코드 본문을 읽는 것은 아니므로, 문서에 관련된 내용을 반드시 명시해 두어야 합니다)</small>.

`require` 함수는 조건을 만족하지 못할 때 무조건적으로 IllegalArgument Exception을 발생시키므로 제한을 무시할 수 없습니다. 일반적으로 이러한 처리는 함수의 가장 앞부분에 하게 되므로, 코드를 읽을 때 쉽게 확인할 수 있습니다. 참고로 코드를 읽지 않는 사람이 있을 수도 있으므로 반드시 문서에 이러한 제한이 있다고 별도로 표시해 놓아야 합니다.

또한 다음과 같은 방법으로 람다를 활용해서 지연 메시지를 정의할 수도 있습니다.

```kotlin
fun factorial(n: Int): Long {
    require(n >=0) { "Cannot calculate factorial of $n " +
        "because it is smaller than 0" }
    return if (n <= 1) 1 else factorial(n - 1) * n
}
```

지금까지 살펴본 것처럼 `require` 함수는 아규먼트와 관련된 제한을 걸 때 사용할 수 있습니다.

이외에도 예외를 활용해 제한을 거는 대표적인 대상으로 상태가 있습니다.

## 상태

어떤 구체적인 조건을 만족할 때만 함수를 사용할 수 있게 해야 할 때가 있습니다. 예를 들어 다음과 같은 경우입니다.

- 어떤 객체가 미리 초기화되어 있어야만 처리를 하게 하고 싶은 함수
- 사용자가 로그인했을 때만 처리를 하게 하고 싶은 함수
- 객체를 사용할 수 있는 시점에 사용하고 싶은 함수

상태와 관련된 제한을 걸 때는 일반적으로 check 함수를 사용합니다.

```kotlin
fun speak(text: String) {
    check(isInitialized)
    //...
}

fun getUserInfo(): UserInfo {
    checkNotNull(token)
    //...
}

fun next(): T {
    check(isOpen)
    //...
}
```

`check` 함수는 `require`와 비슷하지만, 지정된 예측을 만족하지 못할 때, `IllegalStateException`을 throw합니다. 상태가 올바른지 확인할 때 사용합니다. 예외 메시지는 `require`와 마찬가지로 지연 메시지를 사용해서 변경할 수 있습니다. 함수 전체에 대한 어떤 예측이 있을 때는 일반적으로 `require` 블록 뒤에 배치합니다. `check`를 나중에 하는 것입니다.

이러한 확인은 사용자가 규약을 어기고, 사용하면 안 되는 곳에서 함수를 호출하고 있다고 의심될 때 합니다. 사용자가 코드를 제대로 사용할 거라고 믿고 있는 것보다는 항상 문제 상황을 예측하고, 문제 상황에 예외를 throw하는 것이 좋습니다. 이러한 확인은 사용자뿐만 아니라 이를 구현하는 사람에게도 좋습니다. 참고로 스스로 구현한 내용을 확인할 때는 일반적으로 `assert`라는 또다른 함수를 사용합니다.

## Assert 계열 함수 사용

함수가 올바르게 구현되었다면, 확실하게 참을 낼 수 있는 코드들이 있습니다. 예를 들어 어떤 함수가 10개의 요소를 리턴한다면, '함수가 10개의 요소를 리턴하는가?'라는 코드는 참을 낼 것입니다. 그런데 함수가 올바르게 구현되어 있지 않을 수도 있습니다. 처음부터 구현을 잘못했을 수도 있고, 해당 코드를 이후에 다른 누군가가 변경(또는 리팩토링)해서 제대로 작동하지 않게 된 것일 수도 있습니다. 이러한 구현 문제로 발생할 수 있는 추가적인 문제를 예방하려면, 단위 테스트를 사용하는 것이 좋습니다.

```kotlin
class StackTest {

    @Test
    fun 'Stack pops correct number of elements'() {
        val stack = Stack(20) { it }
        val ret = stack.pop(10)
        assertEquals(10, ret.size)
    }

    //...
}
```

단위 테스트는 구현의 정확성을 확인하는 가장 기본적인 방법입니다. 현재 코드에서 스택이 10개의 요소를 pop하면, 10개의 요소가 나온다는 보편적인 사실을 테스트하고 있습니다. 하지만 현재와 같이 한 경우에만 테스트해서 모든 상황에서 괜찮을지는 알 수 없습니다. 따라서 모든 pop 호출 위치에서 제대로 동작하는지 확인해도 좋을 것입니다. 다음 코드와 같이 pop 함수 내부에서 Assert 계열의 함수를 사용해 봅시다.

```kotlin
fun pop(num: Int = 1): List<T> {
    //...
    assert(ret.size == num)
    return ret
}
```

이러한 조건은 현재 코틀린/JVM에서만 활성화되며, `-ea JVM` 옵션을 활성화해야 확인할 수 있습니다. 이러한 코드도 코드가 예상대로 동작하는지 확인하므로 테스트라고 할 수 있습니다. 다만 프로덕션 환경에서는 오류가 발생하지 않습니다. 테스트를 할 때만 활성화되므로, 오류가 발생해도 사용자가 알아차릴 수는 없습니다. 만약 이 코드가 정말 심각한 오류고, 심각한 결과를 초래할 수 있는 경우에는 `check`를 사용하는 것이 좋습니다. 단위 테스트 대신 함수에서 `assert`를 사용하면, 다음과 같은 장점이 있습니다.

- Assert 계열의 함수는 코드를 자체 점검하며, 더 효율적으로 테스트할 수 있게 해 줍니다.
- 특정 상황이 아닌 모든 상황에 대한 테스트를 할 수 있습니다.
- 실행 시점에 정확하게 어떻게 되는지 확인할 수 있습니다.
- 실제 코드가 더 빠른 시점에 실패하게 만듭니다. 따라서 예상하지 못한 동작이 언제 어디서 실행되었는지 쉽게 찾을 수 있습니다.

참고로 이를 활용해도 여전히 단위 테스트는 따로 작성해야 합니다. 표준 애플리케이션 실행에서는 `assert`가 예외를 throw하지 않는다는 것도 기억하세요.

사실 이런 `assert`는 파이썬에서 굉장히 많이 사용되고, 자바에서는 딱히 사용되지 않습니다. 코틀린에서는 코드를 안정적으로 만들고 싶을 때 양념처럼 사용할 수 있다는 것을 기억하세요.

## nullability와 스마트 캐스팅

코틀린에서 `require`와 `check` 블록으로 어떤 조건을 확인해서 `true`가 나왔다면, 해당 조건은 이후로도 true일 거라고 가정합니다.

```kotlin
public inline fun require(value: Boolean): Unit {
    contract {
        returns() implies value
    }
    require(value) { "Failed requirement." }
}
```

따라서 이를 활용해서 타입 비교를 했다면, 스마트 캐스트가 작동합니다. 다음 예에서는 어떤 사람<small>(person)</small>의 복장<small>(person.outfit)</small>이 드레스<small>(Dress)</small>여야 코드가 정상적으로 진행됩니다. 따라서 만약 이러한 outfit 프로퍼티가 final이라면, outfit 프로퍼티가 Dress로 스마트 캐스트 됩니다.

```kotlin
fun changeDress(person: Person) {
    require(person.outfit is Dress)
    val dress: Dress = person.outfit
    //...
}
```

이러한 특징은 어떤 대상이 null인지 확인할 때 굉장히 유용합니다.

```kotlin
class Person(val email: String?)

fun sendEmail(person: Person, message: String) {
    require(person.email != null)
    val email: String = person.email
    //...
}
```

이러한 경우 `requireNotNull`, `checkNotNull`이라는 특수한 함수를 사용해도 괜찮습니다. 둘 다 스마트 캐스트를 지원하므로, 변수를 '언팩<small>(unpack)</small>'하는 용도로 활용할 수 있습니다.

```kotlin
class Person(val email: String?)
fun validateEmail(email: String) { /*..*/ }

fun sendEmail(person: Person, text: String) {
    val email = requireNotNull(person.email)
    validateEmail(email)
    //...
}

fun sendEmail(person: Person, text: String) {
    requireNotNull(person.email)
    validateEmail(person.email)
    //...
}
```

nullability를 목적으로, 오른쪽에 throw 또는 return을 두고 Elvis 연산자를 활용하는 경우가 많습니다. 이러한 코드는 굉장히 읽기 쉽고, 유연하게 사용할 수 잇습니다. 첫 번째로 오른쪽에 return을 넣으면, 오류를 발생시키지 않고 단순하게 함수를 중지할 수도 있습니다.

```kotlin
fun sendEmail(person: Person, text: String) {
    val email: String = Person.email ?: return
    //...
}
```

프로퍼티에 문제가 있어서 null일 때 여러 처리를 해야 할 때도, return/throw와 run 함수를 조합해서 활용하면 됩니다. 이는 함수가 중지된 이유를 로그에 출력해야 할 때 사용할 수 있습니ㅏㄷ.

```kotlin
fun sendEmail(person: Person, text: String) {
    val email: String = person.email ?: run {
        log("Email not sent, no email address")
        return
    }
    //...
}
```

이처럼 return과 throw를 활용한 Elvis 연산자는 nullable을 확인할 때 굉장히 많이 사용되는 관용적인 방법입니다. 따라서 적극적으로 활용하는 것이 좋습니다. 또한 이러한 코드는 함수의 앞부분에 넣어서 잘 보이게 만드는 것이 좋습니다.

## 정리

이번 절에서 활용한 내용을 기반으로, 다음과 같은 이득을 얻을 수 있습니다.

- 제한을 훨씬 더 쉽게 확인할 수 있다.
- 애플리케이션을 더 안정적으로 지킬 수 있다.
- 코드를 잘못 쓰는 상황을 막을 수 있다.
- 스마트 캐스팅을 활용할 수 있다.