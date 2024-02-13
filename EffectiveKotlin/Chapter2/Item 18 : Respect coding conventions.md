Kotlin은 [coding convention 문서](https://kotlinlang.org/docs/coding-conventions.html#verify-that-your-code-follows-the-style-guide)를 통해 잘 정리된 규칙을 제공하고 있다.

해당 컨벤션은 모든 프로젝트에 최적화된 것은 아니지만, Kotlin 커뮤니티 전체가 공통의 컨벤션을 존중하고 따르는 것이 중요하다.  
이런 컨벤션을 덕분에 아래와 같은 이점을 얻을 수 있다.

**1. 프로젝트 간 전환이 용이하다.**  
공통된 코딩 스타일을 유지하여 코드 일관성을 지킴으로써, 프로젝트 전환 시 코드를 더 쉽고 빠르게 이해할 수 있다.

**2. 외부 개발자도 코드 쉽게 읽을 수 있다.**  
오픈 소스 프로젝트에 새롭게 참여하는 외부 개발자가 코드를 더 쉽고 빠르게 이해할 수 있게 하여 협업을 원활하게 할 수 있다.

**3. 코드 동작 방식이 명확해진다.**  
코드가 어떻게 동작하는지에 대한 가정이 더 명확해지며, 그 의도를 더 쉽게 파악할 수 있다.

**4. 코드 병합 및 프로젝트 간 코드 이동이 용이해진다.**  
다른 프로젝트나 공통 레포지토리에서 병합 시 충돌을 최소화하며, 코드의 특정 부분을 다른 프로젝트로 이식하기 용이해진다.

Kotlin 개발자들은 문서에 설명된 컨벤션에 익숙해져야 하며, 시간이 지남에 따라 컨벤션에 변경 사항이 있더라도 적응해야 한다.  
물론, 이를 모두 적응하는 것이 쉽지 않기에, 다음 두 가지 도구가 도움이 될 수 있다.

- IntelliJ 포맷터는 공식 컨벤션 스타일에 따라 자동으로 코드를 포맷팅하도록 설정할 수 있다.   
  이를 위해 'Settings → Editor → Code Style → Kotlin → Set from… → Kotlin style guide'를 선택하자.
- ktlint는 코드를 분석하여 컨벤션 위반 사항을 알려주는 린터로, 이를 통해 코드 일관성을 유지하고, 컨벤션을 쉽게 따를 수 있게 도와준다.

Kotlin 프로젝트를 살펴보면, 대부분의 프로젝트가 코딩 컨벤션을 잘 따르고 있는 것처럼 보여진다.  
이는 Kotlin 컨벤션이 Java 컨벤션을 많이 따르고 있기 떄문이며, Kotlin을 사용하는 개발자 대부분이 Java 개발자 출신이기 때문일 것이다.

그러나, 종종 컨벤션에 위반된다고 보이는 한 가지 규칙은 클래스와 함수의 포맷팅 방식을 어떻게 하는지에 대한 것이다.  
컨벤션에 따르면, 짧은 'primary-constructor'는 한 줄로 정의될 수 있다:

```kotlin
class FullName(val name: String, val surname: String)
```

그러나, 파라미터가 많은 클래스는 각 파라미터가 다른 줄에 오도록 포맷팅되어야 하며, 첫 번째 줄에는 파라미터가 없어야 한다.

```kotlin
// Do
class Person(
    val id: Int = 0,
    val name: String = "",
    val surName: String = ""
): Human(id, name) { 
    // body
}
```

마찬가지로, 함수가 길어질 때에도 동일한 포맷팅 방식을 사용해야 한다.

```kotlin
public fun <T> Iterable<T>.joinToString(
    separator: CharSequence = ", ",
    prefix: CharSequence = "",
    postfix: CharSequence = "",
    limit: Int = -1,
    truncated: CharSequence = "...",
    transform: ((T) -> CharSequence)? = null
): String {
    // ...
}
```

위와 같은 두 가지 방식은 첫 번째 파라미터를 같은 줄에 두고, 이에 맞춰서 그 다음 파라미터들을 모두 들여쓰기하는 컨벤션과는 다르다.

```kotlin
// Don't do that
class Person(val id: Int = 0,
             val name: String = "",
             val surName: String = "") : Human(id, name) {
    // body 
}
```

위와 같은 스타일은 두 가지 문제를 야기할 수 있다.

- 클래스 이름이 변경될 때마다 'primary-constructor' 파라미터 모두 들여쓰기 조정을 해야 하는 번거로움이 생긴다.
- 클래스의 너비가 과도하게 넓어 질 수 있으며, 너비는 다음과 같이 계산 될 수 있다. 
  - 클래스 이름 + class 키워드 + 가장 긴 'primary-constructor' 파라미터 or 마지막 파라미터 + 'super' 클래스 + 구현한 인터페이스

회사마다 컨벤션을 약간 수정하여 사용해도 괜찮지만, 이런 경우엔 회사 전체 프로젝트에 컨벤션들이 일관되게 적용되어야 한다.  
또한, 모든 프로젝트가 개발자 한 사람이 작성된 것처럼 보이는 것이 중요하다.