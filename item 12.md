# 아이템 12. 연산자 오버로드를 할 때는 의미에 맞게 사용하라

연산자 오버로딩은 굉장히 강력한 기능이지만, '큰 힘에는 큰 책임이 따른다'는 말처럼 위험할 수 있습니다.

#### 팩토리얼을 구하는 함수

```kotlin
fun Int.factorial(): Int = (1..this).product()

fun Iterable<Int>.product(): Int =
    fold(1) { acc, i -> acc * i }
```

이 함수는 Int 확장 함수로 정의되어 있으므로, 굉장히 편리하게 사용할 수 있습니다.

```kotlin
print(10 * 6.factorial()) // 7200
```

팩토리얼은 `6!`과 같이 `!` 기호를 사용해서 표기한다는 것을 수학 시간에 배웠을 것입니다. 코틀린은 이런 연산자를 지원하지 않지만, 다음과 같이 연산자 오버로딩을 활용하면, 만들어 낼 수 있습니다.

```kotlin
operator fun Int.not() = factorial()

print(10 * 6!) // 7200
```

이렇게 할 수 있지만, 이렇게 하면 될까요? 당연히 안됩니다. 이 함수의 이름이 *not*이라는 것에 주목해 주세요. 함수의 이름이 not이므로 논리 연산에 사용해야지, 팩토리얼 연산에 사용하면 안됩니다. 코드를 이렇게 작성하면 굉장히 혼란스럽고 오해의 소지가 있습니다. 코틀린의 모든 연산자는 다음 푝와 같은 구체적인 이름을 가진 함수에 대한 별칭일 뿐입니다. 모든 연산자는 연산자 대신 함수로도 호출할 수 있습니다. 다음과 같은 코드는 어떻게 보이나요?

```kotlin
print(10 * 6.not()) // 7200
```
#### 코틀린 연산자에 대응되는 함수 이름

|연산자|대응되는 함수|
|:--|:--|
|+a|a.unaryPlus()|
|-a|a.unaryMinus()|
|!a|a.not()|
|++a|a.inc()|
|--a|a.dec()|
|a+b|a.plus(b)|
|a-b|a.minus(b)|
|a*b|a.time(b)|
|a/b|a.div(b)|
|a..b|a.rangeTo(b)|
|a in b|b.contains(a)|
|a+=b|a.plusAssign(b)|
|a-=b|a.minusAssign(b)|
|a*=b|a.timesAssign(b)|
|a/=b|a.divAssign(b)|
|a==b|a.equals(b)|
|a>b|a.compareTo(b)>0|
|a<b|a.compareTo(b)<0|
|a>=b|a.compareTo(b)>=0|
|a<=b|a.compareTo(b)<=0|

코틀린에서 각 연산자의 의미는 항상 같게 유지됩니다. 이는 매우 중요한 설계 결정입니다. 스칼라와 같은 일부 프로그래밍 언어는 무제한 연산자 오버로딩<small>(unlimited operator overloading)</small>을 지원합니다. 하지만 이 정도의 자유는 많은 개발자가 해당 기능을 오용하게 만듭니다. 예를 들어 `+` 연산자가 일반적인 의미로 사용되지 않고 있다면, 연산자를 볼 때마다 연산자를 개별적으로 이해해야 하기 때문에 코드를 이해하기 어려울 것입니다. 코틀린에서는 각각의 연산자에 구체적인 의미가 있으므로 이러한 문제가 없습니다. 예를 들어 다음 코드를 봅시다.

```kotlin
x + y == z
```

이 코드는 언제나 다음과 같은 코드로 변환됩니다.

```kotlin
x.plus(y).equal(z)
```

참고로 만약 plus의 리턴 타입이 nullable이라면, 다음과 같이 반환됩니다.

```kotlin
(x.plus(y))?.equal(z) ?: (z == null)
```

이는 구체적인 이름을 가진 함수이며, 모든 연산자가 이러한 이름이 나태내는 역할을 할 것이라고 기대됩니다. 이처럼 이름만으로 연산자의 사용이 크게 제한됩니다. 따라서 팩토리얼을 계산하기 위해서 `!` 연산자를 사용하면 안 됩니다. 이는 관례에 어긋나기 때문입니다.

### 분명하지 않은 경우

하지만 관례를 충족하는지 아닌지 확실하지 않을 때가 문제입니다. 예를 들어 함수를 세 배 한다는 것(`*` 연산자)은 무슨 의미일까요? 어떤 사람은 다음과 같이 이 함수를 세 번 반복하는 새로운 함수를 만들어 낸다고 생각할 수 있습니다.

```kotlin
operator fun Int.times(operation: () -> Unit): () -> Unit =
    { repeat(this) { operation() } }

val tripledHello = 3 * { print("Hello") }

tripledHello() // 출력: HelloHelloHello
```

물론 어떤 사람은 다음과 같이 이러한 코드가 함수를 세 번 호출한다는 것을 쉽게 이해할 수 있을 것입니다.

```kotlin
operator fun Int.times(operation: () -> Unit) {
    repeat(this) { operation() }
}

3 * { print("Hello") } // 출력: HelloHelloHello
```

의미가 명확하지 않다면, infix를 활용한 확장 함수를 사용하는 것이 좋습니다. 일반적인 이항 연산자 형태처럼 사용할 수 있습니다.

```kotlin
infix fun Int.timesRepeated(operation: () -> Unit) = {
    repeat(this) { operation() }
}

val tripledHello = 3 timesRepeated { print("Hello") }
tripledHello() // 출력: HelloHelloHello
```

최상위 함수<small>(top-level function)</small>를 사용하는 것도 좋습니다. 사실 함수를 n번 호출하는 것은 다음과 같은 형태로 이미 stdlib에 구현되어 있습니다.

```kotlin
repeat(3) { print("Hello") } // 출력: HelloHelloHello 
```

### 규칙을 무시해도 되는 경우

지금까지 설명한 연산자 오버로딩 규칙을 무시해도 되는 중요한 경우가 있습니다. 바로 도메인 특화 언어<small>(Domain Specific Language, DSL)</small>를 설계할 때입니다. 고전적인 HTML DSL을 생각해 봅시다.

```html
body {
    div {
        +"Some text"
    }
}
```

문자열 앞에 String.unaryPlus가 사용된 것을 볼 수 있습니다. 이렇게 코드를 작성해도 되는 이유는 이 코드가 DSL 코드이기 때문입니다.

### 정리

연산자 오버로딩은 그 이름의 의미에 맞게 사용해 주세요. 연산자 의미가 명확하지 않다면, 연산자 오버로딩을 사용하지 않는 것이 좋습니다. 대신 이름이 있는 일반 함수를 사용하기 바랍니다. 꼭 연산자 같은 형태로 사용하고 싶다면, infix 확장 함수 또는 최상위 함수를 활용하세요.