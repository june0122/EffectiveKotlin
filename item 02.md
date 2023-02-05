# 아이템 2. 변수의 스코프를 최소화하라

**상태**를 정의할 때는 변수와 프로퍼티의 스코프를 최소화하는 것이 좋습니다.
- 프로퍼티보다는 지역 변수 사용
- 최대한 좁은 스코프를 갖도록 변수 사용

> <i>`상태`는 특정 시점에 객체가 가지고 있는 정보의 집합으로 객체의 구조적 특징을 표현한다. - 객체 지향의 사실과 오해 / 2장</i>

코틀린의 스코프는 기본적으로 중괄호로 만들어지며, 내부 스코프에서 외부 스코프에 있는 요소에만 접근할 수 있습니다.

```kotlin
val a = 1

fun fizz() {
    val b = 2
    print(a + b)
}

val buzz = {
    val c = 3
    print(a + c)
}

// 이 위치에서 a는 사용 가능, b와 c는 사용 불가능
```

### 변수의 스코프를 제한하자

#### 케이스 1: 나쁜 예

```kotlin
var user: User

for (i in users.indices) {
    user = users[i]
    print("User at $i is $user")
}
```

#### 케이스 2: 조금 더 좋은 예

```kotlin
for (i in users.indices) {
    val user = users[i]
    print("User at $i is $user")
}
```

#### 케이스 3: 제일 좋은 예

```kotlin
for ((i, user) in users.withIndex()) {
    print("User at $i is $user")
}
```

*케이스 1*에서 변수 `user`는 for 반복문 스코프 내부뿐만 아니라 와부에서도 사용할 수 있습니다. 반면 *케이스 2*와 *케이스 3*에서는 `user`의 스코프를 for 반복문 내부로 제한합니다.

참고로 람다 표현식 내부의 람다 표현식이 중첩되는 상황과 같이, 스코프 내부에 스코프가 있을 수도 있습니다.

> 스코프를 좁게 만드는 것이 좋은 이유: 프로그램을 추적하고 관리하기가 쉬워짐

- 코드를 분석할 때는 어떤 시점에 어떤 요소가 있는지를 알아야 하는데, 스코프가 넓어져 프로그램에서 파악해야하는 변경될 수 있는 부분이 많아지면 프로그램을 이해하기가 어려워집니다.
- **애플리케이션이 간단할수록 읽기도 쉽고 안전합니다.** 이는 **우리가 mutable 프로퍼티보다 immutable 프로퍼티를 선호하는 이유와 비슷**합니다.
- mutable 프로퍼티는 좁은 스코프에 걸쳐 있을수록, 그 변경을 추적하는 것이 쉽습니다. 이렇게 추적이 되어야 코드를 이해하고 변경하는 것이 쉽습니다.
  - 변수의 스코프 범위가 너무 넓으면, 다른 개발자에 의해서 변수가 잘못 사용될 수 있습니다.

### 변수를 정의할 때 초기화하자

변수는 읽기 전용, 읽고 쓰기 전용 여부와 상관 없이, **변수를 정의할 때 초기화되는 것이 좋습니다.** if, when, try-catch, Elvis 표현식 등을 활용하면 최대한 변수를 정의할 때 초기화 할 수 있습니다.

#### 케이스 1: 나쁜 예

```kotlin
val user: User

if (hasValue) {
    user = getValue()
} else {
    user = User()
}
```

#### 케이스 2: 조금 더 좋은 예

```kotlin
val user: User = if (hasValue) {
    getValue()
} else {
    User()
}
```

### 여러 프로퍼티를 한꺼번에 설정할 경우엔 구조분해 선언<small>(destructuring declaration)</small>을 활용하자

#### 케이스 1: 나쁜 예

```kotlin
fun updateWeather(degree: Int) {
    val description: String
    val color: Int
    if (degrees < 5) {
        decription = "cold"
        color = Color.BLUE
    } else if (degress < 23) {
        decription = "mild"
        color = Color.YELLOW
    } else {
        decription = "hot"
        color = Color.RED
    }
    // ...
}
```

#### 케이스 2: 조금 더 좋은 예

```kotlin
fun updateWeather(degree: Int) {
    val (description, color) = when {
        degrees < 5 -> "cold" to Color.BLUE
        degrees < 23 -> "cold" to Color.YELLOW
        else -> "hot" to Color.RED
    }
    // ...
}
```

## 캡쳐링

> `람다 캡처링(capturing lambda)`이란, 파라미터로 넘겨받은 데이터가 아닌 <b><i>람다식 외부에서 정의된 변수</i>를 참조하는 변수를 람다식 내부에 저장하고 사용하는 동작</b>을 의미한다. 

#### 람다 캡쳐링 예시 (Java)

```java
void lambdaCapturing() {
   int localVariable = 1000;

   Runnable r = () -> System.out.println(localVariable);
}
```

#### 에라토스테네스의 체 구현: 간단한 구현

```kotlin
var numbers = (2..100).toList()
var primes = mutableListOf<Int>()

while (numbers.isNotEmpty()) {
    val prime = numbers.first()
    primes.add(prime)
    numbers = numbers.filter { it % prime != 0 }
}

print(primes) // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43,
// 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]
```

#### 에라토스테네스의 체 구현: 시퀀스 이용

```kotlin
val primes: Sequence<Int> = sequence {
	var numbers = generateSequence(2) { it + 1 }

	while (true) {
        val prime = numbers.first()
  	    yield(prime)
  	    numbers = numbers.drop(1)
            .filter { it % prime != 0 }
    }
}

print(primes.take(10).toList()) // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

#### 에라토스테네스의 채 구현: 캡쳐 문제로 인한 잘못된 코드

```kotlin
val primes: Sequence<Int> = sequence {
	var numbers = generateSequence(2) { it + 1 }

    var prime: Int
	while (true) {
        prime = numbers.first()
  	    yield(prime)
  	    numbers = numbers.drop(1)
            .filter { it % prime != 0 }
    }
}

print(primes.take(10).toList()) // [2, 3, 5, 6, 7, 8, 9, 10, 11, 12]
```

`prime` 변수를 캡쳐했기 때문에 잘못된 실행 결과가 나옵니다.

반복문 내부에서 `filter`를 활용해서 `prime`으로 나눌 수 있는 숫자를 필터링합니다. 그런데 시퀀스를 활용하므로 필터링이 지연됩니다. 따라서 최종적인 `prime` 값으로만 필터링되어 위와 같은 결과가 나옵니다.

`prime`이 2로 설정되어 있을 때 필터링된 4를 제외하면, `drop`만 동작하므로 그냥 연속된 숫자가 나와 버립니다.

이러한 문제가 발생할 수 있으므로, 항상 잠재적인 캡처 문제를 주의해야 합니다. 가변성을 피하고 스코프 범위를 좁게 만들면, 이런 문제를 간단하게 피할 수 있습니다.

## 정리

- 변수의 스코프는 좁게 만들어서 활용하는 것이 좋습니다.
- `var` 보다는 `val`을 사용하는 것이 좋습니다.
- 람다에서 변수를 캡쳐한다는 것을 꼭 기억합니다.










