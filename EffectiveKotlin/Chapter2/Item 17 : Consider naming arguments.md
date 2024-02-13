아래 코드를 보면, 'joinToString'의 'argument `|`'가 무엇을 의미하는지 명확하지 않다.

```kotlin
val text = (1..10).joinToString("|")
```

'jointToString'을 잘 알고 있는 개발자라면, `|`가 구분자(separator)라는 것을 쉽게 알 수 있다.  
하지만, 그렇지 않은 사람은 접두사(prefix)라고 이해할 수 있어, 해당 'argument'가 무엇을 의미하는지 명확하지 않다.

이처럼 'argument' 값의 의미가 명확하지 않는 경우, 'named-argument'를 사용하여 코드 가독성을 높일 수 있다.

```kotlin
val text = (1..10).joinToString(separator = "|")
```

또는 'naming-variable'을 사용하여 비슷한 결과를 얻을 수 있다.

```kotlin
val separator = "|"
val text = (1..10).joinToString(separator)
```

하지만, 'naming-variable'은 개발자의 의도를 명확하게 하지만, 정확성을 보장하지는 않는다.  
예를 들어, 변수를 잘못된 위치에 놓았거나, 파라미터의 순서가 변경 되었을 때, 'named-argument'는 정확성 문제를 방지할 수 있지만, 'naming-variable'은 방지할 수 없다.

이러한 이유로, 'named-argument'를 사용하는 것이 더 신뢰성이 높다.

```kotlin
val separator = "|"
val text = (1..10).joinToString(separator = separator)
```

---

## When should we use named arguments?

'named-argument'를 사용하는 것은 코드를 조금 더 길게 만들 수 있지만, 2가지 중요한 이점을 가지게 된다.

1. 예상되는 값이 어떤것인지를 나타내는 이름이 있다.
2. 파라미터 순서에 독립적이어서 안전하다.

또한 'named-argument'는 함수를 사용하거나, 호출 방식을 이해하는데 중요한 정보를 제공한다.  
예를 들어 아래와 같은 함수 호출을 보자.

```kotlin
sleep(100)
```

위 호출만 보고는 '100ms' 인지, '100s'인지 명확하게 알 수 없다.  
하지만, 'named-argument'를 사용하면 이 호출을 더 명확하게 만들 수 있다.

```kotlin
sleep(timeMillis = 100)
```

정적 타입 언어인 Kotlin에서는 'argument' 전달 시, 파라미터 타입을 통해 함수 호출의 의도를 명확하게 전달하고, 타입 안전성을 확보할 수 있다.  
이처럼, 코드 명확성을 제공하는 방법은 'named-argument'만이 유일한 방법이 아니라, 타입을 사용하여 정보를 표현 할 수도 있다.

```kotlin
sleep(Millis(100))
```

위와 같이 타입 시스템은 코드 의도와 구조를 명확하게 전달하는 좋은 방법이다.   
특히, 'inline' 키워드와 같이 활용하면 성능 저하 없이 타입 안전성을 보장 할 수 있다.  

하지만, 타입만으로는 'argument' 역할이나 순서의 명확성을 완전하게 보장하기 어려울 수 있다.  
이로 인해, 아래와 같은 파라미터에는 'named-argument'를 사용하는 것이 좋다. 

**1. 'default arguments'를 가진 파라미터**  
기본 값이 있는 파라미터의 경우, 'named-argument'를 통해 어떤 값이 기본 값인지, 어떤 값이 명시적으로 전달되는지 명확하게 구분 할 수 있다.

**2. 동일한 타입의 파라미터**  
여러 파라미터가 동일한 타입을 가질 때, 'named-argument'를 통해 각 역할의 파라미터에 명확하게 값을 전달하여 실수를 방지할 수 있다.

**3. 마지막 파라미터가 아닌, 함수형 파라미터**  
함수형 파라미터가 '마지막 argument'가 아닌 경우, 'named-argument'를 통해 람다식과 일반 파라미터를 명확하게 구분할 수 있다.

---

## Parameters with default arguments

'default argument'를 가진 파라미터는 함수 호출 시 해당 파라미터에 대해 명시적으로 값을 제공하지 않아도 된다.  
즉, 개발자가 해당 파라미터에 대한 값을 제공하지 않으면 함수에 명시된 기본값이 자동으로 사용된다.

하지만, 이러한 'optional parameter'는 'require parameter' 보다 변경될 가능성이 높다.  
예를 들어, 'optional parameter'는 시간이 지나면서 'default value'가 변경되거나 새로운 'optional parameter'가 추가되는 등의 변경 사항이 생길 수 있다.  
이 때, 함수를 사용하는 코드를 모두 찾아 이런 변경 사항을 반영해야 한다.   
만약, 이런 변경사항을 코드에 제대로 반영하지 않으면 의도한 대로 함수가 동작되지 않을 수 있다.

결과적으로, 위와 같은 상황에서 사전에 'named-argument'를 사용하면, 기본값이나 파라미터 순서의 변경과 같은 사항에 대해 더 유연하게 대응할 수 있다.  
또한, 변경사항이 발생하여도 함수 호출 코드 자체를 수정할 필요가 없어, 코드 유지보수성이 향상된다.

---

## Many parameters with the same type

파라미터가 서로 다른 타입을 가질 때, 값을 잘못된 위치에 전달하면 컴파일러가 타입 불일치를 감지하여 오류 메시지를 표시하기에 안전하다.  
그러나, 몇몇 파라미터가 같은 타입일 때에는 이러한 안전성이 보장되지 않는다.

```kotlin
fun sendEmail(to: String, message: String) { /* ... */ }
```

위 함수처럼 'named-argument'를 사용하지 않으면, 'to'와 'message'의 역할을 명확하게 구분하기 어렵기에 실수가 발생할 수 있다.

```kotlin
sendEmail(
    to = "yongsuk@email.com",
    message = "Hello, Kotlin!"
)
```

---

## Parameters of function type

Kotlin에서 함수형 파라미터가 마지막 위치에 오면, 이를 람다식로 처리 한다.  
때때로 이런 함수의 이름은 람다식이나 함수가 수행할 작업의 성격을 직관적으로 반영하기도 한다.

예를 들어, 'repeat'을 보면, 뒤에 오는 람다식이 반복될 코드 블록이라고 예상할 수 있다.  
또한 'thread'를 보면, 뒤에 오는 블록이 새로운 스레드의 본문이라는 것을 직관적으로 알 수 있다.

```kotlin
thread { 
    // ...
}
```

하지만, 함수형 파라미터가 여러 개 있는 경우, 각각의 역할이 혼동될 수 있다.  
이러한 혼동을 방지하고 코드의 명확성을 유지하기 위해 'named-argument'를 사용하는 것이 좋다.

예를 들어, 아래 간단한 뷰 DSL를 보자.

```kotlin
val view = linearLayout {
    textView("Click")
    button({ /* 1 */  }, { /* 2 */ })
}
```

위 코드 Button을 보면, '빌더 함수'와 '클릭 리스너'가 어느 것인지 명확하지 않다.  
이를 위해 'named-argument'를 사용하여 명확하게 지정할 수 있다.

```kotlin
val view = linearLayout {
    textView("Click")
    button(onClick = { /* 1 */  }) { 
        /* 2 */
    }
}
```

또한 함수형 파라미터가 여러 개가 있는 경우, 더욱 혼란스러울 수 있다.

```kotlin
fun call(
    before: () -> Unit = { }, 
    after: () -> Unit = { }
) {
    before()
    println("Calling")
    after()
}

call({print("CALL")}) // CALLCalling
call { print("CALL") } // CallingCALL
```

위와 같이 함수형 파라미터가 특별한 의미를 갖지 않거나, 여러 개 존재하는 경우, 'named-argument'를 사용하여 혼동을 방지하는 것이 좋다.

```kotlin
call(before = { print("CALL") }) // CALLCalling
call(after = { print("CALL") }) // CallingCALL
```

RxJava와 같은 반응형 프로그래밍 라이브러리에서는 위와 같은 원칙이 더 중요하다.
예를 들어, 'Observable'을 구독할 때, 호출될 함수들은 다음과 같다.

- 수신된 아이템을 처리하는 함수
- 오류 처리 함수
- 완료 이벤트를 처리하는 함수

Java에서는 람다식을 사용하여 'Observable'에 대한 구독을 설정하는 경우가 많다.  
이때 각 람다식이 어떤 메서드에 대응하는지 주석으로 명시하는 방식을 사용하는 경우가 많다.

```java
observable.getUsers()
    .subscribe(
        (List<User> users) -> { // onNext 
            // ...
        }, (Throwable throwable) -> { // onError
            // ...
        }, () -> { // onComplete
            // ...
        }
    );
```

이런 방식은 코드의 의도를 명확하게 전달하기 위해 주석으로 처리되고 있지만, Kotlin에서는 'named-argument'를 사용하여 더 명확하게 표현할 수 있다.

```kotlin
observable.getUsers()
    .subscribeBy(
        onNext = { users: List<User> -> 
            // ...
        },
        onError = { throwable: Throwable -> 
            // ...
        },
        onComplete = { 
            // ...
        }
    )
```