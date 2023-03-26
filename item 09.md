# 아이템 9. use를 사용하여 리소스를 닫아라

더 이상 필요하지 않을 때, close 메서드를 사용해서 명시적으로 닫아야 하는 리소스가 있습니다. 코틀린/JVM에서 사용하는 자바 표준 라이브러리에는 이런 리소스들이 굉장히 많습니다.

예를 들어
- InputStream과 OutputStream
- java.sql.Connection
- java.io.Reader(FileReader, BufferedReader, CSSParser)
- java.new.Socket과 java.util.Scanner

등이 있습니다. 이러한 리소스들은 AutoCloseable을 상속받는 Closeable 인터페이스를 구현<small>(implement)</small>하고 있습니다.

이러한 모든 리소스는 최종적으로 리소스에 대한 레퍼런스가 없어질 때, 가비지 컬렉터가 처리합니다. 하지만 굉장히 느리며(쉽게 처리되지 않음), 그동안 리소스를 유지하는 비용이 많이 들어갑니다. 따라서 더 이상 필요하지 않다면, 명시적으로 close 메서드를 호출해 주는 것이 좋습니다. 전통적으로 이러한 리소스는 다음과 같이 try-finally 블록을 사용해서 처리했습니다.

```kotlin
fun countCharactersInFile(path: String): Int {
    val reader = BufferedReader(FileReader(path))
    try {
        return reader.lineSequence().sumBy { it.length }
    } finally {
        reader.close()
    }
}
```

하지만 이런 코드는 굉장히 복잡하고 좋지 않습니다. 리소스를 닫을 때 예외가 발생할 수도 있는데, 이러한 예외를 따로 처리하지 않기 때문입니다. 또한 try 블록과 finally 블록 내부에서 오류가 발생하면, 둘 중 하나만 전파됩니다. 둘 다 전파될 수 있다면 좋겠지만 이를 직접 구현하려면 코드가 굉장히 길고 복잡해집니다. 그래도 굉장히 많이 사용되는 일반적인 구현이기에, 표준 라이브러리에는 `use`라는 이름의 함수로 포함되어 있습니다. `use` 함수를 사용해서 앞의 코드를 적절하게 변경하면, 다음과 같습니다. 이러한 코드는 모든 Closeable 객체에 사용할 수 있습니다.

```kotlin
fun countCharactersInFile(path: String): Int {
    val reader = BufferedReader(FileReader(path))
    reader.use {
        return reader.lineSequence().sumBy { it.length }
    }
}
```

람다 매개변수로 리시버(현재 코드에서는 reader)가 전달되는 형태도 있으므로, 줄여서 다음과 같이 작성할 수도 있습니다.

```kotlin
fun countCharactersInFile(path: String): Int {
    BufferedReader(FileReader(path)).use { reader ->
        return reader.lineSequence().sumBy { it.length }
    } 
}
```

파일을 리소스로 사용하는 경우가 많고, 파일을 한 줄씩 읽어 들이는 경우도 많으므로, 코틀린 표준 라이브러리는 파일을 한 줄씩 처리할 때 활용할 수 있는 `useLines` 함수도 제공합니다.

```kotlin
fun countCharactersInFile(path: String): Int {
    File(path).useLines { lines ->
        return lines.sumBy { it.length }
    }
}
```

이렇게 처리하면 메모리에 파일의 내용을 한 줄씩만 유지하므로, 대용량 파일도 적절하게 처리할 수 있습니다. 다만 파일의 줄을 한 번만 사용할 수 있다는 단점이 있습니다. 파일의 특정 줄을 두 번 이상 반복 처리하려면, 파일을 두 번 이상 열어야 합니다. 앞의 코드는 다음과 같이 간단하게 작성할 수도 있습니다.

```kotlin
fun countCharactersInFile(path: String): Int =
    File(path).useLines { lines ->
        return lines.sumBy { it.length }
    }
```

지금까지 시퀀스를 활용하여 파일을 처리하는 적절한 방법을 살펴보았는데, 시퀀스 처리와 관련된 추가적인 내용은 `아이템 49: 하나 이상의 처리 단계를 가진 경우에는 시퀀스를 사용하라`에서 알아보도록 합시다.

## 정리

`use`를 사용하면 Closeable/AutoCloseable을 구현한 객체를 쉽고 안전하게 처리할 수 있습니다. 또한 파일을 처리할 때는 파일을 한 줄씩 읽어 들이는 `useLines`를 사용하는 것이 좋습니다.