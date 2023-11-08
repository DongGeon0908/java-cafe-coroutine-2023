# 코루틴 빌더



중단 함수는 일반 함수를 호출 가능, but 일반 함수는 중단 함수를 호출할 수 없음

- 중단 함수는 컨티뉴에이션 객체를 다른 중단함수에게 넘기기 때문!
- 모든 중단 함수는, 또 달른 중단 함수에 의해 호출 가능해야 하기 때문!



### launch builder

- CoroutineScope Interface의 extension function입니다.
- 쉽게 생각하면! 영어 그대로 해석하는게 좋음! (Launch! 발사!)
  - 반환값을 받지 않습니다.



### CoroutineScope

- 부모와 자식 코루틴 사이의 관계 정립



### RunBlocking

- 특정 로직이 빠르게 끝나지 않기 위해 사용합니다.
- 코루틴이 중단된 경우, 시작한 스레드를 중단합니다.
  - 프로그램이 끝나는 걸 방지하기 위한 스레드 블로킹
  - 유닛테스트 (테스트 코드에서 많이 사용)
- 실제 서비스 로직에서는 RunBlocking을 사용하지 않고, 테스트 코드에서 특수한 상황에서 많이 사용합니다.



### async Builder

- launch와 마찬가지로, CoroutineScope Interface의 extension function입니다.

- 값을 반환하는 builder입니다.
- 반환값을 Deferred라는 값에 보관하는데, await을 통해 가져올 수 있습니다.

- 여러 데이터를 코루틴으로 돌리고, 이를 한번에 반환하여 재조립하는 상황에서 많이 사용합니다.



### 구조화된 동시성

- 부모는 자식들을 위한 스코프를 제공, 자식들은 해당 스코프 내에서 비즈니스 로직을 호출
- 자식은 부모로부터 컨텍스트를 상속받습니다! (종속되어있음)
- 부모는 모든 자식이 작업을 마칠때까지 기다림!
- 부모 코루틴이 죽으면, 자식 코루틴도 죽음 (메인 스레드가 죽으면, 하위 스레드가 죽는 것과 같음)
- 자식 코루틴에서 발생하면, 부모 코루틴으로 에러가 전파..
  - 물론, 부모와 자식을 분리 시키는 방법이 있습니다!



### TEST-1

```kotlin
fun test1(){
	runBlocking {
		GlobalScope.async {
			(0..100000000).foreach { println(it) }
			fun test2()
		}
	}
}

suspend fun test2() {
		throw RuntimeException()
}
```

해당 상황에서, test2는 test1의 코루틴에 종속된 부모-자식 관계이므로, 메인 코루틴으로 에러가 전파



### TEST-2

```kotlin
suspend fun test1() {
  // logic 1
	(0..100000000).foreach { println(it) }		
  
  // logic 2
  CoroutineScope(Dispatchers.IO).launch {
 		throw RuntimeException()   
  }
}
```

해당 상황에서, logic 2는 부모와 독립된 다른 컨텍스트이므로, logic1은 계속 수행

- 에러 전파 X

### Reference

- [Composing suspending functions](https://kotlinlang.org/docs/composing-suspending-functions.html)
- [Coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)
