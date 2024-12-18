## The meaning of Composable functions

컴포저블 함수는 Compose의 가장 기본적인 빌딩 블록(atomic building block)이며, 작성할 컴포저블 트리의 기본 구조 입니다.  
여기서 "트리"라는 표현을 사용한 이유는, 컴포저블 함수가 더 큰 트리 구조에서 노드 역할을 하기 떄문입니다.   
이 트리는 Compose 런타임에서 메모리로 표현되며, 이 개념은 차차 자세히 설명하겠습니다. 

문법적으로 보면, 표준 Kotlin 함수는 `@Composable` 어노테이션을 붙이는 것만으로 컴포저블 함수로 변환될 수 있습니다:

```kotlin
@Composable
fun NamePlate(name: String) { ... }
```

이렇게 하면 컴파일러에게 `NamePlate`의 데이터를 노드로 변환하여 컴포저블 트리에 등록할 의도가 있음을 알리는 것입니다.  
즉, 컴포저블 함수를 `@Composable (Input) -> Unit`으로 읽는다면, 입력은 데이터가 되고, 출력은 함수에서 반환되는 값이 아닌 요소를 트리에 삽입하는 작업으로 등록됩니다.
이는 함수 실행의 '사이드 이펙트'로 발생한다고 볼 수 있습니다.

> 입력을 받는 함수에서 `Unit`을 반환한다는 것은, 함수 본문에서 해당 입력을 어떤 방식으로돈 소비하고 있음을 의미합니다.

이와 같은 작업을 Compose 용어로 "발행(emitting)"이라고 부릅니다. 컴포저블 함수는 실행될 때 발행되며, 이 과정은 Composition 중에 일어납니다.  
이 과정에 대한 자세한 내용은 이후 챕터에서 다룰 예정이며, 당분간은 컴포저블 함수를 "컴포징(composing)"한다는 표현을 보면, 이를 "실행한다(executing)"와 동일하게 생각하면 됩니다. 

컴포저블 함수를 실행하는 목적은 메모리 내 트리 구조를 생성하거나 업데이트하는 것입니다.  
컴포저블 함수는 읽은 데이터가 변경될 때마다 다시 실행되어, 트리 구조를 최신 상태로 유지합니다.  
이를 위해 노드를 삽입하는 작업뿐만 아니라, 노드를 제거하거나, 교체하거나, 이동하는 작업도 가능합니다.  
또한, 컴포저블 함수는 트리에서 상태를 읽고 쓰는 기능도 제공합니다.

## Properties of Composable functions

`@Composable` 어노테이션을 사용하면 **함수의 타입이 변경**되며, 이는 다른 타입처럼 몇 가지 제약이나 속성을 부여합니다.  
이러한 속성은 Compose의 라이브러리 기능을 활용하는데 매우 중요합니다.

Compose 런타임은 컴포저블이 부여된 속성들을 준수할 것으로 기대합니다.  
이를 통해 특정 동작을 예측하고, 다음과 같은 다양한 런타임 최적화를 적용할 수 있습니다.

- 병렬 컴포지션 (Parallel composition)
- 우선순위 기반 임의의 컴포지션 (Arbitrary order of composition based on priorities)
- 스마트 리컴포지션 (Smart recomposition)
- 위치 기반 메모이제이션 (Positional memoization)

이 새로운 개념들은 모두 이후 챕터에서 자세히 다룰 것입니다.

> 일반적으로 런타임 최적화는 런타임이 실행해야 할 코드에 대해 몇 가지 확신을 가질 수 있을 때 가능합니다.  
> 이를 통해 특정한 조건이나 동작을 가정하고, 이를 바탕으로 다양한 실행 전략이나 평가 기법을 활용하여 코드를 "소비(consume)"할 기회를 얻게됩니다.
> 
> 이러한 확신의 예로는 코드 내 서로 다른 요소 간의 관계를 들 수 있습니다.  
> 이 요소들이 서로 의존적인가? 혹은 병렬로 실행하거나 다룬 순서로 실행해도 프로그램 결과에 영향을 주지 않는가?  
> 각 논리적인 단위를 완전히 독립적인 요소로 해석할 수 있는가?

## Calling context

컴포저블의 대부분 속성은 Compose 컴파일러에 의해 활성화됩니다.  
Compose 컴파일러는 Kotlin 컴파일러 플러그인으로서, 일반적인 컴파일 과정에서 실행되며, Kotlin 컴파일러가 접근할 수 있는 모든 정보에 접근할 수 있습니다.  
이를 통해 컴포저블의 중간 표현(IR, Intermediate Representation)을 가로채고 변환하여, 함수에 추가적인 정보를 삽입할 수 있습니다.

각 컴포저블에 추가되는 중요한 요소 중 하나는 파라미터 목록의 끝에 추가되는 `Composer`라는 새로운 파라미터입니다.  
`Composer` 파라미터는 개발자가 직접 다루지 않는 **암묵적인** 파라미터로, 인스턴스는 런타임에 주입되며, 모든 자식 컴포저블 호출로 전달되어 트리의 모든 레벨에서 접근할 수 있게 됩니다. 

예를 들어, 다음 컴포저블이 있다고 가정해봅시다:

```kotlin
@Composable
fun NamePlate(
    name: String,
    lastname: String
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text(text = name)
        Text(
          text = lastname,
          style = MaterialTheme.typography.body2
        )
    }
}
```

컴파일러는 이를 다음과 같이 변환합니다:

```kotlin
fun NamePlate(
    name: String,
    lastname: String,
    $composer: Composer
) {
    ...
    Column(
        modifier = Modifier.padding(16.dp),
        $composer
    ) {
        Text(
          text = name,
          $composer
        )
        Text(
          text = lastname,
          style = MaterialTheme.typography.body2,
          $composer
        )
    }
    ...
}
```

보다시피, `Composer`는 컴포저블의 본문 내에서 모든 컴포저블에 전달됩니다.  
이와 더불어, Compose 컴파일러는 컴포저블에 대한 엄격한 규칙을 부여합니다: **컴포저블은 다른 컴포저블에서만 호출될 수 있습니다.**  
이것이 컴포저블의 호출 콘텍스트(calling context)이며, 이를 통해 Composition 트리가 오직 컴포저블로만 구성되도록 보장하며, `Composer`가 계속 하위로 전달될 수 있도록 합니다.

`Composer`는 개발자가 작성한 컴포저블 코드와 Compose 런타임을 연결하는 역할을 합니다.  
컴포저블은 `Composer`를 사용해 트리의 변경 사항을 발행하고, 트리 구조를 Compose 런타임에 알려, 메모리 내의 트리 구조를 생성하거나 업데이트합니다.

## Idempotent

컴포저블은 생성하는 노드 트리에 대한 멱등성(idempotent)을 가져야 합니다.  
동일한 입력 파라미터를 사용하여 컴포저블을 여러 번 실행해도 동일한 트리가 생성되어야 하며, Compose 런타임은 이를 기반으로 Recomposition을 수행합니다.

**Recomposition**이란, 입력값이 변경될 때 컴포저블을 다시 실행하여, 업데이트된 정보를 발행하고 트리를 업데이트하는 작업입니다.  
Compose 런타임은 언제든지 컴포저블을 Recomposition할 수 있어야 하며, 이 과정에서 트리의 노드를 순차적으로 확인하여 입력값이 변경된 노드만 다시 실행합니다.  
입력값이 변경되지 않은 나머지 노드들은 **스킵**되므로 불필요한 Recomposition을 피할 수 있습니다.

이처럼 노드를 스킵하는 것은 해당 컴포저블이 멱등성을 가질 때만 가능합니다.  
왜냐하면, Compose 런타임은 동일한 입력값에 대해 항상 동일한 결과를 반환할 것이라는 가정이 가능하기 떄문입니다.  
이로 인해 Compose는 메모리 내 트리 구조에 이미 저장된 결과가 있다면, 해당 컴포저블을 다시 실행할 필요가 없습니다.

## Free of uncontrolled side effects

사이드 이펙트는 함수의 제어를 벗어나 예상치 못한 작업을 수행하는 모든 동작을 의미합니다.  
로컬 캐시에서 데이터를 읽거나, 네트워크 호출을 수행하거나, 전역 변수를 설정하는 것 등이 대표적인 사이드 이펙트에 해당합니다.  
이러한 사이드 이펙트는 함수가 외부 요인에 의존하게 만들어 함수의 동작에 영향을 줄 수 있습니다.  
예를 들어, 다른 스레드에서 쓰일 수 있는 외부 상태나, 오류를 발생시킬 수 있는 서드파티 API 등이 있습니다.  
즉, 함수가 입력값만으로 결과를 생성하지 않는다는 것을 의미합니다.

사이드 이펙트는 **모호성의 원인**입니다.  
이러한 모호성은 Compose에서 적합하지 않은데, Compose 런타임은 컴포저블이 예측 가능하고(결정론적), 안전하게 여러 번 재실행될 수 있기를 기대하기 때문입니다.  
만약 컴포저블이 사이드 이펙트를 포함하면, 매 실행마다 다른 상태를 초래할 수 있어 함수가 멱등성을 잃게 됩니다.

다음과 같이 컴포저블 본문에서 직접 네트워크 요청을 실행한다고 가정해봅시다:

```kotlin
@Composable
fun EventsFeed(
    networkService: EventsNetworkService
) {
    val events = networkService.loadEvents()
    
    LazyColumn {
        items(events) { event ->
            Text(event.name)
        }
    }
}
```

위 상황은 매우 큰 리스크를 지닙니다. 
Compose 런타임은 짧은 시간 안에 해당 함수를 여러 번 재실행할 수 있기 때문에, 네트워크 요청이 반복적으로 발생하여 통제 불가능한 상태로 빠질 수 있습니다.
게다가, 이러한 실행은 서로 다른 스레드에서 조율되지 않은 상태로 발생할 수 있기 때문에 더 큰 문제가 발생할 수 있습니다.

> Compose 런타임은 컴포저블의 실행 전략을 자유롭게 선택할 수 있습니다.  
> 예를 들어, Recomposition을 여러 스레드로 분산시켜 멀티코어 환경에서 성능을 최적화하거나, 필요에 따라 임의의 우선순위로 실행할 수 있습니다.  
> (e.g : 화면에 표시되지 않는 컴포저블은 우선순위가 낮게 할당될 수 있습니다.)

사이드 이펙트의 또 다른 문제 중 하나는 한 컴포저블이 다른 컴포저블의 결과에 의존하여 실행 순서에 제약을 가할 수 있다는 것입니다.  
이는 반드시 피해야 할 상황이며 다음과 같은 예시를 들 수 있습니다:

```kotlin
@Composable
fun MainScreen() {
    Header()
    ProfileDetail()
    EventList()
}
```

위 예시에서 `Header`, `ProfileDetail`, `EventList`는 임의의 순서로 실행되거나, 병렬로 실행될 수 있습니다.  
때문에 `Header`에서 작성된 외부 변수를 `ProfileDetail`에서 읽는 것과 같은, 특정 실행 순서를 가정하는 로직을 작성해서는 안 됩니다.

일반적으로 컴포저블 내에서 사이드 이펙트는 바람직하지 않습니다.  
가장 이상적인 방식은 컴포저블을 Stateless로 설계하여, 모든 입력을 파라미터로 받아 결과를 생성하는 방식입니다.  
이렇게 하면 컴포저블이 더 단순해지고 재사용성이 높아집니다.   

그러나 상태를 관리하는 프로그램에서는 어느 정도의 사이드 이펙트가 필요하며, 주로 컴포저블 트리의 루트에서 이러한 작업이 수행됩니다.  
예를 들어, 네트워크 요청을 처리하거나, 데이터를 데이터베이스에 저장하고, 메모리 캐시를 사용하는 등의 작업이 필요합니다.  
이 때문에 Compose는 컴포저블에서 사이드 이펙트를 안전하게 처리할 수 있도록 **이펙트 핸들러**(effect handler)라는 메커니즘을 제공합니다.

이펙트 핸들러는 사이드 이펙트를 컴포저블의 라이프사이클과 연동하여 제약을 가하거나 제어할 수 있도록 돕습니다.  
이를 통해 컴포저블이 트리에서 제거될 때 이펙트가 자동으로 폐기되거나 취소되고, 이펙트의 입력이 변경되면 다시 트리거될 수 있습니다.  
또한, 동일한 이펙트가 여러 번 실행되더라도(Recomposition) 단 한번만 호출되도록 처리할 수 있습니다.  
이후 챕터에서 이펙트 핸들러에 대해 깊이 다룰 것이며, 이를 통해 컴포저블 본문에서 아무런 제어 없이 이펙트를 직접 호출하는 일을 방지할 수 있습니다.

## Restartable

몇 차례 언급했듯이, 컴포저블은 재구성될 수 있기 때문에 스탠다드 함수와는 다릅니다.  
스탠다드 함수는 호출 스택의 일부로서 한 번만 호출되지만, 컴포저블은 여러 번 호출될 수 있습니다.  
일반적인 호출 스택에서는 각 함수가 한 번 호출되며, 다른 함수들을 호출할 수 있습니다.

반면, 컴포저블은 여러 번 재시작(재실행, 재구성)될 수 있으므로, Compose 런타임은 이들을 재시작하기 위해 함수에 대한 참조를 유지합니다.  
아래는 컴포저블 호출 트리의 예시입니다:

<img alt="img.png" src="composable_emits.png" width="80%"/>

컴포저블 4와 5는 입력값이 변경된 후 다시 실행됩니다.

Compose는 메모리 내 트리 구조를 항상 최신 상태로 유지하기 위해, 트리에서 어떤 노드를 다시 실행할지 선택적으로 결정합니다.  
컴포저블은 관찰하는 상태의 변화에 반응하여 다시 실행되도록 설계되었습니다.

Compose 컴파일러는 상태를 읽는 모든 컴포저블을 찾아, Compose 런타임에 해당 컴포저블을 다시 실행하는 방법을 가르치기 위한 코드를 생성합니다.  
상태를 읽지 않는 컴포저블은 다시 실행될 필요가 없으므로, 런타임은 이를 재시작하는 방법을 배울 필요가 없습니다. 

## Fast execution

컴포저블과 컴포저블 트리를 빠르고, 선언적이며, 가벼운 방식으로 프로그램의 설명을 구축하여 메모리에 유지되고 이후 단계에서 해석/실체화 시키는 효율적인 방법이라고 생각할 수 있습니다.

컴포저블은 UI를 직접적으로 생성하거나 반환하지 않고, 메모리 구조를 구축하거나 업데이트하기 위한 데이터를 발행할 뿐입니다.  
이러한 방식 덕분에 컴포저블은 매우 빠르게 실행되며, 런타임에서 여러 번 실행해도 성능 저하에 대한 걱정이 없습니다.  
때로는 애니메이션처럼 매 프레임마다 매우 빈번하게 실행될 수 있습니다.

개발자는 이러한 특성을 염두에 두고 코드를 작성해야 하며, 무거운 연산은 코루틴으로 처리하고, 반드시 라이프사이클을 인식하는 이펙트 핸들러로 래핑해야 합니다.

## Positional memoization

위치 기반 메모이제이션은 함수 메모이제이션의 한 형태입니다.    
함수 메모이제이션은 입력값에 따라 결과를 캐시하여, 동일한 입력에 대해 함수가 다시 호출될 때 값을 재계산하지 않고 캐시된 값을 반환하는 기능입니다.  
이는 순수 함수(deterministic)에서만 가능하며, 동일한 입력에 대해 항상 같은 결과를 반호나할 것이라는 확신이 있어야 가능합니다.

> 함수 메모이제이션은 함수형 프로그래밍 세계에서 널리 알려진 기법입니다. 
> 함수형 프로그래밍에서는 프로그램이 순수 함수의 조합으로 정의되므로, 함수의 결과를 메모이제이션하여 성능을 크게 향상시킬 수 있습니다.

함수 메모이제이션에서는 함수의 이름, 타입, 파라미터 값의 조합을 통해 함수 호출을 식별합니다.  
이러한 요소를 기반으로 고유한 키를 생성하고, 이를 통해 캐시된 결과를 저장하거나 나중에 호출할 때 인덱싱하거나 읽을 수 있습니다.  
Compose에서는 여기에 더해 컴포저블의 위치 정보가 추가적으로 고려됩니다.   
동일한 함수가 동일한 파라미터로 호출되더라도 다른 위치에서 호출될 경우, Compose 런타임은 해당 함수 호출에 대해 부모 내에서 고유한 ID를 생성합니다.

```kotlin
@Composable
fun MyComposable() {
    Text()  // id 1
    Text()  // id 2
    Text()  // id 3
}
```

메모리 내 트리는 동일한 컴포저블의 세 가지 서로 다른 인스턴스를 저장하며, 각각은 고유한 아이덴티티를 가집니다.

<img alt="img.png" src="id_identity.png" width="50%"/>

컴포저블의 아이덴티티는 Recomposition이 이루어져도 유지되기에, Compose 런타임은 이를 통해 해당 컴포저블이 이전에 호출된 적이 있는지 확인하고, 가능하다면 호출을 스킵하여 성능을 최적화할 수 있습니다.

하지만, 때로는 고유한 아이덴티티를 부여하는 것이 Compose 런타임에게 어려운 경우도 있습니다.  
그 대표적인 예가 반복문에서 생성된 컴포저블 리스트입니다.

```kotlin
@Composable
fun TalksScreen(talks: List<Talk>) {
    Column {
        for (talk in talks) {
            Talk(talk)
        }
    }
}
```

위 코드에서 `Talk(talk)`는 매번 동일한 위치에서 호출도지만, 각 아이템은 리스트 내에서 서로 다른 요소를 나타내므로 트리 내에서 서로 다른 노드로 처리됩니다.
이런 경우 Compose 런타임은 **호출 순서**를 기준으로 고유 ID를 생성하여 노드를 구분합니다.
리스트의 끝에 새로운 아이템을 추가할 때는 기존 항목들의 위치가 변하지 않으므로 이 방식이 잘 동작합니다.
그러나 상단이나 중간에 아이템을 추가할 경우, 해당 위치 이후의 모든 `Talk`들이 위치가 변경되기에 다시 Recomposition되며, 이는 입력값이 변하지 않았더라도 발생합니다.
이는 리스트가 길어질수록 매우 비효율적이며, 실제로는 이러한 호출들이 스킵되어야 합니다.

이를 해결하기 위해 Compose는 `key` 컴포저블을 제공하여 호출에 대한 명시적인 키를 수동으로 할당할 수 있습니다:

```kotlin
@Composable
fun TalksScreen(talks: List<Talk>) {
    Column {
        for (talk in talks) {
            key(talk.id) { // Unique Key
                Talk(talk)
            }
        }
    }
}
```

이 예시에서는 고유할 가능성이 높은 각 `Talk`의 `talk.id`를 키로 사용하여, 항목의 위치와 상관없이 모든 리스트 아이템의 아이덴티티를 유지할 수 있습니다. 

위치 기반 메모이제이션은 Compose 런타임이 컴포저블을 기억할 수 있도록 설계되었습니다.  
Compose 컴파일러에 의해 restartable로 추론된 모든 컴포저블은 **자동으로 기억**되며, 이를 기반으로 스킵할 수 있습니다.  
Compose는 이러한 메커니즘 위에 구축되었습니다.

때로는 컴포저블의 범위를 넘어 더 세밀하게 메모리 구조에 접근해야할 떄가 있습니다.  
예를 들어, 컴포저블 내에서 무거운 연산의 결과를 캐시하고자 할 때, Compose 런타임은 `remember`를 통해 이를 지원합니다:

```kotlin
@Composable
fun FilteredImage(path: String) {
    val filters = remember { computeFilters(path) }
    ImageWithFiltersApplied(filters)
}

@Composable
fun ImageWithFiltersApplied(filters: Filters) {
    TODO()
}
```

여기서는 `remember`를 통해 사전에 계산된 이미지 필터 결과를 캐시합니다.  
캐시된 값을 인덱싱하는 키는 호출된 위치와 함수의 입력값(이 경우 `path`)을 기반으로 합니다.  
`remember`는 컴포저블 트리의 상태를 저장하는 메모리 구조에서 데이터를 읽고 쓰는 방법을 알고 있으며, 이를 통해 "위치 기반 메모이제이션" 메커니즘을 개발자에게 제공합니다.

Compose에서 메모이제이션은 애플리케이션 전체에 적용되지 않고, 해당 컴포저블의 컨텍스트 내에서만 이루어집니다.  
위 예시에서는 `FilteredImage` 컴포저블 내에서만 유효합니다.  
실제로 Compose는 메모리 구조로 이동해, 컴포저블이 저장된 슬롯 범위 내에서 값을 검색합니다.  
이는 해당 **스코프 내에서 싱글톤**처럼 동작합니다. 동일한 컴포저블이 다른 부모로부터 호출되면, 새로운 인스턴스의 값이 반환됩니다.

## Similarities with suspend functions

Kotlin의 `suspend` 함수는 오직 다른 `suspend` 함수에서만 호출할 수 있으며, 이를 위해서는 특정 호출 컨텍스트가 필요합니다.  
이 방식은 `suspend` 함수들이 서로 연결되어 함께 동작할 수 있도록 하며, Kotlin 컴파일러는 이 과정에서 모든 연산 단계를 통과하는 런타임 환경을 주입하고 전달할 수 있습니다.  
이 런타임 환경은 각 `suspend` 함수에 추가되는 암묵적인 파라미터 `Continuation`을 통해 이루어집니다.  
개발자는 이를 인식하지 않고, `Continuation`을 통해 언어 내에서 강력한 기능을 활용할 수 있습니다.

> Kotlin의 코루틴 시스템에서 `Continuation`은 마치 콜백처럼 동작합니다.  
> 즉, 프로그램이 어떻게 실행을 이어나가야 하는지를 알려주는 역할을 합니다.

다음은 이에 대한 예시입니다:

```kotlin
suspend fun publishTweet(tweet: Tweet): Post = ...
```

Kotlin 컴파일러는 위 코드를 다음과 같이 변환합니다:

```kotlin
fun publishTweet(tweet: Tweet, continuation: Continuation<Post>): Unit
```

`Continuation`은 프로그램의 여러 중단 지점에서 실행을 일시 중단하고 다시 재개하기 위한 필요한 모든 정보를 포함합니다.  
이를 통해 `suspend`는 호출 컨텍스트를 요구하여, 암묵적인 정보를 실행 트리 전반에 걸쳐 전달하는 수단이 될 수 있다는 또 다른 좋은 예시입니다.  
이러한 정보는 런타임에서 고급 언어 기능을 활성화하는데 사용될 수 있습니다.

같은 방식으로, `@Composable`도 Kotlin 함수에 새로운 기능을 부여하는 언어 기능으로 볼 수 있습니다.  
이를 통해 스탠다드 Kotlin 함수는 restartable하고, reactive한 함수로 변환됩니다.

> 이 시점에서, 왜 Compose 팀이 원하는 동작을 구현하는데 `suspend`를 사용하지 않았는지 궁금할 수 있습니다.  
> 두 기능 모두 유사한 패턴으로 구현되지만, 실제로는 언어에서 전혀 다른 기능을 수행합니다.
> 
> `Continuation` 인터페이스는 실행의 중단 및 재개에 매우 특화된 콜백 인터페이스입니다.  
> 여기서 Kotlin은 중단 지점 간의 점프를 관리하고, 데이터를 공유하며 필요한 모든 메커니즘을 갖춘 기본 구현을 자동으로 생성합니다.  
> 반면, Compose는 런타임에서 다양한 방식으로 최적화가 가능한 대규모 호출 그래프의 메모리 내 표현을 생성하는 것이 목적이므로, 사용 목적이 완전히 다릅니다.

## The color of Composable functions

컴포저블은 스탠다드 함수와 다른 기능과 제한이 있습니다. 이들은 다른 타입을 가지며, 특정 목적을 모델링합니다.  
이러한 차이는 "함수 컬러링"이라는 개념으로 설명될 수 있습니다. 컴포저블은 별도의 카테고리를 나타내기 때문입니다.

"함수 컬러링"은 2015년에 구글의 Dart 팀의 블로그 포스트((What color is your function?))에서 설명한 개념입니다.  
이 포스트에서는 비동기 함수와 동기 함수가 함께 잘 조합되지 않는 이유를 설명합니다.  
동기 함수에서 비동기 함수를 호출하려면, 동기 함수도 비동기로 만들어야 하거나, 비동기 함수를 투명하게 호출하고 그 결과를 기다릴 수 있는 메커니즘을 제공해야 합니다.  
그래서 일부 라이브러리와 언어에서는 이 조합의 문제를 해결하기 위해 'Promise'와 `async`/`await`을 도입하는 시도를 했습니다.
이 포스트에서는 두 함수를 서로 다른 "함수 색상"으로 설명했습니다.

Kotlin에서는 `suspend`가 동일한 문제를 해결하려 하지만, `suspend` 함수도 함수 컬러링이 적용된 상태입니다.  
왜냐하면 `suspend` 함수는 다른 `suspend` 함수에서만 호출할 수 있기 때문입니다.  
스탠다드 함수와 `suspend` 함수가 함께 사용될 때는 코루틴 시작점과 같은 통합 메커니즘이 필요하며, 이러한 통합은 개발자에게 명확하게 드러나지 않을 수 있습니다.

이러한 제한은 당연한 것입니다. 앞의 예시는 본질적으로 다른 두 카테고리의 함수를 모델링하고 있으며, 이는 마치 두 개의 서로 다른 언어를 사용하는 것과 같습니다.  
결과를 즉시 제공하는 동기 작업과, 시간이 지나야 결과를 제공하는 비동기 작업이 있으며, 비동기 작업은 결과를 도출하는데 시간이 더 소요될 수 있습니다.

Compose에서 컴포저블도 같은 원칙이 적용됩니다.  
스탠다드 함수에서 컴포저블 함수를 직접 호출할 수 없으며, 이를 위해서는 통합 지점(e.g : `Composition.setContent`)이 필요합니다.  
컴포저블은 프로그램 로직을 작성하는 것이 아닌, 노드 트리의 변경을 기술하기 위해 설계되었습니다.

여기서, 약간 혼란스러운 부분은 컴포저블이 로직을 통해 UI를 선언할 수 있다는 점입니다.  
이 말인즉, 경우에 따라서는 스탠다드 함수에서 컴포저블을 호출할 필요가 있다는 의미입니다.

예를 들어:

```kotlin
@Composable
fun SpeakerList(speakers: List<Speaker>) {
    Column {
        speakers.forEach {
            Speaker(it)
        }
    }
}
```

`Speaker` 컴포저블은 `forEach` 람다 내에서 호출되지만, 컴파일러는 오류를 발생시키지 않습니다.  
이처럼 함수의 색상을 섞는 것이 가능한 이유는 `inline` 때문입니다.  
컬렉션 연산자들은 `inline`으로 선언되어, 람다를 호출자 내에 인라인화하여 추가적인 간접 참조가 없도록 만듭니다.  
위 예제에서는 `Speaker` 호출이 `SpeakerList` 본문 내에 인라인되므로, 둘 다 컴포저블로 호출이 허용됩니다.  
이처럼 `inline`을 사용하면 함수 컬러링 문제를 우회할 수 있으며, 이를 통해 컴포저블 트리 구성이 가능합니다.

함수 컬러링은 두 가지 함수 타입을 계속해서 혼합하고, 이들 사이를 빈번히 전환한다면 문제가 될 수 있습니다.  
그러나 `suspend` 함수나 `@Composable`의 경우, 통합 지점을 거친 이후로는 완전히 컬리링된 호출 스택을 가지게 됩니다. (즉, 모든 함수가 `suspend` 이거나 컴포저블이 됩니다.)  
이는 오히려 장점이 될 수 있는데, 컴파일러와 런타임이 컬러링된 함수를 다르게 처리할 수 있어, 스탠다드 함수에서는 불가능했던 고급 언어 기능을 활성화할 수 있기 때문입니다.

Kotlin에서 `suspend`는 비동기 non-blocking 프로그램을 간결하고 표현력 있는 방식으로 모델링할 수 있도록 도와줍니다.  
단순히 `suspend` 키워드를 함수에 추가하는 것만으로 언어는 복잡한 개념을 간단히 표현할 수 있는 능력을 얻게 됩니다.  
반면에, `@Composable`은 스탠다드 함수를 restartable, skippable, reactive하게 만들어주며, 이는 스탠다드 함수가 갖지 못한 기능들입니다.

## Composable function types

`@Composable` 어노테이션은 컴파일 시점에서 함수의 타입을 변경합니다.  
문법적으로 보면, 컴포저블의 타입은 `@Composable (T) -> A` 형태이며, 여기서 `A`는 함수가 반환하는 값입니다.  
`A`는 `Unit`일 수도 있고, `remember` 함수처럼 다른 타입을 반환할 수도 있습니다.  
개발자는 이 타입을 사용하여 컴포저블 람다를 정의할 수 있으며, 이는 Kotlin의 스탠다드 람다를 선언하는 것과 동일합니다.

```kotlin
// This can be reused from any Composable tree
val textComposable: @Composable (String) -> Unit = { 
    Text(
        text = it,
        style = MaterialTheme.typography.body1
    )
}

@Composable
fun NamePlate(
    name: String,
    lastName: String
) {
    Column(
        modifier = Modifier.padding(16.dp)
    ) {
        Text(
            text = name,
            style = MaterialTheme.typography.h6
        )
        textComposable(lastName)
    }
}
```

컴포저블은 `@Composable Scope.() -> A` 타입을 가질 수 있으며, 이는 특정 컴포저블에만 정보를 스코핑하는데 자주 사용됩니다:

```kotlin
inline fun Box(
    ...,
    content: @Composable BoxScope.() -> Unit
) {
    // ...
    Layout(
        content = { BoxScopeInstance.content() },
        measurePolicy = measurePolicy,
        modifier = modifier
    )
}
```

언어적으로 보면, 타입은 컴파일러에게 정적 검사를 수행할 수 있는 정보를 제공하고, 때로는 편리한 코드를 생성하며, 런타임에서 데이터 사용을 제한하고 세분화하는 역할을 합니다.
`@Composable` 어노테이션은 함수가 런타임에서 어떻게 검증되고 사용되는지를 변경하므로, 컴포저블은 스탠다드 함수와는 다른 타입으로 간주됩니다.