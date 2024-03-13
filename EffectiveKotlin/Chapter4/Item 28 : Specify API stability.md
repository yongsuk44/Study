프로그래밍에서는 안정적이고 표준적인 애플리케이션 프로그래밍 인터페이스(API)를 선호하며, 이유는 다음과 같다.

개발자들은 API가 변경되어 업데이트를 받으면, 자신의 코드를 수동으로 업데이트 해야된다.
이런 점은 이 API에 의존하는 요소들이 많을 때 문제가 될 수 있다.
API 변경에 따라 기존에 사용되던 방식을 수정하거나, 그에 대한 대안을 제공하는 것이 어려울 수 있다.
특히, 프로젝트의 모든 부분을 이해하고 있지 않은 상황에서 다른 개발자가 해당 API를 사용했다면 이는 더욱 어려울 수 있다.

만약, 이것이 공개 라이브러리인 경우, 라이브러리 개발자가 직접 사용자의 코드를 수정할 수 없기에, 사용자가 직접 수정해야 한다.
이는 사용자 관점에서는 불편한 상황이 아닐 수 없다.
라이브러리의 작은 변경사항이 코드베이스의 여러 부분에서 많은 변경이 필요할 수 있다.

라이브러리 사용자는 이런 변경이 두려우면 오래된 라이브러리 버전을 계속해서 사용하게 된다.
업데이트가 점점 어려워지고, 새로운 업데이트에는 버그 수정이나 취약점 보정과 같이 필요한 사항이 포함될 수 있기에 이는 큰 문제이다.
오래된 라이브러리는 더 이상 지원되지 않거나 완전히 동작을 멈출 수 있다.
프로그래머가 안정적인 최신 버전의 라이브러리를 사용하는 것을 두려워하는 상황은 매우 좋지 않은 상황이다.

새로운 기술을 배우는데에는 시간과 노력이 필요하며, 이미 알고 있던 지식이 변경되었을 때 그것을 다시 배우는 것은 더욱 힘든 일이다.
이런 상황은 사용자가 새로운 지식을 습득하는 것을 회피하게 만들고, 결국 오래된 지식을 사용함으로써 보안 문제와 같은 위험한 상황을 초래할 수 있다.

---

좋은 API를 설계하는 일은 매우 어려운 일이며, 시간이 지남에 따라 사용자의 요구사항이 변하거나 기술의 발전으로 인해 기존의 설계를 개선할 필요성이 생긴다. 
이런 상황에서 API 개발자들은 사용자에게 최적의 경험을 제공하기 위해 API를 개선하려고 노력한다.
하지만, 이러한 변경이 사용자에게 혼란을 주거나 기존 시스템과의 호환성 문제를 일으킬 수 있기에, 프로그래밍 커뮤니티는 API의 안정성을 명시하는 방법을 만들었다.

가장 간단한 방법은 문서화를 통해 API 또는 그 일부가 불안정한지 명확하게 표기하는 것이 있다.
이보다 더 공식적으로는 전체 라이브러리 또는 모듈의 안정성을 버전을 통해 명시한다.

많은 버전 관리 시스템이 있지만, 현재 표준처럼 취급되는 시스템은 'SemVer(Semantic Versioning)'이다.
이 시스템에서는 버전 번호를 세 부분으로 구성하며 다음과 같이 표기 한다.

- MAJOR : 호환되지 않는 API 변경을 할 때 증가
- MINOR : 이전 버전과 호환되는 방식으로 기능을 추가할 떄 증가
- PATCH : 이전 버전과 호환되는 버그를 수정할 때 증가

'MAJOR' 버전을 증가시킬 때, 'MINOR'와 'PATCH'를 0으로 설정하며, 'MINOR'를 증가시킬 때는 'PATCH'를 0으로 설정한다.
사전 출시와 빌드 메타데이터를 위한 추가 라벨들은 'MAJOR.MINOR.PATCH' 형식의 확장으로 사용할 수 있다.
'MAJOR' 버전이 0인 경우(0.y.z)는 초기 개발을 위한 것이며, 이 버전에서는 모든 것들이 언제든지 변경될 수 있으며, 공개 API는 이를 안정적이라고 간주해서는 안된다.
따라서, 라이브러리나 모듈이 'SemVer'을 따르고 'MAJOR' 버전이 0이라면 불안정한 상태라고 판단하면 된다.

베타 버전에 오래 머물러 있는것에 대해서 걱정하지 않아도 괜찮다.  
Kotlin은 1.0에 도달하는데 5년이 넘게 걸렸으며, 이 기간 동안 언어에 많은 변화가 있었기 때문에 중요한 시간이였다.

불안정한 새로운 요소를 안정적인 API에 도입하는 경우, 먼저 다른 브랜치에서 일정 시간동안 유지해야 한다.
일부 사용자에게 이를 사용할 수 있도록 허용하고 싶을 땐, `@Experimental`을 사용하여 해당 API가 안정적이지 않다는 것을 경고할 수 있다.
이는 불안정한 새로운 요소들을 보이게 하지만, 이를 사용할 때 경고나 오류를 표시할 수 있다.

<img src="ex_experimental.png" height="200">

이처럼 불안정한 요소들이 언제든지 변경될 수 있음을 예상해야 한다. 
다시 말하지만, 'experimental' 요소를 오래동안 유지하는 것을 걱정하지 않아도 괜찮다.
이렇게 도입을 늦추는 것은 더 오랜 시간 동안 좋은 API를 설계하는데 도움이 된다.

안정적인 API의 일부를 변경해야 하는 경우, 사용자들이 이런 전환을 처리할 수 있도록 도와주기 위해 `@Deprecated`를 사용하여 해당 요소에 주석을 달아야 한다.

```kotlin
// Before
@Deprecated("Use suspending 'getUsers()' instead")
fun getUsers(callback: (List<User>) -> Unit) {
    // ...
}

// After
suspend fun getUsers(): List<User> {
    // ...
}
```

또는, 직접적인 대안이 있는 경우 'ReplaceWith'를 사용하여 IDE가 자동으로 전환할 수 있도록 명시할 수 있다.

```kotlin
@Deprecated(
    message = "Use suspending getUsers instead", 
    replaceWith = ReplaceWith("getUsers()"),
    level = DeprecationLevel.WARNING
)
fun getUsers(callback: (List<User>) -> Unit) {
    // ...
}
```

아래는 표준 라이브러리의 예시이다.

```kotlin
@Deprecated(
    message = "Use readBytes() overload without estimatedSize parameter instead.", 
    replaceWith = ReplaceWith("readBytes()"),
    level = DeprecationLevel.WARNING
)
public fun InputStream.readBytes(
    estimatedSize: Int = DEFAULT_BUFFER_SIZE
): ByteArray {
    // ...
}
```

이 다음 사용자들이 적응할 수 있는 충분한 시간을 제공해야 한다. 
사용자들은 사용 중인 라이브러리의 새로운 버전에 적응하는 것 이외에도 다른 작업들이 있기에, 이 기간은 길어야 한다. 
널리 사용되는 API의 경우, 이 기간은 몇 년이 걸릴 수 있다.
마침내 이 기간이 지난 후, 어떤 주요 릴리스에서 deprecated 요소를 제거할 수 있다.