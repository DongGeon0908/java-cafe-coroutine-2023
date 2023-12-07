
# Round 9

### 학습 내용

1. 플로우의 실제 구현 (20)
2. 플로우 만들기 (21)
3. 플로우 생명주기 함수 (22)

---

# 플로우의 실제 구현

**플로우란**
- 어떤 연산을 실행할지 정의한 것
- 중단 가능한 람다식에 몇 가지 요소를 추가한 것

**플로우 구현 방식**

- collect를 호출하면, flow 빌더를 호출할 때 넣은 람다식이 실행
- 빌더의 람다식이 emit을 호출하면, collect가 호출되었을 때 명시된 람다식이 호출
- 그런데, 왜 이렇게 복잡하게 Detph를 두었을까??

```
fun interface FlowCollector {
    suspend fun emit(value: String)
}

interface Flow {
    suspend fun collect(collector: FlowCollector)
}

fun flow(
    builder: suspend FlowCollector.() -> Unit,
) = object : Flow {
    override suspend fun collect(collector: FlowCollector) {
        collector.builder()
    }
}

suspend fun main() {
    val f: Flow = flow {
        emit("A")
        emit("B")
        emit("C")
    }
    f.collect {
        delay(1000)
        println(it + " " + currentCoroutineContext())
    } // ABC
    f.collect {
        delay(1000)
        println(it + " " + currentCoroutineContext())
    } // ABC
}
```

### Flow 처리 방식

- Flow는 리시버가 있는 중단 람다식에 비해 훨씬 복잡하다고 여겨짐 (극 공감)
- 플로의 강력한 점은 플로우를 생성하고, 처리하고, 감지하기 위해 정의된 함수에 있음

```kotlin
// flow library map
fun <T, R> Flow<T>.map( transformation: suspend (T) -> R
): Flow<R> = flow {
    collect {
        emit(transformation(it))
    }
}

suspend fun main() {
    flowOf("A", "B", "C")
        .map {
            delay(1000)
            it.lowercase()
        }
        .collect { println(it) }
}
```
- 여기서 map은 플로의 각 원소를 변환하는 함수
- mapdms 새로운 플로우를 만들기 때문에, flow 빌더를 사용
- 플로우가 시작되면 래핑하고 있는 플로우를 시작하게 되므로, 빌더 내부에서 collect


