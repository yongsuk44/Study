# Item 28: API 안전성 명시

## 프로그래밍에서 표준화된 API를 선호하는 이유

### 1. API 변경으로 발생되는 문제점

API 변경이 코드의 수동 업데이트를 필요로 하며, 이는 다음과 같은 경우에 문제가 될 수 있습니다.

- API에 의존하는 요소가 많은 경우
- 익숙하지 않은 프로젝트에 API를 사용한 경우

공개 라이브러리를 사용하는 경우 개발자들이 직접 라이브러리를 수정할 수 없으므로, API를 사용하는 코드를 직접 수정해야 합니다.
이러한 수정들이 코드 전반에 큰 변경을 요구하게 되어 개발자들은 오래된 라이브러리 버전을 계속해서 사용하게 되는 경우가 빈번합니다.

개발자들이 오래된 라이브러리를 계속해서 사용하게 되는 경우 다음과 같은 문제가 발생될 수 있습니다.

- 라이브러리를 업데이트 하는것에 더욱 더 어려움이 생깁니다.
- 높은 버전의 라이브러리에서 제공되는 기능을 놓칠 수 있습니다.
- 오래된 라이브러리는 지원이 중단되거나 완전히 작동을 멈출 수 있습니다.

따라서 새로운 안정된 라이브러리 버전을 사용하는것을 두려워하는 개발자들이 있는 상황은 좋은 상황이 아닙니다.

### 2. API 변경사항의 새로운 학습과 보안 이슈

새로운 API 학습은 개발자들에게 부담을 주며, 새로운 지식으로 갱신하는 것을 회피하는 경향이 있습니다.
이러한 점은 좋지 않으며, 오래된 지식은 결국 보안 문제로 이어질 수 있습니다.

---

## API 안정성 명시

개발자들은 좋은 API 설계를 위해 API를 안정성을 명시하는것으로 개선할 수 있습니다.

가장 간단한 방법은 개발자가 문서에 API 혹은 일부가 불안정함을 명확히 표시하는 것으로 API의 안정성을 명시할 수 있습니다.

### SemVer(Semantic Versioning)

좀 더 공식적으로는 라이브러리나 모듈의 전체 안정성을 버전을 통해 명시 할 수 있습니다.  
많은 버전 관리 시스템이 있지만, `SemVer(Semantic Versioning)`가 현재 거의 표준으로 사용되고 있는 버전 관리 시스템 입니다.

이 시스템에서는 버전 번호를 `MAJOR.MINER.PATCH`와 같이 구성되며, 각 부분은 `0`부터 시작하는 양의 정수입니다.

API 중요 변화가 있을 때마다 각 부분을 다음과 같이 증가시킵니다.

- API 변화가 호환되지 않을 때는 `MAJOR` 버전을 올립니다.
- 호환 가능한 방식으로 기능을 추가할 때는 `MINOR` 버전을 올립니다.
- 호환 가능한 버그 수정이 있을 때는 `PATCH` 버전을 올립니다.

|  구성   |                             설명                             |
|:-----:|:----------------------------------------------------------:|
| MAJOR | - API 변화가 호환되지 않을때 증가 <br/> - 증가 시 `MINER`와 `PATCH` 0으로 설정 |
| MINOR |    - 호환 가능한 방식으로 기능 추가 시 증가 <br/> - 증가 시 `PATCH` 0으로 설정    |
| PATCH |                  -  호환 가능한 버그 수정이 있을 때 증가                  |

기능 미리보기와 빌드 메타데이터를 위한 추가적인 레이블들은 `MAJOR.MINER.PATCH` 형식에 확장으로 가능합니다.

`MAJOR` 버전이 `0`과 같은 초기 개발은 언제든지 변화가 발생될 수 있으며 공개 API는 불안정적인 상태 입니다. 
따라서, 라이브러리나 모듈이 `Semver`을 따르고 `MAJOR` 버전이 `0`이라면 불안정적인 상태라고 판단하면 됩니다.

### 안전한 API에 불안전한 요소 추가 

안정적인 API에 아직 안정적이지 않은 새로운 요소를 도입하는 경우, 먼저 다른 브랜치에서 일정 시간동안 유지해야 합니다.
이를 다른 개발자들에게 허용하려면 `@Experimental`을 사용하여 안정적이지 않다는 것을 경고 할 수 있습니다.

```kotlin
@Experimental(level = Experimental.Level.WARNING)
annotation class ExperimentalNewApi

@ExperimentalNewApi
suspend fun getUsers(): List<User> {
    // ...
}
```

안정적인 API에서 일부를 변경하는 경우, 전과의 차이를 이해하는데 도움이 되도록 `@Deprecated`를 붙여 시작하도록 합니다.

```kotlin
@Deprecated("Use suspending getUsers instead")
fun getUsers(callback: (List<User>) -> Unit) {
    // ...
}
```

또한, 직접적인 대체가 가능할 때는 IDE가 자동으로 전환을 돕도록 `ReplaceWith`을 사용하여 명시할 수 있습니다.

```kotlin
@Deprecated("Use suspending getUsers instead", ReplaceWith("getUsers()"))
fun getUsers(callback: (List<User>) -> Unit) {
    // ...
}
```

```kotlin
@Deprecated("Use readBytes() overload without estimatedSize parameter instead.", ReplaceWith("readBytes()"))
public fun InputStream.readBytes(
    estimatedSize: Int = DEFAULT_BUFFER_SIZE
): ByteArray {
    // ...
}
```