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

