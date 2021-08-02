# 일시 중단 함수

launch(), async(), runBlocking()과 같은 코루틴 빌더를 사용해서 일시 중단 알고리즘의 대부분을 작성할 수 있다.

코루틴 빌더를 호출할 때 전달하는 코드는 일시 중단 람다 (`suspending lambda`)이다.
일시 중단 코드를 작성하는 방법을 알아보자.

~~~kotlin
suspend fun greetDelayed(delayMillis: Long) {
    delay(delayMillis)
    println("Hello, World!")
}
~~~

일시 중단 함수를 만들려면 시그니처에 `suspend` 제어자만 추가하면 된다.
일시 중단 함수는 `delay()`와 같은 다른 일시 중단 함수를 직접 호출할 수 있다.
코드를 코루틴 빌더안에 감쌀 필요가 없기 때문에 코드를 명확하고 가독성 있게 만들어준다.

이런 장점이 있지만 코루틴 외부에서 이 함수를 호출하면 동작하지 않는다.

~~~kotlin
fun main(args: Array<String>) {
    greetDelayed(1000)
}
~~~

일시 중단 연산은 다른 일시 중단 연산에서만 호출될 수 있어서 이 코드는 컴파일되지 않는다.

`non-suspending` 코드에서 함수를 호출하려면 다음 예와 같이 코루틴 빌더로 감싸야한다.

~~~kotlin
fun main(args: Array<String>) {
    runBlocking {
        greetDelayed(1000)
    }
}
~~~

## 동작 중인 함수를 일시 중단

앞에서 동시성 코드를 구현할 때 코루틴 빌더 대신 비동기 함수를 사용하는 쪽이 더 편리한지에 대해 설명했다.
이제 일시 중단 함수를 추가해 이 주제를 확장할 차례다.

> 우리는 비동기 함수를 구현한 `Job`을 (`Deferred`를 포함해서) 반환하는 함수라고 했다. 이러한 함수는 보통 `launch()` 또는 `async()` 빌더로 감싸인 함수이지만, 구현한 잡이 반활될 때만 비동기 함수로 본다.

### 비동기 함수로 레파지토리 작성

잡 구현을 반환하는 함수가 있으면 어떤 시나리오에서는 편리할 수 있지만, 코루틴이 실행되는 동안에 일시 중단을 위해서 `join()` 이나 `await()`를 사용하는 코드가 필요하다는 단점이 생긴다.
기분 동작으로 일시 중지를 하고 싶으면 어떻게 해야 할까?

비동기 함수를 사용해 레파지토리를 설계하고 구현하는 방법을 살펴보자.
다음과 같이 데이터 클래스에서부터 시작한다.

~~~kotlin
data class Profile(
    val id: Long,
    val name: String,
    val age: Int
)
~~~

`name` 이나 `id` 를 기준으로 프로파일을 검색하는 클라이언트 인터페이스를 설계해보자.

~~~kotlin
interface ProfileServiceRepository {
    fun fetchByName(name: String): Profile
    fun fetchById(id: Long): Profile
}
~~~

비동기 함수를 갖도록 구현하고 싶으므로 함수 이름을 비동기(`async`)로 바꾸고 `Profile의 Deferred`를 반환하도록 변경한다.
그에 따라 함수의 이름을 바꾸면 다음과 같을 것이다.

~~~kotlin
interface ProfileServiceRepository {
    fun asyncFetchByName(name: String): Deferred<Profile>
    fun asyncFetchById(id: Long): Deferred<Profile>
}
~~~

테스트는 간단하다.

~~~kotlin
class ProfileServiceClient: ProfileServiceRepository {
    override fun asyncFetchByName(name: String) = GlobalScope.launch {
        Profile(1, name, 27)
    }

    override fun asyncFetchById(id: Long) = GlobalScope.async {
        Profile(id, "Susan", 27)
    }
}
~~~

다른 일시 중단 연산에서 이 구현을 호출할 수 있다.

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val client: ProfileServiceRepository = ProfileServiceClient()
    val profile = client.asyncFetchById(12).await()
    println(profile)
}
~~~

결과

~~~kotlin
Profile(id=12, name=Susan, age=28)
~~~

위의 예제에서 관찰할 수 있는 몇 가지 사항이 있다.

- 함수 이름이 이해하기 편리하게 되어있다. 필요할 때 `클라이언트가 진행하는 것을 완료할 때까지 대기해야 한다`는 것을 알 수 있도록, 함수가 비동기(`async`)라는 점을 명시하는 것이 중요하다.
- 이러한 클라이언트의 성질로 인해, 호출자는 항상 요청이 완료될 때까지 일시 정지해야 하므로 보통 함수 호출 직후에 `await()` 호출이 있게 된다.
- 구현은 `Deferred`와 엮이게 될 것이다. 다른 유형의 `Future`로 `ProfileService Repository` 인터페이스를 깔끔하게 구현하기 위한 방법은 없다. 코틀린이 아닌 동시성 기본형(`primitive`)로 구현하면 자칫 지저분해질 수 있다.

## 일시 중단 함수로 업그레이드

일시 중단 함수 (`suspend`)를 사용하기 위해 코드를 리팩토링해보자.

~~~kotlin
data class Profile(
    val id: Long,
    val name: String,
    val age: Int
)
~~~

~~~kotlin
interface ProfileServiceRepository {
    suspend fun fetchByName(name: String): Profile
    suspend fun fetchById(id: Long): Profile
}
~~~

구현 또한 쉽게 바꿀 수 있다.

~~~kotlin
class ProfileServiceClient: ProfileServiceRepository {
    override suspend fun fetchByName(name: String): Profile {
        return Profile(1, name ,28)
    }

    override suspend fun fetchById(id: Long): Profile {
        return Profile(id, "Susan", 28)
    }
}
~~~

> 실제 구현에서는 Future 구현을 ProfileServiceRepository 인터페이스와 연결하지 않고, RxJava, Retrofit 또는 기타 라이브러리를 사용해 요청을 수행할 수 있다.

이 방식은 비동기 구현에 비해 몇 가지 분명한 이점이 있다.
- `유연함` : 인터페이스의 상세 구현 내용은 노출되지 않기 때문에 퓨처를 지원하는 모든 라이브러리를 구현에서 사용할 수 있다. 현재 스레드를 차단하지 않고 예상된 Profile 을 반환하는 구현이라면 어떤 퓨처 유형도 동작할 것이다.

- `간단함` : 순차적으로 수행하려는 작업에 비동기 함수를 사용하면 항상 `await()`를 호출해야 하는 번거로움이 생기고, 명시적으로 `async` 가 포함된 함수의 이름을 지정해야 한다.
일시 중단 함수 (`suspend`)를 이용하면 레파지토리를 사용할 때마다 이름을 변경하지 않아도 되고 `await()`를 호출할 필요가 없어진다.


## 코루틴 컨텍스트 (Coroutine Context)

코루틴은 항상 컨텍스트 안에서 실행된다. 컨텍스트는 코루틴이 어떻게 실행되고 동작해야 하는지를 정의할 수 있게 해주는 요소들의 그룹이다.
살펴본 몇 가지 컨텍스트로 부터 이야기를 시작한다.

### 디스패처 (Dispatcher)

디스패처는 코루틴이 실행될 스레드를 결정하는데, 여기에는 시작될 곳과 중단 후 재개될 곳을 모두 포함한다.

### CommonPool

`CommonPool` 은 `CPU 바운드 작업`을 위해서 `프레임워크에 의해 자동으로 생성되는 스레드 풀`이다.

스레드 풀의 최대 크기는 시스템의 코어 수에서 1을 뺀 값이다. 현재는 기본 디스패처로 사용되지만 용도를 명시하고 싶다면, 다른 디스패처처럼 사용할 수 있다.

~~~kotlin
GlobalScope.launch(CommonPool) {

}
~~~

### 기본 디스패처

현재는 `CommonPool`과 같다. 기본 디스패처 사용을 위해서 디스패처 전달 없이 빌더를 사용할 수 있다.
~~~kotlin
GlobalScope.launch {

}
~~~

혹시 명시적으로 지정한다면

~~~kotlin
GlobalScope.launch(Dispatchers.Default) {

}
~~~

### Unconfined

첫 번째 중단 지점에 도달할 때까지 현재 스레드에 있는 코루틴을 실행한다. 코루틴은 일시 중지된 후에, 일시 중단 연산에서 사용된 기존 스레드에서 다시 시작된다.

~~~kotlin
fun main() = runBlocking {
    GlobalScope.launch(Dispatchers.Unconfined) {
        println("Starting in ${Thread.currntThread().name}")
        delay(500)
        println("Resuming in ${Thread.currentThread().name}")
    }.join()
}
~~~

처음에는 `main()` 에서 실행 중이었지만, 그 다음 일시 중단 연산이 실행된 `Default Executor` 스레드로 이동했다.

### 단일 스레드 컨텍스트

항상 코루틴이 특정 스레드 안에서 실행된다는 것을 보장한다. 이 유형의 디스패처를 생성하려면 `newSingleThreadContext()`를 사용해야 한다.

~~~kotlin
fun main() = runBlocking {
    val dispatcher = newSingleThreadContext("myThread")

    GlobalScope.launch(dispatcher) {
        println("Starting in ${Thread.currentThread().name}")
        delay(500)
        println("Resuming in ${Thread.currentThread().name}")
    }.join()
}
~~~

일시 중지 후에도 항상 같은 스레드에서 실행된다.

### 스레드 풀

스레드 풀을 갖고 있으며 해당 풀에서 가용한 스레드에서 코루틴을 시작하고 재개한다.
런타임이 가용한 스레드를 정하고 부하 분산을 위한 방법도 정하기 때문에, 따로 할 작업은 없다.

~~~kotlin
fun main() = runBlocking {
    val dispatcher = newFixedThreadPoolContext(4, "myPool")

    GlobalScope.launch(dispatcher) {
        println("Starting in ${Thread.currentThread().name}")
        delay(500)
        println("Resuming in ${Thread.currentThread().name}")
    }.join()
}
~~~

앞의 코드에서 볼 수 있듯이 `newFixedThreadPoolContext()`를 사용해 스레드 풀을 만들 수 있다.

~~~kotlin
Starting in myPool - 1
Resuming in myPool - 2
~~~
다음과 같이 출력된다.