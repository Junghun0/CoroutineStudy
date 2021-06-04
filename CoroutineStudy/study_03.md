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





