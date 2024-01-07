# 아이템 48. 더 이상 사용하지 않는 객체의 레퍼런스를 제거하라

메모리 관리를 자동으로 해 주는 프로그래밍 언어에 익숙한 개발자는 객체 해제<small>(free)</small>를 따로 생각하지 않지 않습니다. 자바는 가비지 컬렉터가 객체 해제와 관련된 모든 작업을 해주지만, 그렇다고 메모리 관리를 완전 무시해 버리면 메모리 누수가 발생해서 OOM이 발생하기도 합니다.

따라서 **'더 이상 사용하지 않는 객체의 레퍼런스를 유지하면 안된다'** 라는 규칙 정도는 지켜 주는 것이 좋으며 아래의 경우는 특히 더 그렇습니다.
- 어떤 객체가 메모리를 많이 차지할 경우
- 어떤 객채의 인스턴스가 많이 생성될 경우

### 예시 1. 큰 객체에 대한 레퍼런스를 companion 프로퍼티에 할당

```kotlin
class MainActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //...
        activity = this
    }

    //...

    companion object {
        // DON'T DO THIS! It is a huge memory leak
        var activity: MainActivity? = null
    }
}
```

객체에 대한 참조를 companion<small>(또는 static)</small>으로 유지해 버리면, 가비지 컬렉터가 해당 객체에 대한 메모리를 해제할 수 없습니다.

액티비티는 굉장히 큰 객체이므로 큰 메모리 누수를 발생시킬 수 있기에, 의존 관계를 정적으로 저장하지 말고 다른 방법을 활용해서 적절하게 관리하는게 좋습니다. 객체에 대한 레퍼런스를 다른 곳에 저장할 때는 메모리 누수가 발생할 가능성을 언제나 염두에 두기 바랍니다.

### 예시 2. 컬렉션의 요소를 해제하지 않음

```kotlin
class Stack {
    private var elements: Array<Any?> =
        arrayOfNulls(DEFAULT_INITIAL_CAPACITY)
    private var size = 0

    fun push(e: Any) {
        ensureCapacity()
        elements[size++] = e
    }

    fun pop(): Any? {
        if (size == 0) {
            throw EmptyStackException()
        }
        return elements[--size]
    }

    private fun ensureCapacity() {
        if (elements.size == size) {
            elements = elements.copyOf(2 * size + 1)
        }
    }

    companion object {
        private const val DEFAULT_INITIAL_CAPACITY = 16
    }
}
```

위 코드에선 pop을 할 때 size를 감소시키기만 하고, 배열 위의 요소를 해제하는 부분이 없습니다.

스택에 1000개의 요소가 있고, 이어서 pop을 통해 size를 1까지 줄였다고 가정해 봅시다. 위 코드의 스택은 1000개의 요소를 모두 붙들고 놓아 주지 않으므로, 가비지 컬렉터가 이를 해제하지 못하여 999개의 요소가 메모리를 낭비하게 됩니다. 이는 메모리 누수가 발생한다는 것을 뜻하며 누수가 누적이 되면 OOM이 발생하게 됩니다.

수정은 간단합니다. **객체를 더 이상 사용하지 않을 때, 그 레퍼런스에 null을 설정**하기만 하면 됩니다.

```kotlin
fun pop(): Any? {
    if (size == 0)
        throw EmptyStackException()
    val elem = elements[--size]
    elements[size] = null
    return elem
}
```

### 예시 3. mutableLazy 프로퍼티 델리게이트 구현 예시

```kotlin
fun <T> mutableLazy(
    initializer: () -> T
): ReadWriteProperty<Any?, T> =
    MutableLazy(initializer)

private class MutableLazy<T>(
    val initializer: () -> T
) : ReadWriteProperty<Any?, T> {

    private var value: T? = null
    private var initialized = false

    override fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        synchronized(this) {
            if (!initialized) {
                value = initializer()
                initialized = true
            }
            return value as T
        }
    }

    override fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: T
    ) {
        synchronized(this) {
            this.value = value
            initialized = true
        }
    }
}

// usage
var game: Game? by mutableLazy { readGameFromSave() }

fun setUpActions() {
    startNewGameButton.setOnClickListener {
        game = makeNewGame()
        startGame()
    }
    resumeGameButton.setOnClickListener {
        startGame()
    }
}
```

위의 mutableLazy 구현은 한 가지 결점을 가지고 있습니다. initializer가 사용된 후에도 해제되지 않는다는 것입니다. MutableLazy에 대한 참조가 존재한다면, 이것이 더 이상 필요 없어도 유지됩니다. 이와 관련된 부분을 조금 더 개선해보면 다음과 같습니다.

```kotlin
fun <T> mutableLazy(
    initializer: () -> T
): ReadWriteProperty<Any?, T> =
    MutableLazy(initializer)

private class MutableLazy<T>(
    var initializer: (() -> T)?
) : ReadWriteProperty<Any?, T> {

    private var value: T? = null

    override fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): T {
        synchronized(this) {
            val initializer = initializer
            if (initializer != null) {
                value = initializer()
                this.initializer = null
            }
            return value as T
        }
    }

    override fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: T
    ) {
        synchronized(this) {
            this.value = value
            this.initializer = null
        }
    }
}
```

initializer를 null로 설정하기만 하면, 가비지 컬렉터가 이를 처리할 수 있습니다.

#### 이런 최적화 처리가 과연 중요할까요?

거의 사용되지 않는 객체까지 이런 것을 신경 쓰는 것은 오히려 좋지 않을 수도 있습니다. 쓸데없는 최적화가 모든 악의 근원이라는 말도 있습니다. 하지만 오브젝트에 null을 설정하는 것은 그렇게 어려운 일이 아니므로, 무조건 하는 것이 좋습니다. 특히 많은 [변수를 캡쳐할 수 있는 함수 타입](https://stackoverflow.com/a/22025344), Any 또는 제네릭 타입과 같은 미지의 클래스일 때는 이러한 처리가 중요합니다.

예를 들어 Stack과 같이 범용적으로 사용되는 것들은 어떤 식으로 사용될지 예측하기 어렵습니다. 따라서 이런 것들은 최적화에 더 신경을 써야 합니다. 즉, 라이브러리를 만들 때 이런 최적화가 중요합니다. 예를 들어 코틀린 stdlib에 구현되어 있는 lazy 델리게이트는 사용 후에 모두 initializer를 null로 초기화합니다.

```kotlin
private class SynchronizedLazyImpl<out T>(
    initializer: () -> T, lock: Any? = null
) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    private var _value: Any? = UNINITIALIZED_VALUE
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    override fun isInitialized(): Boolean =
        _value !== UNINITIALIZED_VALUE

    override fun toString(): String =
        if (isInitialized()) value.toString()
        else "Lazy value not initialized yet."

    private fun writeReplace(): Any =
        InitializedLazyImpl(value)
}
```

절대 사용되지 않는 객체를 캐시해서 저장해 두는 경우 메모리 누수가 발생할 수 있습니다. 물론 캐시를 해 두는 것이 나쁜 것은 아니지만 이것이 OOM을 일으킬 수 있다면 아무런 도움이 되지 않습니다. 해결 방법은 소프트 레퍼런스<small>(soft reference)</small>를 사용하는 것입니다.

소프트 레퍼런스를 활용하면, 메모리가 필요한 경우에는 가비지 컬렉터가 이를 알아서 해제합니다. 하지만 메모리가 부족하지 않아서 해제되지 않았다면, 이를 활용할 수 있습니다.

화면 위의 대화상자와 같은 일부 객체는 약한 레퍼런스<small>(weak reference)</small>를 사용하는 것이 좋을 수 있습니다. 대화상자가 출력되는 동안에는 가비지 컬렉터가 이를 수집하지 않지만, 대화상자를 닫을 이후에는 이에 대한 참조를 유지할 필요가 전혀 없으므로 약한 레퍼런스를 사용하면 좋습니다.

사실 객체를 수동으로 해제해야하는 경우는 굉장히 드뭅니다. 일반적으로 스코프를 벗어나면서, 어떤 객체를 가리키는 레퍼런스가 제거될 때 객체가 자동으로 해제됩니다. 따라서 메모리와 관련된 문제를 피하는 가장 좋은 방법은 `아이템 2. 변수의 스코프를 최소화하라`에서 언급했던 것처럼 변수를 지역 스코프<small>(local scope)</small>에 정의하고, 톱레벨 프로퍼티 또는 객체 선언<small>(companion 객체 포함)</small>으로 큰 데이터를 저장하지 않는 것입니다.