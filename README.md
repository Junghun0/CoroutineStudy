# 코루틴 스터디
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
