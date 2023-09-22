# Hot and cold data sources

## Hot vs cold

### Hot Stream

1. 데이터를 미리 생성하고 저장하려는 경향이 있는데, 이는 데이터 생성의 주체와 소비의 주체가 분리되기 때문입니다.
2. 소비자의 구독 여부와 관계없이 데이터를 계속해서 생성합니다.
3. 생성된 데이터를 저장하고 있다가 필요할 때 제공할 수 있습니다.   
   예를 들어 `List`는 미리 생성된 요소들을 저장하고 있습니다.

### Cold Stream

1. 실제 데이터가 필요할 때까지 어떠한 연산도 수행하지 않습니다.
2. 구독자가 데이터를 요청할 때만 해당 데이터를 생성하고 연산을 수행합니다.
3. 생성된 데이터를 저장하지 않습니다. 대신 필요할 때마다 다시 생성하거나 연산을 수행합니다.  
   예를 들어 `Sequence`는 요청에 따라 데이터를 생성합니다.

```kotlin
fun main() {
    val l = buildList {
        repeat(3) {
            add("User$it")
            println("L: Added User")
        }
    }

    val l2 = l.map {
        println("L: Processing")
        "Processed $it"
    }

    val s = sequence {
        repeat(3) {
            yield("User$it")
            println("S: Added User")
        }
    }

    val s2 = s.map {
        println("S: Processing")
        "Processed $it"
    }
}

// L: Added User
// L: Added User
// L: Added User
// L: Processing
// L: Processing
// L: Processing
```

결과적으로 Cold 스트림은 다음과 같은 특징을 지님을 알 수 있습니다.

- 무한할 수 있습니다. 즉, 요청을 받을 때 마다 데이터를 생성하므로 이론적으로 무한한 길이의 데이터를 생성할 수 있습니다.
- 최소한의 연산만을 수행합니다. 실제로 데이터가 필요한 시점에만 연산을 수행합니다. 이는 연산의 최적화 및 효율성을 의미합니다.
- 더 적은 메모리를 사용합니다. 중간 결과를 저장하는 컬렉션을 만들 필요가 없기에 메모리 사용량이 줄어듭니다.