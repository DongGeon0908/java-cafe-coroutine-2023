### Round 8

- 17장 셀렉트
- 18장 핫 데이터소스와 콜드 데이터소스
- 19장 플로우란 무었인가?

### 셀렉트

- 가장 먼저 완료되는 코루틴의 결과를 기다립니다.

### 사용예시

- 코루틴의 채널, 코루틴을 통한 경합등에서 사용합니다.

### 지연되는 값 선택!

예시) 특정 IO 요청을 여러 개 보내고, 그중에서 가장 빨리 답이 온 것만 반환할 때, select를 사용할 수 있습니다.

```kotlin
suspend fun requestData1(): String {
  delay(100_000)
  return "Data1"
}

suspend fun requestData2(): String {
  delay(1000)
  return "Data2"
}

val scope = CoroutineScope(SupervisorJob())


suspend fun askMultipleForData(): String {
val defData1 = scope.async { requestData1() }
val defData2 = scope.async { requestData2() }

return select {
        defData1.onAwait { it }
        defData2.onAwait { it }
    }
}

suspend fun main(): Unit = coroutineScope {
    println(askMultipleForData())
}

// (1 sec)
// Data2
```

그런데, 위의 코드에서 발생할 수 있는 문제는, 자식 코루틴이 다 끝날 때까지 대기해야 한다는 문제가 있습니다.
그래서, 명시적으로 해당 경합이후의 코루틴에 대해서는 취소시키는 작업을 선언해야 합니다.

```kotlin
suspend fun askMultipleForData(): String = coroutineScope { select<String> {
        async { requestData1() }.onAwait { it }
        async { requestData2() }.onAwait { it }
    }.also { coroutineContext.cancelChildren() }
}
```

책에서는, 해당 해결이 복잡하다고 하는데, 이런 구조면 충분히 사용가능하지 않을까.. 생각합니다..

- [splitties library](https://splitties.louiscad.com/modules/coroutines/)

찾아보니, 아직 이슈가 살아 있네요.
https://github.com/Kotlin/kotlinx.coroutines/issues/2867


### select code 탐색

```kotlin
/**
 * Waits for the result of multiple suspending functions simultaneously, which are specified using _clauses_
 * in the [builder] scope of this select invocation. The caller is suspended until one of the clauses
 * is either _selected_ or _fails_.
 *
 * At most one clause is *atomically* selected and its block is executed. The result of the selected clause
 * becomes the result of the select. If any clause _fails_, then the select invocation produces the
 * corresponding exception. No clause is selected in this case.
 *
 * This select function is _biased_ to the first clause. When multiple clauses can be selected at the same time,
 * the first one of them gets priority. Use [selectUnbiased] for an unbiased (randomized) selection among
 * the clauses.

 * There is no `default` clause for select expression. Instead, each selectable suspending function has the
 * corresponding non-suspending version that can be used with a regular `when` expression to select one
 * of the alternatives or to perform the default (`else`) action if none of them can be immediately selected.
 *
 * ### List of supported select methods
 *
 * | **Receiver**     | **Suspending function**                           | **Select clause**
 * | ---------------- | ---------------------------------------------     | -----------------------------------------------------
 * | [Job]            | [join][Job.join]                                  | [onJoin][Job.onJoin]
 * | [Deferred]       | [await][Deferred.await]                           | [onAwait][Deferred.onAwait]
 * | [SendChannel]    | [send][SendChannel.send]                          | [onSend][SendChannel.onSend]
 * | [ReceiveChannel] | [receive][ReceiveChannel.receive]                 | [onReceive][ReceiveChannel.onReceive]
 * | [ReceiveChannel] | [receiveCatching][ReceiveChannel.receiveCatching] | [onReceiveCatching][ReceiveChannel.onReceiveCatching]
 * | none             | [delay]                                           | [onTimeout][SelectBuilder.onTimeout]
 *
 * This suspending function is cancellable. If the [Job] of the current coroutine is cancelled or completed while this
 * function is suspended, this function immediately resumes with [CancellationException].
 * There is a **prompt cancellation guarantee**. If the job was cancelled while this function was
 * suspended, it will not resume successfully. See [suspendCancellableCoroutine] documentation for low-level details.
 *
 * Note that this function does not check for cancellation when it is not suspended.
 * Use [yield] or [CoroutineScope.isActive] to periodically check for cancellation in tight loops if needed.
 */
public suspend inline fun <R> select(crossinline builder: SelectBuilder<R>.() -> Unit): R {
    contract {
        callsInPlace(builder, InvocationKind.EXACTLY_ONCE)
    }
    return suspendCoroutineUninterceptedOrReturn { uCont ->
        val scope = SelectBuilderImpl(uCont)
        try {
            builder(scope)
        } catch (e: Throwable) {
            scope.handleBuilderException(e)
        }
        scope.getResult()
    }
}
```

```kotlin

    /**
     * Specifies that the function parameter [lambda] is invoked in place.
     *
     * This contract specifies that:
     * 1. the function [lambda] can only be invoked during the call of the owner function,
     *  and it won't be invoked after that owner function call is completed;
     * 2. _(optionally)_ the function [lambda] is invoked the amount of times specified by the [kind] parameter,
     *  see the [InvocationKind] enum for possible values.
     *
     * A function declaring the `callsInPlace` effect must be _inline_.
     *
     */
    /* @sample samples.contracts.callsInPlaceAtMostOnceContract
    * @sample samples.contracts.callsInPlaceAtLeastOnceContract
    * @sample samples.contracts.callsInPlaceExactlyOnceContract
    * @sample samples.contracts.callsInPlaceUnknownContract
    */
    @ContractsDsl public fun <R> callsInPlace(lambda: Function<R>, kind: InvocationKind = InvocationKind.UNKNOWN): CallsInPlace
```

```kotlin
    @PublishedApi
    internal fun getResult(): Any? {
        if (!isSelected) initCancellability()
        var result = _result.value // atomic read
        if (result === UNDECIDED) {
            if (_result.compareAndSet(UNDECIDED, COROUTINE_SUSPENDED)) return COROUTINE_SUSPENDED
            result = _result.value // reread volatile var
        }
        when {
            result === RESUMED -> throw IllegalStateException("Already resumed")
            result is CompletedExceptionally -> throw result.cause
            else -> return result // either COROUTINE_SUSPENDED or data
        }
    }
```

### 채널에서 값 선택하기

- onReceive : 채널이 값을 가지고 있을 선택, select는 람다식의 결과값을 반환
- onReceiveCatching: 채널이 값을 가지고 있거나 닫혔을 때 선택, select는 람다식의 결과값을 반환
- onSend: 채널의 버퍼에 공간이 있을 때 선택, select는 Unit을 반환

```kotlin
suspend fun CoroutineScope.produceString(s: String, time: Long ) = produce {
  while (true) {
  delay(time)
  send(s)
}
}
fun main() = runBlocking {

val fooChannel = produceString("foo", 210L) val barChannel = produceString("BAR", 500L)
    repeat(7) {
        select {
            fooChannel.onReceive {
              println("From fooChannel: $it")
            }
            barChannel.onReceive {
              println("From barChannel: $it")
            }
        }
    }
    coroutineContext.cancelChildren()
}
// From fooChannel: foo
// From fooChannel: foo
// From barChannel: BAR
// From fooChannel: foo
// From fooChannel: foo
// From barChannel: BAR
// From fooChannel: foo
```

결론적으로 select는 로직의 경합, 채널의 송수신 등에서 사용할 수 있습니다.
그런데, kotlin coroutine issue에 select 관련해서 내용이 있는 것을 보니.. 쓰기 좀 망설여집니다.
