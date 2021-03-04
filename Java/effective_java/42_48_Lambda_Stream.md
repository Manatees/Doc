# Lambda 和 Stream

在 Java 8 中，增加了函数接口 ( functional interface )、`Lambda` 和方法引用 (Method reference )，使得创建函数对象 ( function object ) 变得很容易。与此同时，还增加了 `Stream` API，为处理数据元素的序列提供了类库级别的支持。在本章中，将讨论如何最佳地利用这些机制。



## 42. Lambda 优先于匿名类

根据以往的经验，是用带有单个抽象方法的接口 (或者，几乎都不是抽象类) 作为函数类型 (function type)。它们的实例称作函数对象(function object)，表示函数或者要采取的动作。自从 1997 年发布JDK 1.1 以来，创建函数对象的主要方式是通过匿名类(anonymous class ，详见第24 条) 。下面是一个按照字符串的长度对字符串列表进行排序的代码片段，它用一个匿名类创建了排序的比较函数 (加强排列顺序):

```java
// Anonymous class instance as a function object - obsolete!
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

匿名类满足了传统的面向对象的设计模式对函数对象的需求，最著名的有策略 (Strategy) 模式［ Gamma95 ］ 。`Comparator` 接口代表一种排序的抽象策略 (abstract strategy)；上述的匿名类则是为字符串排序的一种具体策略(concrete strategy)。但是，匿名类的烦琐使得在 Java 中进行函数编程的前景变得十分黯淡。

在 Java 8 中，形成了 “带有单个抽象方法的接口是特殊的，值得特殊对待” 的观念。这些接口现在被称作函数接口 (functional interface),  Java 允许利用 `Lambda` 表达式 (Lambda expression ，简称 `Lambda`) 创建这些接口的实例。`Lambda` 类似于匿名类的函数，但是比它简洁得多。以下是上述代码用 `Lambda` 代替匿名类之后的样子。样板代码没有了，其行为也十分明确:

```java
// Lambda expression as function object (replaces anonymous class)
Collections.sort(words, 
         (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

注意， `Lambda` 的类型 (`Comparator<String>`)、其参数的类型 ( `s1` 和 `s2` ，两个都 是 `String`) 及其返回值的类型(`int`)，都没有出现在代码中。编译器利用一个称作类型推导 (type inference )的过程，根据上下文推断出这些类型。在某些情况下，编译器无法确定类型，你就必须指定。类型推导的规则很复杂：在JLS [JLS,18 ］中占了整章的篇幅。几乎没有程序员能够详细了解这些规则，但是没关系。**删除所有Lambda 参数的类型吧，除非它们的存在能够使程序变得更加清晰**。如果编译器产生一条错误消息，告诉你无法推导出 `Lambda` 参数的类型，那么你就指定类型。有时候还需要转换返回值或者整个 `Lambda` 表达式，但是这种情况很少见。

关于类型推导应该增加一条警告。第 26 条告诉你不要使用原生态类型，第 29 条说过要支持泛型类型，第 30 条说过要支持泛型方法。在使用 `Lambda` 时，这条建议确实非常重要，因为编译器是从泛型获取到得以执行类型推导的大部分类型信息的。如果你没有提供这些信息，编译器就无法进行类型推导，你就必须在 `Lambda` 中手工指定类型，这样极大地增加了它们的烦琐程度。如果上述代码片段中的变量 `words` 声明为原生态类型 `List` ，而不是参数化的类型 `List<String>`，它就不会进行编译。

当然，如果用 `Lambda` 表达式（详见第 14 条和第 43 条）代替比较器构造方法 (comparator construction method)，有时这个代码片段中的比较器还会更加简练:

`Collections.sort(words, comparingInt(String::length));`

事实上，如果利用 Java 8 在 `List` 接口中添加的 `sort` 方法，这个代码片段还可以更加简短一些：

`words.sort(comparingInt(String::length));`

Java 中增加了 `Lambda` 之后，使得之前不能使用函数对象的地方现在也能使用了。例如，以第 34 条中的`Operation` 枚举类型为例。由于每个枚举的 `apply `方法都需要不同的行为，我们用了特定于常量的类主体，并覆盖了每个枚举常量中的 `apply` 方法。通过以下代码回顾一下：

```java
// Enum type with constant-specific class bodies & data (Item 34)
public enum Opeartion {
    PLUS("+") {
        public double apply(double x, double y) { return x + y;}
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y;}
    }, 
    TIMES("*") {
        public double apply(double x, double y) { return x * y;}
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y;}
    };
    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }
    @Override public String toString() { retrn symbol; }
    public abstract double apply(double x, double y);
}
```

由第 34 条可知，枚举实例域优先于特定于常量的类主体。`Lambda` 使得利用前者实现特定于常量的行为变得比用后者来得更加容易了。只要给每个枚举常量的构造器传递一个实现其行为的 `Lambda` 即可。构造器将 `Lambda` 保存在一个实例域中， apply 方法再将调用转给 `Lambda` 。由此得到的代码比原来的版本更简单，也更加清晰:

```java
// Enum with funciton object fields & constant-specific behavior
public enum Operation {
    PLUS("+", (x, y) -> x + y),
    MINUS("-", (x, y) -> x - y),
    TIMES("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);
    
    private final String symbol;
    private final DoubleBinaryOperator op;
    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }
    @Override public String toString() { return symbol; }
    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

注意，这里给 `Lambda` 使用了 `DoubleBinaryOperator` 接口，代表枚举常量的行为。这是在 `java.util.function` (详见第44 条) 中预定义的众多函数接口之一。它表示一个带有两个 `double` 参数的函数并返回一个`double` 结果。

看看基于 `Lambda` 的 `Operation` 枚举，你可能会想，特定于常量的方法主体已经形同虚设了，但是实际并非如此。与方法和类不同的是， `Lambda` 没有名称和文档；如果一个计算本身不是自描述的， 或者超出了几行， 那就不要把它放在一个 `Lambda` 中。对于 `Lambda` 而言，一行是最理想的， 三行是合理的最大极限。如果违背了这个规则，可能对程序的可读性造成严重的危害。如果 `Lambda` 很长或者难以阅读，要么找一种方法将它简化，要么重构程序来消除它。而且，传入枚举构造器的参数是在静态的环境中计算的。因而，枚举构造器
中的 `Lambda` 无法访问枚举的实例成员。如果枚举类型带有难以理解的特定于常量的行为，或者无法在几行之内实现，又或者需要访问实例域或方法，那么特定于常量的类主体仍然是首选。

同样地，你可能会认为，在 `Lambda` 时代，匿名类已经过时了。这种想法比较接近事实，但是仍有一些工作用`Lambda` 无法完成，只能用匿名类才能完成。`Lambda` 限于函数接口。如果想创建抽象类的实例，可以用匿名类来完成，而不是用 `Lambda` 。同样地，可以用匿名类为带有多个抽象方法的接口创建实例。最后一点， `Lambda` 无法获得对自身的引用。在 `Lambda` 中，关键字 `this` 是指外围实例，这个通常正是你想要的。在匿名类中，关键字 `this` 是指匿名类实例。如果需要从函数对象的主体内部访问它，就必须使用匿名类。

`Lambda` 与匿名类共享你无法可靠地通过实现来序列化和反序列化的属性。因此， 尽可能不要（除非迫不得已）序列化一个 `Lambda` （或者匿名类实例） 。如果想要可序列化的函数对象，如 `Comparator` ，就使用私有静态嵌套类（详见第24 条）的实例。

总而言之，从 Java 8 开始， `Lambda` 就成了表示小函数对象的最佳方式。千万不要给函数对象使用匿名类， 除非必须创建非函数接口的类型的实例。同时，还要记住， `Lambda` 使得表示小函数对象变得如此轻松，因此打开了之前从未实践过的在 Java 中进行函数编程的大门。



## 43. 方法引用优先于 Lambda

与匿名类相比， `Lambda` 的主要优势在于更加简洁。Java 提供了生成比 `Lambda` 更简洁函数对象的方法： 方法引用(method reference)。以下代码片段的源程序是用来保持从任意键到 `Integer` 值的一个映射。如果这个值为该键的实例数目那么这段程序就是一个多集合的实现。这个代码片段的作用是，当这个键不在映射中时，将数字 1 和键关联起来；或者当这个键已经存在，就负责递增该关联值：

`map.merge(key, 1, (count, incr) -> count + incr);`

注意，这行代码中使用了 `merge` 方法，这是 Java 8 版本在 `Map` 接口中添加的。如果指定的键没有映射， 该方法就会插入指定值；如果有映射存在， `merge` 方法就会将指定的函数应用到当前值和指定值上，并用结果覆盖当前值。这行代码代表了 `merge` 方法的典型用例。

这样的代码读起来清晰明了，但仍有些样板代码。参数 `count` 和 `incr` 没有添加太多价值，却占用了不少空间。实际上， `Lambda` 要告诉你的就是，该函数返回的是它两个参数的和。从 Java 8 开始， `Integer` （以及所有其他的数字化基本包装类型都）提供了一个名为 `sum` 的静态方法，它的作用也同样是求和。我们只要传人一个对该方法的引用，就可以更轻松地得到相同的结果：

`map.merge(key, 1, Integer::sum);`

方法带的参数越多能用方法引用消除的样板代码就越多。但在有些 `Lambda` 中，即便它更长，但你所选择的参数名称提供了非常有用的文档信息，也会使得 `Lambda` 的可读性更强，并且比方法引用更易于维护。

只要方法引用能做的事，就没有 `Lambda` 不能完成的（只有一种情况例外，有兴趣的读者请参见JLS,9.9-2 ）。也就是说，使用方法引用通常能够得到更加简短、清晰的代码。如果 `Lambda` 太长，或者过于复杂，还有另一种选择： 从 `Lambda` 中提取代码，放到一个新的方法中，并用该方法的一个引用代替 `Lambda` 。你可以给这个方法起一个有意义的名字，并用自己满意的方式编写进入文档。

如果是用IDE 编程，则可以在任何可能的地方都用方法引用代替 `Lambda` 。通常（但并非总是）应该让 IDE 把握机会好好表现一下。有时候， `Lambda` 也会比方法引用更加简洁明了。这种情况大多是当方法与 `Lambda` 处在同一个类中的时候。比如下面的代码片段，假定发生在一个名为 `GoshThisClassNameisHumongous` 的类中:

`service.execute(GoshThisClassIsHumongous::action);`

`Lambda` 版本的代码如下: 

`service.execute(() -> action());`

这个代码片段使用了方法引用，但是它既不比 `Lambda` 更简短，也不比它更清晰，因此应该优先考虑 `Lambda `。类似的还有Function 接口，它用一个静态工厂方法返回 `M ` 函数 `Function .identity()`。如果它不用这个方法，而是在行内编写同等的 `Lambda` 表达式：`x -> x` ，一般会比较简洁明了。

许多方法引用都指向静态方法，但其中有4 种没有这么做。其中两个是有限制(bound)和无限制(unbound)的实例方法引用。在有限制的引用中，接收对象是在方法引用中指定的。有限制的引用本质上类似于静态引用：函数对象与被引用方法带有相同的参数。在无限制的引用中，接收对象是在运用函数对象时，通过在该方法的声明函数前面额外添加一个参数来指定的。无限制的引用经常用在流管道(Stream pipeline) （详见第45 条）中作为映射和过滤函数。最后，还有两种构造器(constructor)引用，分别针对类和数组。构造器引用是充当工厂对象。这五种方法引用概括如下：

| 方法引用类型 | 范例                     | Lambda等式                                                  |
| ------------ | ------------------------ | ----------------------------------------------------------- |
| 静态         | `Integer::parseInt`      | `str -> Integer.parseInt(str)`                              |
| 有限制       | `Instant.now()::isAfter` | `Instant then=Instant.now();`<br /> `t -> then.isAfter(t);` |
| 无限制       | `String::toLowerCase`    | `str -> str.toLowerCase()`                                  |
| 类构造器     | `TreeMap<K, V>::new`     | `() -> new TreeMap<K, V>`                                   |
| 数组构造器   | `int[]::new`             | `len -> new int[len]`                                       |

总而言之，方法引用常常比 `Lambda`  表达式更加简洁明了。**只要方法引用更加简洁、清晰，就用方法引用; 如果方法引用并不简洁，就坚持使用 `Lambda`** 。





## 44. 坚持使用标准的函数接口

在 Java 具有 `Lambda` 表达式之后，编写 API 的最佳实践也做了相应的改变。例如在模板方法(Template Method) 模式［ Gamma95 ］中，用一个子类覆盖基本类型方法(primitive method)，来限定其超类的行为，这是最不讨人喜欢的。现在的替代方法是提供一个接受函数对象的静态工厂或者构造器，便可达到同样的效果。在大多数情况下，需要编写更多的构造器和方法，以函数对象作为参数。需要非常谨慎地选择正确的函数参数类型。

以 `Linke dHashMap` 为例。每当有新的键添加到映射中时， `put` 就会调用其受保护的 `removeEldestEntry` 方法。如果覆盖该方法，便可以用这个类作为缓存。当该方法返回 `true` ，映射就会删除最早传入该方法的条目。下列覆盖代码允许映射增长到 100 个条目，然后每添加一个新的键，就会删除最早的那个条目，始终保持最新的 100 个条目:

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > 100;
}
```

这个方法很好用，但是用 `Lambda` 可以完成得更漂亮。假如现在编写 `LinkedHash-Map`，它会有一个带函数对象的静态工厂或者构造器。看一下 `removeEldestEntry` 的声明，你可能会以为该函数对象应该带一个`Map.Entry<K,V>`，并且返回一个`boolean`，但实际并非如此： `removeEldestEntry` 方法会调用 `size()`，获取映射中的条目数量，这是因为`removeEldestEntry` 是映射中的一个实例方法。传到构造器的函数对象则不是映射中的实例方法，无法捕提到，因为调用其工厂或者构造器时，这个映射还不存在。所以，映射必须将它自身传给函数对象，因此必须传入映射及其最早的条目作为 `remove` 方法的参数。声明一个这样的函数接口的代码如下:

```java
// Unnecessary functional interface: use a standard one instead.
@FunctionalInterface interface EldestEntryRemovalFunction<K, V> {
    boolean remove(Map<K,V>map, Map.Entry<K,V> eldest);
}
```

这个接口可以正常工作，但是不应该使用，因为没必要为此声明一个新的接口。`java.util.function` 包已经为此提供了大量标准的函数接口。只要标准的函数接口能够满足需求，通常应该优先考虑，而不是专门再构建一个新的函数接口。这样会使 API 更加容易学习，通过减少它的概念内容，显著提升互操作性优势，因为许多标准的函数接口都提供了有用的默认方法。如 `Predicate` 接口提供了合并断言的方法。对于上述`LinkedHashMap`范例，应该优先使用标准的 `BiPredicate<Map<K,V>, Map.Entry<K,V>>`接口，而不是定制 `EldestEntryRemovalFunction` 接口。

`java.util.Function` 中共有 43 个接口。别指望能够全部记住它们，但是如果能记住其中6 个基础接口，必要时就可以推断出其余接口了。基础接口作用于对象引用类型。`Operator` 接口代表其结果与参数类型一致的函数。`Predicate` 接口代表带有一个参数并返回一个`boolean` 的函数。`Function` 接口代表其参数与返回的类型不一致的函数。`Supplier` 接口代表没有参数并且返回（或“提供”）一个值的函数。最后， `Consumer` 代
表的是带有一个函数但不返回任何值的函数，相当于消费掉了其参数。这 6 个基础函数接口概述如下:

| 接口                | 函数签名              | 范例                  |
| ------------------- | --------------------- | --------------------- |
| `UnaryOperator<T>`  | `T apply(T t)`        | `String::toLowerCase` |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | `BigInteger::add`     |
| `Predicate<T>`      | `boolean test(T t)`   | `Collection::isEmpty` |
| `Function<T, R>`    | `R apply(T t)`        | `Arrays::asList`      |
| `Supplier<T>`       | `T get()`             | `Instant::now`        |
| `Consumer<T>`       | `void accept(T t)`    | `System.out::println` |

这 6 个基础接口各自还有 3 种变体，分别可以作用于基本类型 `int` 、`long` 和 `double`。 它们的命名方式是在其基础接口名称前面加上基本类型而得。因此，以带有 `int` 的 `predicate`接口为例， 其变体名称应该是`IntPredicate` ，它是一个二进制运算符带有两个 `long` 值参数并返回一个`long` 值`LongBinaryOperator` 。这些变体接口的类型都不是参数化的，除`Function` 变体外，后者是以返回类型作为参数。例如， `LongFunction<int[]>`表示带有一个 `long` 参数，并返回一个`int[]`数组。

`Function` 接口还有 9 种变体，用于结果类型为基本类型的情况。源类型和结果类型始终不一样，因为从类型到自身的函数就是 `UnaryOperator` 。如果源类型和结果类型均为基本类型， 就是在 `Function` 前面添加格式如`ScrToResult`，如 `LongTointFunction`（有6 种变体） 。如果源类型为基本类型，结果类型是一个对象参数， 则要在 `Function` 前添加 `<Src>ToObj` ，如`DoubleToObjFunction` (有3 种变体)。

这三种基础函数接口还有带两个参数的版本，如 `BiPredicate<T,U>`、`Bi Function<T,U,R＞` 和`BiConsurner<T,U>` 。还有 `BiFunction` 变体用于返回三个相关的基本类型：`TointBiFunction<T,U>`、`ToLongBiFunction<T,U>`和`ToDoubleBiFunction<T,U>` 。`Consumer` 接口也有带两个参数的变体版本，它们带一个对象引用和一个基本类型： `Obj DoubleConsumer<T>`、`ObjintConsumer<T>`和`ObjLongConsumer<T>` 。总之，这些基础接口有9 种带两个参数的版本。

最后， 还有 `BooleanSupplier` 接口，它是 `Supplier` 接口的一种变体，返回 `boolean` 值。这是在所有的标准函数接口名称中唯一显式提到`boolean` 类型的，但 `boolean` 返回值是通过 `Predicate` 及其4 种变体来支持的。`BooleanSupplier` 接口和上述段落中提及的 42 个接口，总计 43 个标准函数接口。显然，这是个大数目，但是它们之间并非纵横交错。另一方面，你需要的函数接口都替你写好了，它们的名称都是循规蹈矩的，需要的时候并不难找到。

现有的大多数标准函数接口都只支持基本类型。千万不要用带包装类型的基础函数接口来代替基本函数接口。虽然可行，但它破坏了第 61 条的规则 “基本类型优于装箱基本类型” 。使用装箱基本类型进行批量操作处理，最终会导致致命的性能问题。

现在知道了，通常应该优先使用标准的函数接口，而不是用自己编写的接口。但什么时候应该自己编写接口呢?当然，是在如果没有任何标准的函数接口能够满足你的需求之时，如需要一个带有三个参数的 `predicate` 接口，或者需要一个抛出受检异常的接口时，当然就需要自己编写啦。但是也有这样的情况： 有结构相同的标准函数接口可用，却还是应该自己编写函数接口。

还是以咱们的老朋友 `Comparator<T>`为例吧。它与`ToIntBiFunction<T,T>`接口在结构上一致，虽然前者被添加到类库中时，后一个接口已经存在，但如果用后者就错了。`Comparator` 之所以需要有自己的接口，有三个原因。首先，每当在 API 中使用时，其名称提供了良好的文档信息，并且被大量使用。其次， `Comparator` 接口对于如何构成一个有效的实例，有着严格的条件限制，这构成了它的总则（ general contract ） 。实现该接口相当于承诺遵守其契约。第三，这个接口配置了大量很好用的缺省方法，可以对比较器进行转换和合并。

如果你所需要的函数接口与`Comparator` 一样具有一项或者多项以下特征， 则必须认真考虑自己编写专用的函数接口，而不是使用标准的函数接口：

* 通用，并且将受益于描述性的名称。
* 具有与其关联的严格的契约。
* 将受益于定制的缺省方法。

如果决定自己编写函数接口， 一定要记住，它是一个接口，因而设计时应当万分谨慎 (详见第 21 条)。

注意， `EldestEntryRemovalFunction` 接口（详见第 199 页）是用 `@FunctionalInterface`注解进行标注的。这个注解类型本质上与`@Override` 类似。这是一个标注了程序员设计意图的语句，它有三个目的：告诉这个类及其文档的读者，这个接口是针对 `Lambda` 设计的；这个接口不会进行编译，除非它只有一个抽象方法；避免后续维护人员不小心给该接口添加抽象方法。必须始终用 `@FunctionalInteface` 注解对自己编写的函数接口进行标注。

最后一点是关于函数接口在API 中的使用。不要在相同的参数位置，提供不同的函数接口来进行多次重载的方法，否则可能在客户端导致歧义。这不仅仅是理论上的问题。比如 `ExecutorService` 的 `submit` 方法就可能带有`Callable<T>`或者 `Runnable` ，并且还可以编写一个客户端程序，要求进行一次转换，以显示正确的重载(详见第 52 条) 。避免这个问题的最简单方式是，不要编写在同一个参数位置使用不同函数接口的重载。这是该建议的一个特例，详情请见第 52 条。

总而言之，既然 Java 有了 `Lambda` ，就必须时刻谨记用 `Lambda` 来设计API 。输入时接受函数接口类型，并在输出时返回之。一般来说，最好使用 `java.util.function.Function` 中提供的标准接口，但是必须警惕在相对罕见的几种情况下，最好还是自己编写专用的函数接口。