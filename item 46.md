# 아이템 46. 함수 타입 파라미터를 갖는 함수에 inline 한정자를 붙여라

### inline 한정자

inline 한정자의 역할은 컴파일 시점에 '함수를 호출하는 부분'을 '함수의 본문'으로 대체하는 것입니다.

예를 들어 repeat 함수를 다음과 같이 구현되어 있습니다.

```kotlin
inline fun repeat(times: Int, action: (Int) -> Unit){
	for(index in 0 until times){
		action(index)
	}
}
```
 
다음과 같이 repeat 함수를 호출하는 코드가 있다면,

```kotlin
repeat(10){
	println(it)
}
```

컴파일 시점 다음과 같이 대체됩니다.

```kotlin
for(index in 0 until 10){
	println(index)
}
```
 
이처럼 inline 한정자를 붙여 함수를 만들면, 굉장히 큰 변화가 일어납니다. 일반적인 함수를 호출하면 함수 본문으로 점프하고, 본문의 모든 문장을 호출한 뒤에 함수를 호출했던 위치로 다시 점프하는 과정을 거치빈다. 하지만 '함수를 호출하는 부분'을 '함수의 본문'으로 대체하면, 이러한 점프가 일어나지 않습니다.

#### inline 한정자의 장점

1. 타입 아규먼트에 reified 한정자를 붙여 사용할 수 있다 (제네릭의 타입정보를 runtime 시점에 알 수 있음)
2. 함수 타입 파라미터를 가진 함수가 훨씬 빠르게 동작합니다. (점프하는 과정이 없음)
3. 비지역<small>(non-local)</small> 리턴을 사용할 수 있습니다. (non-inline이라면 return이 불가능하지만 inline은 return을 할 수 있다)