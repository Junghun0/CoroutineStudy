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
