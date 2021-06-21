## 코루틴 인 액션


### 안드로이드의 UI 스레드

안드로이드는 UI 를 업데이트하고 사용자와의 상호작용을 리스닝하며, 메뉴를 클릭하는 것과 같은 사용자에 의해 생성된 이벤트 처리를 전담하는 스레드가 있다.
할 수 있는 최선의 방법으로 UI 스레드와 백그라운드 스레드를 확실하게 분리하기 위해서, UI 스레드의 기본적인 사항들을 검토해본다.


### CallFromWrongThreadException

안드로이드는 뷰 계층을 생성하지 않은 스레드가 관련 뷰를 업데이트하려고 할 때마다 `CallFromWrongThreadException` 을 발생시킨다.
실제로 이 예외는 UI 스레드가 아닌 다른 스레드가 뷰를 업데이트할 때마다 발생한다. `UI 스레드` 만이 뷰 계층을 생성할 수 있는 스레드이며 뷰를 항상 업데이트할 수 있다.

UI를 업데이트하는 코드가 UI 스레드에서 실행되도록 보장하는 것이 중요하다.

### NetworkOnMainThreadException

자바에서의 네트워크 동작은 기본적으로 블로킹된다. UI 스레드가 블로킹된다는 것은 애니메이션이나 기타 상호작용을 포함한 모든 UI가 멈추는 것을 의미하므로, UI 스레드에서 네트워크 작업을 수행할 때마다 안드로이드는 중단된다.

### 백그라운드에서 요청하고, UI 스레드에서 업데이트할 것

두 가지를 합쳐서 서비스 호출을 구현하려면 백그라운드 스레드가 웹 서비스를 호출하고, 응답이 처리된 후에 UI 스레드에서 UI를 업데이트하도록 해야 한다.

## 스레드 생성

코틀린은 스레드 생성 과정을 단순화해서 쉽고 간단하게 스레드를 생성할 수 있다.
지금은 단일 스레드만으로도 충분하지만, 이후 과정에서는 CPU 바운드와 I/O 바운드 작업을 모두 효율적으로 수행하기 위해 스레드 풀도 생성할 것이다.

### CoroutineDispatcher

코틀린에서는 스레드와 스레드 풀을 쉽게 만들 수 있지만 직접 액세스하거나 제어하지 않는다는 점을 알아야 한다.
여기서는 CoroutineDispatcher 를 만들어야 하는데, 이것은 기본적으로 가용성, 부하, 설정을 기반으로 스레드 간에 코루틴을 분산하는 오케스트레이터다.

여기에서는 스레드를 하나만 갖는 `CoroutineDispatcher`를 생성할 것이며, 거기에 추가하는 모든 코루틴은 그 특정 스레드에서 실행된다.

그렇게 하려면 단 하나의 스레드만 갖는 `CoroutineDispatcher`를 확장한 `ThreadPoolDispatcher`를 생성한다.

### 디스패처에 코루틴 붙이기

디스패처가 만들어졌고 이제 이 디스패처를 사용하는 코루틴을 시작할 수 있다. 디스패처는 코루틴이 정의한 스레드를 강제로 사용하도록 할 것이다.

코루틴을 시작하는 두가지 방법을 살펴볼 텐대, 결과와 에러를 처리하려면 둘 사이의 차이를 알아야 한다.

### async 코루틴 시작

결과 처리를 위한 목적으로 코루틴을 시작했다면 `async()` 를 사용해야 한다.
`async()`는 `Deferred<T>`를 반환하는데, 디퍼드 코루틴 프레임워크에서 제공하는 취소불가능한 넌 블로킹 퓨처 (non-blocking cancellable future)를 의미하며, 
`T`는 그 결과의 유형을 나타낸다.

`async`를 사용할 때 처리하는 것을 잊어서는 안 되며, 잊기 쉽기 때문에 주의하자.

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val task = GlobalScope.async {
        doSomething()
    }
    task.join()
    println("Completed")
}
~~~

doSomething() 은 단순히 예외를 던진다.

~~~kotlin
fun doSomething() {
    throw UnsupportedOperationException("Can't do")
}
~~~

예외를 통해 애플리케이션 실행이 멈추고 예외 스택이 출력되며 또한 애플리케이션의 종료 코드는 0이 아닐 것이라고 생각할 수 있다.
여기서 0은 오류가 발생하지 않았다는 것을 의미하며, 그 외 다른 코드는 오류를 뜻한다.

결과는
~~~kotlin
Completed
~~~

`async()` 블록 안에서 발생하는 예외는 그 결과에 첨부되는데, 그 결과를 확인해야 예외를 찾을 수 있다.
이를 위해서 `isCancelled` 와 `getCancellationException()` 메소드를 함께 사용해 안전하게 예외를 가져올 수 있다.

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val task = GlobalScope.async {
        doSomething()
    }
    task.join()
    if (task.isCancelled) {
        val exception = task.getCancellationException()
        println("Error with message: ${exception.cause}")
    } else {
        println("Success")
    }
}
~~~

결과는
~~~kotlin
Error with message: Can't do
~~~

예외를 전파하기 위해서 디퍼드에서 `await()`를 호출할 수 있다. 예를 들면

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val task = GlobalScope.async {
        doSomething()
    }
    task.await()
}
~~~
그러면 애플리케이션이 비정상적으로 중단된다.

`join()`으로 대기한 후 검증하고 어떤 오류를 처리하는 것과 `await()`를 직접 호출하는 방식의 주요 차이는 
`join()` **은 예외를 전파하지 않고 처리하는 방면,**
`await()` **는 단지 호출하는 것만으로 예외가 전파된다는 점**

`await()`를 사용한 예제는 실행 중 에러를 의미하는 코드 1을 반환하는 반면, `join()`으로 대기하고 `iscancelled`와 `getCancellationException()`을 사용해 에러를 처리한 경우는 성공을 의미하는 코드 0이 나온다.


### launch 코루틴 시작

결과를 반환하지 않는 코루틴을 시작하려면 `launch()`를 사용해야 한다. `launch()`는 연산이 실패한 경우에만 통보 받기를 원하는 파이어-앤-포겟 (fire-and-forget) 시나리오를 위해 설계되었으며,
필요할 때 취소할 수 있는 함수도 함께 제공된다.

~~~kotlin
fun main(args: Array<String>) = runBlocking {
    val task = GlobalScope.launch {
        doSomething()
    }
    task.join()
    println("Completed")
}
~~~

여기에서 doSomething()은 예외를 발생시킨다.

~~~kotlin
fun doSomething() {
    throw UnsupportedOperationException("Can't do")
}
~~~

**예상한 대로 예외가 스택에 출력되지만 실행이 중단되지 않았고, 애플리케이션은 main()의 실행을 완료했다는 것을 알 수 있다.**



