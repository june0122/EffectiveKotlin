# 아이템 11. 가독성을 목표로 설계하라

> "개발자가 코드를 작성하는 데는 1분 걸리지만, 이를 읽는 데는 10분이 걸린다." - *로버트 마틴, <클린 코드>*

개발자는 코드를 작성하는 것보다 읽는 데 많은 시간을 소모하므로 항상 가독성을 생각하면서 코드를 작성해야 합니다.

## 인식 부하 감소

```kotlin
// 구현 A
if (person != null && person.isAdult) {
    view.showPerson(person)
} else {
    view.showError()
}

// 구현 B
person?.takeIf { it.Adult }
    ?.let(view::showPerson)
    ?: view.showError()
```

가독성이란 코드를 읽고 얼마나 빠르게 이해할 수 있는지를 의미합니다. 이는 우리의 뇌가 얼마나 많은 관용구(구조, 함수, 패턴)에 익숙해져 있는지에 따라서 다릅니다.

코틀린 초보자에게는 *구현 A*가 더 읽고 이해하기 쉽습니다. 일반적인 관용구인 `if/else`, `&&`, `메서드 호출`을 사용하고 있기 때문입니다.

*구현 B*는 코틀린에서는 꽤 일반적으로 사용되는 관용구인 `safe call ?.`, `takeIf`, `let`, `Elvis 연산자`, `제한된 함수 레퍼런스 view::showPerson`을 사용하고 있습니다. 코틀린에서 일반적으로 사용되는 관용구이므로, 경험이 많은 코틀린 개발자라면 그래도 코드를 쉽게 읽을 수 있을 것이지만, 숙련된 개발자만을 위한 코드는 좋은 코드가 아닙니다.

구현 A와 구현 B는 사실 비교조차 할 수 없을 정도로 A가 훨씬 가독성이 좋은 코드입니다.

- 사용 빈도가 적은 관용구는 코드를 복잡하게 만듭니다.
- 구현 A는 수정하기 쉽지만 구현 B는 수정해야 할 경우, 함수 참조를 더 이상 사용할 수 없으므로 코드를 수정해야 합니다.
  - 그리고 B는 Elvis 연산자의 오른쪽 부분이 하나 이상의 표현식을 갖게 하려면, `run`과 같은 함수를 추가로 사용해야 합니다.

```kotlin
if (person != null && person.isAdult) {
    view.showPerson(person)
    view.hideProgressWithSuccess()
} else {
    view.showError()
    view.hideProgress()
}

person?.takeIf { it.Adult }
    ?.let {
        view.showPerson(it)
        view.hideProgressWithSuccess()
    } ?: run {
        view.showError()
        view.hideProgress()
    }
```

- 구현 A는 디버깅도 더 간단합니다.

참고로 구현 A와 구현 B는 실행 결과가 다릅니다. <b>`let`은 람다식의 결과를 리턴합니다.</b> 즉, showPerson이 null을 리턴하면, 두 번째 구현 때는 showError도 호출합니다. 익숙하지 않은 구조를 사용하면, 이처럼 잘못된 동작을 코드를 보면서 확인하기 어렵습니다. 

기본적으로 **인지 부하**를 줄이는 방향으로 코드를 작성하세요. 우리의 뇌는 패턴을 인식하고, 패턴을 기반으로 프로그램의 작동 방식을 이해합니다. 가독성은 **뇌가 프로그램의 작동 방식을 이해하는 과정**을 더 짧게 만드는 것입니다. 자주 사용되는 패턴을 활용하면, 이와 같은 과정을 더 짧게 만들 수 있습니다. **뇌는 기본적으로 짧은 코드를 빠르게 읽을 수 있겠지만, 익숙한 코드는 더 빠르게 읽을 수 있습니다.**

## 극단적이 되지 않기

방금 `let`으로 인해서 예상하지 못한 결과가 나올 수 있다고 하였는데, 이 이야기를 **let은 절대로 쓰면 안 된다**로 이해하는 사람들이 꽤 많습니다. 극단적이 되지 말기 바랍니다. `let`은 좋은 코드를 만들기 위해서 다양하게 활용되는 인기 있는 관용구입니다.

예를 들어 nullable 가변 프로퍼티가 있고, null이 아닐 때만 어떤 작업을 수행해야 하는 경우가 있다고 합시다. 가변 프로퍼티는 쓰레드와 관련된 문제를 발생시킬 수 있으므로, 스마트 캐스팅이 불가능합니다. 여러 가지 해결 방법이 있는데, 일반적으로 당므과 같이 safe call let, `?.let`을 사용합니다.

```kotlin
class Person(val name: String)
var person: Person? = null

fun printName() {
    person?.let {
        print(it.name)
    }
}
```

이런 관용구는 널리 사용되며, 많은 사람이 쉽게 인식합니다. 이외에도 다음과 같은 경우에 `let`을 많이 사용합니다.

- 연산을 아규먼트 처리 후로 이동시킬 때
  - `print(students.filter{}.joinToString{})`라는 코드에서 `print`연산을 뒤로 이동시켜서 `students.filter{}.joinToString{}.let(::print)`처럼 만든다는 의미
- 데코레이터를 사용해서 객체를 랩할 때


```kotlin
// 연산을 아규먼트 처리 후로 이동
students
    .filter { it.result >= 50 }
    .joinToString(separator = "\n") {
        "${it.name} ${it.surname}, ${it.result}"
    }
    .let(::print)

// 데코레이터 사용하여 객체를 랩
var obj = FileInputStream("/file.gz")
    .let(::BufferedInputStream)
    .let(::ZipInputStream)
    .let(::ObjectInputStream)
    .readObject() as SomeObject
```

이 코드들은 디버기하기 어렵고, 경험이 적은 코틀린 개발자는 이해하기 어렵습니다. 따라서 비용이 발생합니다. 하지만 이 비용은 지불할 만한 가치가 있으므로 사용해도 괜찮습니다. 문제가 되는 경우는 비용을 지불할 만한 가치가 없는 코드에 비용을 지불하는 경우(정당한 이유 없이 복잡성을 추가할 때)입니다.

물론 어떤 것이 비용을 지불할 만한 코드인지 아닌지는 항상 논란이 있을 수 있습니다. 균형을 맞추는 것이 중요합니다. 일단 어떤 구조들이 어떤 복잡성을 가져오는지 등을 파악하는 것이 좋습니다. 또한 두 구조를 조합해서 사용하면 복잡성이 단순히 더해지는 것이 아니라, 훨씬 커진다는 것을 기억해주세요.