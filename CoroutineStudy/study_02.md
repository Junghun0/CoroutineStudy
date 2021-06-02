## 동시성에 대해

올바른 동시성 코드는 (특정 입력이 들어오면 언제나 똑같은 과정을 거쳐서 항상 똑같은 결과를 내놓는다) , 실행 순서에서는 약간의 가변성을 허용하는 코드다.

그러려면 코드의 서로 다른 부분이 어느 정도 독립성이 있어야 하며 약간의 조정도 필요할 것이다.
동시성을 이해하기 위해서는 순차적인 코드와 비교해보며 이해해보자.

~~~kotlin
fun getProfile(id: Int) : Profile {
    val basicUserInfo = getUserInfo(id)
    val contactInfo = getContactInfo(id)

    return createProfile(basicUserInfo, contactInfo)
}
~~~

UserInfo 와 ContactInfo 중 어느 정보를 먼저 얻게 될까 ?
당연히 UserInfo 가 될 것이다. 
여기서 가장 중요한 것은 사용자 정보가 반환되기 전까지 연락처 정보를 요청하지 않는다는 사실이다.

이것이 순차 코드의 장점이다. 정확한 실행 순서를 쉽게 알 수 있어서 예측하지 못한 일이 벌어지지 않을 것이다.
하지만 순차 코드에는 두 가지 문제점이 있다.

- 동기성 코드에 비해 성능이 저하될 수 있음
- 코드가 실행되는 하드웨어를 제대로 활용하지 못할 수 있음

동시성 구현을 해보자
~~~kotlin
suspend fun getProfile(id: Int) : Profile {
    val basicUserInfo = asyncGetUserInfo(id)
    val contactInfo = asyncGetContactInfo(id)

    return createProfile(basicUserInfo.await(), contactInfo.await())
}
~~~

getProfile() 함수는 suspend function 로서 정의에 `suspend` 수식어 있음을 알 수 있으며, 내부의 두 함수는 `async` 키워드를 통해 비동기임을 알 수 있다.
이 두개의 함수는 서로 다른 스레드에서 실행되도록 작성 되었기 때문에 동시성이라고 한다. 만약에 두 함수가 동시에 실행된다고 생각해보자.
asyncGetUserInfo() 의 실행이 asyncGetContactInfo() 의 완료에 좌우되지 않으므로 웹 서비스에 대한 요청이 동시에 이뤄질 수 있다.
createProfile() 에서 `await()` 을 통해서 어떤 동시성 호출이 먼저 종료되는지에 대해 상관없이 항상 같은 결과를 얻도록 된다.

## 동시성은 병렬성이 아니다

흔히 동시성과 병렬성을 혼동하고 한다.

- 동시성은 두 개 이상의 알고리즘의 실행 시간이 겹쳐질 때 발생한다. 중첩이 발생하려면 두 개 이상의 실행 스레드가 필요하다. 이런 스레드들이 단일 코어에서 실행되면 병렬이 아니라 동시에 실행되는데, 단일 코어가 서로 다른 스레드의 인스트럭션을 교차 배치해서, 스레드들의 실행을 효율적으로 겹쳐서 실행한다.

- 병렬은 두 개의 알고리즘이 정확히 같은 시점에 실행될 때 발생한다.
이것이 가능하려면 2개 이상의 코어와 2개 이상의 스레드가 있어야 각 코어가 동시에 스레드의 인스터럭션을 실행할 수 있다. 병렬은 동시성을 의미하지만 동시성은 병렬성이 없이도 발생할 수 있다는 점에 유의하자.

## CPU 바운드와 I/O 바운드

병목 현상은 다양한 유형의 성능저하가 발생하는 지점을 나타낸다. 
Application 성능을 최적화할 때 가장 중요한 사항이다. 동시성과 병렬성이 CPU 나 I/O 연산에 바인딩됐는지 여부에 따라 알고리즘의 성능에 어떻게 영향을 미칠 수 있는지를 알아본다.

### CPU 바운드

예제로 단어를 가져와서 좌우가 같은 단어인지를 판별하는 간단한 알고리즘을 살펴보자.

~~~kotlin
fun isPalindrome(word: String) : Boolean {
    val lcWord = word.toLowerCase()
    return lcWord == lcWord.reversed()
}
~~~

단어 목록을 가져와서 좌우가 같은 단어를 반환하는 filterPalindromes() 에서 위의 함수를 호출한다고 생각해보자.

~~~kotlin
fun filterPalindromes(words: List<String>) : List<String> {
    return words.filter { isPalindrome(it) }
}
~~~

마지막으로 단어 목록이 이미 정의된 애플리케이션의 main 메소드에서 filterPalindromes()를 호출한다.

~~~kotlin
val words = listOf("level", "pope", "needle", "Anna", "Pete", "noon", "stats")

fun main(args: Array<String>) {
    filterPalindromes(words).forEach {
        println(it)
    }
}
~~~

예제에서는 실행의 모든 부분이 CPU 성능에 의존적이다. 수십 만 개의 단어를 보내도록 코드를 바꾸면 filterPalindromes() 는 더 오래 걸릴 것이다.
코드를 더 빠른 CPU에서 실행하면 코드의 변경 없이도 성능이 향상된다.


### 원자성 위반

원자성 작업 이란 `작업이 사용하는 데이터를 간섭 없이 접근할 수 있음` 을 말한다. 
단일 스레드 Application 에서는 모든 코드가 순차적으로 실행되기 때문에 모든 작업이 모두 원자일 것이다. 즉, 스레드가 하나만 실행되므로 간성ㅂ이 있을 수 없다.

원자성은 객체의 상태가 동시에 수정될 수 있을 때 필요하며 그 상태의 수정이 겹치지 않도록 보장해야한다.
코루틴을 사용할 때 이부분을 유의하자! 코루틴이 다른 코루틴이 숮어하고 있는 데이터를 바꿀 수 있다는 것이다.
예제를 통해 살펴보자
~~~kotlin
var count = 0
fun main(args: Array<String>) = runBlocking {
    val workerA = asyncIncrement(2000)
    val workerB = asyncIncrement(100)
    workerA.await()
    workerB.await()
    print("counter [$counter]")
}

fun asyncIncrement(by: Int) = GlobalScope.async {
    for (i in 0 until by) {
        counter++
    }
}
~~~

위 코드에서 asyncIncrement() 코루틴을 두 번 동시에 실행한다.
한 호출에서는 counter 를 2000 번 증가시키고 다른 호출에서는 100 번 증가시킬 것이다.
문제는 asyncIncrement()의 두 실행이 서로간섭 할 수 있으며, 서로 다른 코루틴 인스턴스가 값을 재정의할 수 있다는 것이다.
main()을 실행하면 대부분 counter [2100] 을 출력하지만, 꽤 많은 실행에서 2100 보다 적은 값을 인쇄한다는 것을 의미한다.

![22](https://user-images.githubusercontent.com/30828236/120473041-8fa5d580-c3e1-11eb-8227-3b8fa1908a2c.PNG)

예제에서는 counter++ 의 원자성이 부족해서 workerA 와 workerB 에서 각각 한 번씩 counter 값을 증가시켜야 하지만 한 번만 증가시켰다.
이런 현상이 발생할 때마다 예상 값이 2100 에서 1씩 줄어들게 된다.

코루틴에서 명령이 중첩되는 것은 counter++ 작업이 원자적이지 않기 때문이다.
실제로 이 작업은 counter의 현재 값을 읽은 다음 그 값을 1씩 증가시킨 후에 그 결과를 counter 에 다시 저장하는 세 가지 명령어로 나눌 수 있다.

counter++ 의 원자성이 없기 때문에 두 코루틴이 다른 코루틴이 하는 조작을 무시하고 값을 읽고 수정할 수 있다.