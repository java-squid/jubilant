# Kotlin Collection vs Kotlin Sequence
## Kotlin Collection vs Kotlin Sequence
* Collection과 Sequence를 비교하기 위하여 Crayon Factory 예시를 들어보고자 함
    * Crayon Factory Example: 크래용을 담고 있는 랜덤 박스에서, 원하는 색상만 골라낸 다음에, 해당 색상 크래용에다가 라벨을 붙이고, 다시 해당 라벨 중 가장 왼쪽에서 5개 있는 크래용을 선별하여 박스에 넣는다.
* Crayon Factory를 Collection으로 구현하면 다음과 같음
    * 매 작업마다 별도의 Array(그림에서는 Box)에 중간 결과값을 넣고, 새로운 작업을 시작할 때마다 이전 Array를 unboxing하여 작업을 수행하고 있음

![Image1](https://typealias.com/img/guides/sequences-illustrated-guide/full-process-inefficient-40p.png)

* 이를 코드로 구현하면 다음과 같음.
    * 일단 아래와 같은 코드가 기본적으로 제공되어 있다고 생각하자.
```kotlin=
data class Crayon(
    val color: String, 
    val label: String? = null
)
```
```kotlin=
val BigBoxOfCrayons = setOf(
    Crayon("marigold"),
    Crayon("lime"),
    Crayon("red"),
    Crayon("yellow"),
    Crayon("blue"),
    Crayon("orange"),
    Crayon("grape"),
    Crayon("white"),
    Crayon("red-violet"),
    Crayon("pond"),
    Crayon("cinnamon"),
    Crayon("neon lightning"),
    Crayon("metal"),
    Crayon("violet"),
    Crayon("charcoal"),
    Crayon("brick"),
    Crayon("green"),
    Crayon("silver")
)
```
```kotlin=
val includedColors =
    setOf("brick", "marigold", "neon lightning", "orange", "red", "red-violet", "white", "yellow")
```
### Kotlin Collection
* Kotlin Collection으로 처리하는 코드는 다음과 같음.
    * filter, map, take는 **collection operations** 인데, collection에 applying some forms of transformation을 해서 또 다른 collection을 render하는 것이다.
    * filter, map, take는 각각 Collection Operation이라고 부르며, 이 operations를 연결하는 것을 Collection Operation Chain이라고 부른다.
    * FP가 다 그렇지만, 코드가 declarative하고, 내부 구현은 Kotlin library가 다 해준다.
    * 단점은 각 operation 마다 새로운 collection(여기서는 ArrayList)을 생성하고 파기하므로 이에 들어가는 비용이 중복된다는 점이다.
```kotlin
val fireSet = BigBoxOfCrayons               // 1. Start with the big box
    .filter { it.color in includedColors }  // 2. Filter out colors that don't apply
    .map { it.copy(label = "New!") }        // 3. Label the crayons
    .take(5)                                // 4. Take just the first five crayons
    .toSet()                                // 5. Collect them into their final box
```
* 사진으로 나타내면 다음과 같음.
![Image2](https://typealias.com/img/guides/sequences-illustrated-guide/full-process-inefficient-code-40p.png)

* 즉, 도식화하면, 크래용 전체 콜렉션에 대해 하나의 연산이 이루어지고 이에 대한 결과값을 ArrayList로 반환한다. 그리고, 그 다음 연산을 반환받은 ArrayList를 활용하여 수행한 뒤, 새로운 ArrayList로 반환하는 형태이다.)

### Kotlin Sequence
* ArrayList로 shoveling data하는게 너무 expensive operation이라면, operation chain 별로 iteration 하는 것이 아니라, 크래용 원소마다 iteration 하는 것이 어떨까?

![Image3](https://typealias.com/img/guides/sequences-illustrated-guide/full-process-efficient-40p.png)

![Image4](https://typealias.com/img/guides/sequences-illustrated-guide/full-process-efficient-trace-flow-40p.png)

* 하나의 크래용에 대해 실행해야 하는 모든 연산을 수행한 다음, 다음 크래용으로 넘어가는 효율적인 구조이다.
* 이를 코드로 표현하면 다음과 같다.

```kotlin=
val fireSet = BigBoxOfCrayons
    .asSequence()  // <--------------------- the one easy change
    .filter { it.color in includedColors }
    .map { it.copy(label = "New!") }
    .take(5)
    .toSet() // executing!
```
* Eager -> lazy 변경을 쉽게 하기 위해 기존 Collection에 대한 chain calls 앞에 asSequence() 만 붙여주면 만사 OK다.

![Image5](https://typealias.com/img/guides/sequences-illustrated-guide/full-process-efficient-code-40p.png)

* 하나의 크래용에 대해 전체 연산을 apply하고, 그 다음 크래용에 전체 연산을 apply하고, ... , 반복한 뒤 그 결과값을 하나의 ArrayList로 집어넣어 반환하게 된다.
* 이러한 과정을 Collection과 Sequence 별로 비교하면 다음과 같다.

![Image6](https://typealias.com/img/guides/sequences-illustrated-guide/order-of-operations.png)

* In the collection operation chain, the letters are grouped together - the line passes through all the F’s and then all the M’s, and then all the T’s. In other words, all of the filtering happens, then all of the mapping happens, then all of the taking happens.
* On the other hand, in the sequence operation chain, the colors are grouped together - the line passes through all of the marigold circles, then all of the lime circles, then all of the red circles, and so on. This means that each crayon is processed entirely before moving onto the next.

## Sequence 동작 원리 알아보기
* Kotlin의 Sequence는 하나의 원소에 대하여 전체 연산을 적용하는 것으로, ```one item at a time``` 이라고 부를 수 있다. 이는 Iterable과 유사하다.
* Iterable과 Sequence 인터페이스는 실제로 비슷하다.

```kotlin=
public interface Iterable<out T> {
    public operator fun iterator(): Iterator<T>
}

public interface Sequence<out T> {
    public operator fun iterator(): Iterator<T>
}
```
* Iterable과 Sequence의 차이점은 구조에 있다.
    * Iterable은 List와 Set 두 자식으로 나뉘어지고, 다시 List는 ArrayList 등을 가지고, Set은 HashSet 등을 가진다. 즉, 구조화(structured) 되어 있다. 각 item은 서로 상관관계를 갖는다. 예를 들어, List에는 index가 있고, Set 원소 간에는 distinct하다는 특징이 있다.
    * Sequence는 item에 대해 어떠한 연산을 진행할 것인가에 대한 정의로 이루어져 있다. 예를 들어, filter operation에는 filteringSequence라는 구현체가 있다. map operation에는 TransformingSequence라는 구현체가 있다.

![Image7](https://typealias.com/img/guides/inside-kotlin-sequences/iterable-sequences-naming-uml.png)

* Sequence에서 우리는 다양한 operation을 wrapping 함으로써, 각 Iteration turn마다 적용해야 하는 전체 연산의 차례를 지정하고 생성할 수 있다.
* 이를 그림으로 굳이 표현하면 다음과 같다.

```kotlin=
val fireSet = BigBoxOfCrayons
    .asSequence()
    .filter { it.color in includedColors }
    .map { it.copy(label = "New!") }
    .take(5)   
```
![Image8](https://typealias.com/img/guides/inside-kotlin-sequences/seq-wrapping-take.png)

* 새로운 연산을 호출할 때마다, 이전 sequence를 또 한 번 wrapping하는 것이다.
* 실제로 이 코드를 다음과 같이 작성할 수도 있다.
```kotlin=
TakeSequence(
    TransformingSequence(
        FilteringSequence(
            BigBoxOfCrayons.asSequence(),
            true,
            { it.color in includedColors }
        ),
        { it.copy(label = "New!") }
    ),
    5
)
```
* 그렇다면 이 연산들은 어떻게 수행되는 것일까? Sequence Operation은 두 과정을 거친다.
    * The Building Phrase - intermediate operation: wrapping하는 과정이다.
        * filter, map, take(5)
    * The Execution Phrase - terminal operation: sequence의 Iterator가 실행되는 과정이다.
        * toSet()

```kotlin=
val fireSet = BigBoxOfCrayons
    .asSequence()                            // Building...
    .filter { it.color in includedColors }   // Building...
    .map { it.copy(label = "New!") }         // Building...
    .take(5)                                 // Building...
    .toSet()                                 // Executing!
```

* Intermediate Operation만 저장하고 각기 다르게 Terminal operation을 수행할 수 있다.

```kotlin=
    val fireSequence = BigBoxOfCrayons
        .asSequence()                            // Building...
        .filter { it.color in includedColors }   // Building...
        .map { it.copy(label = "New!") }         // Building...
        .take(5)                                 // Building...

    val fireSet = fireSequence.toSet()           // Executing!
    val crayonCount = fireSequence.count()       // Executing!
```

* Intermediate Operation은 오직 Sequence만 반환한다.
* Terminal Operation은 List, Set과 같은 Collection이나 String과 같은 single value, 또는 Boolean이나 Int와 같은 primitive type을 반환할 수도 있다.

* 예를 들어, 다음과 같이 코드를 구현하면 println()은 수행되지 않는다. 왜냐하면 terminal operation이 수행되지 않았기 때문이다.

```kotlin
sequenceOf("Hello", "Kotlin", "World")
    .onEach { println("I spy: $it") }
```

* 다음과 같이 toSet()을 통해 terminal operation을 수행해야 println()이 수행된다.

```kotlin=
sequenceOf("Hello", "Kotlin", "World")
    .onEach { println("I spy: $it") }
    .toSet()
```

* 여기서 forEach는 terminal operation이고 이 점에서 onEach와 다르다는 사실을 알아두면 좋겠다.

```kotlin=
sequenceOf("Hello", "Kotlin", "World")
    .forEach { println("I spy: $it") }
```

### Custom Sequence를 만들어보자
* 위에서 println으로 I spy...를 반환하는 intermediate operation을 만들어보자.
    * Creating a class that implements Sequence, and then…
    * Creating an extension function that calls that class’ constructor.
* 다음과 같이 사용하고 싶다고 가정해보자.

```kotlin=
sequenceOf("Hello", "Kotlin", "World")
    .spy()                            
    .toSet()
```
* Step 1: Sequence Interface를 implement 한다.
```kotlin=
class SpyingSequence<T> : Sequence<T> {
    override fun iterator() = TODO()
}
```
* Step 2: Wrapping another sequence
    * 우리의 sequence는 이전 sequence를 wrapping 해야 하므로, 이전 sequence를 constructor에 매개변수로 받아들여야 한다.
    * 여기서는 underlyingSequence로 쓰였다. underylingSequence의 Iterator로 wrapping 해야 한다.
```kotlin=
class SpyingSequence<T>(private val underlyingSequence: Sequence<T>) : Sequence<T> {
    override fun iterator() = underlyingSequence.iterator()
}
```
* Step 3: Adding the Functionality
    * hasNext와 next를 구현해 주어 functionality가 있는 코드를 구현해주어야 한다.
```kotlin=
class SpyingSequence<T>(private val underlyingSequence: Sequence<T>) : Sequence<T> {
    override fun iterator() = object : Iterator<T> {
        val iterator = underlyingSequence.iterator()

        override fun hasNext() = iterator.hasNext()
        
        override fun next(): T {
            val item = iterator.next()
            println("I spy: $item")
            return item
        }
    }
}
```
* Step 4: Extension Function으로 활용하도록 한다.
    * sequence에 spy로 선언하여, SpyingSequence에 현재 클래스를 주입하도록 한다.
```kotlin=
fun <T> Sequence<T>.spy() = SpyingSequence(this)
```
* 여기까지 했으면 다음과 같이 사용할 수 있다.

```kotlin=
val fireSet = BigBoxOfCrayons
    .asSequence()
    .filter { it.color in includedColors }
    .map { it.copy(label = "New!") }
    .spy()  // <---- here's our sequence!   
    .take(5)
    .toSet()
```

* 요약하자면, Sequence의 Extension Function은 Sequence Class(es) 로 구현되어 있다.
    * 예시
        * Sequence Extension Function	Sequence Class(es)
        * .map()	TransformingSequence()
        * .mapIndexed()	TransformingIndexedSequence()
        * .mapIndexedNotNull()	FilteringSequence(TransformingIndexedSequence())
        * .filter()	FilteringSequence()
        * .filterNot()	FilteringSequence()
        * .filterNotNull()	FilteringSequence()
        * .filterIsInstance()	FilteringSequence()
        * .filterIndexed()	TransformingSequence(FilteringSequence(IndexingSequence()))

### Kotlin Sequence API vs Kotlin Collection API 비교
```
// _Collections.map
public inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R> {
    return mapTo(ArrayList<R>(collectionSizeOrDefault(10)), transform)
}

// _Collections.mapTo
public inline fun <T, R, C : MutableCollection<in R>> Iterable<T>.mapTo(destination: C, transform: (T) -> R): C {
    for (item in this)
        destination.add(transform(item))
    return destination
}

// _Sequences.map
public fun <T, R> Sequence<T>.map(transform: (T) -> R): Sequence<R> {
    return TransformingSequence(this, transform)
}

// Sequences.TransformingSequence
internal class TransformingSequence<T, R>
constructor(private val sequence: Sequence<T>, private val transformer: (T) -> R) : Sequence<R> {
    override fun iterator(): Iterator<R> = object : Iterator<R> {
        val iterator = sequence.iterator()
        override fun next(): R {
            return transformer(iterator.next())
        }

        override fun hasNext(): Boolean {
            return iterator.hasNext()
        }
    }

    internal fun <E> flatten(iterator: (R) -> Iterator<E>): Sequence<E> {
        return FlatteningSequence<T, R, E>(sequence, transformer, iterator)
    }
}
```
* 위 코드를 보면 알 수 있지만, Stream API는 inline function이고 람다를 바로 실행한다. 이와 달리, Sequence API는 일반 함수이고 람다를 저장한다는 특징이 있다.
* [inline function](https://agrawalsuneet.github.io/blogs/inline-function-kotlin/)은 매개 변수로 SAM interface를 받는 함수를 익명 함수롤 통하여 호출하게 되면, 해당 함수 호출마다 새로운 객체를 생성하게 되므로 오버헤드가 발생하게 된다. [참고](https://medium.com/@mook2_y2/%EC%BD%94%ED%8B%80%EB%A6%B0-%EC%9E%85%EB%AC%B8-%EC%8A%A4%ED%84%B0%EB%94%94-14-inline-functions-f3628bb347ca)
   * 하지만 캡쳐링하지 않은 람다, 즉 람다 외부의 지역 변수를 참조하지 않은 람다는 람다 자체가 stateless 하므로 람다를 따로 저장해두었다가 계속 사용하는 방식으로 바이트코드를 컴파일 할 수 있게 된다.
   * 이와 달리 캡쳐링한 람다, 즉 람다 외부의 지역 변수를 참조하는 람다는 람다 자체가 stateful 하므로 람다가 가지고 있는 변수가 각기 다르므로 람다를 미리 저장해두었다가 재사용하는 방식으로 사용할 수 없으므로 계속 함수 호출마다 새롭게 객체를 생성해야만 할 것이다.
   * inline function은 바이트코드로 컴파일할 때 람다 함수의 본문을 함수를 호출한 곳으로 복사하여 바이트코드를 생성하도록 강제하는 옵션이다.
   * 만약 함수의 본문이 그리 크지 않다면 바이트코드가 많아져서 무거워질 일도 없으니 캡처링하지 않은 람다를 받는 함수를 인라인 함수로 만들면 속도의 향상이 있을 수 있겠다.
* Collection의 Stream API는 캡쳐링하지 않은 람다를 받으므로 인라인 함수로 구현할 수 있고, Sequence의 Stream API는 람다를 실행하지 않고 새로운 클래스에 저장하게 되므로 람다가 stateful 하게 되어 인라인 함수로 구현할 수 없다. Sequence 의 경우는 map 이 inline 이 아닌데, 해당 lambda 를 저장하고 있다가 final operation 이 수행되었을 때 이 lambda 를 사용해야 하기 때문이다.
* 따라서 Collection을 사용하게 되면 람다를 실행할 때마다 새로운 스트림을 생성해야 하고, Sequence를 사용하게 되면 람다를 실행할 때마다 새로운 익명 객체를 생성해야 한다.
* 원소의 수가 적을 때는 람다 객체를 만드는 것이 오버헤드가 더 크고, 원소의 수가 많을 때는 스트림을 매번 생성하는 것이 오버헤드가 더 크다. 결론적으로, Collection의 크기가 클 때 asSequence()를 사용하여 시퀀스를 사용해야 한다.

* 하기 익명 객체 생성의 예시는 다음 [StackOverFlow](https://stackoverflow.com/questions/48140788/kotlin-higher-order-functions-costs) 예시를 참조한다.
```java
// 익명 객체 생성 예시
// Definition of higher-order function and caller code:

fun hoFun(func: (Int) -> Boolean) {
    func(1337)
}

//invoke with lambda
val mod = 2
hoFun { it % mod == 0 }

// Bytecode Java Representation:

public static final void hoFun(@NotNull Function1 func) {
  Intrinsics.checkParameterIsNotNull(func, "func");
  func.invoke(1337);
}

final int mod = 2;
hoFun((Function1)(new Function1() {

     public Object invoke(Object var1) {
        return this.invoke(((Number)var1).intValue());
     }

     public final boolean invoke(int it) {
        return it % mod == 0;
     }
}));
```
* Lambda는 Bytecode에서 Function 형태로 컴파일 된다. inline을 활용하면 훨씬 간단해진다.
```java
int mod = 2;
int it = 1337;
if (it % mod == 0) {
   ;
}
```

### 그러면 언제 Sequence를 사용해야 하는가?
#### Performance-based
* Operation Count
    * Collection Operation Chain은 operation chain 간의 불필요한 ArrayList를 생성함
    * 이와 달리, Sequence는 operation chain 간의 불필요한 ArrayList를 생성하지 않음
    * 따라서 operation counts가 많을 경우에는 Sequence를 사용하는 것이 비용 절감에 있어서 바람직함
    * 사실 counts가 중요한 것이 아니고, CPU-incentive한 operation이 얼마나 있는가에 따라 결과가 달라질 것이다.

* Short-Circuiting Operation
    * 앞선 예시에서 take(5)를 생각하면, 결국 최상위 5개 원소만 뽑는 것인데, 이러면 최상위 5개 원소에 대해서만 연산을 진행하면 된다.
    * 따라서 모든 원소에 대해 operation chain을 일일이 걸어주는 collection보다, 원소 별로 operation chain을 걸어주는 sequence가 훨씬 short-circuiting operation에서는 유리하다.
    * 앞의 예시에서 collection은 31번 operation을 수행했고, sequence는 18번 operation을 수행했다.
    * take 말고도, contains(), indexOf(), any(), none(), find(), first()도 short-circuiting operation이 되는 대표적인 예시라고 꼽아볼 수 있다.

#### Resizable Collection

* 다음 예시를 살펴보자.
    ```kotlin
    val shades = colors
    .asSequence()
    .map { it.toGrayscale() }
    .toList()  
    ```
    * 위 코드에 따르면, colors가 grayscale로 map 되면, grayscale 결과값은 ArrayList로 변환된다.
    * ArrayList는 JVM에서 regular Java array로 동작하는데, 이 array는 fixed size로 동작한다.
    * 즉, 다시 말해서 array 생성 시점에서 크기를 선택하는데, array의 크기를 절대 늘릴 수 없다.
    * ArrayList는 initial capacity가 있고, initial capacity가 늘어나는 만큼의 새로운 원소가 추가될 경우 underlying array를 새로 만들고, 기존 array에서 기존 원소를 복사하고 뒤에 새로운 원소를 붙인다. 새롭게 만들어지는 array의 initial capacity는 현재 담고자 하는 원소의 150% 만큼 할당한다.


    ![Image8](https://typealias.com/img/guides/when-to-use-sequences/arraylist-empty.png)
    
    * add 함수를 실행했다!
    ![Image9](https://typealias.com/img/guides/when-to-use-sequences/arraylist-full.png)
    
    * 크기를 150% 늘린 새로운 array를 생성하고, 기존 원소를 복사해서 새롭게 붙인다.
    ![Image10](https://typealias.com/img/guides/when-to-use-sequences/arraylist-copy.png)
    
    * 이제 뒤에 새로운 원소를 추가해본다.
    ![Image11](https://typealias.com/img/guides/when-to-use-sequences/arraylist-vacancy.png)
    
    * 기존 array 크기를 10개라고 가정해보자. 그러면 11번째 원소를 넣으려고 하면, 크기가 15개인 새로운 array를 만든다. 이 과정은 array 생성 시간과 공간 차지를 비효율적으로 만든다.

* map 함수의 실행
    * Collection 버전: map 함수는 이전 array에서 정확히 같은 size로 연산을 수행한 뒤 새로운 array를 만든다. 무조건 이전 array와 새로운 array는 array의 크기가 같음을 보장할 수 있다. 이는 collection이 이전 array의 size를 정확하게 알기에 가능한 일이다.
    ![Image12](https://typealias.com/img/guides/when-to-use-sequences/isometric-collection.png)
    * Sequence 버전: Iterator에 바탕을 두어 implement 되었으므로, 이전 array의 size를 정확하게 알 수 없다. 따라서 Sequence.toList를 호출할 때 ArrayList의 initial capacity로 설정한다. 새로운 아이템이 Iterator를 거치면서 계속 추가될 수록, array size를 늘리고 array elements를 복사하는 과정이 비효율적으로 중복 발생하게 된다.
    ![Image13](https://typealias.com/img/guides/when-to-use-sequences/isometric-sequence.png)

* 결론
    * forEach처럼 elements collection을 하지 않을 때는 sequence가 유리하나, toList나 toSet 처럼 결국 resizable collection을 활용해서 elements collection을 해야 하는 경우에는 collection이 유리하다.

#### Stateless and Stateful operations
* Sequence는 one item at a time이지만, sorting 기능도 제공한다 (좀 놀랍죠?)
* operation chain에서 어떻게 sequence는 order를 바꿀 수 있을까?

```kotlin=
val fireSet = BigBoxOfCrayons
    .asSequence()
    .filter { it.color in includedColors }
    .sortedBy { it.color }                  
    .map { it.copy(label = "New!") }
    .take(5)
    .toSet()
```

* 그 방법은 다음과 같다.
    * sequence를 collection으로 변환한다.
    * collection을 sorting한다.
    * collection의 iterator를 활용하여 sequence operation으로 바꾼다.

* sequence의 sorted 함수를 살펴보면 다음과 같다.
```kotlin=
public fun <T : Comparable<T>> Sequence<T>.sorted(): Sequence<T> {
    return object : Sequence<T> {
        override fun iterator(): Iterator<T> {
            val sortedList = this@sorted.toMutableList()
            sortedList.sort()
            return sortedList.iterator()
        }
    }
}
```
* sorted가 intermediate operation이지만, toMutableList라는 terminal operation을 내부적으로 호출한다.
    * 만약 4GB log file을 sequence로 다루고 있다고 생각해보자. 이를 sort한다고 보면.. 결국 ArrayList로 4GB 되는 인스턴스를 새로 생성하는 꼴이 된다. 이게 바로 terminal operation인 toMutableList를 실행하고 이를 sort한 뒤 다시 되돌리는 과정이다.
    * Stateless operation이란, sequence에 대한 지식이 필요가 없다는 뜻이다. 다시 말해서, sequence items는 각기 독립적으로 operation이 이루어질 수 있다. 즉, vertical processing이 가능한 것이다. 예시로는 map, filter, take가 있다.
    * Stateful operation은 시작 전에 이전 sequence에 대한 지식이 필요하다는 뜻이다. 다시 말해, horizontal processing이 가능하다는 것이다. 예시로는 toList와 sorted, HashSet 등을 꼽아볼 수 있다.
    * Sequence는 stateless operation에 적합하게 설계되었다. 이와 달리, Collection은 stateful operation에 적합하다고 말할 수 있겠다.

#### Other Considerations: Generators
* collection에는 isEmpty나 contains, size 같은 함수나 프로퍼티가 존재하지만, sequence는 iterator로 구성되어 있어 이러한 properties나 function이 존재하지 않는다. 따라서 생산성 측면에서 collection의 손을 들어줄 수 있겠다.
![Image14](https://typealias.com/img/guides/when-to-use-sequences/sequence-collection-uml.png)

* 사실 sequence를 사용하면 infinite sequence를 만들 수 있다. [Sequence는 terminal operation을 진행하기 전까지 연산이 일어나지 않기 때문에 무한히 원소를 생성하는 방식으로 sequence를 만들어도 최종 결과물이 유한한 형태로 제한된다면 overflow 문제는 없다.](https://medium.com/@mook2_y2/%EC%BD%94%ED%8B%80%EB%A6%B0-%EC%9E%85%EB%AC%B8-%EC%8A%A4%ED%84%B0%EB%94%94-15-sequences-52cfca1805c8) 다음의 코드를 보자.
```kotlin=
val sequence = generateSequence { Random.nextInt() }
```
* take 함수를 활용하여 연산 호출에 제한을 걸어보자.
```kotlin=
sequence.take(10).forEach(::println)
```
* sequence generator는 무한 루프를 돌도록 만들 수 있기 때문에, 요청 전까지 production을 defer하고 싶다면 쓰일 수 있다. sequence를 end하면 null이 반환된다. 이는 코루틴 기반 제너레이터를 활용하는 측면에서 유용하다고 볼 수 있다고 한다.

#### Other Considerations: Available Operations
* collection은 collection에 대한 지식을 알기에 left-fold나 right-fold operation을 모두 지원한다.
* 이와 달리 sequence는 iterator를 통해 이루어지므로 right-fold operation은 보통 지원하지 않는다.
```
Sequences support…	Sequences do not support…
fold()	foldRight()
reduce()	reduceRight()
take()	takeLast()
drop()	dropLast()
```
* 하기 사항도 지원하지 않는다.
    * Slicing
    * Reversing
    * Set theory-ish operations like union() and intersect()
    * Many operations that get a single element out of a sequence, such as getOrNull() or random()
* sequence에서 sorted를 구현하려면 다음과 같은 collection 변환의 꼼수를 부리면 되지만, 이럴 거면 collection을 애시당초 쓰면 되겠다.
```kotlin=
fun <T> Sequence<T>.reversed() = object : Sequence<T> {
    override fun iterator(): Iterator<T> =
        this@reversed.toMutableList().also { it.reverse() }.iterator()
}
```

### Sequence vs Java Stream
* 자바 스트림은 자바 8 이상에만 지원되며, 일부 안드로이드 버전에서는 자바 8을 지원하지 않는다.
* 코틀린 쓸거면 자바 스트림을 쓸 이유가 없다.
* 자바 스트림은 코틀린 시퀀스에 대비해서 병행 처리가 가능하다. sequence는 말 그대로 sequential하기 때문이다.
* 코드에서 operation chain이 CPU-incentive process를 해야 하는 경우, 자바의 parallel stream을 고려해볼 수 있다.
    * Kotlin Corutines, Flow는 sequence를 parallel하게 처리하도록 만들어주는 커스텀 라이브러리다.

### 더 공부할 거리 (다음 Chapter)
* Java Stream과 Kotlin Sequence의 장,단점 비교 (null safety, performance 등)
* 왜 Kotlin Sequence는 parallel이 안되는데 Java Stream은 될까요? (Keyword: Collection)
* Kotlin Coroutine vs Kotlin Flow vs Kotlin Sequence 비교해보기

### Reference
* https://typealias.com/guides/kotlin-sequences-illustrated-guide/
* https://proandroiddev.com/java-streams-vs-kotlin-sequences-c9ae080abfdc
* https://typealias.com/guides/when-to-use-sequences/
* https://bcp0109.tistory.com/359
* https://blog.frankel.ch/kotlin-collections-sequences/
* https://velog.io/@dhwlddjgmanf/Kotlin-asSequence-vs-non-asSequence
* https://www.codementor.io/@kotlin_academy/effective-kotlin-use-sequence-for-bigger-collections-with-more-than-one-processing-step-jqgbmnllp
* https://umbum.dev/599
* https://jo5ham.tistory.com/14
* https://codechacha.com/ko/kotlin-sequences/
* https://discuss.kotlinlang.org/t/kotlin-sequences-vs-java-streams/14415
* https://minz.dev/25
* https://medium.com/mobile-app-development-publication/kotlin-flow-a-much-better-version-of-sequence-d2555ba9eb94
