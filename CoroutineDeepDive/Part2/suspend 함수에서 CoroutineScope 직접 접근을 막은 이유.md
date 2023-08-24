## suspend 함수에서 `CoroutineScope`에 직접 접근을 막은 이유

### 1. Scope와 Lifecycle

`CoroutineScope`는 코루틴의 범위와 생명 주기를 정의하고 관리하는 역할을 가지며 이는 코루틴을 제어하고 생명 주기를 관리하기 위한 목적으로 사용됩니다.

그러나 suspend 함수는 이런 특정 Scope 밖에서도 동작될 수 있으므로 `CoroutineScope`에 직접 접근하는 것은 의미가 없는 행위 입니다.

### 2. 캡슐화

코루틴은 특정 작업을 비동기로 수행하며 `coroutineScope`는 이러한 코루틴 작업들의 실행 범위를 제어합니다.

만약 suspend 함수가 `coroutineScope`에 직접 접근할 수 있다면 Lifecycle과 Scope 제어가 복잡해질 수 있어 캡슐화가 깨질 수 있는 위험이 있습니다.

### 3. 결합도

`coroutineScope`에 직접 접근하는 것을 허용하면 suspend 함수와 `coroutineScope` 사이의 결합도가 증가하게 됩니다.  
이는 유연성과 재사용성을 저하시킬 수 있습니다.