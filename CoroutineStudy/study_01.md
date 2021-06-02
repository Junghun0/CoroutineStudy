## 코루틴

코틀린 문서에서는 코루틴을 '경량 스레드' 라고도 한다.
코루틴은 스레드 안에서 실행된다. 스레드 하나에 많은 코루틴이 있을 수 있지만 주어진 시간에 하나의 스레드에서 하나의 명령만이 실행될 수 있다.
즉 같은 스레드에 10개의 코루틴이 있다면 해당 시점에는 하나의 코루틴만 실행된다.

스레드와 코루틴의 가장 큰 차이점은 코루틴이 빠르고 적은 비용으로 생성할 수 있다는 것이다.
수천 개의 스레드를 생성하는 것보다 빠르고 자원도 훨씬 적게 사용한다.

~~~kotlin
suspend fun createCoroutines(amount: Int) {
    val jobs = ArrayList<Job>()
    for(i in 1..amount) {
        jobs += luanch {
            delay(1000)
        }
    }
    jobs.forEach {
        it.join()
    }
}
~~~

함수는 파라미터 amount 에 지정된 수만큼 코루틴을 생성해 각 코루틴을 1초 간 지연시킨 후 모든 코루틴이 종료될 때까지 기다렸다가 반환한다.
예를 들어 10,000으로 설정해서 호출해보자.

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val time = measureTimeMillis {
        createCoroutines(10_000)
    }
    println("Took $time ms")
}
~~~

코틀린은 고정된 크기의 스레드 풀을 사용하고 코루틴을 스레드들에 배포하기 때문에 실행 시간이 매우 적게 증가한다. 따라서 수 천개의 코루틴을 추가하는 것은 거의 영향이 없다.
코루틴이 일시 중단되는 동안 (예제의 경우 delay()를 호출) 실행 중인 스레드는 다른 코루틴을 실행하는 데 사용되며 코루틴은 시작 또는 재개될 준비 상태가 된다.

- 코루틴이 특정 스레드 안에서 실행되더라도 스레드와 묶이지 않는다
- 코루틴의 일부를 특정 스레드에서 실행하고, 실행을 중지한 다음 나중에 다른 스레드에서 계속 실행하는 것이 가능
- 위의 예제에서 코틀린이 실행 가능한 스레드로 코루틴을 이동시키기 때문

~~~kotlin
suspend fun createCoroutines(amount: Int) {
    val jobs = ArrayList<Job>()
    for(i in 1..amount) {
        jobs += luanch {
            println("Start $i in ${Thread.currentThread().name}")
            delay(1000)
            println("Finished $i in ${Thread.currentThread().name}")
        }
    }
    jobs.forEach {
        it.join()
    }
}
~~~

스레드는 한 번에 하나의 코루틴만 실행할 수 있기 때문에 프레임워크가 필요에 따라 코루틴을 스레드들 사이에 옮기는 역할을 한다.

**정리**
Application 이 하나 이상의 프로세스로 구성돼 있고 각 프로세스가 하나 이상의 스레드를 갖고 있음
스레드를 블록한다는 것은 그 스레드에서 코드의 실행을 중지한다는 의미인데, 사용자와 상호작용하는 스레드는 블록되지 않아야 한다.
코루틴이 기본적으로 스레드 안에 존재하지만 스레드에 얽매이지 않은 가벼운 스레드 라는 것을 알게 되었다.
각각의 코루틴이 특정 스레드에서 시작되지만 어느 시점이 지나 다른 스레드에서 다시 시작되는 점을 확인할 수 있다.
