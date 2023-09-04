# 아이템 32. 추상화 규약을 지켜라

규약은 개발자들의 단순한 합의입니다. 따라서 한쪽에서 규약을 위반할 수도 있습니다. 기술적으로 모든 부분에서 이런 규약 위반이 발생할 수 있습니다. 예를 들어 다음과 같이 리플렉션을 활용하면, 우리가 원하는 것을 열고 사용할 수 있습니다.

> 리플렉션을 활용해서 클래스 외부에서 private 함수를 호출하는 예

```kotlin
class Employee {
    private val id: Int = 2
    override fun toString() = "User(id=$id)"

    private fun privateFunction() {
        println("Private function called")
    }
}

fun callPrivateFunction(employee: Employee) {
    employee::class.declaredMemberFunctions
        .first { it.name = "privateFunction" }
        .apply { isAccessible = true }
        .call(employee)
}

fun changeEmployeeId(employee: Employee, newId: Int) {
    employee::class.java.getDeclaredField("id")
        .apply { isAccessible = true }
        .set(employeee, newId)
}

fun main() {
    val employee = Employee()
    callPrivateFunction(employee) // 출력: Private function called
    changeEmployeeId(employee, 1)
    print(employee) // 출력: User(id=1)
}
```

무언가를 할 수 있다는 것이 그것을 해도 괜찮다는 의미는 아닙니다. 현재 코드는 private 프로퍼티와 private 함수의 이름과 같은 세부적인 정보에 매우 크게 의존하고 있습니다. 이러한 이름은 규약이라고 할 수 없기 때문에, 언제든지 변경될 수 있습니다. 따라서 이런 코드를 프로젝트에서 사용한다면, 프로젝트 내부에 시한 폭탄을 설치한 것과 같습니다.

규약은 보증<small>(warranty)</small>과 같습니다. 스마트폰을 그냥 사용했다면 AS를 받을 수 있지만, 스마트폰을 뜯거나 해킹했다면 AS를 받을 수 없습니다. 코드도 마찬가지입니다. 규약을 위반하면, 코드가 작동을 멈췄을 때 문제가 됩니다.

### 상속된 규약

클래스를 상속하거나, 다른 라이브러리의 인터페이스를 구현할 때는 규약을 반드시 지켜야 합니다. 예를 들어 모든 클래스는 eqauls와 hashCode 메서드를 가진 Any 클래스를 상속받습니다. 이러한 메서드는 모두 우리가 반드시 존중하고 지켜야 하는 규약을 갖고 있습니다. 만약 규약을 지키지 않는다면, 객체가 제대로 동작하지 않을 수 있습니다. 예를 들어 hashCode가 제대로 구현되지 않으면, HashSet과 함께 사용할 때 제대로 동작하지 않습니다. 다음 코드를 살펴봅시다. 원래 세트는 중복을 허용하지 않는데, equals가 제대로 구현되지 않으므로 중복을 허용해 버립니다.

```kotlin
class Id(val id: Int) {
    override fun equals(other: Any?) =
        other is Id && other.id == id
}

val set = mutableSetOf(Id(1))
set.add(Id(1))
set.add(Id(1))
print(set.size) // 3
```

참고로 현재 hashCode와 equals 구현에 일관성이 없습니다. 이와 관련된 내용은 '6장 클래스 설계'에서 코틀린의 중요한 규약을 다룰 때 다시 설명됩니다.

### 정리

프로그램을 안정적으로 유지하고 싶다면, 규약을 지키세요. 규약을 깰 수 밖에 없다면, 이를 잘 문서화하세요. 이러한 정보는 코드를 유지하고 관리하는 사람에게 큰 도움이 됩니다. 그 사람이 몇 년 뒤의 당신이 될 수도 있다는 것을 기억하세요.