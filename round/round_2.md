# 2회차

- 코틀린 기본
  - 클래스, 상속, 인터페이스
  - Aggregation, Composition,
  - Dependency Injection



>  들어가기에 앞서, 코틀에서는 바에서 겪었던 불편함?을 최대한 줄일려고 했다고 생각합니다.

>  더불어, 불확실성에 대한 안정성을 고민, 고려했다고 생각합니다.



<br>

<br>

### Interface

코틀린에서는 다음과 같이 인터페이스를 정의한다.

```kotlin
interface JavaCafe {
		fun study()
}
```

<br>



상속의 경우는 다음과 같이 진행한다.

```kotlin
class CoroutineStudy : JavaCafe {
		override fun study = "coroutine study"
}
```

<br>

자바와 다른 점은, extends나 implements가 없다는 것이다.

상속이 진행된 경우, override메서드를 꼭 선언해야 한다. (이 점은 자바와 다릅니다.)

<br>

<br>

### override 선언을 강제하는 이유는 무엇 때문일까?

제가 생각하기에는, 코틀린의 설계 원칙에서 나왔다고 생각한다.

코틀린은 안정성과 가독성 그리고 모호함을 줄이기 위해 많은 노력을 했고, 그것 중 하나가 override를 강제하는 것이라고 생각한다.

ex) 프로퍼티에 대해 명시적으로 어떤 값인지 표현 가능

```kotlin
Goofy(
	name = "김동건"
)
```

<br>

<br>

### Default Method

자바와 마찬가지로 코틀린도 디폴트 메서드를 사용할 수 있다.

단, default 를 붙이지 않는다.

<br>

```kotlin
interface CoroutineStudy {
    fun study() = "coroutine study"
}
```



<br><br>

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



<br>

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



<br><br>

### 코틀린의 변경자

코틀린은 모든 클래스가 기본적으로 final로 만들어진다. (상속 불가..)

만약, 상속을 받고 싶다라고 하면, class 앞에 open을 붙여, 명시적으로 상속 가능임을 알려줘야 한다.



<br><br>

### 코틀린이 왜 기본적으로 Final로 클래스를 만들까?

처음 class를 만든 개발자의 의도와 다르게 기반 클래스를 상속 및 변경하여 사용함으로서, 의도하지 않은 버그와 이슈들이 많이 발생했다.

코틀린은 이런 예기치 않은 문제들에 대한 해결책으로 기본적으로 final로 class를 생성한다. (Method도 동일)



<br>

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





<br><br>

### 회사에서 상속을 통해 구현하다가, 문제가 발생한 적이 있는가?

공통 모듈을 커스텀하게 사용하고 싶었다. 그래서 해당 코드를 상속하고, 내 입맛에 맞추어 변경하였다.

잘되는 것 같았다. 디비에 데이터가 이상하게 들어가는 것을 파악하기 전까지는..

분명히 테스트 코드도 잘 작성했다.. 분명히 잘되어야 하는데...

기존 코드를 더 확인해보았는데, 내가 하려했던 작업에 대해서는 지원하지 않는 것을 파악했다...



<br><br>

### 추상 클래스

자바와 마찬가지로 코틀린도 추상 클래스를 선언할 때, abstract 키워드를 사용한다.

```kotlin
abstract class JavaCafe {
		abstract fun study()
}
```

자바와 마찬가지로, 코틀린의 추상 클래스도 인스턴스화 불가능하다. 추상화를 통해 구현체를 만들어야 한다.



<br><br>

### 가시성 변경자

기본적으로는 코틀린의 가시성 변경자는 자바와 거의 비슷하다.

조금의 차이점은, 코틀린에서 패키지 단위의 가시성 변경자는 없다. (패키지는 단지 네임스테이스를 관리하기 위한 용도라고 한다.)

그리고 public이 기본적인 가시성 변경자이다.



<br><br>

### 모듈 단위의 가시성 변경자

패키지 단위의 접근 제한자가 없다. 대신, 모듈 단위의 접근 제한자가 있다.

여기서 모듈은 한 번에 컴파일되는 코틀린 파일들의 집합을 의미한다. (프로젝트가 모듈이 될 수 있음)





<br><br>

### 가시성 변경자와 접근 제한자

자바에서는 public, protected, private 등을 접근 제한자라고 표현한다.

그런데, 코틀린에서는 가시성 변경자라고 표현한다. 



그럼 왜? 코틀린에서는 가시성 변경자라고 표현할까? 그리고 이 둘의 차이는??





<br>

**접근 제한자**

>클래스 멤버(클래스 내부의 변수, 함수 등)가 다른 클래스에서 얼마나 "접근"할 수 있는지를 결정하는 키워드를 의미,
>
>다른 클래스에서 멤버를 볼 수 있는지 여부를 결정



<br>

**가시성 변경자**

> 클래스 정의 블록 내부의 코드 블록에서 어떻게 클래스 멤버를 "볼 수 있는지" 결정
>
> 클래스 내부에서 사용되며 클래스 외부에서 멤버를 참조하는 경우에는 접근 제한자가 적용



<br><br>

### 중첩 클래스

자바와 가장 큰 차이점은 코틀린의 중첨 클래스는 명시적으로 요청이 없다면, 바깥쪽 클래스 인스턴스에 대한 접근 권한이 없다.

기본적으로 코틀린의 중첩 클래스는 자바의 static 중첩 클래스와 동일하다. 그렇기 때문에, 바깥 클래스에 대한 참조가 없다.

만약, 바깥 클래스에 대한 참조를 하고 싶다면, Inner를 붙여야 한다!



<br>

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



<br>

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



<br><br>

### Sealed Class

제가 생각하기에 sealed class가 kotlin에서 추가된 가장 큰 이유는, **상속을 제한할 수 있다** 입니다. 

(정확하게는, 상속을 받은 클래스에 대해서 제한할 수 있다.)



<br>

일반적으로, 특정 class 상속하고, 인스턴스화 하여 비교하는 로직을 구성한다면, 다음과 같이 작성할 수 있다.





<br>

```kotlin
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



<br>

```kotlin
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





<br><br>



반면에, sealed class를 이용하는 경우에는, 다음과 같이 상속을 받는 하위 클래스 들에 대해 제한을 할 수 있습니다.

```kotlin
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



<br><br>

**Sealed Class의 장점들**

1. enum 클래스처럼 사용할 수 있습니다. 위에 제시한 예시처럼, Sealed Class를 이용하게 되면,  enum과 같은 형식으로 코드를 구현할 수 있습니다.
2. 상속과 인스턴스화에 안전합니다. open class와 동일하지만, 상속을 제한할 수 있다는 점에 안전하게 사용 가능합니다.
3. 그 밖에 어떠한 장점들이 있다고 생각하시나요?





<br><br>

### class의 초기화 (주 생성자와 초기화 블록)

클래스 이름 뒤에 나오는 괄호 사이의 선언을 kotlin에서는 **주 생성자** 라고 합니다.

```kotlin
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

```kotlin
open class A
```



<br><br>

**비공개 생성자에 대해..**

한번만 인스턴스화하고 다른 곳에서는 인스턴스화를 못하게 해야 하는 경우가 있는데, 코틀린에서는 다음과 같은 방법으로 비공개 생성자를 만들어 관리할 수 있습니다.

```kotlin
class A private constructor() {}
```



<br><br>



### 클래스를 생성할 때 좋은점

- 자바와 다르게, 코틀린에서 클래스를 새롭게 생성할 때 new 키워드를 쓸 필요가 없습니다.

  - ```kotlin
    val a = A()
    ```

    

- 생성자의 순서를 개발자가 정의 혹은 변경도 가능합니다. 더불어 생성자의 인자에 대한 이름을 지정할 수 있습니다.

  - ```kotlin
    val a = A(name = "김동건")
    ```



<br><br>

### 부 생성자

부 생성자도 주 생성자와 마찬가지로 constructor로 시작합니다. 

```kotlin
class A(
	val name : String
) {
	constructor(human: Human) {
		this.name = human.name
	}
}
```



<br><br>

### 인터페이스에 프로퍼티

코틀린에서는 인터페이스에 프로퍼티를 추가할 수 있다. 단, 해당 프로퍼티는 추상 프로퍼티로서, 해당 값들을 어디선가 초기화할 수 있도록 로직을 구성해야 한다.

```kotlin
interface User {
		val name: String
}

class DongGeon(override val name: String) : User
class Goofy(val request: Request) : User {
		override val name: String
			get() = request.name
}
```

위와 같은 방식으로 인터페이스의 프로퍼티를 초기화할 수 있다.



<br><br>



### ==과 equals에 대해

자바에서 **==** 원시타입과 참조 타입을 비교할 때 사용한다. (동등성)

equals의 경우에는 두 피연산자의 주소가 같은지 비교한다. (참보 비교)



그럼 kotlin은 어떤 식을 사용할까?

기본적으로 ==을 사용한다, 그런데 자바에서처럼 버그가 발생하는 경우는 없다. 왜냐하면, ==의 내부 연산 구조에서 equals가 포함되기 때문이다.



<br><br>

### 데이터 클래스 

equasls, toString, hashcode 등 기본적으로 모든 클래스가 생성해야 하는(혹은 오버라이드를 통해 구현을 변경해줘야 하는) 요소들을 kotlin의 dataclass가 알아서 잘! 만들어준다.

모든 프로퍼티에 대해 적용하여 구현체를 만들기 때문에, 자바에서 있었던 불편함..줄어든다.



**데이터 클래스의 사기적인 copy()**

특정 객체에 대해, 모든 프로퍼티를 바꾸는 것이 아닌, 특정 프로퍼티만 바꿀 수 있도록 코틀린은 지원한다.

여기서 사기인 점은, val 로 선언된 프로퍼티도 변경이 가능하다..



```kotlin
fun a() {
    val a = AB(name = "hello", age = 10)
    val b = a.copy(name = "goofy")
}
```



<br><br>

### by??? 

해당 키워드에 대해 명확히 이해가 되지 않는데,,,.. 도움이 필요합니다..



<br><br>

<br><br>

---

`by` 키워드는 Kotlin에서 델리게이션(delegation) 패턴을 쉽게 구현하기 위한 기능을 제공합니다. 델리게이션 패턴은 객체 지향 프로그래밍에서 하나의 클래스가 다른 클래스의 일부 기능을 위임하거나 재사용하기 위해 사용되는 패턴입니다. `by` 키워드는 이러한 델리게이션을 쉽게 구현할 수 있도록 도와줍니다.

`by` 키워드를 사용하여 델리게이션을 구현할 때, 클래스 A가 다른 클래스 B의 인스턴스를 내부적으로 가지고, 클래스 B의 메서드 호출을 클래스 A에서 호출하도록 위임합니다. 이를 통해 코드 재사용과 모듈화를 강화할 수 있습니다.

예를 들어, 다음은 `by` 키워드를 사용하여 델리게이션을 구현하는 간단한 예제입니다:

```kotlin
interface Printer {
    fun print(text: String)
}

class ConsolePrinter : Printer {
    override fun print(text: String) {
        println("Printing: $text")
    }
}

class Document(private val printer: Printer) : Printer by printer {
    fun generateDocument() {
        print("Document content")
    }
}

fun main() {
    val consolePrinter = ConsolePrinter()
    val document = Document(consolePrinter)
    document.generateDocument()
}
```

이 예제에서 `Document` 클래스는 `Printer` 인터페이스를 구현하는 `ConsolePrinter` 클래스의 인스턴스를 가지고 있습니다. `Document` 클래스는 `Printer by printer` 구문을 사용하여 `print` 메서드를 `printer` 객체에 위임하고, `generateDocument` 메서드를 호출하면 실제로 `printer` 객체에서 `print` 메서드가 실행됩니다.

`by` 키워드를 사용하면 델리게이션을 통해 코드를 재사용하고 클래스 간의 결합도를 낮추는데 도움이 됩니다.



**(chatgpt의 도움)**





<br><br>

<br><br>

----



<br><br>

<br><br>

### object

kotlin에서 object라는 키워드가 있는데, 해당 키워드 블락은 클래스 정의와 생성을 동시에 진행하는 특징이 있다.

주로, 싱글톤과 동반 객체, 무명 내부 클래스 등에서 사용한다.

(저의 경우에는 싱글톤, 혹은 상수를 관리할때도 사용합니다.)



<br><br>

**싱글톤 만들기**

```kotlin
object Goofy {
		fun name() = "goofy"
}

Goofy.name()
```

어떻게 보면, java의 static method와 흡사하다.



<br><br>

**동반객체**

클래스 내부에 정적인 멤버와 함수를 저장하고, 클래스와 관련된 공유 데이터나 동작을 제공하는 용도로 사용되는 것을 동반 객체라고 합니다.

```kotlin
class Goofy {
		companion object {
				private const val name = "GOOFY GOOD!"
		}
}
```

저의 경우에는 주로, 팩토리 메서드를 만들거나, 혹은 상수를 관리할 때 사용합니다.

혹은, 동반객체를 통한 확장함수를 만들어 사용합니다.



<br><br>

### Aggreagate Operations(코틀린의 연산자 집합)

코틀린에서 정말 다양한 연사들을 제공하고, 이를 통해 코드 개발을 진행시 가독성과 라인수를 화끈하게 줄일 수 있습니다.

다양한 메서드가 있지만, 저의 경우에는 max(), min()을 가장 많이 사용합니다. (다들 가장 많이 쓰는 연산자가 있을까요?)

```kotlin
fun a() {
    val max = listOf(1,5,11,2).maxOrNull() // max is 11
    val min = listOf(1,5,11,2).minOrNull() // min is 1
}
```



- [baeldung](https://www.baeldung.com/kotlin/aggregate-operations)
- [kotlin docs](https://kotlinlang.org/docs/collection-aggregate.html)



<br><br>

# 

### Aggregation, Composition



<br><br>

**Aggregation (has-a)**

- Aggregation은 하나의 객체가 다른 객체들을 모아놓은 "집합"을 의미
- Aggregation은 포함된 객체가 독립적으로 존재할 수 있으며, 하나의 객체가 다른 객체를 참조할 뿐
- Composition에 비해서 느슨한 관계이며, 포함된 관계가 독립적 존재 가능
- **객체의 수명주기에는 영향을 주지는 않지만, 이를 관리함으로서 유용한 기능을 제공한다.** 
  - **(내 손가락이 10개여서 타자를 잘침, 8개면 좀 힘들고, 2개면 힘들고, 그런데 지금 독수리 타자인...)**

```kotlin
class Book(val title: String)

class Library(val books: List<Book>) {
    fun listBooks() {
        for (book in books) {
            println(book.title)
        }
    }
}

fun main() {
    val book1 = Book("Book 1")
    val book2 = Book("Book 2")

    val booksInLibrary = listOf(book1, book2)
    val library = Library(booksInLibrary)

    library.listBooks()
}
```

라이브러리는 다수의 Book 객체를 가질 수 있으며, 이에 대한 관리 작업을 수행할 수 있음





<br><br>

**Composition (has-a)**

- Composition은 하나의 객체가 다른 객체를 "합성"하여 더 높은 수준의 객체
- Composition은 포함된 객체가 더 높은 수준의 객체에 의존하며, 높은 수준의 객체가 생성될 때 포함된 객체를 생성
- **강하게 응집되었으며, 더 높은 수준의 객체에 LifeCycle이 맞춰짐 **
  - **(건물 안에 방이 있음. 근데 건물이 부셔지면, 방도 부셔지는 거임)**

```kotlin
class Engine {
    fun start() {
        println("Engine started")
    }
}

class Car {
    private val engine: Engine = Engine()

    fun startCar() {
        engine.start()
        println("Car started")
    }
}

fun main() {
    val myCar = Car()
    myCar.startCar()
}
```

Engine은 Car에 LifeCycle을 맞춰감..



<br><br>



- [baeldung](https://www.baeldung.com/java-composition-aggregation-association)

---

### Dependency Injection

kotlin 기반의 spring di 방법은 다음과 같다.

**constructor Injection (생성자 주입)**

```kotlin
@Service
class MyService(
  private val myRepository: MyRepository
) {
    // ...
}
```

생성자를 통해 의존성을 주입하는 방법



<br><br>

**property Injection (속성 주입):**

```kotlin
@Service
class MyService {
    @Autowired
    lateinit var myRepository: MyRepository
    // ...
}
```

`@Autowired`나 `@Inject` 어노테이션을 사용하여 의존성을 클래스의 프로퍼티에 주입하는 방법



<br><br>

**Method Injection (메서드 주입)**

```kotlin
@Service
class MyService {
    private lateinit var myRepository: MyRepository

    @Autowired
    fun setMyRepository(myRepository: MyRepository) {
        this.myRepository = myRepository
    }
    // ...
}
```

메서드를 사용하여 의존성을 주입하는 방법



<br><br>



**Constructor-Based DI with `@ConstructorBinding`**

```kotlin
@ConstructorBinding
@ConfigurationProperties("my")
data class MyProperties(
  val property1: String, 
  val property2: String
)
```

spring Boot 2.2 이상에서는 `@ConstructorBinding` 어노테이션을 사용하여 생성자 기반의 DI를 지원



<br>

<br>



- [baeldung code](https://github.com/Baeldung/kotlin-tutorials)

