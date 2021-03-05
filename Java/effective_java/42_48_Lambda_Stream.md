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



## 45. 谨慎使用 Stream

在 Java 8中增加了 `Stream API` ，简化了串行或并行的大批量操作。这个 API 提供了两个关键抽象： `Stream` (流) 代表数据元素有限或无限的顺序， `Stream pipeline`  (流管道) 则代表这些元素的一个多级计算。`Stream` 中的元素可能来自任何位置。常见的来源包括集合、数组、文件、正则表达式模式匹配器、伪随机数生成器，以及其他 `Stream` 。`Stream` 中的数据元素可以是对象引用，或者基本类型值。它支持三种基本类型： `int`、`long` 和 `double` 。

一个 `Stream pipeline` 中包含一个源 `Stream` ，接着是 0 个或者多个中间操作 (intermediate operation) 和一个终止操作 (terminal operation) 。每个中间操作都会通过某种方式对 `Stream` 进行转换，例如将每个元素映射到该元素的函数，或者过滤掉不满足某些条件的所有元素。所有的中间操作都是将一个 `Stream` 转换成另一个 `Stream` ，其元素类型可能与输入的 `Stream` 一样，也可能不同。终止操作会在最后一个中间操作产生的`Stream` 上执行一个最终的计算，例如将其元素保存到一个集合中，并返回某一个元素，或者打印出所有元素等。

`Stream pipeline` 通常是 `lazy` 的： 直到调用终止操作时才会开始计算，对于完成终止操作不需要的数据元素，将永远都不会被计算。正是这种 `lazy` 计算，使无限 `Stream` 成为可能。注意，没有终止操作的 `Stream pipeline` 将是一个静默的无操作指令，因此千万不能忘记终止操作。

`Stream API` 是流式 (fluent) 的：所有包含 `pipeline` 的调用可以链接成一个表达式。事实上，多个 `pipeline ` 也可以链接在一起，成为一个表达式。

在默认情况下， `Stream pipeline` 是按顺序运行的。要使`pipeline`  并发执行，只需在该 `pipeline` 的任何`Stream` 上调用 `parallel` 方法即可，但是通常不建议这么做（详见第48 条） 。

`Stream API` 包罗万象，足以用 `Stream` 执行任何计算，但是 "可以" 并不意味着 "应该" 。如果使用得当， `Stream` 可以使程序变得更加简洁、清晰；如果使用不当，会使程序变得混乱且难以维护。对于什么时候应该使用 `Stream` ，并没有硬性的规定，但是可以有所启发。

以下面的程序为例，它的作用是从词典文件中读取单词，并打印出单词长度符合用户指定的最低值的所有换位词。记住，包含相同的字母，但是字母顺序不同的两个词，称作换位词 (anagram)。该程序会从用户指定的词典文件中读取每一个词，并将符合条件的单词放入一个映射中。这个映射键是按字母顺序排列的单词，因此`staple` 的键是 `aelpst` ， `petals` 的键也是 `aelpst`：这两个词就是换位词，所有换位词的字母排列形式是一样的(有时候也叫alphagram) 。映射值是包含了字母排列形式一致的所有单词。词典读取完成之后，每一个列表就是一个完整的换位词组。随后，程序会遍历映射的 `values()`，预览并打印出单词长度符合极限值的所有列表。

```java
// Prints all large anagram groups in a dictionary iteratively
public class Anagrams {
    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        
        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word), 
                                       (unused) -> new TreeSet<>()).add(word);
            }
        }
        for (Set<String>group : groups.values())
            if (group.size() >= minGroupSize)
                System.out.println(group.size() + ": " + group);
    }
    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```

这个程序中有一个步骤值得注意。被插人到映射中的每一个单词都以粗体显示，这是使用了 Java 8 中新增的`computeifAbsent` 方法。这个方法会在映射中查找一个键：如果这个键存在，该方法只会返回与之关联的值。如果键不存在，该方法就会对该键运用指定的函数对象算出一个值，将这个值与键关联起来，并返回计算得到的值。`computeifAbsent` 方法简化了将多个值与每个键关联起来的映射实现。

下面举个例子，它也能解决上述问题，只不过大量使用了 `Stream` 。注意，它的所有程序都是包含在一个表达式中，除了打开词典文件的那部分代码之外。之所以要在另一个表达式中打开词典文件，只是为了使用 `try-with-resources` 语句，它可以确保关闭词典文件：

```java
// Overuse of streams - don't do this!
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
              groupingBy(word -> word.chars().sorted()
                        .collect(StringBuilder::new, 
                                 (sb, c) -> sb.append((char) c),
                                StringBuilder::append).toString()))
                .values().stream()
                .filter(group -> group.size() >= minGroupSize)
                .map(group -> group.size() + ": " + group)
                .forEach(System.out::println);
        }
    }
}
```

如果你发现这段代码好难懂，别担心，你并不是唯一有此想法的人。它虽然简短，但是难以读懂，对于那些使用 `Stream` 还不熟练的程序员而言更是如此。滥用 `Stream` 会使程序代码更难以读懂和维护。

好在还有一种舒适的中间方案。下面的程序解决了同样的问题，它使用了 `Stream` ，但是没有过度使用。结果，与原来的程序相比，这个版本变得既简短又清晰：

```java
// Tasteful use of streams enhances clarity and conciseness
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
        	words.collect(groupingBy(word -> alphabetize(word)))
                .values().stream()
                .filter(group -> group.size() >= minGroupSize)
                .forEach(g -> System.out.println(g.size() + ": " + g));
        }
    }
    // alphbetize method is the same in original version
}
```

即使你之前没怎么接触过 `Stream` ，这段程序也不难理解。它在 `try-with-resources` 块中打开词典文件，获得一个包含了文件中所有代码的 `Stream` 。`Stream` 变量命名为 `words` , 是建议`Stream` 中的每个元素均为单词。这个 `Stream` 中的 `pipe line` 没有中间操作；它的终止操作将所有的单词集合到一个映射中，按照它们的字母排序形式对单词进行分组 (详见第 46 条)。这个映射与前面两个版本中的是完全相同的。随后，在映射的`values()` 视图中打开了一个新的 `Stream<List<String>>`。当然，这个 `Stream` 中的元素都是换位词分组。`Stream` 进行了过滤，把所有单词长度小于 `minGroupSize` 的单词都去掉了，最后，通过终止操作的 `forEach` 打印出剩下的分组。

注意， `Lambda` 参数的名称都是经过精心挑选的。实际上参数应当以 `group` 命名，只是这样得到的代码行对于书本而言太宽了。在没有显式类型的情况下，仔细命名 `Lambda`参数， 这对于`Stream pipeline` 的可读性至关重要。
还要注意单词的字母排序是在一个单独的 `alphabetize` 方法中完成的。给操作命名，并且不要在主程序中保留实现细节，这些都增强了程序的可读性。在 `Stream pipeline` 中使用 `helper`方法，对于可读性而言，比在迭代化代码中使用更为重要，因为 `pipeline` 缺乏显式的类型信息和具名临时变量。

可以重新实现 `alphabetize` 方法来使用 `Stream` ，只是基于 `Stream` 的 `alphabetize` 方法没那么清晰，难以正确编写，速度也可能变慢。这些不足是因为 Java 不支持基本类型的 `char Stream` （这并不意味着Java 应该支持 `char Stream` ；也不可能支持） 。为了证明用 `Stream` 处理 `char` 值的各种危险，请看以下代码：

`"Hello world!".chars().forEach(System.out::print);`

或许你以为它会输出 `Hello world!` ，但是运行之后发现，它输出的是 `721011081081113211911111410810033` 。这是因为 `Hello world !”.chars()`返回的 `Stream` 中的元素，并不是 `char` 值，而是 `int` 值，因此调用了`print` 的 `int` 覆盖。名为 `chars` 的方法，却返回 `int` 值的`Stream` ，这固然会造成困扰。修正方法是利用转换强制调用正确的覆盖：

`"Hello world!".chars().forEach(x->System.out.print((char) x));`

但是， 最好避免利用 `Stream` 来处理 `char` 值。

刚开始使用 `Stream` 时，可能会冲动到恨不得将所有的循环都转换成 `Stream` ，但是切记，千万别冲动。这可能会破坏代码的可读性和易维护性。一般来说即使是相当复杂的任务，最好也结合 `Stream` 和迭代来一起完成，如上面的 `Anagrams` 程序范例所示。因此， 重构现有代码来使用 `Stream` ，并且只在必要的时候才在新代码中使用。

如本条目中的范例程序所示， `Stream pipeline` 利用函数对象（一般是 `Lambda` 或者方法引用）来描述重复的计算，而迭代版代码则利用代码块来描述重复的计算。下列工作只能通过代码块，而不能通过函数对象来完成：

* 从代码块中，可以读取或者修改范围内的任意局部变量；从 `Lambda` 则只能读取 `final` 或者有效的 `final` 变量［ JLS 4.12.4 ］，并且不能修改任何 `local` 变量。
* 从代码块中，可以从外国方法中 `return` 、`break` 或 `continue` 外围循环，或者抛出该方法声明要抛出的任何受检异常；从 `Lambda` 中则完全无法完成这些事情。

如果某个计算最好要利用上述这些方法来描述，它可能并不太适合 `Stream` 。反之， `Stream` 可以使得完成这些工作变得易如反掌：

* 统一转换元素的序列
* 过滤元素的序列
* 利用单个操作（如添加、连接或者计算其最小值）合并元素的顺序
* 将元素的序列存放到一个集合中，比如根据某些公共属性进行分组
* 搜索满足某些条件的元素的序列

如果某个计算最好是利用这些方法来完成，它就非常适合使用 `Stream` 。

利用 `Stream` 很难完成的一件事情就是，同时从一个 `pipeline` 的多个阶段去访问相应的元素：一旦将一个值映射到某个其他值，原来的值就丢失了。一种解决办法是将每个值都映射到包含原始值和新值的一个对象对（ pair object ），不过这并非万全之策，当`pipeline` 的多个阶段都需要这些对象对时尤其如此。这样得到的代码将是混乱、繁杂的，违背了 `Stream` 的初衷。最好的解决办法是，当需要访问较早阶段的值时，将映射颠倒过来。

例如，编写一个打印出前20 个梅森素数 (Mersenne primes) 的程序。解释一下，梅森素数是一个形式为2<sup>p</sup> - 1 的数字。如果 `p` 是一个素数，相应的梅森数字也是素数；那么它就是一个梅森素数。作为 `pipeline` 的第一个`Stream` ，我们想要的是所有素数。下面的方法将返回（无限） `Stream` 。假设使用的是静态导入，便于访问 `Biginteger` 的静态成员：

```java
static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

方法的名称（ `primes` ）是一个复数名词，它描述了 `Stream` 的元素。强烈建议返回 `Stream` 的所有方法都采用这种命名惯例，因为可以增强 `Stream pipeline` 的可读性。该方法使用静态工厂 `Stream.iterate` ，它有两个参数： `Stream` 中的第一个元素，以及从前一个元素中生成下一个元素的一个函数。下面的程序用于打印出前 20 个梅森素数。

```java
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
        .filter(mersenne -> mersenne.isProbablePrime(50))
        .limit(20)
        .forEach(System.out::println);
}
```

这段程序是对上述内容的简单编码示范： 它从素数开始，计算出相应的梅森素数，过滤掉所有不是素数的数字(其中 50 是个神奇的数字，它控制着这个概率素性测试)，限制最终得到的 `Stream` 为20 个元素，并打印出来。

现在假设想要在每个梅森素数之前加上其指数 (p)。这个值只出现在第一个 `Stream` 中，因此在负责输出结果的终止操作中是访问不到的。所幸将发生在第一个中间操作中的映射颠倒过来，便可以很容易地计算出梅森数字的指数。该指数只不过是一个以二进制表示的位数， 因此终止操作可以产生所要的结果：

`.forEach(mp -> System.out.println(mp.bitLength() + ": " + mp));`

现实中有许多任务并不明确要使用 `Stream` ，还是用迭代。例如有个任务是要将一副新纸牌初始化。假设 `Card` 是一个不变值类，用于封装 `Rank` 和 `Suit` ，这两者都是枚举类型。这项任务代表了所有需要计算从两个集合中选择所有元素对的任务。数学上称之为两个集合的笛卡尔积。这是一个迭代化实现，嵌入了一个`for-each` 循环，大家对此应当都非常熟悉了：

```java
// Iterative Cartesian product computation
private static List<Card> new Deck() {
    List<Card> result = new ArrayList<>();
    for (Suit suit : Suit.values())
        for (Rank rank : Rank.values())
            result.add(new Card(suit, rank));
    return result;
}
```

这是一个基于 `Stream` 的实现，利用了中间操作 `flatMap` 。这个操作是将 `Stream `中的每个元素都映射到一个`Stream` 中，然后将这些新的 `Stream` 全部合并到一个`Stream`  (或者将它们扁平化)。注意，这个实现中包含了一个嵌入式的 `Lambda` ，如以下粗体部分所示：

```java
// Stram-based Cartesian product computation
private static List<Card> newDeck() {
    return Stream.of(Suit.values())
        .flatMap(suit -> Stream.of(Rank.values())
                 .map(rank -> new Card(suit, rank)))
        .collect(toList());
}
```

这两种 `newDeck` 版本哪一种更好？这取决于个人偏好，以及编程环境。第一种版本比较简单，可能感觉比较自然，大部分 Java 程序员都能够理解和维护，但是有些程序员可能会觉得第二种版本(基于 `Stream` 的)更舒服。这个版本可能更简洁一点，如果已经熟练掌握 `Stream` 和函数编程，理解起来也不难。如果不确定要用哪个版本，或许选择迭代化版本会更加安全一些。如果更喜欢 `Stream` 版本，并相信后续使用这些代码的其他程序员也会喜欢，就应该使用 `Stream` 版本。

总之，有些任务最好用 `Stream` 完成，有些则要用迭代。而有许多任务则最好是结合使用这两种方法来一起完成。具体选择用哪一种方法，并没有硬性、速成的规则，但是可以参考一些有意义的启发。在很多时候，会很清楚应该使用哪一种方法；有些时候，则不太明显。如果实在不确定用 `Stream` 还是用迭代比较好，那么就两种都试试，看看哪一种更好用吧。



## 46. 优先选择 Stream 中无副作用的函数

如果刚接触 `Stream` ，可能比较难以掌握其中的窍门。就算只是用 `Stream pipeline` 来表达计算就困难重重。当你好不容易成功了，运行程序之后，却可能感到这么做并没有享受到多大益处。`Stream`并不只是一个 API，它是一种基于函数编程的模型。为了获得 `Stream` 带来的描述性和l速度，有时还有并行性，必须采用范型以及 API 。

`Stream` 范型最重要的部分是把计算构造成一系列变型，每一级结果都尽可能靠近上一级结果的纯函数 (pure function)。纯函数是指其结果只取决于输入的函数： 它不依赖任何可变的状态，也不更新任何状态。为了做到这一点，传入 `Stream` 操作的任何函数对象，无论是中间操作还是终止操作，都应该是无副作用的。

有时会看到如下代码片段，它构建了一张表格，显示这些单词在一个文本文件中出现的频率：

```java
// Uses the streams API but not the paradigm - Don't do this!
Map<String, Long> freq = new HashMap<>();
try (Stream<String> words = new Scanner(file).tokens()) {
    words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
}
```

以上代码有什么问题吗？它毕竟使用了 `Stream` 、`Lambda` 和方法引用，并且得出了正确的答案。简而言之，这根本不是 `Stream` 代码；只不过是伪装成 `Stream` 代码的迭代式代码。它并没有享受到 `Stream API` 带来的优势，代码反而更长了点，可读性也差了点，并且比相应的迭代化代码更难维护。因为这段代码利用一个改变外部状态（频率表）的 `Lambda` ，完成了在终止操作的 `forEach` 中的所有工作。`forEach` 操作的任务不只展示由`Stream` 执行的计算结果，这在代码中并非好事，改变状态的 `Lambda` 也是如此。那么这段代码应该是什么样的呢？

```java
// Proper use of streams to initialize a frequency table
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {
    freq = words.collect(groupingBy(String::toLowerCase, counting()));
}
```

这个代码片段的作用与前一个例子一样，只是正确使用了`Stream API` ，变得更加简洁、清晰。那么为什么有人会以其他的方式编写呢？这是为了使用他们已经熟悉的工具。Java 程序员都知道如何使用 for-each 循环，终止操作的 `forEach` 也与之类似。但 `forEach` 操作是终止操作中最没有威力的，也是对 `Stream` 最不友好的。它是显式迭代，因而不适合并行。**`forEach` 操作应该只用于报告 `Stream` 计算的结果，而不是执行计算**。有时候，也可以将 `forEach` 用于其他目的，比如将 `Stream` 计算的结果添加到之前已经存在的集合中去。

改进过的代码使用了一个收集器 (collector)，为了使用 `Stream` ，这是必须了解的一个新概念。`Collectors API` 很吓人：它有 39 种方法，其中有些方法还带有 5 个类型参数！好消息是，你不必完全搞懂这个 API 就能享受它带来的好处。对于初学者，可以忽略 `Collector` 接口，并把收集器当作封装缩减策略的一个黑盒子对象。在这里，缩减的意思是将 `Stream` 的元素合并到单个对象中去。收集器产生的对象一般是一个集合(即名称收集器)。

将 `Stream` 的元素集中到一个真正的 `Collection` 里去的收集器比较简单。有三个这样的收集器： `toList()`、`toSet()`和 `toCollection(collectionFactory)` 。它们分别返回一个列表、一个集合和程序员指定的集合类型。了解了这些，就可以编写 `Stream pipeline`时，从频率表中提取排名前十的单词列表了：

```java
// Pipeline to get a top-ten list of words from a frequency table
List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList());
```

注意，这里没有给 `toList` 方法配上它的 `Collectors` 类。**静态导入`Collectors` 的所有成员是惯例也是明智的，因为这样可以提升`Stream pipeline` 的可读性**。

这段代码中唯一有技巧的部分是传给 `sorted` 的比较器 `comparing(freq::get).reversed()` 。 `comparing `方法是一个比较器构造方法(详见第14 条)，它带有一个键提取函数。函数读取一个单词，“提取” 实际上是一个表查找：有限制的方法引用`freq:: get` 在频率表中查找单词，并返回该单词在文件中出现的次数。最后，在比较器上调用 `reversed`, 按频率高低对单词进行排序。后面的事情就简单了，只要限制 `Stream` 为 10 个单词，并将它们集中到一个列表中即可。

上一段代码是利用 `Scanner` 的 `Stream` 方法来获得 `Stream` 。这个方法是在 Java 9 中增加的。如果使用的是更早的版本，可以把实现`Iterator` 的扫描器，翻译成使用了类似于第 47 条中适配器的 `Stream(streamOf(iterable<E>))` 。

`Collectors` 中的另外 36 种方法又是什么样的呢？它们大多数是为了便于将 `Stream` 集合到映射中，这远比集中到真实的集合中要复杂得多。每个 `Stream` 元素都有一个关联的键和值，多个 `Stream` 元素可以关联同一个键。

最简单的映射收集器是 `toMap(keyMapper,valueMapper)`，它带有两个函数，其中一个是将 `Stream` 元素映射到键，另一个是将它映射到值。我们采用第 34 条 `fromString` 实现中的收集器，将枚举的字符串形式映射到枚举本身：

```java
// Using a toMap collector to make a map from string to enum
private static final Map<String, Operation> stringToEnum = 
    Stream.of(values()).collect(toMap(Object::toString, e -> e));
```

如果 `Stream` 中的每个元素都映射到一个唯一的键，那么这个形式简单的 `toMap` 是很完美的。如果多个`Stream` 元素映射到同一个键， `pipeline` 就会抛出一个 `IllegalStateException`异常将它终止。

`toMap` 更复杂的形式，以及 `groupingBy` 方法，提供了更多处理这类冲突的策略。其中一种方式是除了给`toMap` 方法提供了键和值映射器之外，还提供一个合并函数 (merge function) 。合并函数是一个`BinaryOperator<V >`，这里的 `V`  是映射的值类型。合并函数将与键关联的任何其他值与现有值合并起来，因此，假如合并函数是乘法，得到的值就是与该值映射的键关联的所有值的积。

带有三个参数的 `toMap` 形式，对于完成从键到与键关联的被选元素的映射也是非常有用的。假设有一个`Stream` ，代表不同歌唱家的唱片，我们想得到一个从歌唱家到最畅销唱片之间的映射。下面这个收集器就可以完成这项任务。

```java
// Collector to generate a map from key to chosen element for key
Map<Artist, Album> topHits = albums.collect(
	toMap(Album::artist, a -> a, maxBy(comparing(Album::sales))));
```

注意，这个比较器使用了静态工厂方法 `maxBy` ，这是从 `BinaryOperator` 静态导入的。该方法将`Comparator<T>` 转换成一个 `BinaryOperator<T>`， 用于计算指定比较器产生的最大值。在这个例子中，比较器是由比较器构造器方法 `comparing` 返回的，它有一个键提取函数 `Album : : sales` 。这看起来有点绕，但是代码的可读性良好。不严格地说，它的意思是 “将唱片的 `Stream` 转换成一个映射，将每个歌唱家映射到销量最佳的唱片“ 这就非常接近问题陈述了。

带有三个参数的 `toMap` 形式还有另一种用途，即生成一个收集器，当有冲突时强制 “保留最后更新”（ last-write -wins ）。对于许多 `Stream` 而言，结果是不确定的，但如果与映射函数的键关联的所有值都相同，或者都是可接受的，那么下面这个收集器的行为就正是你所要的：

```java
// Collector to impose last-write-wins policy
toMap(keyMapper, valueMapper, (oldVal, newVal) -> new?Val)
```

`toMap` 的第三个也是最后一种形式是，带有第四个参数，这是一个映射工厂，在使用时要指定特殊的映射实现，如 `EnurnMap` 或者 `TreeMap` 。

`toMap` 的前三种版本还有另外的变换形式命名为 `toConcurrentMap` ，能有效地并行运行，并生成`ConcurrentHashMap` 实例。

除了 `toMap` 方法， `Collectors API` 还提供了`groupingBy` 方法，它返回收集器以生成映射，根据分类函数将元素分门别类。分类函数带有一个元素，并返回其所属的类别。这个类别就是元素的映射键。`groupingBy` 方法最简单的版本是只有一个分类器，并返回一个映射，映射值为每个类别中所有元素的列表。下列代码就是在第 45 条的`Anagram` 程序中用于生成映射（从按字母排序的单词，映射到字母排序相同的单词列表）的收集器：

`words.collect(groupingBy(word -> alphzbetize(word)))`

如果要让 `groupingBy` 返回一个收集器用它生成一个值而不是列表的映射，除了分类器之外，还可以指定一个下游收集器（ downstream collector ） 。下游收集器从包含某个类别中所有元素的 `Stream` 中生成一个值。这个参数最简单的用法是传入`toSet()`，结果生成一个映射，这个映射值为元素集合而非列表。

另一种方法是传入`toCollection` (collectionFactory)，允许创建存放各元素类别的集合。这样就可以自由选择自己想要的任何集合类型了。带两个参数的 `groupingBy` 版本的另一种简单用法是，传入`counting()` 作为下游收集器。这样会生成一个映射，它将每个类别与该类别中的元素数量关联起来，而不是包含元素的集合。这正是在本条目开头处频率表范例中见到的：

`Map<String, Long> freq = words.collect(groupingBy(String::toLowerCase, counting()));`

`groupingBy` 的第三个版本，除了下游收集器之外，还可以指定一个映射工厂。注意，这个方法违背了标准的可伸缩参数列表模式： 参数`mapFactory` 要在 `down Stream`参数之前，而不是在它之后。`groupingBy` 的这个版本可以控制所包围的映射，以及所包围的集合，因此，比如可以定义一个收集器， 让它返回值为`TreeSets` 的 `TreeMap` 。

`groupingByConcurrent` 方法提供了`groupingBy` 所有三种重载的变体。这些变体可以有效地并发运行，生成`ConcurrentHashMap` 实例。还有一种比较少用到的 `groupingBy` 变体叫作 `partitioningBy` 。除了分类方法之外，它还带一个断言(predicate)，并返回一个键为 `Boolean` 的映射。这个方法有两个重载，其中一个除了带有断言之外，还带有下游收集器。

`counting` 方法返回的收集器仅用作下游收集器。通过在 `Stream` 上的 `count` 方法，直接就有相同的功能，因此压根没有理由使用 `collect(counting())`。这个属性还有15 种 `Collectors` 方法。其中包含 9 种方法其名称以 `summing` 、`averaging` 和 `summarizing`开头 (相应的 `Stream` 基本类型上就有相同的功能) 。它们还包`reducing` 、`filtering` 、`mapping` 、`flatMapping` 和 `collectingAndThen` 方法。大多数程序员都能安全地避开这里的大多数方法。从设计的角度来看，这些收集器试图部分复制收集器中 `Stream` 的功能，以便下游收集器可以成为 “ ministream ” 。

目前已经提到了 3 个 `Collectors` 方法。虽然它们都在 `Collectors` 中，但是并不包含集合。前两个是 `minBy` 和 `maxBy` ，它们有一个比较器并返回由比较器确定的 `Stream` 中的最少元素或者最多元素。它们是`Stream` 接口中 `min` 和 `max` 方法的粗略概括，也是 `BinaryOperator` 中同名方法返回的二进制操作符，与收集器相类似。回顾一下在最畅销唱片范例中用过的 `BinaryOperator.maxBy` 方法。

最后一个 `Collectors` 方法是 `joining` ，它只在 `CharSequence` 实例的 `Stream` 中操作，例如字符串。它以参数的形式返回一个简单地合并元素的收集器。其中一种参数形式带有一个名为 `delimiter` （分界符）的`CharSequence` 参数，它返回一个连接 `Stream` 元素并在相邻元素之间插入分隔符的收集器。如果传入一个逗号作为分隔符，收集器就会返回一个用逗号隔开的值字符串（但要注意，如果 `Stream` 中的任何元素中包含逗号，这个字符串就会引起歧义） 。这三种参数形式，除了分隔符之外，还有一个前缀和一个后缀。最终的收集
器生成的字符串，会像在打印集合时所得到的那样，如［came, saw, conquered ] 。

总而言之，编写 `Stream pipeline` 的本质是无副作用的函数对象。这适用于传入 `Stream`及相关对象的所有函数对象。终止操作中的 `forEach` 应该只用来报告由 `Stream` 执行的计算结果，而不是让它执行计算。为了正确地使用 `Stream` ，必须了解收集器。最重要的收集器工厂是 `toList` 、`toSet` 、`toMap` 、`groupingBy` 和`joining` 。