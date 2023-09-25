# 아이템 35. 복잡한 객체를 생성하기 위한 DSL을 정의하라

코틀린을 활용하면 DSL<small>(Domain Specific Language)</small>을 직접 만들 수 있습니다. DSL은 복잡한 객체, 계층 구조를 갖고 있는 객체들을 정의할 때 굉장히 유용합니다. DSL을 만드는 것은 약간 힘든 일이지만, 한 번 만들고 나면 보일러 플레이트와 복잡성을 숨기면서 개발자의 의도를 명확하게 표현할 수 있습니다.

예를 들어 코틀린 DSL은 다음과 같은 형태로 HTML을 표현할 수 있습니다. 이는 고전적인 HTML과 리액트 HTML 모두에서 활용할 수 있습니다.

```HTML
body {
    div {
        a("https://kotlinlang.org") {
            target = ATarget.blank
            + "Main site"
        }
    }
    +"Some content"
}
```

다른 플랫폼의 뷰도 이와 같은 형태로 DSL을 사용해 만들 수 있습니다.
- Anko 라이브러리를 활용한 안드로이드의 뷰
- JavaFX를 기반으로 만들어진 TornadoFX를 사용해 만든 뷰

DSL은 자료 또는 설정을 표현할 때도 활용될 수 있습니다. 다음 코드는 Ktor를 활용해서 만든 API 정의 예입니다.

```kotlin
fun Routing.api() {
    route("news") {
        get {
            val newsData = NewsUseCase.getAcceptedNews()
            call.respond(newsData)
        }
        get("propositions") {
            requireSecret()
            val newsData = NewsUseCase.getPropositions()
            call.respond(newsData)
        }
    }
    // ...
}
```

- 코틀린 테스트를 활용해서 테스트 케이스를 정의
- Gradle 설정을 정의할 때도 Gradle DSL이 사용됨

DSL을 활용하면 복잡하고 계층적인 자료 구조를 쉽게 만들 수 있습니다. 참고로 DSL 내부에서도 코틀린이 제공하는 모든 것을 활용할 수 있습니다. 코틀린의 DSL은 type-safe이므로, (그루비 등과 다르게) 여러 가지 유용한 힌트를 활용할 수 있습니다. 이미 존재하는 코틀린 DSL을 활용하는 것도 좋지만, 사용자 정의 DSL을 만드는 방법도 알아 두면 좋습니다.

### 사용자 정의 DSL 만들기

사용자 정의 DSL을 만드는 방법을 이해하려면, 리시버를 사용하는 함수 타입에 대한 개념을 이해해야 합니다. 이와 관련된 내용을 알아보기 전에 일단 함수 자료형 자체에 대한 개념을 간단하게 알아봅시다. 함수 타입은 함수로 사용할 수 있는 객체를 나타내는 타입입니다. 예를 들어 filter 함수를 살펴봅시다. predicate에 함수 타입이 활용되고 있습니다.

```kotlin
incline fun <T> Iterable<T>.filter(
    predicate: (T) -> Boolean
): List<T> {
    val list = arrayListOf<T>()
    for (elem in this) {
        if (predicate(elem)) {
            list.add(elem)
        }
    }
    return list
}
```

함수 타입의 몇 가지 예를 살펴봅시다.

- ()->Unit : 아규먼트를 갖지 않고, Unit을 리턴하는 함수
- (Int)->Unit : Int를 아규먼트로 받고, Unit을 리턴하는 함수
- (Int)->Int : Int를 아규먼트로 받고, Int를 리턴하는 함수
- (Int, Int)->Int : Int 2개를 아규먼트로 받고, Int를 리턴하는 함수
- (Int)->()->Unit : Int를 아규먼트로 받고, 다른 함수를 리턴하는 함수입니다. 이때 다른 함수는 아규먼트를 아무것도 받지 않고, Unit를 리턴하는 함수
- (()->Unit)->Unit : 다른 함수를 아규먼트로 받고, Unit을 리턴하는 함수입니다. 이때 다른 함수는 아규먼트를 아무것도 받지 않고, Unit을 리턴합니다.

함수 타입을 만드는 기본적인 방법은 다음과 같습니다.

- 람다 표현식
- 익명 함수
- 함수 레퍼런스

예를 들어 다음과 같은 함수가 있다고 해 봅시다.

```kotlin
fun plus(a: Int, b: Int) = a + b
```

유사 함수(analogical function)는 다음과 같은 방법으로 만듭니다.

```kotlin
val plus1: (Int, Int)->Int = { a, b -> a + b }
val plus2: (Int, Int)->Int = fun(a, b) = a + b
val plus3: (Int, Int)->Int = ::plus
```

위의 예에서는 프로퍼티 타입이 지정되어 있으므로, 람다 표현식과 익명 함수의 아규먼트 타입을 추론할 수 있습니다. 반대로 다음과 같이 아규먼트 타입을 지정해서 함수의 형태를 추론하게 할 수도 있습니다.

```kotlin
val plus4 = { a: Int, b: Int -> a + b }
val plus5 = fun(a: Int, b: Int) = a + b
```

함수 타입은 '함수를 나타내는 객체'를 표현하는 타입입니다. 익명 함수는 일반적인 함수처럼 보이지만, 이름을 갖고 있지 않습니다. 람다 표현식은 익명 함수를 짧게 작성할 수 있는 표기 방법입니다.

함수를 나타내는 타입이 있다면, 확장 함수의 경우는 어떨까요? 확장 함수는 어떻게 표현할 수 있을까요?

```kotlin
fun Int.myPlus(other: Int) = this + other
```

익명 함수를 만들 떄는 일반 함수처럼 만들고, 이름만 빼면 된다고 했습니다. 익명 확장 함수도 이와 같은 방법으로 만들 수 있습니다.

```kotlin
val myPlus = fun Int.(other: Int) = this + other
```

이 함수의 타입은 어떻게 될까요? 확장 함수를 나타내는 특별한 타입이 됩니다. 이를 **리시버를 가진 함수 타입**이라고 부릅니다. 일반적인 함수 타입과 비슷하지만, 파라미터 앞에 리시버 타입이 추가되어 있으며, 점(`.`) 기호로 구분되어 있습니다.

### 정리

DSL은 언어 내부에서 사용할 수 있는 특별한 언어입니다. 복잡한 객체는 물론이고 HTML 코드, 복잡한 설정 등의 계층 구조를 갖는 객체를 간단하게 표현할 수 있게 해 줍니다. 하지만 DSL 구현은 해당 DSL이 익숙하지 않은 개발자에게 혼란과 어려움을 줄 수 있습니다. 따라서 DSL은 복잡한 객체를 만들거나, 복잡한 계층 구조를 갖는 객체를 만들 때만 활용하는 것이 좋습니다. 좋은 DSL을 만드는 작업은 굉장히 힘듭니다. 하지만 잘 정의된 DSL은 프로젝트에 굉장히 큰 도움을 줍니다.