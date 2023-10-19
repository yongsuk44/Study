# Generics: `in`, `out`, `where`

Kotlin의 클래스는 Java와 마찬가지로 타입 파라미터를 가질 수 있으며, 해당 클래스의 인스턴스를 만드려면 타입 인자를 제공하면 됩니다.  
하지만 파라미터가 생성자에서 타입이 추론이 된다면 타입 파라미터를 생략할 수 있습니다.

```kotlin
class Box<T>(t: T) {
    var value = t
}

val box: Box<Int> = Box(1)
val box1 = Box(1)
```

---

## Variance

Java의 제네릭은 클래스, 인터페이스, 함수 등에서 동일한 코드를 재사용하고 싶을 때 여러 타입을 지원하는 중요한 기능 중 하나입니다.  
이처럼 중요한 기능 중 하나인 제네릭 타입을 사용할 때, Java 타입 시스템은 와일드카드 타입을 필요로 합니다.  
(Kotlin은 와일드카드 대신 [declaration-site variance](#declaration-site-variance)와 [Type projection](#type-projections)가 존재합니다.)

자세히 알아보면 Java의 제네릭 타입은 불변(invariance)입니다.  
타입 불변성은 제네릭 타입을 사용하는 클래스, 인터페이스에는 해당 타입의 상위, 하위를 대입할 수 없고 오직 일치하는 타입만을 대입하는 것을 의미합니다. 

이처럼 Java의 `List<T>`를 활용한 `List<String>`와 `List<Object>`는 서로 다른 타입으로 취급됩니다.  
만약 `List<String>`과 `List<Object>`가 같은 타입으로 취급된다면, 런타임에서 문제가 발생할 수 있기 때문입니다.

```java
List<String> strs = new ArrayList<String>();
List<Object> objs=strs; // A compile-time error here saves us from runtime exception later.
objs.add(1); // Put an Integer into a list of Strings
String s=strs.get(0); // ClassCastException: Cannot cast Integer to String
```

위 상황와 같이 Java는 런타임 안전성을 확보하기 위해 일부 타입 관련 작업을 제한합니다.  
이로 인해 `Collection`의 `addAll()`과 같은 메서드의 시그니처가 복잡해질 수 있습니다.

`addAll()`을 직관적으로 작성하면 다음과 같을 수 있을겁니다.

```java
interface Collection<E> {
    void addAll(Collection<E> items);
}
```

그러나 위처럼 작성되면, 아래와 같은 작업을 할 수 없게 됩니다.

```java
void copyAll(Collection<Object> to,Collection<String> from){
    to.addAll(from);
}
```

왜냐하면 `Collection<String>`은 `Collection<Object>`의 하위 타입임을 확인할 수 없기 때문입니다.  
그래서 `addAll()`의 실제 시그니처는 다음과 같습니다.

```java
interface Collection<E> {
    void addAll(Collection<? extends E> items);
}
```

`? extends E` 타입의 와일드카드는 Java에서 공변성(covariance)을 구현하게 도와줍니다.

`? extends E` 와일드카드는 `E`와 `E`의 하위 타입들을 안전하게 읽을 수 있게(read) 해주지만,   
알려지지 않은 `E`의 하위 타입에 어떤 객체가 부합하는지 알 수 없기에 쓸 수는(write) 없게 만듭니다.

`Collection<String>`은 `Collection<? extends Object>`의 하위 타입으로 취급되기 때문에,  
상한(extends-bound)을 가진 와일드카드는 타입을 공변(covariance)으로 만듭니다.

반대로 컬렉션에 항목을 추가만 할 수 있다면, `Collection<? super String>`로 선언하여 `String`또는 그 상위 타입만을 수용할 수 있도록 할 수 있습니다.

`Collection<? super String>`와 같은 와일드카드를 반공변(contravariance)이라고 부릅니다.  
이 와일드카드는 `add(String)`, `set(int, String)`을 호출할 수 있습니다.

그러나 `List<T>`에서 `T`를 반환을 호출하면 `String`이 아닌, `Object`를 얻게 됩니다.
