# 아이템 3. 최대한 플랫폼 타입을 사용하지 말라

> <b>플랫폼 타입<small>(platform type)</small></b>이란, 다른 프로그래밍 언어에서 전달되어서 nullable인지 아닌지 알 수 없는 타입을 말합니다.

코틀린의 특징이나 장점에 대해 이야기를 할 때 <b>널 안정성<small>(null safety)</small></b>는 항상 주요하게 언급되어지는 기능 중 하나입니다. 자바에서 자주 경험했던 <b>널 포인터 예외<small>(Null-Pointer Exception, NPE)</small></b>는 코틀린의 null-safety 매커니즘으로 인해 거의 찾아보기 힘듭니다. 하지만 null-safety 메커니즘이 없는 자바, C 등의 프로그래밍 언어와 코틀린을 연결해서 사용할 때는 이러한 예외가 발생할 수 있습니다. 특히 회사의 기간이 오래된 프로젝트 코드에서 자바와 코틀린이 혼용되어 사용되는 경우를 쉽게 찾아볼 수 있습니다.

만약 자바에서 `String` 타입을 리턴하는 메서드가 있다면, 코틀린에서는 이를 사용하려면 어떻게 해야 할까요? `@Nullable` 어노테이션이 붙어 있다면, 이를 nullable로 추정하고, `String?`으로 변경하면 됩니다. `@NotNull` 어노테이션이 붙어 있다면, String으로 변경하면 됩니다.

그런데 만약 다음과 같이 어노테이션이 붙어 있지 않다면 어떻게 해야 할까요?

#### 어노테이션이 붙어 있지 않은 자바 코드

```java
public class JavaTest {
    public String giveName() {
        // ...
    }
}
```

자바에서는 모든 것이 nullable일 수 있으므로 최대한 안전하게 접근한다면, 이를 nullable로 가정하고 다루어야 합니다. 하지만 어떤 메서드는 null을 리턴하지 않을 것이 확실할 수 있기에 not-null 단정을 나타내는 `!!`를 마지막에 붙입니다.

nullable과 관련하여 자주 문제가 되는 부분은 바로 자바의 제네릭 타입입니다. 자바 API에서 `List<User>`를 리턴하고, 어노테이션이 따로 붙어 있지 않은 경우를 생각해 봅시다. 코틀린이 디폴트로 모든 타입을 nullable로 다룬다면, 우리는 이를 사용할 때 이러한 리스트 내부의 `User` 객체들이 널이 아니라는 것을 알아야 합니다. 따라서 리스트 자체만 널인지 확인해서는 안되고, 그 내부에 있는 것들도 널인지 확인해야 합니다.

```java
// 자바
public class UserRepo {

    public List<User> getUsers() {
        // ...
    }
}
```

```kotlin
// 코틀린
val users: List<User> = UserRepo().users!!.filterNotNull()
```

조금 더 나아가서, 만약 함수가 `List<List<User>>`를 리턴한다면 훨씬 복잡해질 것입니다.

```kotlin
val users: List<List<User>> = UserRepo().groupedUsers!!
        .map { it!!.filterNotNull() }
```

리스트는 적어도 `map`과 `filterNotNull` 등의 메서드를 제공하지만, 다른 제네릭 타입이라면 널을 확인하는 것 자체가 정말 복잡한 일이 됩니다. 그래서 코틀린은 자바 등의 다른 프로그래밍 언어에서 넘어온 타입들을 특수하게 다루며, 이러한 타입을 <b>플랫폼 타입<small>(platform type)</small></b>이라고 부릅니다.

플랫폼 타입은 `String!`처럼 타입 이름 뒤에 `!` 기호를 붙여서 표기합니다. 물론 이러한 노테이션이 직접적으로 코드에 나타나지는 않습니다. 대신 다음 코드와 같은 형태로 이를 선택적으로 사용합니다.

```java
// 자바
public class UserRepo {
    public User getUser() {
        // ...
    }
}
```

```kotlin
// 코틀린
val repo = UserRepo()
val user1 = repo.user           // user1의 타입은 User!
val user2: User = repo.user     // user2의 타입은 User
val user3: User? = repo.user    // user3의 타입은 User?
```

이러한 코드를 사용할 수 있으므로, 이전에 언급했던 문제가 사라집니다.

```kotlin
val users: List<User> = UserRepo().users
val users: List<List<User>> = UserRepo().groupedUsers
```

문제는 null이 아니라고 생각되는 것이 null일 가능성이 있으므로, 여전히 위험하다는 것입니다. 그래서 플랫폼 타입을 사용할 때는 항상 주의를 기울여야 합니다. 설계자가 명시적으로 어노테이션을 표시하거나, 주석으로 달아두지 않으면, 언제든지 동작이 변경될 가능성이 있습니다. 따라서 함수가 지금 당장 null을 리턴하지 않아도, 미래에는 변경될 수도 있다는 것을 염두해 둬야 합니다.

자바를 코틀린과 함께 사용할 때, 자바 코드를 직접 조작할 수 있다면, 가능한 `@Nullable`과 `@NotNull` 어노테이션을 붙여서 사용하기 바랍니다.

#### 자바 코드에 어노테이션 사용

```java
import org.jetbrains.annotations.NotNull;

public class UserRepo {
    public @NotNull User getUser() {
        // ...
    }
}
```

이는 코틀린 개발자를 지원하고 싶을 경우, 가장 중요한 단계 중 하나라고 할 수 있습니다. 안드로이드 개발에서 코틀린을 메인 언어로 변경했을 때, 가장 중요한 변경 사항으로 이러한 어노테이션들이 언급되고 있습니다. 이러한 어노테이션들이 붙어 있어서, 안드로이드 API가 코틀린과 친화적이라고 불리는 것입니다.

다만 플랫폼 타입은 안전하지 않으므로 최대한 빨리 제거하는 것이 좋습니다. 왜 플랫폼 타입이 안전하지 않은 지 다음 코드에서 `statedType`과 `platformType`의 동작을 살펴봅시다.

```java
// 자바
public class JavaClass {
    public String getValue() {
        return null;
    }
}
```

```kotlin
// 코틀린
fun statedType() {
    val value: String = JavaClass().value // NPE 발생
    // ...
    println(value.length)
}

fun platformType() {
    val value = JavaClass().value
    // ...
    println(value.length) // NPE 발생
}
```

이처럼 플랫폼 타입은 더 많은 위험 가능성을 갖고 있습니다. 추가적인 예로 인터페이스에서 다음과 같이 플랫폼 타입을 사용했다고 해 봅시다.

```kotlin
interface UserRepo {
    fun getUserName() = JavaClass().value
}
```

이러한 경우 메서드의 inferred 타입<small>(추론된 타입)</small>이 플랫폼 타입이고, 이는 누구나 nullable 여부를 지정할 수 있다는 것입니다. 예를 들어 어떤 사람이 이를 활용해서 nullable을 리턴하게 됐는데, 사용하는 사람이 nullable이 아닐 거라고 받아들였다면 문제가 됩니다.

```kotlin
class RepoImpl: UserRepo {
    override fun getUserName(): String? {
        return null
    }
}

fun main() {
    val repo: UserRepo = RepoImpl()
    val text: String = repo.getUserName() // 런타임 때 NPE
    print("User name length is ${text.length}")
}
```

이처럼 플랫폼 타입이 전파<small>(다른 곳에서 사용)</small>되는 일은 굉장히 위험합니다. 항상 위험을 내포하고 있으므로, 안전한 코드를 원한다면 이런 부분을 제거하는 것이 좋습니다. 참고로 IntelliJ에서는 다음과 같은 경고를 출력해줍니다.

<p align = 'center'>
<img width = '700' src = 'https://user-images.githubusercontent.com/113880311/218317759-ddebc907-7031-41d3-a762-ed42ac1f4de4.png'>
</p>


## 정리

다른 프로그래밍 언어에서 전달되어 nullable 여부를 알 수 없는 타입을 플랫폼 타입이라 부릅니다. 플랫폼 타입을 사용하는 코드는 해당 부분만 위험할 뿐만 아니라, 이를 활용하는 곳까지 영향을 줄 수 있는 위험한 코드입니다.

- 따라서 이런 코드를 사용하고 있다면 빨리 해당 코드를 제거하는 것이 좋습니다.
- 연결되어 있는 자바 생성자, 메서드, 필드에 nullable 여부를 지정하는 어노테이션을 활용하는 것도 좋습니다.
   - 이러한 정보는 코틀린 개발자뿐만 아니라 자바 개발자에게도 유용한 정보입니다.