# 아이템 45. 불필요한 객체 생성을 피해라

### 객체 생성 비용은 항상 클까?

어떤 객체를 랩<small>(wrap)</small>하면, 크게 세 가지 비용이 발생합니다.

- 객체는 더 많은 용량을 차지합니다.
- 요소가 캡슐화되어 있다면, 접근에 추가적인 함수 호출이 필요합니다. 함수를 사용하는 처리는 굉장히 빠르므로 마찬가지로 큰 비용이 발생하지는 않지만, 티끌 모아 태산이 되는 것처럼 수많은 객체를 처리한다면, 이 비용도 굉장히 커집니다.
- 객체는 생성되어야 합니다. 객체는 생성되고, 메모리 영역에 할당되고, 이에 대한 레퍼런스를 만드는 등의 작업이 필요합니다. 마찬가지로 적은 비용이지만, 모이면 굉장히 큰 비용이 됩니다.

### 싱글톤(객체 선언 / 재사용)

매 순간 객체를 생성하지 않고, 객체를 재사용하는 간단한 방법은 싱글톤을 사용하는 것입니다.

#### LinkedList 구현 예제

```kotlin
sealed class  LinkedList<T>

class Node1<T>(
    val head: T,
    val tail: LinkedList<T>
): LinkedList<T>()

class Empty<T>: LinkedList<T>()

// 사용 예시
val list = Node1(1, Node1(2, Node1(3, Empty())))
val list2 = Node1(1, Node1(2, Node1(3, Empty())))
```

위 구현은 리스트를 만들 때마다 Empty 인스턴스를 만들어야 한다는 문제점이 있습니다. Empty 인스턴스를 하나만 만들고, 다른 모든 리스트에서 활용할 수 있게 구현하면 해결이 되겠지만 어떤 제네릭 타입을 지정해야 할까요? 빈 리스트는 다른 모든 리스트의 서브타입이 되어야 하므로, 이를 해결하려면 Nothing 리스트를 만들어서 사용하면 됩니다.

Nothing은 모든 타입의 서브타입입니다. 따라서 `LinkedList<Nothing>`은 리스트가 covariant이라면<small>(out 한정자)</small>, 모든 LinkedList의 서브타입이 됩니다. 리스트는 immutable이고, 이 타입은 out 위치에서만 사용되므로, 현재 상황에서는 타입 아규먼트를 covariant로 만드는 것은 의미 있는 일입니다<small>(아이템 24: 제네릭 타입과 variance 한정자를 참고하라)</small>.

개선된 코드는 다음과 같습니다.

```kotlin
sealed class  LinkedList<out T>

class Node1<T>(
    val head: T,
    val tail: LinkedList<T>
): LinkedList<T>()

object Empty : LinkedList<Nothing>()

val list = Node1(1, Node1(2, Node1(3, Empty)))
val list2 = Node1(1, Node1(2, Node1(3, Empty)))
```

이러한 트릭은 immutable sealed 클래스를 정의할 때 자주 사용됩니다. 만약 mutable 객체에 사용하면 공유 상태 관리와 관련된 버그를 검출하기 어려울 수 있으므로 좋지 않습니다. mutable 객체는 캐시하지 않는다는 규칙을 지키는 것이 좋습니다<small>(아이템 1: 가변성을 제한하라)</small>. 객체 선언 이외에도 객체를 재사용하는 다양한 방법이 있습니다. 바로 캐시를 활용하는 팩토리 함수입니다.

### 캐시를 사용하는 팩토리 함수

일반적으로 객체는 생성자를 사용해서 만듭니다. 하지만 팩토리 메서드를 사용해서 만드는 경우도 있습니다. 팩토리 함수는 캐시를 가질 수 있습니다. 그래서 팩토리 함수는 항상 같은 객체를 리턴하게 만들 수도 있습니다. 실제로 stdlib의 emptyList는 이를 활용해서 구현되어 있습니다.

```kotlin
fun <T> List<T> emptyList() {
    return EMPTY_LIST
}
```

객체 세트가 있고, 그중에서 하나를 리턴하는 경우를 생각해 봅시다.

- 코틀린 코루틴 라이브러리에 있는 default dispatcher인 `Dispatchers.Default`는 쓰레드 풀을 갖고 있으며, 어떤 처리를 시작하라고 명령하면, 사용하고 있지 않은 쓰레드 하나를 사용해 명령을 수행합니다.
- 데이터베이스도 비슷한 형태로 커넥션 풀을 사용합니다.

객체 생성이 무겁거나, 동시에 여러 mutable 객체를 사용해야 하는 경우에는 이처럼 객체 풀을 사용하는 것이 좋습니다.

parameterized 팩토리 메서드도 캐싱을 활용할 수 있습니다. 예를 들어 객체를 다음과 같이 map에 저장해 둘 수 있을 것입니다.

```kotlin
private val connections = mutableMapOf<String, Connection>()

fun getConnection(host: String) = connections.getOrPut(host) { createConnection(host) }
```

모든 순수 함수는 캐싱을 활용할 수 있습니다. 이를 메모이제이션<small>(memoization)</small>이라고 부릅니다.