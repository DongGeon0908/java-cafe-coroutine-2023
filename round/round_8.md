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

### 핫데이터 소스, 콜드 데이터 소스, FLOW

**핫데이터란?**

- 데이터를 소비하는 것과 무관하게 원소를 생성
- Channel, List, Set

**콜드데이터란?**

- 요청이 있을 떄만 작업을 수행
- 아무것도 저장하지 않음
- Flow, Stream, Sequence
- 중간에 생성되는 값들을 보관할 필요 없어, 메모리 사용에 좋음

**Sequence**

- 원소를 지연 처리, 적은 연산
- 특정 연산을 수행하고 나서, 그것에 대한 임시 데이터를 기록하지 않고, 모든 연산을 처리후에 데이터를 반환

```kotlin
fun m(i: Int): Int { print("m$i ")
return i * i }
fun f(i: Int): Boolean { print("f$i ")
return i >= 10 }
fun main() {
    listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) }
        .find { f(it) }
        .let { print(it) }

    // m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 f1 f4 f9 f16 16
    println()

  sequenceOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    .map { m(it) }
    .find { f(it) }
    .let { print(it) }
// m1 f1 m2 f4 m3 f9 m4 f16 16
}

```
- 리스트는 원소의 컬렉션
- **Sequence는 원소를 어떻게 계산할 것인지 정의한 것**

**핫 데이터 스트림**

```kotlin
fun m(i: Int): Int { print("m$i ")

return i * i }

fun main() {

val l = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) } // m1 m2 m3 m4 m5 m6 m7 m8 m9 m10
    println(l) // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    println(l.find { it > 10 }) // 16
    println(l.find { it > 10 }) // 16
    println(l.find { it > 10 }) // 16

val s = sequenceOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10) .map { m(it) }
    println(s.toList())
    // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    println(s.find { it > 10 }) // m1 m2 m3 m4 16
    println(s.find { it > 10 }) // m1 m2 m3 m4 16
    println(s.find { it > 10 }) // m1 m2 m3 m4 16
}
```

- 항상 사용 가능한 상태 (모든 연산이 최종 연산이 될 수 있음)
- 여러번 사용되었을 때 매번 결과를 다시 계산할 필요 없음

### 핫 채널, 콜드 플로우

**기억을 되짚어, channel 생성 방법**

```kotlin
val channel = produce { while (true) {
val x = computeNextValue()
  send(x) }
}
```
- 다음과 같이 채널을 생성할 수 있음
- 채널은 바로 값을 계산
- 별도의 코루틴에서 계산을 수행
- 코루틴의 확장함수어야 함 (코루틴 빌더)
- 소비되는 것과 상관없이 값을 생성

**플로우를 생성하는 일반적인 방법**

```kotlin

val flow = flow { while (true) {
val x = computeNextValue()
  emit(x) }
}
```
- 반면에 flow는 코루틴 빌더가 아님
- 값이 필요할 때만 생성
- 최종 연산이 호출될 때 원소가 어떻게 생성되어야 하는 지 정의
- flow 빌더는 빌더를 호출한 최정 연산의 스코프에서 동작


flow code
```kotlin
/**
 * Creates a _cold_ flow from the given suspendable [block].
 * The flow being _cold_ means that the [block] is called every time a terminal operator is applied to the resulting flow.
 *
 * Example of usage:
 *
 * ```
 * fun fibonacci(): Flow<BigInteger> = flow {
 *     var x = BigInteger.ZERO
 *     var y = BigInteger.ONE
 *     while (true) {
 *         emit(x)
 *         x = y.also {
 *             y += x
 *         }
 *     }
 * }
 *
 * fibonacci().take(100).collect { println(it) }
 * ```
 *
 * Emissions from [flow] builder are [cancellable] by default &mdash; each call to [emit][FlowCollector.emit]
 * also calls [ensureActive][CoroutineContext.ensureActive].
 *
 * `emit` should happen strictly in the dispatchers of the [block] in order to preserve the flow context.
 * For example, the following code will result in an [IllegalStateException]:
 *
 * ```
 * flow {
 *     emit(1) // Ok
 *     withContext(Dispatcher.IO) {
 *         emit(2) // Will fail with ISE
 *     }
 * }
 * ```
 *
 * If you want to switch the context of execution of a flow, use the [flowOn] operator.
 */
public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

// Named anonymous object
private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}
```

여기서 이슈는... 이런 FLow를 어떤 상황에서 사용해야할지..
(정말 필요한 상황???)

### 플로우란 무엇인가?

**플로우**

- 비동기적으로 계산해야 할 값의 스트림
- Flow 인터페이스 자체는 떠다니는 원소들을 모으는 역할
- 플로우의 끝에 도달할 떄까지 각 값을 처리하는 것을 의미

**collect**
```
/**
 * An asynchronous data stream that sequentially emits values and completes normally or with an exception.
 *
 * _Intermediate operators_ on the flow such as [map], [filter], [take], [zip], etc are functions that are
 * applied to the _upstream_ flow or flows and return a _downstream_ flow where further operators can be applied to.
 * Intermediate operations do not execute any code in the flow and are not suspending functions themselves.
 * They only set up a chain of operations for future execution and quickly return.
 * This is known as a _cold flow_ property.
 *
 * _Terminal operators_ on the flow are either suspending functions such as [collect], [single], [reduce], [toList], etc.
 * or [launchIn] operator that starts collection of the flow in the given scope.
 * They are applied to the upstream flow and trigger execution of all operations.
 * Execution of the flow is also called _collecting the flow_  and is always performed in a suspending manner
 * without actual blocking. Terminal operators complete normally or exceptionally depending on successful or failed
 * execution of all the flow operations in the upstream. The most basic terminal operator is [collect], for example:
 *
 * ```
 * try {
 *     flow.collect { value ->
 *         println("Received $value")
 *     }
 * } catch (e: Exception) {
 *     println("The flow has thrown an exception: $e")
 * }
 * ```
 *
 * By default, flows are _sequential_ and all flow operations are executed sequentially in the same coroutine,
 * with an exception for a few operations specifically designed to introduce concurrency into flow
 * execution such as [buffer] and [flatMapMerge]. See their documentation for details.
 *
 * The `Flow` interface does not carry information whether a flow is a _cold_ stream that can be collected repeatedly and
 * triggers execution of the same code every time it is collected, or if it is a _hot_ stream that emits different
 * values from the same running source on each collection. Usually flows represent _cold_ streams, but
 * there is a [SharedFlow] subtype that represents _hot_ streams. In addition to that, any flow can be turned
 * into a _hot_ one by the [stateIn] and [shareIn] operators, or by converting the flow into a hot channel
 * via the [produceIn] operator.
 *
 * ### Flow builders
 *
 * There are the following basic ways to create a flow:
 *
 * * [flowOf(...)][flowOf] functions to create a flow from a fixed set of values.
 * * [asFlow()][asFlow] extension functions on various types to convert them into flows.
 * * [flow { ... }][flow] builder function to construct arbitrary flows from
 *   sequential calls to [emit][FlowCollector.emit] function.
 * * [channelFlow { ... }][channelFlow] builder function to construct arbitrary flows from
 *   potentially concurrent calls to the [send][kotlinx.coroutines.channels.SendChannel.send] function.
 * * [MutableStateFlow] and [MutableSharedFlow] define the corresponding constructor functions to create
 *   a _hot_ flow that can be directly updated.
 *
 * ### Flow constraints
 *
 * All implementations of the `Flow` interface must adhere to two key properties described in detail below:
 *
 * * Context preservation.
 * * Exception transparency.
 *
 * These properties ensure the ability to perform local reasoning about the code with flows and modularize the code
 * in such a way that upstream flow emitters can be developed separately from downstream flow collectors.
 * A user of a flow does not need to be aware of implementation details of the upstream flows it uses.
 *
 * ### Context preservation
 *
 * The flow has a context preservation property: it encapsulates its own execution context and never propagates or leaks
 * it downstream, thus making reasoning about the execution context of particular transformations or terminal
 * operations trivial.
 *
 * There is only one way to change the context of a flow: the [flowOn][Flow.flowOn] operator
 * that changes the upstream context ("everything above the `flowOn` operator").
 * For additional information refer to its documentation.
 *
 * This reasoning can be demonstrated in practice:
 *
 * ```
 * val flowA = flowOf(1, 2, 3)
 *     .map { it + 1 } // Will be executed in ctxA
 *     .flowOn(ctxA) // Changes the upstream context: flowOf and map
 *
 * // Now we have a context-preserving flow: it is executed somewhere but this information is encapsulated in the flow itself
 *
 * val filtered = flowA // ctxA is encapsulated in flowA
 *    .filter { it == 3 } // Pure operator without a context yet
 *
 * withContext(Dispatchers.Main) {
 *     // All non-encapsulated operators will be executed in Main: filter and single
 *     val result = filtered.single()
 *     myUi.text = result
 * }
 * ```
 *
 * From the implementation point of view, it means that all flow implementations should
 * only emit from the same coroutine.
 * This constraint is efficiently enforced by the default [flow] builder.
 * The [flow] builder should be used if the flow implementation does not start any coroutines.
 * Its implementation prevents most of the development mistakes:
 *
 * ```
 * val myFlow = flow {
 *    // GlobalScope.launch { // is prohibited
 *    // launch(Dispatchers.IO) { // is prohibited
 *    // withContext(CoroutineName("myFlow")) { // is prohibited
 *    emit(1) // OK
 *    coroutineScope {
 *        emit(2) // OK -- still the same coroutine
 *    }
 * }
 * ```
 *
 * Use [channelFlow] if the collection and emission of a flow are to be separated into multiple coroutines.
 * It encapsulates all the context preservation work and allows you to focus on your
 * domain-specific problem, rather than invariant implementation details.
 * It is possible to use any combination of coroutine builders from within [channelFlow].
 *
 * If you are looking for performance and are sure that no concurrent emits and context jumps will happen,
 * the [flow] builder can be used alongside a [coroutineScope] or [supervisorScope] instead:
 *  - Scoped primitive should be used to provide a [CoroutineScope].
 *  - Changing the context of emission is prohibited, no matter whether it is `withContext(ctx)` or
 *    a builder argument (e.g. `launch(ctx)`).
 *  - Collecting another flow from a separate context is allowed, but it has the same effect as
 *    applying the [flowOn] operator to that flow, which is more efficient.
 *
 * ### Exception transparency
 *
 * When `emit` or `emitAll` throws, the Flow implementations must immediately stop emitting new values and finish with an exception.
 * For diagnostics or application-specific purposes, the exception may be different from the one thrown by the emit operation,
 * suppressing the original exception as discussed below.
 * If there is a need to emit values after the downstream failed, please use the [catch][Flow.catch] operator.
 *
 * The [catch][Flow.catch] operator only catches upstream exceptions, but passes
 * all downstream exceptions. Similarly, terminal operators like [collect][Flow.collect]
 * throw any unhandled exceptions that occur in their code or in upstream flows, for example:
 *
 * ```
 * flow { emitData() }
 *     .map { computeOne(it) }
 *     .catch { ... } // catches exceptions in emitData and computeOne
 *     .map { computeTwo(it) }
 *     .collect { process(it) } // throws exceptions from process and computeTwo
 * ```
 * The same reasoning can be applied to the [onCompletion] operator that is a declarative replacement for the `finally` block.
 *
 * All exception-handling Flow operators follow the principle of exception suppression:
 *
 * If the upstream flow throws an exception during its completion when the downstream exception has been thrown,
 * the downstream exception becomes superseded and suppressed by the upstream exception, being a semantic
 * equivalent of throwing from `finally` block. However, this doesn't affect the operation of the exception-handling operators,
 * which consider the downstream exception to be the root cause and behave as if the upstream didn't throw anything.
 *
 * Failure to adhere to the exception transparency requirement can lead to strange behaviors which make
 * it hard to reason about the code because an exception in the `collect { ... }` could be somehow "caught"
 * by an upstream flow, limiting the ability of local reasoning about the code.
 *
 * Flow machinery enforces exception transparency at runtime and throws [IllegalStateException] on any attempt to emit a value,
 * if an exception has been thrown on previous attempt.
 *
 * ### Reactive streams
 *
 * Flow is [Reactive Streams](http://www.reactive-streams.org/) compliant, you can safely interop it with
 * reactive streams using [Flow.asPublisher] and [Publisher.asFlow] from `kotlinx-coroutines-reactive` module.
 *
 * ### Not stable for inheritance
 *
 * **The `Flow` interface is not stable for inheritance in 3rd party libraries**, as new methods
 * might be added to this interface in the future, but is stable for use.
 *
 * Use the `flow { ... }` builder function to create an implementation, or extend [AbstractFlow].
 * These implementations ensure that the context preservation property is not violated, and prevent most
 * of the developer mistakes related to concurrency, inconsistent flow dispatchers, and cancellation.
 */
public interface Flow<out T> {

    /**
     * Accepts the given [collector] and [emits][FlowCollector.emit] values into it.
     *
     * This method can be used along with SAM-conversion of [FlowCollector]:
     * ```
     * myFlow.collect { value -> println("Collected $value") }
     * ```
     *
     * ### Method inheritance
     *
     * To ensure the context preservation property, it is not recommended implementing this method directly.
     * Instead, [AbstractFlow] can be used as the base type to properly ensure flow's properties.
     *
     * All default flow implementations ensure context preservation and exception transparency properties on a best-effort basis
     * and throw [IllegalStateException] if a violation was detected.
     */
    public suspend fun collect(collector: FlowCollector<T>)
}
```

- 유일한 멤버 함수, 기타 플로우 로직은 extension function으로 구현되어 있음

### 플로우와 시퀀스 비교, 시퀀스 먼저

**list,set**

- 모든 원소의 계산이 완료된 컬렉션
- 값들을 계산하는 과정에 시간이 걸리기 때문에, 원소들이 채워질 떄까지 모든 값이 생성되길 기다려야 함

```kotlin
fun getList(): List<Int> = List(3) { Thread.sleep(1000)
"User$it"
}
fun main() {
val list = getList() println("Function started") list.forEach { println(it) }
}
// (3 sec)
// Function started
// User0
// User1
// User2

```
- 원소를 하나씩 계산할 때는 원소가 나오자마자 얻을 수 있는 것이 좋아서 -> Sequence가 이때 좋음

**Sequence**

```kotlin
fun getList(): List<Int> = List(3) {
  Thread.sleep(1000)
  "User$it"
}

fun main() {
  val list = getList()
  println("Function started")

  list.forEach { println(it) }
}
// (3 sec)
// Function started
// User0
// User1
// User2

```

- CPU 집약적인 연산, 블로킹 연산일 때 필요시마다 값을 계산하는 플로우에 적용하기 적합
- 시퀀스의 최종연산은 중단함수가 아니기 때문에 (foreach, map) 시퀀스 내부에 중단점이 있으면 값을 기다리는 스레드가 블로킹
- 시퀀스 빌더의 스코프 안에서는 SequenceScope의 리시버에서 호출되는 yield, yieldAll 외에 다른 중단 함수 사용 불가
- 시퀀스 로직 안에 중단 함수 있으면, 예상하지 못한 스레드 블로킹을 통해 지옥을 맛봄
- 데이터 소스의 개수가 많거나, 원소가 무거운 경우, 시퀀스 사용에 적합

```kotlin
val fibonacci: Sequence<BigInteger> = sequence {
  var first = 0.toBigInteger()
var second = 1.toBigInteger()

while (true) {
  yield(first)
  val temp = first
  first += second
  second = temp
}
}
fun countCharactersInFile(path: String): Int = File(path).useLines { lines ->
        lines.sumBy { it.length }
    }
```

- 이전에 파일 입출력 관련 로직을 만들때 Sequence를 사용했는데, 확실히 성능상 좋았음..

**시퀀스 로직의 이슈**
- 같은 스레드에서 launch로 시작된 코루틴이 대기하게 되면서, 하나의 코루틴이 다른 코루틴을 블로킹

```kotlin
fun getSequence(): Sequence<String> = sequence { repeat(3) {
Thread.sleep(1000)
// the same result as if there were delay(1000) here yield("User$it")
} }

suspend fun main() { withContext(newSingleThreadContext("main")) {
launch {
            repeat(3) {
                delay(100)
                println("Processing on coroutine")
            }
}
val list = getSequence()
        list.forEach { println(it) }
    }
}
// (1 sec)
// User0
// (1 sec)
// User1
// (1 sec)
// User2
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
```

- 이런 상황에서, Flow 사용해야 함 (다른 말로는.. 논블리킹 코드가 들어가야하는 경우에 사용하는 것이 좋다?)

**플로우를 사용하자!**

- Sequence에서 제공하는 기능을 전부 사용 가능
- 구조화된 동시성 그리고 예외 처리 기능이 수월
- Sequence로직과 같이 동작해야 하는데, 코루틴이 섞여 있는 경우라면, FLow를 사용하는 것이 이득!!

```kotlin
fun getFlow(): Flow<String> = flow { repeat(3) {
delay(1000)
emit("User$it") }
}
suspend fun main() { withContext(newSingleThreadContext("main")) {
        launch {
            repeat(3) {

  delay(100)
                println("Processing on coroutine")
            }
}
val list = getFlow()
        list.collect { println(it) }
    }
}
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (0.1 sec)
// Processing on coroutine
// (1 - 3 * 0.1 = 0.7 sec)
// User0
// (1 sec)
// User1
// (1 sec)
// User2
```

**플로우의 특징**

- 플로우의 최종 연산은 스레드를 블로킹하는 대신 코루틴을 중단
- 코루틴 컨텍스트를 활용하고, 예외를 처리하는 코루틴 기능 제공
- 취소가 가능, 구조화된 동시성 적용 가능
- flow 빌더는 중단 함수가 아니며, 어떤 스코프도 불필요
- 최종 연산은 중단 가능, 연산이 실행될 때 부모 코루틴과의 관계 정립
- flow의 연산은 가장 마지막 최종 연산의 코루틴을 통해 진행

**launch를 취소하면 플로우 처리도 적절하게 취소**

```kotlin
// Notice, that this function is not suspending // and does not need CoroutineScope
fun usersFlow(): Flow<String> = flow {
    repeat(3) {
        delay(1000)
val ctx = currentCoroutineContext() val name = ctx[CoroutineName]?.name emit("User$it in $name")
} }
suspend fun main() {
val users = usersFlow()
withContext(CoroutineName("Name")) { val job = launch {
            // collect is suspending
            users.collect { println(it) }
        }
launch {

} }
}
delay(2100)
println("I got enough")
job.cancel()
// (1 sec)
// User0 in Name
// (1 sec)
// User1 in Name
// (0.1 sec)
// I got enough
```

**플로우의 실제 사용 사례**

- 실시간으로 데이터를 처리하는 경우'
- 웹소켓과 같은 알림 서버
- 처리율 제한 장치 (flatMapMerge)


