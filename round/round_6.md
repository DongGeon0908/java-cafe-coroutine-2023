# 코틀린 코루틴 테스트하기

> 코틀린 코루틴 테스트 연습을 진행해보자



보통 중단 함수를 테스트할 때, runBlocking을 통해 코루틴을 열어주고 진행합니다.

```kotlin
inner class Test {
		@Test
  	fun `suspend function test`() = runBlocking {
				// suspend function
    }
}
```





dependency가 묶여있는 경우 mock 객체를 사용하거나, Test를 위한 Fake객체를 만들어 사용합니다.



보통은 mock Library를 많이 사용하는데, kotlin의 경우에는 [mockk](https://mockk.io/)라는 라이브러리를 주로 사용합니다.

```groovy
testImplementation("io.mockk:mockk:${mockkVersion}")
```

[spring tutorials](https://spring.io/guides/tutorials/spring-boot-kotlin/) 에서도 mockk 관련 예제가 있습니다. (mockk를 표준으로 잡은게 아닐지..)





mockk library의 예시입니다.

```kotlin
val car = mockk<Car>()

every { car.drive(Direction.NORTH) } returns Outcome.OK

car.drive(Direction.NORTH) // returns OK

verify { car.drive(Direction.NORTH) }

confirmVerified(car)
```

kotlin dsl이 적용되서, 좀더 직관적이고 편리하게 mock test가 가능합니다.



---



```kotlin
// 구현체가 UserRepository인 경우
class FakeTest : UserRepository {
		// override function
}
```

Fake객체의 경우에는, 조금 특수한 상황에서 많이 사용됩니다.





예를 들어, 코루틴의 성능 테스트나, 코루틴의 동작 과정에 대한 테스트를 진행할 때 사용됩니다.

```kotlin
// 구현체가 UserRepository인 경우
class FakeTest : UserRepository {
		override getUser(): Any {
				delay(1000) // 성능 테스트를 위해, 일부러 delay를 추가
		}
}
```



그런데, 이런 식의 테스트를 진행하는 경우에는, test 자체를 실행하는데 많은 시간이 소요될 수 있습니다. 그래서, 코루틴은 TestCoroutineScheduler와 StandardTestDispatcher를 지원하고 있습니다.



```kotlin
// 책속 예제
fun main() {
	val scheduler = TestCoroutineScheduler()
    	println(scheduler.currentTime) // 0
    	scheduler.advanceTimeBy(1_000)
    	println(scheduler.currentTime) // 1000
    	scheduler.advanceTimeBy(1_000)
    	println(scheduler.currentTime) // 2000
}
```

TestCoroutineScheduler는 가상 타임라인 안에서 실행되게하여 테스트를 진행합니다.



TestCoroutineScheduler

```kotlin
/**
     * Moves the virtual clock of this dispatcher forward by [the specified amount][delayTimeMillis], running the
     * scheduled tasks in the meantime.
     *
     * Breaking changes from [TestCoroutineDispatcher.advanceTimeBy]:
     * * Intentionally doesn't return a `Long` value, as its use cases are unclear. We may restore it in the future;
     *   please describe your use cases at [the issue tracker](https://github.com/Kotlin/kotlinx.coroutines/issues/).
     *   For now, it's possible to query [currentTime] before and after execution of this method, to the same effect.
     * * It doesn't run the tasks that are scheduled at exactly [currentTime] + [delayTimeMillis]. For example,
     *   advancing the time by one millisecond used to run the tasks at the current millisecond *and* the next
     *   millisecond, but now will stop just before executing any task starting at the next millisecond.
     * * Overflowing the target time used to lead to nothing being done, but will now run the tasks scheduled at up to
     *   (but not including) [Long.MAX_VALUE].
     *
     * @throws IllegalStateException if passed a negative [delay][delayTimeMillis].
     */
    @ExperimentalCoroutinesApi
    public fun advanceTimeBy(delayTimeMillis: Long) {
        require(delayTimeMillis >= 0) { "Can not advance time by a negative delay: $delayTimeMillis" }
        val startingTime = currentTime
        val targetTime = addClamping(startingTime, delayTimeMillis)
        while (true) {
            val event = synchronized(lock) {
                val timeMark = currentTime
                val event = events.removeFirstIf { targetTime > it.time }
                when {
                    event == null -> {
                        currentTime = targetTime
                        return
                    }
                    timeMark > event.time -> currentTimeAheadOfEvents()
                    else -> {
                        currentTime = event.time
                        event
                    }
                }
            }
            event.dispatcher.processEvent(event.time, event.marker)
        }
    }
```

(시간 데이터 마킹을 통해, delay test가 가능한 것 같습니다..)



코루틴에서 해당 기능을 사용하기 위해서는 [StandaradTestDispatcher](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/kotlinx.coroutines.test/-standard-test-dispatcher.html)를 사용해야 합니다. 해당 디스패처를 사용하는 경우, 지정된 가상 시간만큼 멈추고, 실행됩니다.



```kotlin
// 책속 예제
fun main() {
val scheduler = TestCoroutineScheduler()
val testDispatcher = StandardTestDispatcher(scheduler)
    CoroutineScope(testDispatcher).launch {
        println("Some work 1")
        delay(1000)
        println("Some work 2")
delay(1000)
        println("Coroutine done")
    }
println("[${scheduler.currentTime}] Before") scheduler.advanceUntilIdle() println("[${scheduler.currentTime}] After")
}
// [0] Before
// Some work 1
// Some work 2
// Coroutine done
// [2000] After
```



```kotlin
/**
 * Creates an instance of a [TestDispatcher] whose tasks are run inside calls to the [scheduler].
 *
 * This [TestDispatcher] instance does not itself execute any of the tasks. Instead, it always sends them to its
 * [scheduler], which can then be accessed via [TestCoroutineScheduler.runCurrent],
 * [TestCoroutineScheduler.advanceUntilIdle], or [TestCoroutineScheduler.advanceTimeBy], which will then execute these
 * tasks in a blocking manner.
 *
 * In practice, this means that [launch] or [async] blocks will not be entered immediately (unless they are
 * parameterized with [CoroutineStart.UNDISPATCHED]), and one should either call [TestCoroutineScheduler.runCurrent] to
 * run these pending tasks, which will block until there are no more tasks scheduled at this point in time, or, when
 * inside [runTest], call [yield] to yield the (only) thread used by [runTest] to the newly-launched coroutines.
 *
 * If no [scheduler] is passed as an argument, [Dispatchers.Main] is checked, and if it was mocked with a
 * [TestDispatcher] via [Dispatchers.setMain], the [TestDispatcher.scheduler] of the mock dispatcher is used; if
 * [Dispatchers.Main] is not mocked with a [TestDispatcher], a new [TestCoroutineScheduler] is created.
 *
 * One can additionally pass a [name] in order to more easily distinguish this dispatcher during debugging.
 *
 * @see UnconfinedTestDispatcher for a dispatcher that is not confined to any particular thread.
 */
@Suppress("FunctionName")
public fun StandardTestDispatcher(
    scheduler: TestCoroutineScheduler? = null,
    name: String? = null
): TestDispatcher = StandardTestDispatcherImpl(
    scheduler ?: TestMainDispatcher.currentTestScheduler ?: TestCoroutineScheduler(), name)

private class StandardTestDispatcherImpl(
    override val scheduler: TestCoroutineScheduler = TestCoroutineScheduler(),
    private val name: String? = null
) : TestDispatcher() {

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        scheduler.registerEvent(this, 0, block, context) { false }
    }

    override fun toString(): String = "${name ?: "StandardTestDispatcher"}[scheduler=$scheduler]"
}
```

내부적으로 TestCoroutineScheduler를 생성하고 있어, 명시적 선언을 할 필요 없습니다.

(대신, 가상화된 타임라인에서 시간이 흐르고 있음을 명시해야 합니다, 그렇지 않으면...계속 실행되어서 끝나지 않습니다.)



delay 이후에, 얼마나 시간이 지났는지 표현하는 방식은 다음과 같습니다.

```kotlin
    /**
     * Runs the enqueued tasks in the specified order, advancing the virtual time as needed until there are no more
     * tasks associated with the dispatchers linked to this scheduler.
     *
     * A breaking change from [TestCoroutineDispatcher.advanceTimeBy] is that it no longer returns the total number of
     * milliseconds by which the execution of this method has advanced the virtual time. If you want to recreate that
     * functionality, query [currentTime] before and after the execution to achieve the same result.
     */
    @ExperimentalCoroutinesApi
    public fun advanceUntilIdle(): Unit = advanceUntilIdleOr { events.none(TestDispatchEvent<*>::isForeground) }

    /**
     * [condition]: guaranteed to be invoked under the lock.
     */
    internal fun advanceUntilIdleOr(condition: () -> Boolean) {
        while (true) {
            if (!tryRunNextTaskUnless(condition))
                return
        }
    }

    /**
     * Runs the tasks that are scheduled to execute at this moment of virtual time.
     */
    @ExperimentalCoroutinesApi
    public fun runCurrent() {
        val timeMark = synchronized(lock) { currentTime }
        while (true) {
            val event = synchronized(lock) {
                events.removeFirstIf { it.time <= timeMark } ?: return
            }
            event.dispatcher.processEvent(event.time, event.marker)
        }
    }

    /**
     * Moves the virtual clock of this dispatcher forward by [the specified amount][delayTimeMillis], running the
     * scheduled tasks in the meantime.
     *
     * Breaking changes from [TestCoroutineDispatcher.advanceTimeBy]:
     * * Intentionally doesn't return a `Long` value, as its use cases are unclear. We may restore it in the future;
     *   please describe your use cases at [the issue tracker](https://github.com/Kotlin/kotlinx.coroutines/issues/).
     *   For now, it's possible to query [currentTime] before and after execution of this method, to the same effect.
     * * It doesn't run the tasks that are scheduled at exactly [currentTime] + [delayTimeMillis]. For example,
     *   advancing the time by one millisecond used to run the tasks at the current millisecond *and* the next
     *   millisecond, but now will stop just before executing any task starting at the next millisecond.
     * * Overflowing the target time used to lead to nothing being done, but will now run the tasks scheduled at up to
     *   (but not including) [Long.MAX_VALUE].
     *
     * @throws IllegalStateException if passed a negative [delay][delayTimeMillis].
     */
    @ExperimentalCoroutinesApi
    public fun advanceTimeBy(delayTimeMillis: Long) {
        require(delayTimeMillis >= 0) { "Can not advance time by a negative delay: $delayTimeMillis" }
        val startingTime = currentTime
        val targetTime = addClamping(startingTime, delayTimeMillis)
        while (true) {
            val event = synchronized(lock) {
                val timeMark = currentTime
                val event = events.removeFirstIf { targetTime > it.time }
                when {
                    event == null -> {
                        currentTime = targetTime
                        return
                    }
                    timeMark > event.time -> currentTimeAheadOfEvents()
                    else -> {
                        currentTime = event.time
                        event
                    }
                }
            }
            event.dispatcher.processEvent(event.time, event.marker)
        }
    }
```



---

### RunTest

TestScope에서 코루틴을 시작하고, 유후상태가 될 때까지 시간을 흐르게 합니다.

그래서, 경과시간 파악할때 currentTime으로 바로 확인 가능합니다.





```kotlin
@ExperimentalCoroutinesApi
public fun runTest(
    context: CoroutineContext = EmptyCoroutineContext,
    dispatchTimeoutMs: Long = DEFAULT_DISPATCH_TIMEOUT_MS,
    testBody: suspend TestScope.() -> Unit
): TestResult {
    if (context[RunningInRunTest] != null)
        throw IllegalStateException("Calls to `runTest` can't be nested. Please read the docs on `TestResult` for details.")
    return TestScope(context + RunningInRunTest).runTest(dispatchTimeoutMs, testBody)
}
```



```kotlin
/**
 * The current virtual time on [testScheduler][TestScope.testScheduler].
 * @see TestCoroutineScheduler.currentTime
 */
@ExperimentalCoroutinesApi
public val TestScope.currentTime: Long
    get() = testScheduler.currentTime

```



```kotlin
    /** The current virtual time in milliseconds. */
    @ExperimentalCoroutinesApi
    public var currentTime: Long = 0
        get() = synchronized(lock) { field }
        private set

    /** A channel for notifying about the fact that a dispatch recently happened. */
    private val dispatchEvents: Channel<Unit> = Channel(CONFLATED)

    /**
     * Registers a request for the scheduler to notify [dispatcher] at a virtual moment [timeDeltaMillis] milliseconds
     * later via [TestDispatcher.processEvent], which will be called with the provided [marker] object.
     *
     * Returns the handler which can be used to cancel the registration.
     */
    internal fun <T : Any> registerEvent(
        dispatcher: TestDispatcher,
        timeDeltaMillis: Long,
        marker: T,
        context: CoroutineContext,
        isCancelled: (T) -> Boolean
    ): DisposableHandle {
        require(timeDeltaMillis >= 0) { "Attempted scheduling an event earlier in time (with the time delta $timeDeltaMillis)" }
        checkSchedulerInContext(this, context)
        val count = count.getAndIncrement()
        val isForeground = context[BackgroundWork] === null
        return synchronized(lock) {
            val time = addClamping(currentTime, timeDeltaMillis)
            val event = TestDispatchEvent(dispatcher, count, time, marker as Any, isForeground) { isCancelled(marker) }
            events.addLast(event)
            /** can't be moved above: otherwise, [onDispatchEvent] could consume the token sent here before there's
             * actually anything in the event queue. */
            sendDispatchEvent(context)
            DisposableHandle {
                synchronized(lock) {
                    events.remove(event)
                }
            }
        }
    }
```



```
val time = addClamping(currentTime, timeDeltaMillis)
            val event = TestDispatchEvent(dispatcher, count, time, marker as Any, isForeground)...
```



testCoroutineScheduler에서 코루틴 관련 job들이 실행되면서, 수행시간을 기록하는데, 해당 기록 결과물을 확인할 수 있습니다.



---

### 백그라운드 스코프

runTest는 모든 자식 코루틴이 완료될 떄까지 기다립니다. runcTest가 스코프가 종료될 떄까지 기다리지 않기 위해 backgroudScope를 사용할 수 있습니다. (부모 자식간의 종속관계를 가지지 않음)



```kotlin
@Test
fun `should increment counter`() = runTest { var i = 0
backgroundScope.launch { while (true) {
delay(1000)
i++ }
}
    delay(1001)
    assertEquals(1, i)
    delay(1000)
    assertEquals(2, i)
}
```



**TestScope의 변수**

```kotlin
    override val backgroundScope: CoroutineScope =
        CoroutineScope(coroutineContext + BackgroundWork + ReportingSupervisorJob {
            if (it !is CancellationException) reportException(it)
        })

```





```
백그라운드 작업의 범위입니다.

이 범위는 테스트가 완료되면 자동으로 취소됩니다. 이 범위의 코루틴은 advanceTimeBy 및 runCurrent 를 사용할 때 평소와 같이 실행됩니다 . 반면에 advanceUntilIdle 은 이 범위의 코루틴만 처리되지 않은 채로 남겨지면 가상 시간의 진행을 중지합니다.
이 범위의 코루틴에 오류가 발생해도 테스트가 종료되지 않습니다. 대신, 테스트가 끝나면 보고됩니다. 마찬가지로, TestScope 자체 의 실패는 backgroundScope 사이에 부모-자식 관계가 없기 때문에 영향을 미치지 않습니다 .
이 범위의 일반적인 사용 사례는 프로덕션 환경에서 테스트된 코드보다 오래 지속되는 작업을 시작하는 것입니다.
이 예에서는 새 요소를 채널에 지속적으로 보내는 코루틴이 취소됩니다.
```



---

### 취소와 컨텍스트 전달 테스트하기

동시성 이슈가 발생할 수 있는 로직에 대한 값 테스트를 진행할 떄, [mapAsync](https://github.com/MarcinMoskala/kotlin-coroutines-recipes/blob/master/src/commonMain/kotlin/mapAsync.kt)를 사용합니다.

```kotlin
suspend fun <T, R> Iterable<T>.mapAsync( transformation: suspend (T) -> R
): List<R> = coroutineScope {
this@mapAsync.map { async { transformation(it) } }
.awaitAll()
}
```

순서를 보장하면서, 비동기적으로 원소를 매핑합니다. (그런데, 찾아보니, 해당 function은 kotlin 공식에는 없는 것 같아요.)



```kotlin
@Test
fun `should map async and keep elements order`() = runTest { val transforms = listOf(
        suspend { delay(3000); "A" },
        suspend { delay(2000); "B" },
        suspend { delay(4000); "C" },
        suspend { delay(1000); "D" },
)
val res = transforms.mapAsync { it() } assertEquals(listOf("A", "B", "C", "D"), res) assertEquals(4000, currentTime)
}
```

async 확인이 가능합니다.



중단 함수의 컨텍스트를 확인할 때는 currentCoroutineContext 또는 coroutineContext의 property를 까서 확인합니다.

```kotlin
@Test
fun `should support context propagation`() = runTest { var ctx: CoroutineContext? = null
val name1 = CoroutineName("Name 1") withContext(name1) {
        listOf("A").mapAsync {
            ctx = currentCoroutineContext()
            it
}
        assertEquals(name1, ctx?.get(CoroutineName))
    }
val name2 = CoroutineName("Some name 2") withContext(name2) {
        listOf(1, 2, 3).mapAsync {
            ctx = currentCoroutineContext()
            it
}
        assertEquals(name2, ctx?.get(CoroutineName))
    }
}

```



가장 많이 사용하는 방법은 내부 함수를 만들고, 부모가 내부함수를 취소하여, 취소된 job의 reason을 파악하는 방법입니다.

```kotlin
@Test
fun `should support cancellation`() = runTest { var job: Job? = null
val parentJob = launch { listOf("A").mapAsync {
            job = currentCoroutineContext().job
delay(Long.MAX_VALUE) }
}

delay(1000)
parentJob.cancel() assertEquals(true, job?.isCancelled)
}
```



---

### UnconfinedTestDispatcher

```kotlin
fun main() { CoroutineScope(StandardTestDispatcher()).launch {
        print("A")
        delay(1)
        print("B")
    }
    CoroutineScope(UnconfinedTestDispatcher()).launch {
} }
// C
print("C")
delay(1)
print("D")
```



StandardTestDispatcher는 스케줄러 사용전까지 연산 수행을 진행하지 않는데,

UnconfinedTestDispatcher는 첫번째 지연이 일어나기 전까지 연산을 즉시 수행합니다.

---

### 디스패처를 바꾸는 함수 테스트하기



디스패처가 바뀌어 호출했는지, 확인할 때, Thread의 CurrentName을 통해 확인 가능합니다.

```kotlin
@Test
fun `should change dispatcher`() = runBlocking { // given
val csvReader = mockk<CsvReader>() val startThreadName = "MyName"
var usedThreadName: String? = null every {
            csvReader.readCsvBlocking(
                aFileName,
                GameState::class.java
            )
        } coAnswers {
            usedThreadName = Thread.currentThread().name
            aGameState
}
val saveReader = SaveReader(csvReader)
// when
        withContext(newSingleThreadContext(startThreadName)) {
            saveReader.readSave(aFileName)
}
// then
        assertNotNull(usedThreadName)
val expectedPrefix = "DefaultDispatcher-worker-"
        assert(usedThreadName!!.startsWith(expectedPrefix))
    }

```





디스패처를 바꾸는 함수에서 시간 관련 테스가 진행되는 경우에는, 디스패처를 별도로 주입하여, 갈아끼우며 테스트를 진행하면 됩니다.

```kotlin
class FetchUserUseCase(
private val userRepo: UserDataRepository, private val ioDispatcher: CoroutineDispatcher =
){
Dispatchers.IO
suspend fun fetchUserData() = withContext(ioDispatcher) { val name = async { userRepo.getName() }
val friends = async { userRepo.getFriends() }
val profile = async { userRepo.getProfile() }
    User(
        name = name.await(),
        friends = friends.await(),
        profile = profile.await()
) }
}

```



---

### 함수 실행 중에 일어나는 일 테스트



```kotlin
@Test
fun `should show progress bar when sending data`() = runTest { // given
val database = FakeDatabase() val vm = UserViewModel(database)
// when
    launch {
        vm.sendUserData()
}
// then
assertEquals(false, vm.progressBarVisible.value) // when
    advanceTimeBy(1000)
    // then
assertEquals(false, vm.progressBarVisible.value) // when
    runCurrent()
// then
assertEquals(true, vm.progressBarVisible.value) // when
    advanceUntilIdle()
// then
assertEquals(false, vm.progressBarVisible.value) }
```

(요기 제대로 이해가..)



---



### 새로운 코루틴을 시작하는 함수 테스트하기

```kotlin
@Test
fun testSendNotifications() { // given
val notifications = List(100) { Notification(it) } val repo = FakeNotificationsRepository(
        delayMillis = 200,
        notifications = notifications,
    )
val service = FakeNotificationsService( delayMillis = 300,
)
val testScope = TestScope()
val sender = NotificationsSender(
        notificationsRepository = repo,
        notificationsService = service,
        notificationsScope = testScope
)
// when
    sender.sendNotifications()
    testScope.advanceUntilIdle()
    // then all notifications are sent and marked
    assertEquals(
        notifications.toSet(),
        service.notificationsSent.toSet()
    )
    assertEquals(
        notifications.map { it.id }.toSet(),
        repo.notificationsMarkedAsSent.toSet()
    )

    // and notifications are sent concurrently
    assertEquals(700, testScope.currentTime)
}

```

launch된 코루틴의 실제 로직에 어느정도의 delay를 주어, 시간차를 기반으로 새로운 코루틴에서 로직들이 수행되는 것을 확인할 수 있습니다.



---

### 메인 디스패처 교체하기

단위 테스트에서는 main function이 없기 때문에 Dispatcher.Main을 쓸 수 없는데,

이를 kotlin-test에서 setMain을 통해 만들어 쓸 수 있습니다.



```kotlin
/**
 * Sets the given [dispatcher] as an underlying dispatcher of [Dispatchers.Main].
 * All subsequent usages of [Dispatchers.Main] will use the given [dispatcher] under the hood.
 *
 * Using [TestDispatcher] as an argument has special behavior: subsequently-called [runTest], as well as
 * [TestScope] and test dispatcher constructors, will use the [TestCoroutineScheduler] of the provided dispatcher.
 *
 * It is unsafe to call this method if alive coroutines launched in [Dispatchers.Main] exist.
 */
@ExperimentalCoroutinesApi
public fun Dispatchers.setMain(dispatcher: CoroutineDispatcher) {
    require(dispatcher !is TestMainDispatcher) { "Dispatchers.setMain(Dispatchers.Main) is prohibited, probably Dispatchers.resetMain() should be used instead" }
    getTestMainDispatcher().setDispatcher(dispatcher)
}
```



---

### 룰이 있는 테스트 디스패처 설정하기

[Junit Rule](https://junit.org/junit4/javadoc/4.12/org/junit/ClassRule.html)을 통해,클래스의 수명 동안 반드시 실행되어야 할 로직을 포함할 수 있습니다.



```kotlin
class MainCoroutineRule : TestWatcher() {
lateinit var scheduler: TestCoroutineScheduler
private set
lateinit var dispatcher: TestDispatcher
private set
override fun starting(description: Description) { scheduler = TestCoroutineScheduler() dispatcher = StandardTestDispatcher(scheduler) Dispatchers.setMain(dispatcher)
}
override fun finished(description: Description) { Dispatchers.resetMain()
} }
```

어떻게 보면, @Before, @After와 같은 로직들과 비슷하다고 생각됩니다.
