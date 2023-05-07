# 아이템 13. Unit?을 리턴하지 말라

> "함수에서 Unit?을 리턴한다면, 그 이유는 무엇일까요? 그렇다면 이러한 코드를 사용해도 괜찮을까요?"

Boolean이 true 또는 false를 갖는 것처럼, Unit?은 Unit 또는 null이라는 값을 가질 수 있습니다.

일반적으로 Unit?을 사용한다는 것은 아래와 같은 경우 입니다.

```kotlin
fun keyIsCorrect(key: String): Boolean  = //...

if(!keyIsCorrect(key)) return
```

위의 코드를 아래와 같은 코드처럼 사용할 수도 있습니다.

```kotlin
fun verifyKey(key: String): Unit? = //...

verifyKey(key) ?: return
```

Unit?으로 Boolean 값을 표현하는 것에는 오해의 소지가 있으며, 예측하기 어려운 오류를 만들 수 있습니다.

기본적으로 Unit?을 리턴하거나, 이를 기반으로 연산하지 않는 것이 좋습니다.