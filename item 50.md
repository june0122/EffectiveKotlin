# 아이템 50. 컬렉션 처리 단계 수를 제한하라

모든 컬렉션 처리 메서드는 비용이 많이 듭니다.

표준 컬렉션 처리는 내부적으로 요소들을 활용해 반복을 돌며, 내부적으로 계산을 위해 추가적인 컬렉션을 만들어 사용합니다.

시퀀스 처리도 시퀀스 전체를 랩하는 객체가 만들어지며, 조작을 위해서 또 다른 추가적인 객체를 만들어 냅니다.
- 연산 내용이 시퀀스 객체로 전달되므로, 인라인으로 사용할 수 없습니다. 따라서 람다 표현식을 객체로 만들어 사용해야 합니다.

따라서 적절한 메서드를 활용해서 컬렉션 처리 단계 수를 적절하게 제한하는 것이 좋습니다.

```kotlin
class Student(val name: String?)

// Works
fun List<Student>.getNames(): List<String> = this
   .map { it.name }
   .filter { it != null }
   .map { it!! }
 
// Better
fun List<Student>.getNames(): List<String> = this
   .map { it.name }
   .filterNotNull()

// Best
fun List<Student>.getNames(): List<String> = this
   .mapNotNull { it.name }
```

컬렉션 처리와 관련해서 비효율적인 코드를 작성하더라도 IDE의 Lint를 통해서 어느 정도 방지할 수 있지만, 컬렉션 처리를 어떤 형태로 줄일 수 있는지 알아두면 좋습니다.

|이 코드보다는|이 코드가 좋습니다.|
|:--|:--|
|.filter { it != null }<br>.map { it!! }|.filterNotNull()|
|.map { <Transformation> }<br>.filterNotNull()|.mapNotNull { <Transformation> }|
|.map { <Transformation> }<br>.joinToString()|.joinToString { <Transformation> }|
|.filter { <Predicate 1> }<br>.filter { <Predicate 2> }|.filter { <Predicate 1> && <Predicate 2> }|
|.filter { it is Type }<br>.map { it as Type }|.filterIsInstance<Type>()|
|.sortedBy { <Key 2> }<br>.sortedBy { <Key 1> }|.sortedWith( compareBy( { <Key 1> }, { <Key 2> } ) )|
|listOf(...)<br>.filterNotNull()|listOfNotNull(...)|
|.withIndex()<br>.filter { (index, elem) -> <Predicate using index> }<br>.map { it.value }|.filterIndexed { index, elem -> <Predicate using index> }<br>(map, forEach, reduce, fold도 비슷)|