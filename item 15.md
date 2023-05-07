# 아이템 15. 리시버를 명시적으로 참조하라

무언가를 더 자세하게 설명하기 위해서, 명시적으로 긴 코드를 사용할 때가 있습니다. 대표적으로 함수와 프로퍼티를 지역 또는 톱레벨 변수가 아닌 다른 리시버로부터 가져온다는 것을 나타낼 때가 있습니다. 예로 클래스의 메서드라는 것을 나타내기 위한 `this`가 있습니다.

```kotlin
class User: Person() {
    private var beersDrunk: Int = 0

    fun drinkBeers(num: Int) {
        // ...
        this.beersDrunk += num
        // ...
    }
}
```

비슷하게 확장 리시버<small>(확장 메서드에서의 `this`)</small>를 명시적으로 참조하게 할 수도 있습니다. 비교를 위해서 일단 리시버를 명시적으로 표시하지 않은 퀵소트 구현을 살펴봅시다.

#### 리시버를 명시적으로 표시하지 않은 퀵소트 구현

```kotlin
fun <T: Comparable<T>> List<T>.quickSort(): List<T> {
    if (size < 2) return this
    val pivot = first()
    val (smaller, bigger) = drop(1)
        .partition { it < pivot }
    return smaller.quickSort() + pivot + bigger.quickSort()
}
```

#### 리시버를 명시적으로 표시한 퀵소트 구현

```kotlin
fun <T: Comparable<T>> List<T>.quickSort(): List<T> {
    if (this.size < 2) return this
    val pivot = this.first()
    val (smaller, bigger) = this.drop(1)
        .partition { it < pivot }
    return smaller.quickSort() + pivot + bigger.quickSort()
}
```

두 함수의 사용에 차이는 없습니다.

## 여러 개의 리시버

스코프 내부에 둘 이상의 리시버가 있는 경우, 리시버를 명시적으로 나타내면 좋습니다. `apply`, `with`, `run` 함수를 사용할 때가 대표적인 예입니다. 상황을 이해할 수 있게 다음 코드를 살펴봅시다.

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply {
                print("Created $name")
            }
    
    fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("Parent")
    node.makeChild("child")
}
```


일반적으로 위 코드의 결과가 'Created parent.child'가 출력된다고 예상하지만 실제로는 'Created Parent'가 출력됩니다. 왜 이런 결과가 나오는지 조금 더 이해할 수 있도록 앞에 명시적으로 리시버를 붙여봅시다.

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply { print("Created ${this.name}") }
            // 컴파일 오류

    fun create(name: String): Node? = Node(name)
}
```

문제는 `apply` 함수 내부에서 `this`의 타입이 `Node?`라서, 이를 직접 사용할 수 없다는 것입니다. 이를 사용하려면 언팩<small>(unpack)</small>하고 호출해야 합니다. 이렇게 하면 일반적으로 생각하는 답이 나옵니다.

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply {
                print("Created ${this?.name}")
            }

    fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("Parent")
    node.makeChild("child")
    // Created parent.child
}
```

사실 이것은 `apply`의 잘못된 사용 예입니다. 만약 `also` 함수와 파라미터 `name`을 사용했다면, 이런 문제 자체가 일어나지 않습니다. `also`를 사용하면, 이전과 마찬가지로 명시적으로 리시버를 지정하게 됩니다. 일반적으로 `also` 또는 `let`을 사용하는 것이 nullable 값을 처리할 때 훨씬 좋은 선택지입니다.

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName")
            .also { print("Created ${this?.name}") }

    fun create(name: String): Node? = Node(name)
}
```

리시버가 명확하지 않다면, 명시적으로 리시버를 적어서 이를 명확하게 해 주세요. 레이블 없이 리시버를 사용하면, 가장 가까운 리시버를 의미합니다. 외부에 있는 리시버를 사용하려면, 레이블을 사용해야 합니다. 둘 모두를 사용하는 예를 살펴봅시다.

```kotlin
class Node(val name: String) {
    fun makeChild(childName: String) =
        create("$name.$childName")
            .apply {
                print("Created ${this?.name} in ${this@Node.name}")
            }

    fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("Parent")
    node.makeChild("child")
    // Created parent.child in parent
}
```

어떤 리시버를 활용하는지 의미가 훨씬 명확해졌습니다. 이렇게 명확하게 작성하면 코드를 안전하게 사용할 수 있을 뿐만 아니라 가독성도 향상됩니다.

## 정리

짧게 적을 수 있다는 이유만으로 리시버를 제거하지 말기 바랍니다. 여러 개의 리시버가 있는 상홍 등에는 리시버를 명시적으로 적어 주는 것이 좋습니다. 리시버를 명시적으로 지정하면, 어떤 리시버의 함수인지를 명확하게 알 수 있으므로, 가독성이 향상됩니다. DSL에서 외부 스코프에 있는 리시버를 명시적으로 적게 강제하고 싶다면, `DslMarker` 메타 어노테이션을 사용합니다.