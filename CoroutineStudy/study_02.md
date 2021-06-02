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


