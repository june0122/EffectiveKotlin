# 아이템 47. 인라인 클래스의 사용을 고려하라

### inline 클래스

```kotlin
inline class Name(private val value: String){
	//...
}
```

String을 name으로 래핑 해서 사용하고 있습니다.

하지만 inline을 활용하기 때문에 어떠한 오버헤드도 발생하지 않습니다. (inline으로 바로 호출해서 사용하기 때문에 래핑의 비용이 들지 않음)

### inline 클래스의 활용

보통 측청 단위를 표현할 때
타입 오용으로 발생하는 문제를 막을 때 

측정 단위를 표현할 때
만약 time이라는 단어를 마주하면 이게 ms인지 s 인지 min인지 명확하지 않습니다.

가장 간단한 방법으로 파라미터 이름에 측정 단위를 붙여 줄 수 있습니다.

하지만 함수를 사용할 때 프로퍼티 이름이 표현되지 않는다면 여전히 실수할 수 있습니다.

타입에 제한을 걸고 inline 클래스를 활용하면 코드를 가독성이 높고 효율적으로 만들 수 있습니다.

### 타입 오용으로 발생하는 문제를 막을 때

학생의 ID, 교사의 ID, 학교의 ID 를 모두 int로 표현할 수 있습니다.

이렇게 되면 모든 ID가 int 자료형이여 실수로 잘못된 값을 넣을 수 있습니다.

이를 막기위해 inline class로 명시적인 studentId, teacherId, schoolId 클래스를 만들어서 ID활용을 안정적으로 사용할 수 있습니다.

이를 대체하는 방법으로 typealias를 활용할 수도 있습니다.

하지만 typealias의 경우 studnetId에 teacherId를 넣을 수 있게 되므로 안전하지 않습니다.

즉, inline class를 활용하면 비용과 안전이라는 두 마리 토끼를 모두 잡을 수 있습니다.


### interface와 inline class

inline class를 interface를 override 해서 사용하는 경우 inline으로 동작하지 않습니다.