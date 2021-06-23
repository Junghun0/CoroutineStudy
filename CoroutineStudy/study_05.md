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



