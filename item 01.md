# 아이템 1. 가변성을 제한하라

코틀린은 모듈로 프로그램을 설계하고 모듈에는 다음과 같은 다양한 구성요소가 있습니다.
- 클래스
- 객체
- 함수
- 타입 별칭<small>(type alias)</small>
- 최상위 프로퍼티<small>(top-level property)</small>
- etc…

이러한 요소 중 일부는 <b>상태<small>(state)</small></b>를 가질 수 있습니다.
- 읽고 쓸 수 있는 프로퍼티인 `var`을 사용하거나 **mutable 객체**를 사용하면 상태를 가질 수 있습니다.

```kotlin
var a = 10
var list: MutableList<Int> = mutableListOf()
```

상태를 갖는 것은 시간의 변화에 따라서 변하는 요소를 표현할 수 있기에 유용하지만, 상태를 적절히 관리하는 것이 생각보다 꽤 어렵기 때문에 양날의 검이라 할 수 있습니다.

1. 프로그램을 이해하고 **디버그하기 힘들어집니다.**
   - 상태 변경이 많아지면 이를 추적하기 힘들어짐
   - 클래스를 이해하기도 어렵고, 이후에 코드 수정도 힘듦 
2. 가변성<small>(mutablility)</small>이 있으면 **코드의 실행을 추론하기 어려워집니다.**
   - 시점에 따라 값이 달라질 수 있어 코드의 실행을 예측하기가 어려움
3. **멀티스레드** 프로그램일 경우 **적절한 동기화가 필요**합니다.
   - 변경이 발생하는 모든 부분에서 충돌이 발생할 수 있음
4. **테스트하기 어렵습니다.**
   - 모든 상태를 테스트해야 하므로, 변경이 많아질수록 더 많은 조합을 테스트해야 함
5. **상태 변경이 일어날 때,** 이러한 변경을 **다른 부분에게 알려야**하는 경우가 있습니다.
   - 정렬된 리스트에 가변 요소를 추가하면 요소에 변경이 일어날 때마다 리스트 전체를 재정렬해야 함

## 코틀린에서 가변성 제한하기

코틀린은 가변성을 제한할 수 있도록 설계되어 있기에 불변<small>(immutable)</small> 객체를 만들거나, 프로퍼티를 변경할 수 없게 막는 것이 매우 쉽습니다.

이를 위해 많은 방법이 존재하지만 그 중에서 많이 사용되고 중요한 것들은 다음과 같습니다.

1. 읽기 전용 프로퍼티 `val`
2. 가변 컬렉션과 읽기 전용 컬렉션 구분하기
3. data class의 `copy`

### 1. 읽기 전용 프로퍼티 `val`

#### 케이스 1

코틀린은 `val`을 사용하여 읽기 전용 프로퍼티를 만들 수 있으며, 선언된 프로퍼티는 마치 값<small>(value)</small>처럼 동작합니다. **일반적인 방법으로는** 값이 변하지 않습니다.

```kotlin
val a = 10
a = 20 // 오류
```

#### 케이스 2

> 읽기 전용 ≠ 불변(immutable)

읽기 전용 프로퍼티는 **완전히 변경 불가능한 것이 아닙니다.** 읽기 전용 프로퍼티가 mutable 객체를 담고 있다면, 내부적으로 변할 수 있습니다.

```kotlin
val list = mutableListOf(1, 2, 3)
list.add(4)

print(list) // [1, 2, 3, 4]
```

위의 코드 스니펫에서 `list = mutableListOf(4, 5, 6)`과 같은 코드를 뒤에 추가하여 재할당하는 것이 불가능할 뿐입니다. **우리는 읽기 전용과 가변성을 구분해서 생각해야 합니다.**

- 프로퍼티를 읽을 수만 있다는 속성: 읽기 전용
- 값이 변할 수 없는 것: 가변성

#### 케이스 3

읽기 전용 프로퍼티는 다른 프로퍼티를 활용하는 사용자 정의 게터로도 정의할 수 있습니다. 이렇게 `var` 프로퍼티를 사용하는 `val` 프로퍼티는 `var` 프로퍼티가 변할 때 함께 변할 수 있습니다.

```kotlin
var name: String = "Christopher"
var surname: String = "Nolan"
val fullName
    get() = "$name $surname"

fun main() {
    println(fullName) // Christopher Nolan
    name = "Jonathan"
    println(fullName) // Jonathan Nolan
}
```

#### 케이스 4

값을 추출할 때마다 사용자 정의 게터가 호출되므로 아래와 같은 코드를 사용할 수 있습니다.

```kotlin
fun calculate(): Int {
    print("계산합니다...")
    return 42
}

val fizz = calculate() // 계산합니다...
val buzz
    get() = calculate()

fun main() {
    println(fizz) // 42
    println(fizz) // 42
    println(buzz) // 계산합니다... 42
    println(buzz) // 계산합니다... 42
}
```

```console
계산합니다...42
42
계산합니다...42
계산합니다...42
```

- 코틀린의 프로퍼티는 기본적으로 캡슐화되어 있고, 추가적으로 사용자 정의 접근자<small>(게터와 세터)</small>를 가질 수 있습니다.
  - 이러한 특성으로 코틀린은 API를 변경하거나 정의할 때 굉장이 유연합니다.
  - `아이템 16: 프로퍼티는 동작이 아니라 상태를 나타내야 한다`와 연관 내용

#### 케이스 5

- `var`은 게터와 세터를 모두 제공하지만, `val`은 변경이 불가능하므로 게터만 제공합니다.
  - 그래서 `val`을 `var`로 오버라이드할 수 있습니다.

```kotlin
interface Element {
    var active: Boolean
}

class ActualElement: Element {
    override var active: Boolean = false
}
```

> `var`보다 `val`을 일반적으로 더 많이 사용하는 이유: 읽기 전용 프로퍼티 `val`의 값은 변경될 수 있기는 하지만, **프로퍼티 레퍼런스 자체를 변경할 수는 없으므로 동기화 문제 등을 줄일 수 있기 때문**입니다.


#### 케이스 6

- `val`은 게터 또는 델리게이트<small>(delegate)</small>로 정의할 수 있습니다.
- 만약 변경할 필요가 완전히 없다면 *final 프로퍼티*를 사용하는 것이 좋습니다.
  - `val`은 정의 옆에 상태가 바로 적히므로 코드의 실행을 예측하는 것이 훨씬 간단합니다. 또한 스마트 캐스트 등의 추가적인 기능을 활용할 수 있습니다.

```kotlin
var name: String = "Christopher"
var surname: String = "Nolan"
val fullName: String?
    get() = name?.let { "$it $surname" }

val fullName2: String? = name?.let { "$it $surname" }

fun main() {
    if (fullName != null) {
        println(fullName.length) // 오류
    }

    if (fullName2 != null) {
        println(fullName2.length)
    }
}
```

- `fullName`은 게터로 정의했으므로 스마트 캐스트가 불가능합니다.
  - 게터를 활용하므로 값을 사용하는 시점의 `name`에 따라서 다른 결과가 나올 수 있기 때문.
- `fullName2`처럼 <i>지역 변수가 아닌 프로퍼티<small>(non-local property)</small></i>가 final이고, 사용자 정의 게터를 갖지 않을 경우 스마트 캐스트가 가능합니다.


### 2. 가변 컬렉션과 읽기 전용 컬렉션 구분하기

코틀린에는 읽고 쓸 수 있는 프로퍼티와 읽기 전용 프로퍼티로 구분되는데, 컬렉션도 마찬가지로 읽고 쓸 수 있는 컬렉션과 읽기 전용 컬렉션으로 구분됩니다.

<p align = 'center'>
<img width = '600' src = 'https://user-images.githubusercontent.com/113880311/215347451-1a74dcfc-2b78-4177-bb29-dd241ac6536c.png'>
</p>


### 3. data class의 `copy`

- mutable 객체 단점: 예측하기 어려우며 위험하다
- immutable 객체 단점: 변경할 수 없다

따라서 immutable 객체는 자신의 일부를 수정한 새로운 객체를 만들어 내는 메서드를 가져야 합니다.

```kotlin
data class User(
    val name: String,
    val surname: String
)

var user = User("Camille", "Claudel")
user = user.copy(surname = "Bidan")
print(user) // User(name=Camille, surname=Bidan)
```

코틀린에서는 이와 같은 형태로 immutable 특성을 가지는 데이터 모델 클래스를 만듭니다.
- 변경을 할 수 있다는 측면만 보면 mutable 객체가 더 좋아보이지만, 이렇게 데이터 모델 클래스를 만들어 immutable 객체로 만드는 것이 더 많은 장점을 가집니다.

## 정리

> 가변성을 제한한 immutable 객체를 사용하는 것이 좋은 이유

> 코틀린은 가변성을 제한하기 위해 다양한 도구들을 제공, 이를 활용해 가변 지점을 제한하며 코드를 작성하자

- `var`보다는 `val`을 사용
- mutable 프로퍼티보단 immutable 프로퍼티를 사용
- mutalbe 객체와 클래스보다는 immutable 객체와 클래스를 사용
- 변경이 필요한 대상을 만들어야 한다면, immutable 데이터 클래스로 만들고 `copy`를 활용
- 컬렉션에 상태를 저장해야 한다면, mutable 컬렉션보다는 읽기 전용 컬렉션을 사용
- 변이 지점을 적절하게 설계하고, 불필요한 변이 지점은 만들지 말자
- mutable 객체를 외부에 노출하지 말자

#### 예외

- 효율성 때문에 immutable 객체보다 mutable 객체를 사용하는 것이 좋을 때가 있습니다.
  - 이러한 최적화는 코드에서 성능이 중요한 부분에서만 사용하는 것이 좋습니다.
  - 관련 내용은 `3부: 효율성`에서.
- immutable 객체를 사용할 때는 언제나 멀티스레드 때에 더 많은 주의를 기울여야 합니다.
  - 그래도 일단 immutable 객체와 mutable 객체를 구분하는 기준은 가변성입니다.