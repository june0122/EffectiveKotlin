# 아이템 20. 일반적인 프로퍼티 패턴은 프로퍼티 위임으로 만들어라

코틀린은 코드 재사용과 관련해서 프로퍼티 위임<small>(property delegation)</small>이라는 새로운 기능을 제공합니다. 프로퍼티 위임을 사용하면 일반적인 프로퍼티의 행위를 추출해서 재사용할 수 있습니다.

대표적인 예로 지연 프로퍼티가 있습니다. `lazy` 프로퍼티는 이후에 처음 사용하는 요청이 들어올 때 초기화되는 프로퍼티를 의미합니다. 이러한 패턴은 굉장히 많이 사용됩니다. 일반적으로 대부분의 언어<small>(자바스크립트 등)</small>에서는 필요할 때마다 이를 복잡하게 구현해야 하지만, 코틀린에서는 프로퍼티 위임을 활용해 간단하게 구현할 수 있습니다. 코틀린의 stdlib는 `lazy` 프로퍼티 패턴을 쉽게 구현할 수 있게 `lazy` 함수를 제공합니다.

```kotlin
val value by lazy { createValue() }
```

프로퍼티 위임을 사용하면, 이외에도 변화가 있을 때 이를 감지하는 observable 패턴을 쉽게 만들 수 있습니다. 예를 들어 목록을 출력하는 리스트 어댑터가 있다면, 내부 데이터가 변경될 때마다 변경된 내용을 다시 출력해야 할 것입니다. 또한 프로퍼티의 변경 사항을 로그로 출력하고 싶은 경우도 있을 것입니다. 이러한 것들은 다음과 같이 stdlib의 observable 델리게이트를 기반으로 간단하게 구현할 수 있습니다.

```kotlin
var items: List<Item> by Delegates.observable(listOf()) { _, _, _ ->
    notifyDataSetChanged()
}

var key: String? by Delegates.observable(null) { _, old, new ->
    Log.e("key changed from $old to $new")
}
```

lazy와 observable 델리게이터는 언어적인 관점에서 보았을 때, 그렇게 특별한 것은 아닙니다. 일반적으로 프로퍼티 위임 매커니즘을 활용하면, 다양한 패턴들을 만들 수 있습니다.

#### 예시

- 뷰와 리소스 바인딩
- 데이터 바인딩
- 의존성 주입

일반적으로 이런 패턴들을 사용할 때 자바 등에서는 어노테이션을 많이 활용해야 하지만 코틀린은 프로퍼티 위임을 사용해서 간단하고 type-safe하게 구현할 수 있습니다.

```kotlin
// 안드로이드에서의 뷰와 리소스 바인딩
private val button: Button by bindView(R.id.button)
private val textSize by bindDimension(R.dimen.font_size)
private val doctor: Doctor by argExtra(DOCTOR_ARG)

// Koin에서의 종속성 주입
private val presenter: MainPresenter by inject()
private val repository: NetworkRepository by inject()
private val vm: MainViewModel by viewModel()

// 데이터 바인딩
private val port by bindConfiguration("port")
private val token: String by preferences.bind(TOKEN_KEY)
```

어떻게 이런 코드가 가능하고, 프로퍼티 위임을 어떻게 활용할 수 있는지 살펴볼 수 있게, 간단한 프로퍼티 델리게이트를 만들어 보겠습니다. 예를 들어 일부 프로퍼티가 사용될 때, 간단한 로그를 출력하고 싶다고 해 봅시다. 가장 기본적인 구현 방법은 다음과 같이 게터와 세터에서 로그를 출력하는 방법입니다.

```kotlin
var token: String? = null
	get(){
    	print("token returned value $field")
        return field
    }
    set(value){
    	print("token changed from $field to $value")
        field = value
    }

var attempts: Int = 0
	get(){
    	print("attempts returned value $field")
        return field
    }
    set(value){
    	print("attempts changed from $field to $value")
        field = value
    }
```

두 프로퍼티는 타입이 다르지만, **내부적으로 거의 같은 처리**를 합니다. 또한 **프로젝트에서 자주 반복될 것 같은 패턴**처럼 보입니다. 따라서 프로퍼티 위임을 활용해서 추출하기 좋은 부분입니다.

프로퍼티 위임은 다른 객체의 메서드를 활용해서 프로퍼티의 접근자<small>(게터와 세터)</small>를 만드는 방식입니다. 이때 다른 객체의 메서드 이름이 중요한데요. 게터는 `getVaule`, 세터는 `setValue` 함수를 사용해서 만들어야 합니다. 객체를 만든 뒤에는 `by` 키워드를 사용해서 `getVaule`와 `setValue`를 정의한 클래스와 연결해 주면 됩니다.

#### 이전의 코드를 프로퍼티 위임을 활용해 변경한 예

```kotlin
var token: String? by LoggingProperty(null)
var attempts: Int by LoggingProperty(0)

private class LoggingProperty<T>(var value: T) {
    operator fun getValue(
        thisRef: Any?,
        prop: KProperty<*>
    ): T {
        print("${prop.name} returned value $value")
        return value
    }

    operator fun setValue(
        thisRef: Any?,
        prop: KProperty<*>,
        newValue: T
    ) {
        val name = prop.name
        print("$name changed from $value to $newValue")
        value = newValue
    }
}
```

프로퍼티 위임이 어떻게 동작하는지 이해하려면, `by`가 어떻게 컴파일되는지 보는 것이 좋습니다. 위의 코드에서 token 프로퍼티는 다음과 비슷한 형태로 컴파일됩니다<small>(프로퍼티가 톱레벨에서 사용될 때는 `this` 대신 `null`로 바뀝니다.)</small>.

```kotlin
@JvmField
private val 'token$delegate' = 
	LoggingProperty<String?>(null)

var token: String?
	get() = 'token$delegate'.getValue(this, ::token)
    set(Value){
    	'token$delegate'.setValue(this, ::token, value)
    }
```

코드를 보면 알 수 있는 것처럼 `getVaule`와 `setValue`는 단순하게 값만 처리하게 바뀌는 것이 아니라, 컨텍스트<small>(`this`)</small>와 프로퍼티 레퍼런스의 경계도 함께 사용하는 형태로 바뀝니다. 프로퍼티에 대한 레퍼런스는 이름, 어노테이션과 관련된 정보 등을 얻을 때 사용됩니다. 그리고 컨텍스트는 함수가 어떤 위치에서 사용되는지와 관련된 정보를 제공해줍니다.

이러한 정보로 인해서 `getVaule`와 `setValue` 메서드가 여러 개 있어도 문제 없습니다. `getVaule`와 `setValue` 메서드가 여러 개 있어도 컨텍스트를 활용하므로, 상황에 따라서 적절한 메서드가 선택됩니다. 이는 굉장히 다양하게 활용됩니다.

예를 들어 *여러 종류의 뷰와 함께 사용할 수 있는 델리게이트가 필요한 경우*에도 다음과 같이 구현해서 컨텍스트의 종류에 따라서 적절한 메서드가 선택되게 만들 수 있습니다.

```kotlin
class SwipeRefreshBinderDelegate(val id: Int) {
	private var cache: SwipeRefreshLayout? = null

	operator fun getValue(
		activity: Activity,
		prop: KProperty<*>,
	): SwipeRefreshLayout {
	return cache?: activity
		.findViewById<SwipeRefreshLayout>(id)
		.also { cache = it }
	}

	operator fun getValue(
		fragment: Fragment,
		prop: KProperty<*>
	): SwipeRefreshLayout {
		return cache?: fragment.view
		.findViewById<SwipeRefreshLayout>(id)
		.also { cache = it }
	}
}
```

## 정리

프로퍼티 델리게이트는 프로퍼티와 관련된 다양한 조작을 할 수 있으며, 컨텍스트와 관련된 대부분의 정보를 갖습니다. 이러한 특징으로 인해서 다양한 프로퍼티의 동작을 추출해서 재사용할 수 있습니다.

표준 라이브러리의 `lazy`와 `observable`이 대표적인 예입니다. 프로퍼티 위임은 프로퍼티 패턴을 추출하는 일반적인 방법이라 많이 사용되고 있습니다. 따라서 코틀린 개발자라면 프로퍼티 위임이라는 강력한 도구와 관련된 내용을 잘 알고 있어야 합니다. 이를 잘 알면, 일반적인 패턴을 추출하거나 더 좋은 API를 만들 때 활용할 수 있을 것입니다.