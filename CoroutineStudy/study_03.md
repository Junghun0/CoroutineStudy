## 코틀린에서 동시성

### 넌 블로킹

스레드는 무겁고 생성하는 데 비용이 많이 들며 제한된 수의 스레드만 생성할 수 있다.


스레드가 블로킹되면 어떻게 보면 자원이 낭비되는 셈이어서 코틀린은 중단 가능한 연산이라는 기능을 제공한다.
스레드의 실행을 블로킹하지 않으면서 실행을 잠시 중단하는 것이다.
예를 들어 스레드 Y에서 작업이 끝나기를 기다리려면 스레드X를 블로킹하는 대신, 대기해야 하는 코드를 일시 중단하고 그동안 스레드 X를 다른 연산 작업에 이용하기도 한다.

### 명시적인 선언

일시 중단 가능한 연산은 기본적으로 순차적으로 실행된다. 연산은 일시 중단될 때 스레드를 블로킹하지 않기 때문에 직접적인 단점은 아니다.

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val time = measureTimeMillis {
        val name = getName()
        val lastName = getLastName()
        println("Hello, $name $lastName")
    }
    println("Execution took $time ms")
}

suspend fun getName(): String {
    delay(1000)
    return "Jung Hoon"
}

suspend fun getLastName(): String {
    delay(1000)
    return "Park"
}
~~~

위의 코드에서 main() 은 현재 스레드에서 일시 중단 가능한 연산 getName() 과 getLastName() 을 순차적으로 실행한다.

실행 스레드를 블록하지 않는 비 동시성 코드를 작성할 수 있어서 편리하다.
하지만 getLastName() 과 getName() 간에 서로 의존성이 없기 때문에, getLastName() 이 getName() 이 실행될 때까지 기다려야 할 필요가 없음을 알게된다.

동시에 수행하는 코드
~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val time = measureTimeMillis {
        val name = async { getName() }
        val lastName = async { getLastName() }
        println("Hello, ${name.await()} ${lastName.await()}")
    }
    println("Execution took $time ms")
}
~~~

async{...} 를 호출해 두 함수를 동시에 실행해야 하며 await() 를 호출해 두 연산에 모두 결과가 나타날 때까지 main() 이 일시 중단되도록 요청한다.


## 코틀린 동시성 관련 개념과 용어

### 일시 중단 연산

일시 중단 연산은 해당 스레드를 차단하지 않고 실행을 일시 중지할 수 있는 연산이다.
 스레드를 차단하는 것은 좀 불편하기 때문에 자체 실행을 일시 중단하면 일시 중단 연산을 통해 스레드를 다시
시작해야 할 때까지 스레드를 다른 연산에서 사용할 수 있다.

### 일시 중단 함수

일시 중단 함수는 함수 형식의 일시 중단 연산이다. `suspend` 키워드 사용

~~~kotlin
suspend fun greetAfter(name: String, delayMillis: Long) {
    delay(delayMillis)
    println("Hello, $name")
}
~~~

위의 예제는 delay() 가 호출될 때 일시 중단된다. delay() 자체가 일시 중단 함수이며, 주어진 시간 동안 실행을 일시 중단한다.
delay() 가 완료되면 greetAfter() 가 실행을 정상적으로 다시 시작한다. greetAfter() 가 일시 중지된 동안 실행 스레드가 다른 연산을 수행하는 데 사용될 수 있다.

### 코루틴 디스패치

코루틴을 시작하거나 재개할 스레드를 결정하기 위해 코루틴 디스패처가 사용된다.
모든 코루틴 디스패처는 CoroutineDispatcher 인터페이스를 구현해야 한다.

- DefaultDispathcer : 현재는 CommonPool과 같다.
- CommonPool : 공유된 백그라운드 스레드 풀에서 코루틴을 실행하고 다시 시작한다. 기본 크기는 CPU 바운드 작업에서 사용하기에 적합하다.
- Unconfined : 현재 스레드 (코루틴이 호출된 스레드) 에서 코루틴을 시작하지만 어떤 스레드에서도 코루틴이 다시 재개될 수 있다.

디스패처와 함께 필요에 따라 풀 또는 스레드를 정의하는 데 사용할 수 있는 몇 가지 빌더가 있다.

- newSingleThreadContext() : 단일 스레드로 디스패처를 생성한다. 여기에서 실행되는 코루틴은 항상 같은 스레드에서 시작되고 재개된다.

- newFixedThreadPoolContext : 지정된 크기의 스레드 풀이 있는 디스패처를 만든다. 런타임은 디스패처에서 실행된 코루틴을 시작하고 재개할 스레드를 결정한다.

### 코루틴 빌더

코루틴 빌더는 -> 일시 중단 (suspend) 람다를 받아 그것을 실행시키는 코루틴을 생성하는 함수다.

- async() : 결과가 예상되는 코루틴을 시작하는데 사용된다. async() 는 코루틴 내부에서 일어나는 모든 예외를 캡쳐해서 결과에 넣기 때문에 조심해서 사용해야 한다.
결과 또는 예외를 포함하는 Deferred<T> 를 반환한다.

- launch() : 결과를 반환하지 않는 코루틴을 시작한다. 자체 혹은 자식 코루틴의 실행을 취소하기 위해 사용할 수 있는 Job을 반환한다.

- runBlocking() : 블로킹 코드를 일시 중지 가능한 코드로 연결하기 위해 작성되었다. 보통 main() 메소드와 유닛 테스트에서 사용된다. runBlocking() 은 코루틴의 실행이
끝날 때까지 현재 스레드를 차단한다.




