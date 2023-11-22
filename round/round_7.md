# Round 7, Channel
> 채널


<br>
<br>
<br>

### Channel이란?

- 채널은 코루틴끼리의 통신을 위한 기술입니다.
- 채널은 송신자와 수신자의 수에 제한이 없고, 채널을 통해 전송된 모든 값은 단 한번만 받을 수 있음
  - Message Queue와 비슷

![image](https://github.com/DongGeon0908/java-cafe-coroutine-2023/assets/50691225/dcb92083-dd04-4ed1-ab29-e2119ed14c7a)
![image](https://github.com/DongGeon0908/java-cafe-coroutine-2023/assets/50691225/5225f185-69c3-49f4-9efe-f4bd91bcde99)


<br>
<br>
<br>

### Channel Interface

- 상위 인터페이스로, SendChannel과 ReceiveChannel이 존재

```kotlin
interface SendChannel<in E> { suspend fun send(element: E) fun close(): Boolean
//...
}
interface ReceiveChannel<out E> {
suspend fun receive(): E
fun cancel(cause: CancellationException? = null) // ...
}
interface Channel<E> : SendChannel<E>, ReceiveChannel<E>
```


<br>
<br>
<br>

### receive, send
- receive를 호출했는데 채널에 원소가 없다면, 코루틴은 원소가 들어올 때 까지 중지
- send는 채널의 용량이 다 차게 되는 경우 중단
- 중단 함수 가 아닌 경우에는 trySend, tryReceive를 사용해도 되지만, 이것들은 버퍼가 없는 채널에서 동작하므로 용량 확인 유의


<br>
<br>
<br>

### 생성자와 소비자

> 생성자는 원소를 채널로 보내고, 소비자는 채널에서 원소를 받음


<br>
<br>
<br>

### Default Example

```
suspend fun main(): Unit = coroutineScope { val channel = Channel<Int>()
launch {
        repeat(5) { index ->
            delay(1000)
            println("Producing next one")
            channel.send(index * 2)
        }
}
    launch {
        repeat(5) {
} }
```

위의 기본 예제에서는, 수신자는 송신자가 얼마의 데이터를 보낼 것인지 알고 있어야 하므로, 불편함
(얼마나 받을지 고려하지 않고, for문 혹은 consumeEach를 사용하는 것을 선호)

### loop example

```kotlin
suspend fun main(): Unit = coroutineScope { val channel = Channel<Int>()
launch {
        repeat(5) { index ->
            println("Producing next one")
            delay(1000)
            channel.send(index * 2)
}
        channel.close()
    }
launch {
for (element in channel) {
// or
// channel.consumeEach { element ->
//     println(element)
// }
} }
```
- consumeEach의 경우, 모든 원소를 가지고 온 다음에 채널을 취소함
- 해당 방식의 단점은, 채널은 close하는 것을 계속 선언해야 함


<br>
<br>
<br>

### produce coroutine builder

```kotlin
fun CoroutineScope.produceNumbers(
max: Int
): ReceiveChannel<Int> = produce {
var x = 0
while (x < 5) send(x++) }
```
- 해당 빌더는 ReceiveChannel을 반환하는데, 시작된 코루틴이 어떤 상황이던 채널을 종료시킴
- 채널의 소멸을 알아서 관리

ReceiveChannel.kt
```kotlin
public interface ReceiveChannel<out E> {
    /**
     * Returns `true` if this channel was closed by invocation of [close][SendChannel.close] on the [SendChannel]
     * side and all previously sent items were already received. This means that calling [receive]
     * will result in a [ClosedReceiveChannelException]. If the channel was closed because of an exception, it
     * is considered closed, too, but is called a _failed_ channel. All suspending attempts to receive
     * an element from a failed channel throw the original [close][SendChannel.close] cause exception.
     *
     * **Note: This is an experimental api.** This property may change its semantics and/or name in the future.
     */
    @ExperimentalCoroutinesApi
    public val isClosedForReceive: Boolean

    /**
     * Returns `true` if the channel is empty (contains no elements), which means that an attempt to [receive] will suspend.
     * This function returns `false` if the channel [is closed for `receive`][isClosedForReceive].
     */
    @ExperimentalCoroutinesApi
    public val isEmpty: Boolean

    /**
     * Retrieves and removes an element from this channel if it's not empty, or suspends the caller while the channel is empty,
     * or throws a [ClosedReceiveChannelException] if the channel [is closed for `receive`][isClosedForReceive].
     * If the channel was closed because of an exception, it is called a _failed_ channel and this function
     * will throw the original [close][SendChannel.close] cause exception.
     *
     * This suspending function is cancellable. If the [Job] of the current coroutine is cancelled or completed while this
     * function is suspended, this function immediately resumes with a [CancellationException].
     * There is a **prompt cancellation guarantee**. If the job was cancelled while this function was
     * suspended, it will not resume successfully. The `receive` call can retrieve the element from the channel,
     * but then throw [CancellationException], thus failing to deliver the element.
     * See "Undelivered elements" section in [Channel] documentation for details on handling undelivered elements.
     *
     * Note that this function does not check for cancellation when it is not suspended.
     * Use [yield] or [CoroutineScope.isActive] to periodically check for cancellation in tight loops if needed.
     *
     * This function can be used in [select] invocations with the [onReceive] clause.
     * Use [tryReceive] to try receiving from this channel without waiting.
     */
    public suspend fun receive(): E

    /**
     * Clause for the [select] expression of the [receive] suspending function that selects with the element
     * received from the channel.
     * The [select] invocation fails with an exception if the channel
     * [is closed for `receive`][isClosedForReceive] (see [close][SendChannel.close] for details).
     */
    public val onReceive: SelectClause1<E>

    /**
     * Retrieves and removes an element from this channel if it's not empty, or suspends the caller while this channel is empty.
     * This method returns [ChannelResult] with the value of an element successfully retrieved from the channel
     * or the close cause if the channel was closed. Closed cause may be `null` if the channel was closed normally.
     * The result cannot be [failed][ChannelResult.isFailure] without being [closed][ChannelResult.isClosed].
     *
     * This suspending function is cancellable. If the [Job] of the current coroutine is cancelled or completed while this
     * function is suspended, this function immediately resumes with a [CancellationException].
     * There is a **prompt cancellation guarantee**. If the job was cancelled while this function was
     * suspended, it will not resume successfully. The `receiveCatching` call can retrieve the element from the channel,
     * but then throw [CancellationException], thus failing to deliver the element.
     * See "Undelivered elements" section in [Channel] documentation for details on handling undelivered elements.
     *
     * Note that this function does not check for cancellation when it is not suspended.
     * Use [yield] or [CoroutineScope.isActive] to periodically check for cancellation in tight loops if needed.
     *
     * This function can be used in [select] invocations with the [onReceiveCatching] clause.
     * Use [tryReceive] to try receiving from this channel without waiting.
     */
    public suspend fun receiveCatching(): ChannelResult<E>

    /**
     * Clause for the [select] expression of the [onReceiveCatching] suspending function that selects with the [ChannelResult] with a value
     * that is received from the channel or with a close cause if the channel
     * [is closed for `receive`][isClosedForReceive].
     */
    public val onReceiveCatching: SelectClause1<ChannelResult<E>>

    /**
     * Retrieves and removes an element from this channel if it's not empty, returning a [successful][ChannelResult.success]
     * result, returns [failed][ChannelResult.failed] result if the channel is empty, and [closed][ChannelResult.closed]
     * result if the channel is closed.
     */
    public fun tryReceive(): ChannelResult<E>

    /**
     * Returns a new iterator to receive elements from this channel using a `for` loop.
     * Iteration completes normally when the channel [is closed for `receive`][isClosedForReceive] without a cause and
     * throws the original [close][SendChannel.close] cause exception if the channel has _failed_.
     */
    public operator fun iterator(): ChannelIterator<E>

    /**
     * Cancels reception of remaining elements from this channel with an optional [cause].
     * This function closes the channel and removes all buffered sent elements from it.
     *
     * A cause can be used to specify an error message or to provide other details on
     * the cancellation reason for debugging purposes.
     * If the cause is not specified, then an instance of [CancellationException] with a
     * default message is created to [close][SendChannel.close] the channel.
     *
     * Immediately after invocation of this function [isClosedForReceive] and
     * [isClosedForSend][SendChannel.isClosedForSend]
     * on the side of [SendChannel] start returning `true`. Any attempt to send to or receive from this channel
     * will lead to a [CancellationException].
     */
    public fun cancel(cause: CancellationException? = null)

    /**
     * @suppress This method implements old version of JVM ABI. Use [cancel].
     */
    @Deprecated(level = DeprecationLevel.HIDDEN, message = "Since 1.2.0, binary compatibility with versions <= 1.1.x")
    public fun cancel(): Unit = cancel(null)

    /**
     * @suppress This method has bad semantics when cause is not a [CancellationException]. Use [cancel].
     */
    @Deprecated(level = DeprecationLevel.HIDDEN, message = "Since 1.2.0, binary compatibility with versions <= 1.1.x")
    public fun cancel(cause: Throwable? = null): Boolean

    /**
     * **Deprecated** poll method.
     *
     * This method was deprecated in the favour of [tryReceive].
     * It has proven itself as error-prone method in Channel API:
     *
     * * Nullable return type creates the false sense of security, implying that `null`
     *    is returned instead of throwing an exception.
     * * It was used mostly from non-suspending APIs where CancellationException triggered
     *   internal failures in the application (the most common source of bugs).
     * * Its name was not aligned with the rest of the API and tried to mimic Java's queue instead.
     *
     * See https://github.com/Kotlin/kotlinx.coroutines/issues/974 for more context.
     *
     * ### Replacement note
     *
     * The replacement `tryReceive().getOrNull()` is a default that ignores all close exceptions and
     * proceeds with `null`, while `poll` throws an exception if the channel was closed with an exception.
     * Replacement with the very same 'poll' semantics is `tryReceive().onClosed { if (it != null) throw it }.getOrNull()`
     *
     * @suppress **Deprecated**.
     */
    @Deprecated(
        level = DeprecationLevel.ERROR,
        message = "Deprecated in the favour of 'tryReceive'. " +
            "Please note that the provided replacement does not rethrow channel's close cause as 'poll' did, " +
            "for the precise replacement please refer to the 'poll' documentation",
        replaceWith = ReplaceWith("tryReceive().getOrNull()")
    ) // Warning since 1.5.0, error since 1.6.0
    public fun poll(): E? {
        val result = tryReceive()
        if (result.isSuccess) return result.getOrThrow()
        throw recoverStackTrace(result.exceptionOrNull() ?: return null)
    }

    /**
     * This function was deprecated since 1.3.0 and is no longer recommended to use
     * or to implement in subclasses.
     *
     * It had the following pitfalls:
     * - Didn't allow to distinguish 'null' as "closed channel" from "null as a value"
     * - Was throwing if the channel has failed even though its signature may suggest it returns 'null'
     * - It didn't really belong to core channel API and can be exposed as an extension instead.
     *
     * ### Replacement note
     *
     * The replacement `receiveCatching().getOrNull()` is a safe default that ignores all close exceptions and
     * proceeds with `null`, while `receiveOrNull` throws an exception if the channel was closed with an exception.
     * Replacement with the very same `receiveOrNull` semantics is `receiveCatching().onClosed { if (it != null) throw it }.getOrNull()`.
     *
     * @suppress **Deprecated**
     */
    @Suppress("INVISIBLE_REFERENCE", "INVISIBLE_MEMBER")
    @LowPriorityInOverloadResolution
    @Deprecated(
        message = "Deprecated in favor of 'receiveCatching'. " +
            "Please note that the provided replacement does not rethrow channel's close cause as 'receiveOrNull' did, " +
            "for the detailed replacement please refer to the 'receiveOrNull' documentation",
        level = DeprecationLevel.ERROR,
        replaceWith = ReplaceWith("receiveCatching().getOrNull()")
    ) // Warning since 1.3.0, error in 1.5.0, will be hidden in 1.6.0
    public suspend fun receiveOrNull(): E? = receiveCatching().getOrNull()

    /**
     * This function was deprecated since 1.3.0 and is no longer recommended to use
     * or to implement in subclasses.
     * See [receiveOrNull] documentation.
     *
     * @suppress **Deprecated**: in favor of onReceiveCatching extension.
     */
    @Deprecated(
        message = "Deprecated in favor of onReceiveCatching extension",
        level = DeprecationLevel.ERROR,
        replaceWith = ReplaceWith("onReceiveCatching")
    ) // Warning since 1.3.0, error in 1.5.0, will be hidden or removed in 1.7.0
    public val onReceiveOrNull: SelectClause1<E?>
        get() {
            return object : SelectClause1<E?> {
                @InternalCoroutinesApi
                override fun <R> registerSelectClause1(select: SelectInstance<R>, block: suspend (E?) -> R) {
                    onReceiveCatching.registerSelectClause1(select) {
                        it.exceptionOrNull()?.let { throw it }
                        block(it.getOrNull())
                    }
                }
            }
        }
}
```

<br>
<br>
<br>

### 채널 타입
> 선정한 용량 크기에 따라 4가지의 채널로 구분

- Unlimited : 제한이 없는 용량 버퍼 가짐, send 중단 X
- Buffered : 특정 용량 크기(64) 또는 오버라이드한 설정 값으로 설정된 채널
- Rendezvous : 용량이 0인 채널, 송신자와 수신자가 만날 때만 원소를 교환
- Conflated : 버퍼 크기가 1인 채널, 새로운 원소가 이전 원소를 대체

```kt
public interface Channel<E> : SendChannel<E>, ReceiveChannel<E> {
    /**
     * Constants for the channel factory function `Channel()`.
     */
    public companion object Factory {
        /**
         * Requests a channel with an unlimited capacity buffer in the `Channel(...)` factory function.
         */
        public const val UNLIMITED: Int = Int.MAX_VALUE

        /**
         * Requests a rendezvous channel in the `Channel(...)` factory function &mdash; a channel that does not have a buffer.
         */
        public const val RENDEZVOUS: Int = 0

        /**
         * Requests a conflated channel in the `Channel(...)` factory function. This is a shortcut to creating
         * a channel with [`onBufferOverflow = DROP_OLDEST`][BufferOverflow.DROP_OLDEST].
         */
        public const val CONFLATED: Int = -1

        /**
         * Requests a buffered channel with the default buffer capacity in the `Channel(...)` factory function.
         * The default capacity for a channel that [suspends][BufferOverflow.SUSPEND] on overflow
         * is 64 and can be overridden by setting [DEFAULT_BUFFER_PROPERTY_NAME] on JVM.
         * For non-suspending channels, a buffer of capacity 1 is used.
         */
        public const val BUFFERED: Int = -2

        // only for internal use, cannot be used with Channel(...)
        internal const val OPTIONAL_CHANNEL = -3

        /**
         * Name of the property that defines the default channel capacity when
         * [BUFFERED] is used as parameter in `Channel(...)` factory function.
         */
        public const val DEFAULT_BUFFER_PROPERTY_NAME: String = "kotlinx.coroutines.channels.defaultBuffer"

        internal val CHANNEL_DEFAULT_CAPACITY = systemProp(DEFAULT_BUFFER_PROPERTY_NAME,
            64, 1, UNLIMITED - 1
        )
    }
}
```

<br>
<br>
<br>

### Unlimited
> 정해진 크기의 용량을 가지고 있다면 버퍼가 가득 찰 때까지 원소가 생성, 생성자는 수신자가 원소를 소비하기를 기다리기 시작

### Rendezvous
> 기본 용량을 가진 채널의 경우 송신자는 항상 수신자를 기다림

### CONFLATED
> 이전 원소를 더 이상 저장하지 않음, 새로운 원소가 이전 원소를 대체, 최근 원소만 받을 수 있음 (먼저 보내진 원소는 유실, 버려짐)

<br>
<br>
<br>

### Buffer Overflow
- SUSPEND (default) : 버퍼가 가득 찼을 때, send 메서드 중단
- DROP_OLDEST : 버퍼가 가득 찼을 때, 가장 오래된 원소가 제거
- DROP_LATEST : 버퍼가 가득 찼을 때, 가장 최근의 원소가 제거

### 전달되지 않은 원소 핸들러 (onUndeliveredElement)
- 원소가 어떠한 이유로 처리되지 않을 때 호출 진행
- 채널이 닫히거나, 취소된 경우
- send, receive, receiveOrNull, hasNext에서 에러가 발생한 경우

### Fan-out

![image](https://github.com/DongGeon0908/java-cafe-coroutine-2023/assets/50691225/305e96d5-b750-43c7-b47a-f07f32f5ec39)


- 여러 개의 코루틴이 하나의 채널로부터 원소를 받을 수 있음
- 원소를 적절하게 처리하려면 for loop tkdyd
- consumeEach는 다수의 코루틴 사용시, 안전하지 않음
- 원소는 공평하게 배분
- 채널은 원소를 기다리는 코루틴을 FIFO 큐로 가지고 있음

```kotlin

@ObsoleteCoroutinesApi
public inline fun <E, R> ReceiveChannel<E>.consume(block: ReceiveChannel<E>.() -> R): R {
    var cause: Throwable? = null
    try {
        return block()
    } catch (e: Throwable) {
        cause = e
        throw e
    } finally {
        cancel(cause)
    }
}

@ObsoleteCoroutinesApi
public suspend inline fun <E> ReceiveChannel<E>.consumeEach(action: (E) -> Unit) =
    consume {
        for (e in this) action(e)
    }
```

왜 안전하지 않지??

<br>
<br>
<br>


### Fan-in

![image](https://github.com/DongGeon0908/java-cafe-coroutine-2023/assets/50691225/9f5cf5e3-a7d3-45ab-940c-78e9edbec075)

- 여러 개의 코루틴이 하나의 채널로 원소를 전송할 수 있음
- 다수의 채널을 하나의 채널로 합치는 경우 사용

### 더 궁금한 점..

- 왜 Fan-out에서 consumeEach가 불안정할까?
- EventListener 기반으로 동작하는 로직들을 코루틴의 Channel을 통해 개선할 수 있을까?
