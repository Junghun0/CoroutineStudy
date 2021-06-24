# 라이프 사이클과 에러 핸들링

- Job 과 그 사용사례
- Job 과 Deferred 의 라이프 사이클
- Deferred 사용사례
- Job 의 각 상태별 예상되는 사항
- Job 의 현재 상태를 산출하는 방법
- 예외 처리 방법

## Job 과 Deferred

비동기 함수를 다음과 같이 두 그룹으로 나눠 볼 수 있다.

- **결과가 없는 비동기 함수** : 일반적인 시나리오로는 로그에 기록하고 분석 데이터를 전송하는 것과 같은 `백그라운드 작업`을 들 수 있다. 완료 여부를 모니터링할 수 있지만 결과를 갖지 않는 백그라운드 작업이 이런 유형에 속한다.
- **결과를 반환하는 비동기 함수** : 예를 들어 비동기 함수가 웹 서비스에서 정보를 가져올 때 거의 대부분 해당 함수를 사용해 정보를 반환하고자 할 것이다.

두 가지 중 어떤 경우이건 해당 작업에 접근하고 예외가 발생하면 그에 대응하거나 해당 작업이 더 이상 필요하지 않을 때는 취소한다.
두 가지 유형을 어떻게 생성하고 상호 작용할 수 있는지 알아보자!

## Job

Job 은 fire and forget 작업이다.

한 번 시작된 작업은 예외가 발생하지 않는 한 대기하지 않는다. 다음과 같이 코루틴 빌더인 `launch()`를 사용해 잡을 생성하는 방법이 가장 일반적이다.

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val job = GlobalScope.launch {
        // Do background task here
    }
}
~~~

다음과 같이 `Job()` 팩토리 함수를 사용할 수도 있다.

~~~kotlin
fun main(args: Arrays<String>) = runBlocking {
    val job = Job()
}
~~~

>Job 은 인터페이스로, launch() 와 Job()은 모두 JobSupport의 구현체를 반환한다.

## 예외 처리

기본적으로 `Job` 내부에서 발생하는 예외는 잡을 생성한 곳까지 전파된다.
`Job`이 완료되기를 기다리지 않아도 발생한다.

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    GlobalScope.launch {
        TODO("Not Implemented!")
    }
    delay(500)
}
~~~

이렇게 하면 현재 스레드의 포착되지 않은 예외 처리기(`Uncaught Exception Handler`)에 예외가 전파된다.
JVM 애플리케이션이라면 표준 오류 출력에 예외가 출력됨을 의미한다.


## 라이프 사이클

![33](https://user-images.githubusercontent.com/30828236/123254104-3553ec80-d529-11eb-801e-6ee53867e4a8.PNG)

다이어그램에는 다섯 가지 상태가 있다.

- `New(생성)` : 존재하지만 아직 실행되지 않는 잡
- `Active(활성)` : 실행 중인 잡, 일시 중단된 잡도 활성으로 간주된다.
- `Completed(완료됨)` : 잡이 더 이상 실행되지 않는 경우
- `Cancelling(취소중)` : 실행 중인 잡에서 cancel() 이 호출되면 취소가 완료될 때까지 시간이 걸리기도 한다. 이것은 활성과 취소 사이의 중간 상태다.
- `Cancelled(취소됨)` : 취소로 인해 실행이 완료된 잡, 취소된 잡도 완료로 간주될 수 있다.

## 생성

잡은 기본적으로 `launch()` 나 `Job()`을 사용해 생성될 때 자동으로 시작된다. 잡을 생성할 때 자동으로 시작되지 않게 하려면 `CoroutineStart.LAZY`를 사용해야 한다.

~~~kotlin
fun main() = runBlocking {
    GlobalScope.launch(start = CoroutineStart.LAZY) {
        TODO("Not implemented yet!")
    }
    delay(500)
}
~~~

코드를 실행하면 오류가 출력되지 않는다. 작업이 생성됐지만 시작된 적이 없으므로 예외가 발생하지 않는다.

## 활성

`start()` - 잡이 완료될 때까지 기다리지 않고 잡을 시작한다.
`join()` - 잡이 완료될 때까지 실행을 일시 중단한다.

~~~kotlin
fun main() {
    val job = GlobalScope.launch(start = CoroutineStart.LAZY) {
        delay(3500)
    }
    job.start()
}
~~~

~~~kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch(start = CoroutineStart.LAZY) {
        delay(3500)
    }
    job.join()
}
~~~

## 취소 중

취소 요청을 받은 활성 잡은 취소 중(Canceling) 이라고 하는 스테이징 상태로 들어갈 수 있다.
잡에 실행을 취소하도록 요청하려면 `cancel()` 함수를 호출해야 한다.

~~~kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch {
        delay(5000)
    }
    delay(2000)
    job.cancel()
}
~~~

잡은 2초 후에 취소된다. cancel()에는 선택적 매개변수인 `cause`가 있다.
예외가 취소의 원인일 때는 원인을 같이 제공해 주면 나중에 찾아볼 수 있다.

~~~kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch {
        delay(5000)
    }
    delay(2000)
    job.cancel(cause = Exception("Timeout!"))
}
~~~

## 취소 됨

취소 또는 처리되지 않은 예외로 인해 실행이 종료된 잡은 취소됨(cancelled)으로 간주된다.
잡이 취소되면, `getCancellationException()` 함수를 통해 취소에 대한 정보를 얻을 수 있다.

~~~kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch {
        delay(5000)
    }
    delay(2000)
    job.cancel(cause = CancellationException("Tired of waiting!"))

    val cancellation = job.getCancellationException()
    printf(cancellation.message)
}
~~~


CoroutineException Handler 를 설정해 취소 작업을 예외처리 할 수 있다.

~~~kotlin
fun main() = runBlocking {
    val exceptionHandler = CoroutineExceptionHandler {
        _: CoroutineContext, throwable: Throwable ->
        println("Job cancelled due to ${throwable.message}")
    }

    GlobalScope.launch(exceptionHandler) {
        TODO("Not implemented yet!")
    }

    delay(2000)
}
~~~

## 완료 됨

실행이 중지된 잡은 완료됨(completed)으로 간주된다.

이는 실행이 정상적으로 종료됐거나 취소됐는지 또는 예외 때문에 종료됐는지 여부에 관계없이 적용된다.

### 잡의 현재 상태 확인
- `isActive` : 잡이 활성 상태인지 여부, 잡이 일시 중지인 경우도 true 를 반환한다.
- `isCompleted` : 잡이 실행을 완료했는지 여부
- `isCancelled` : 잡 취소 여부, 취소가 요청되면 즉시 true 가 된다.



## 디퍼드

디퍼드(Deferred, 자연) 은 결과를 갖는 비동기 작업을 수행하기 위해 잡을 확장한다.

다른 언어에서 `Future`, `Promise`라고 하는 것의 코틀린 구현체가 바로 디퍼드 (`Deferred`) 이다.

기본적인 콘셉트는 연산이 객체를 반환할 것이며, 객체는 비동기 작업이 완료될 때까지 비어 있다는 것이다.

~~~kotlin
fun main() = runBlocking {
    val headlinesTask = GlobalScope.async {
        getHeadlines()
    }

    healinesTask.await()
}
~~~

`async 를 사용하여 deferred 를 사용할 수 있다.` 또는 `CompletableDeferred`의 생성자를 사용할 수 있다.


## 상태는 한 방향으로만 이동

일단 잡이 특정 상태에 도달하면 이전 상태로 되돌아가지 않는다. 

~~~kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        val job = GlobalScope.luanch {
            delay(2000)
        }

        //Wait for it to complete one
        job.join()

        //Restart the job
        job.start()
        job.join()
    }
    println("Took $time ms")
}
~~~

코드는 2초 동안 실행을 일시 중단하는 잡을 만든다.
처음 호출한 `job.join()`이 완료되면 잡을 다시 시작하기 위해 `start()` 함수가 호출되고,
두 번째 `join()`을 호출해서 두 번째 실행이 끝날 때까지 대기한다.

~~~kotlin
Took 2021 ms
~~~

총 실행에는 약 2초가 걸렸으므로 잡이 한 번만 실행 됐음을 보여준다.
완료된 잡에서 `start()`를 호출해 다시 시작했다면 총 실행 시간은 약 4초가 될 것이다.

잡은 완료됨(completed) 상태에 도달했으므로 `start()`를 호출해도 아무런 변화가 없다.

> 완료됨 잡에서 join()을 호출해도 아무 일도 일어나지 않는다. 작업이 이미 완료됐으므로 일시 중단이 일어나지 않는다.

