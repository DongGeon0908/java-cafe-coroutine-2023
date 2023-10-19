# 2회차

- 코틀린 기본
  - 클래스, 상속, 인터페이스
  - Aggregation, Composition,
  - Dependency Injection



### Interface

코틀린에서는 다음과 같이 인터페이스를 정의한다.

```kotlin
interface JavaCafe {
		fun study()
}
```



상속의 경우는 다음과 같이 진행한다.

```kotlin
class CoroutineStudy : JavaCafe {
		override fun study = "coroutine study"
}
```



자바와 다른 점은, extends나 implements가 없다는 것이다.

상속이 진행된 경우, override메서드를 꼭 선언해야 한다. (이 점은 자바와 다릅니다.)



### override 선언을 강제하는 이유는 무엇 때문일까?

제가 생각하기에는, 코틀린의 설계 원칙에서 나왔다고 생각한다.

코틀린은 안정성과 가독성 그리고 모호함을 줄이기 위해 많은 노력을 했고, 그것 중 하나가 override를 강제하는 것이라고 생각한다.

ex) 프로퍼티에 대해 명시적으로 어떤 값인지 표현 가능

```kotlin
Goofy(
	name = "김동건"
)
```



### Default Method

자바와 마찬가지로 코틀린도 디폴트 메서드를 사용할 수 있다.

단, default 를 붙이지 않는다.



```kotlin
interface CoroutineStudy {
    fun study() = "coroutine study"
}
```



### Default Method에 대한 구현 강제

```kotlin
class JavaCafe : CoroutineStudy, WebfluxStudy {
    override fun study(name: String): String {
        return name
    }

}

interface CoroutineStudy {
    fun study()
    fun study(name: String) = "hellllo"
}

interface WebfluxStudy {
    fun study(name: String) = "hellllo"
}
```

다음과 같은 형태에서, 코틀린 컴파일러는 하위 클래스 (JavaCafe)에서 default method에 대한 구현을 강제한다.

**(구현을 진행하지 않은 경우, 컴파일러 오류가 발생)**



```kotlin
class JavaCafe : CoroutineStudy, WebfluxStudy {
    override fun study() {
				super<CoroutineStudy>.study()
				super<WebfluxStudy>.study()
    }
}

interface CoroutineStudy {
    fun study() = println("coroutine study")
}

interface WebfluxStudy {
    fun study() = println("webflux study")
}
```

자바와 다르게, kotlin에서는 super<Type>.signature() 구조로 상위 타입 구현을 진행합니다. (java는 Type.super.signature())



### 코틀린의 변경자

코틀린은 모든 클래스가 기본적으로 final로 만들어진다. (상속 불가..)

만약, 상속을 받고 싶다라고 하면, class 앞에 open을 붙여, 명시적으로 상속 가능임을 알려줘야 한다.



### 코틀린이 왜 기본적으로 Final로 클래스를 만들까?

처음 class를 만든 개발자의 의도와 다르게 기반 클래스를 상속 및 변경하여 사용함으로서, 의도하지 않은 버그와 이슈들이 많이 발생했다.

코틀린은 이런 예기치 않은 문제들에 대한 해결책으로 기본적으로 final로 class를 생성한다. (Method도 동일)



```kotlin
open class CoroutineStudy : JavaCafe {
	fun study1()
	open fun study2()
	override fun study3()
	final	override fun study4()
}
```

- study1은 final method여서 메소드 오버라이드 불가능
- study2는 메소드 오버라이드 가능
- study3와 같이 override method는 메소드 오버라이드 가능
- study4는 override 가 붙여있지만, final 키워드로 인해 메소드 오버라이드 불가능





### 회사에서 상속을 통해 구현하다가, 문제가 발생한 적이 있는가?

공통 모듈을 커스텀하게 사용하고 싶었다. 그래서 해당 코드를 상속하고, 내 입맛에 맞추어 변경하였다.

잘되는 것 같았다. 디비에 데이터가 이상하게 들어가는 것을 파악하기 전까지는..

분명히 테스트 코드도 잘 작성했다.. 분명히 잘되어야 하는데...

기존 코드를 더 확인해보았는데, 내가 하려했던 작업에 대해서는 지원하지 않는 것을 파악했다...



### 추상 클래스

자바와 마찬가지로 코틀린도 추상 클래스를 선언할 때, abstract 키워드를 사용한다.

```kotlin
abstract class JavaCafe {
		abstract fun study()
}
```

자바와 마찬가지로, 코틀린의 추상 클래스도 인스턴스화 불가능하다. 추상화를 통해 구현체를 만들어야 한다.



### 가시성 변경자

기본적으로는 코틀린의 가시성 변경자는 자바와 거의 비슷하다.

조금의 차이점은, 코틀린에서 패키지 단위의 가시성 변경자는 없다. (패키지는 단지 네임스테이스를 관리하기 위한 용도라고 한다.)

그리고 public이 기본적인 가시성 변경자이다.



### 모듈 단위의 가시성 변경자

패키지 단위의 접근 제한자가 없다. 대신, 모듈 단위의 접근 제한자가 있다.

여기서 모듈은 한 번에 컴파일되는 코틀린 파일들의 집합을 의미한다. (프로젝트가 모듈이 될 수 있음)



### 가시성 변경자와 접근 제한자

자바에서는 public, protected, private 등을 접근 제한자라고 표현한다.

그런데, 코틀린에서는 가시성 변경자라고 표현한다. 



그럼 왜? 코틀린에서는 가시성 변경자라고 표현할까? 그리고 이 둘의 차이는??



**접근 제한자**

>클래스 멤버(클래스 내부의 변수, 함수 등)가 다른 클래스에서 얼마나 "접근"할 수 있는지를 결정하는 키워드를 의미,
>
>다른 클래스에서 멤버를 볼 수 있는지 여부를 결정



**가시성 변경자**

> 클래스 정의 블록 내부의 코드 블록에서 어떻게 클래스 멤버를 "볼 수 있는지" 결정
>
> 클래스 내부에서 사용되며 클래스 외부에서 멤버를 참조하는 경우에는 접근 제한자가 적용



### 중첩 클래스

자바와 가장 큰 차이점은 코틀린의 중첨 클래스는 명시적으로 요청이 없다면, 바깥쪽 클래스 인스턴스에 대한 접근 권한이 없다.

기본적으로 코틀린의 중첩 클래스는 자바의 static 중첩 클래스와 동일하다. 그렇기 때문에, 바깥 클래스에 대한 참조가 없다.

만약, 바깥 클래스에 대한 참조를 하고 싶다면, Inner를 붙여야 한다!



```kotlin
class A {
  private val name : String

  class B {
    private val name : String
  }
  
  inner class C {
    private val name : String
  }
}
```



추가적으로, 코틀린에서 바깥쪽 클래스를 참조할려면 다음과 같이 진행해야 한다.

```kotlin
class A {
  private val name : String
  
  inner class C {
    private val name : String
    
    fun getA() : A = this@A
  }
}
```



### Sealed Class

제가 생각하기에 sealed class가 kotlin에서 추가된 가장 큰 이유는, **상속을 제한할 수 있다** 입니다. 

(정확하게는, 상속을 받은 클래스에 대해서 제한할 수 있다.)



일반적으로, 특정 class 상속하고, 인스턴스화 하여 비교하는 로직을 구성한다면, 다음과 같이 작성할 수 있다.



```
open class A {
}

class B(
    val name: String
) : A() {

}

class C(
    val name: String
) : A() {

}

fun type(type: A) = when (type) {
    is B -> {

    }

    is C -> {

    }
    else -> {
        
    }
}

```

해당 부분에서, 가장 크게 주의할 점은 else 입니다. 일반적인 추상 클래스의 상속을 기반으로 한 인스턴스 비교에서는



```
fun type(type: A) = when (type) {
    is B -> {

    }

    is C -> {

    }
    else -> {
        
    }
}

```

else 문구가 들어갑니다. 이유는 상위 클래스가 하위의 어떤 클래스를 상속하고 제공하는지에 대한 제한이 없기 때문입니다.

그렇기 때문에, 불필요한 else 문구가 계속 들어가게 됩니다.







반면에, sealed class를 이용하는 경우에는, 다음과 같이 상속을 받는 하위 클래스 들에 대해 제한을 할 수 있습니다.

```
sealed class A {
}

class B(
    val name: String
) : A() {

}

class C(
    val name: String
) : A() {

}

fun type(type : A) = when(type) {
    is B -> {

    }

    is C -> {

    }
}
```



**Sealed Class의 장점들**

1. enum 클래스처럼 사용할 수 있습니다. 위에 제시한 예시처럼, Sealed Class를 이용하게 되면,  enum과 같은 형식으로 코드를 구현할 수 있습니다.
2. 상속과 인스턴스화에 안전합니다. open class와 동일하지만, 상속을 제한할 수 있다는 점에 안전하게 사용 가능합니다.
3. 그 밖에 어떠한 장점들이 있다고 생각하시나요?



### class의 초기화 (주 생성자와 초기화 블록)

클래스 이름 뒤에 나오는 괄호 사이의 선언을 kotlin에서는 **주 생성자** 라고 합니다.

```
class A1(name: String)

class A2 constructor(_name: String) {
    val name: String

    init {
        name = _name
    }
}
```

여기서 constructor는 주, 부 생성자 정의를 시작할 때 사용하며, init은 인스턴스화되면서, 초기화를 진행하는 block입니다.

A2의 주 생성자 표기에서는 _name을 사용했는데, 해당 이유는 생성자 파라미터랑 클래스의 프로퍼티를 구분하기 위함입니다.

주 생성자는 생성자 파라미터를 지정하고, 그 생성자 파라미터에 의해 초기화되는 프로퍼티를 정의합니다.



만약, 생성자를 별도로 지정하지 않았다면, 컴파일러가 자동으로 default constructor를 생성합니다.

```
open class A
```



**비공개 생성자에 대해..**

한번만 인스턴스화하고 다른 곳에서는 인스턴스화를 못하게 해야 하는 경우가 있는데, 코틀린에서는 다음과 같은 방법으로 비공개 생성자를 만들어 관리할 수 있습니다.

```
class A private constructor() {}
```





### 클래스를 생성할 때 좋은점

- 자바와 다르게, 코틀린에서 클래스를 새롭게 생성할 때 new 키워드를 쓸 필요가 없습니다.

  - ```
    val a = A()
    ```

    

- 생성자의 순서를 개발자가 정의 혹은 변경도 가능합니다. 더불어 생성자의 인자에 대한 이름을 지정할 수 있습니다.

  - ```
    val a = A(name = "김동건")
    ```



