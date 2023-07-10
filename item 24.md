# 아이템 24. 제네릭 타입과 variance 한정자를 활용하라

#### invariant<small>(불공변성)</small>

```kotlin
class Cup<T>
```

위의 코드에서 타입 파라미터 `T`는 variance 한정자인 `out` 또는 `in`이 없으므로 기본적으로 invariant<small>(불공변성)</small>입니다.

- invariant라는 것은 제네릭으로 만들어지는 타입들이 서로 관련성이 없다는 의미
- 예로 `Cup<Int>`와 `Cup<Number>, Cup<Any>와 Cup<Nothing>`은 어떠한 관련성도 갖지 않음


```kotlin
fun main() {
    val anys: Cup<Any> = Cup<Int>() // 오류 : Type mismatch
    val nothing: Cup<Nothing> = Cup<Int>() // 오류
}
```

> 만약 어떤 관련성을 원한다면, `out` 또는 `in`과 같은 variance 한정자를 붙입니다.

#### covarinat<small>(공변성)</small>

`out`은 타입 파라미터를 covarinat<small>(공변성)</small>로 만듭니다.

- 이는 A가 B의 서브타입일 때, `Cup<A>`가 `Cup<B>`의 서브타입이라는 의미

```kotlin
class Cup<out T>
open class Dog
class Puppy: Dog()

fun main(args: Array<String>) {
    val b: Cup<Dog> = Cup<Puppy>() // OK
    val b: Cup<Puppy> = Cup<Dog>() // 오류

    val anys: Cup<Any> = Cup<Int>() // OK
    val nothings: Cup<Noting> = Cup<Int>() // 오류
}
```

#### contravariant<small>(반공변성)</small>

`in` 한정자는 반대 의미로, 타입 파라미터를 contravariant<small>(반공변성)</small>으로 만듭니다.

- 이는 A가 B의 서브타입일 때, `Cup<A>`가 `Cup<B>`의 슈퍼타입이라는 것을 의미

```kotlin
class Cup<in T>
open class Dog
class Puppy: Dog()

fun main(args: Array<String>) {
    val b: Cup<Dog> = Cup<Puppy>() // 오류
    val b: Cup<Puppy> = Cup<Dog>() // OK

    val anys: Cup<Any> = Cup<Int>() // 오류
    val nothings: Cup<Noting> = Cup<Int>() // 오류OK
}
```

variance 한정자를 그림으로 나타내면, 다음과 같습니다.

<p align = 'center'>
<img width = '800' src = 'https://github.com/june0122/algorithm_study/assets/39554623/8c846094-04e6-4421-b7c7-755ee93c7d69'>
</p>