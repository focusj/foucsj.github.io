## 背景

自从2013年放弃了Java就再也没有碰过它。期间Java还发布了重大更新：引入lambda，但是那会儿我已经玩了一段时间Scala了，对Java已经瞧不上眼了。相比Scala所支持的函数式特性Java 8的lambda还是too young， too naive！再后来，就已经是2015年左右了，我有机会学习了一下C#。虽然C#师承Java，但它已经是一门非常现代化的语言了，青出于蓝而胜于蓝。说先是LINQ，提供了一致的，并且非常有表达力的数据操作API。还有像扩展函数，匿名类这些语法特性，用起来也是非常趁手，不像Java还是那么死板。在技术层面微软比甲骨文强了不知道多少倍。而且这几年Java社区也稍微有点萧条，国内再也没有出现过类似JavaEye这样的高质量技术社区。



今年五月份，公司要做新的业务，团队里的一些老成员已经被动态语言（主要是Python，还有一些Ruby）折磨的无可奈何，决定重新用回Java。如果不是这个原因，我恐怕再也不会捡起Java了。但是工作嘛，没有太多选择的余地。既然团队决定了那就硬着头皮上吧。第一步肯定是要了解一下Java8的函数式特性（这可是千呼万唤始出来）。这段时间用下来的总体感觉是，没我想象中的那么糟，还凑合。下边总结了一些Java8函数式编程的要点：包括Optional，lambda表达式，Stream





## Optional

NullPointerException，这个应该是Java里被人诟病最多的了，有的公司应为这个损失的钱可不是小数。那在Java 8里引入了一个Optional类型来解决这个问题。Optional是什么呢？在函数式编程里Optional其实就是一个Monad。和其他编程语言中，比如Scala的Option，Haskell的Maybe异曲同工。



在没有Optional之前，通常做法是返回null，或者程序抛出异常，具体使用那种看团队的规范。但这两种方式都有各自的问题，我们在数落一遍。



对于返回null，试想如果所有的方法都由可能返回null，那对方法使用者来说非常恐怖的。处处防，处处防。而且每一个null都是一个地雷，哪一个地方疏忽了都有可能被“炸”到。



抛出异常呢，不会有处处写防御代码的问题了。但是这种解决方案也是非常”凑合“。Java中异常分为两种：受检异常和非受检异常。如果使用了非受检异常程序就会直接被异常中断，通常在Spring中是提倡直接抛出非受检异常的，再搭配上Spring的拦截器，省去了程序员不少麻烦。



受检异常就不一样了，使用受检异常绝不会比使用null好到哪里去。在方法的签名上附带上函数可能抛出的异常，让方法使用者去判断如何处理这个异常。看起来这是一种负责任的做法，事实确是把自己不想做的事情（异常处理）交给了方法调用者。Jackson类库就是这样，每次使用它序列化对象时都得考虑是try-catch还是修改方法签名（告知外部方法处理）。非常不友好。



再有一点，抛出Exception意味着这个函数（方法）是带有副作用的。函数副作用会给程序设计带来不必要的麻烦，引入潜在的bug，并降低程序的可读性。试想一个函数从其签名上来看应该返回一个订单，但是结果却是它不总能够返回订单，有时还抛出异常。



终于可以开始说Optional了。用白话形容，Optional就是一个盒子。当你拿到这个盒子的时候，盒子可能里边有想要的东西，但也可能只是一个空盒子。所以原来返回null，或者跑出异常的方法我们可以直接返回一个盒子。来一个例子，这个例子来改进一下Jackson的API：



```

public Optional<String> json(Object object) {

    String jsonString = null;

    try {

        jsonString = new ObjectMapper().writeValueAsString(object);

    } catch (JsonProcessingException e) {

        e.printStackTrace();

    }



    return Optional.ofNullable(jsonString);

}

```

这个新的方法不再让调用者处理异常，也不存在返回null的风险。但是问题来了，拿到了这个“盒子”我们接下来该怎么办啊？别找急，Optional提供了非常丰富的接口，几乎能够满足你日常全部的使用场景。以下接口都是非常常用的：

1. 判断Optional是否有值：`op.isPresent()`

2. 基于Optional封装的结果做一些操作，并继续返回一个Optional类型的结果：`op.map(str -> str.trim())`

3. 合并两个Optional为一个：`op.flatMap(s -> op2.map(s2 -> s1 + s2))`

4. 只有在有值的情况下才进行一些操作：`op.ifPresent(s -> s.trim())`

5. 如果Optional有值则返回，没有返回给定的默认值：`op.orElse("hello")`

还有一些其它的接口，请自行查阅Optional API文档。注意在使用Optional的时候不推荐直接使用`get()`，因为这样可能会抛出异常。



我曾经把方法的参数也置为Optional类型，但是的到了IDEA的警告：Optional不推荐用作方法的参数。我随后Google了一些帖子，得到的答案就是：对于方法的参数，null可能比Optional更好用一些。在Scala和Haskell里是没有这样的限制的，你可以任意的把Option和Maybe当做参数。因为在Scala和Haskell中Option和Maybe是基本的数据类型。Java中的Optional则只是一种受限的实现，主要的目的是提供一种清晰，友好的表达“空”的方式。



## Lambda 表达式

lambda表达式在上边我们已经用到了，比如`op.map(str -> str.trim())`，`str -> str.trim()`就是一个lambda表达式。Java和其他大多数支持lambda的语言一样采用使用箭头：-> 来标示lambda。在箭头的的左边是0或多个参数，箭头的右边是一个表达式或者代码块。我们来看几个lambda的实例：

```



() -> {}                // No parameters; result is void

() -> 42                // No parameters, expression body

() -> null              // No parameters, expression body

() -> { return 42; }    // No parameters, block body with return

() -> { System.gc(); }  // No parameters, void block body



() -> {                 // Complex block body with returns

  if (true) return 12;

  else {

    int result = 15;

    for (int i = 1; i < 10; i++)

      result *= i;

    return result;

  }

}                          



(int x) -> x+1              // Single declared-type parameter

(int x) -> { return x+1; }  // Single declared-type parameter

(x) -> x+1                  // Single inferred-type parameter

x -> x+1                    // Parentheses optional for

                            // single inferred-type parameter



(String s) -> s.length()      // Single declared-type parameter

(Thread t) -> { t.start(); }  // Single declared-type parameter

s -> s.length()               // Single inferred-type parameter

t -> { t.start(); }           // Single inferred-type parameter



(int x, int y) -> x+y  // Multiple declared-type parameters

(x, y) -> x+y          // Multiple inferred-type parameters

(x, int y) -> x+y    // Illegal: can't mix inferred and declared types

(x, final y) -> x+y  // Illegal: no modifiers with inferred types

```



忆苦思甜，我们先来回忆一下在没有lambda之前如果我们想为Optional实现map功能我们应该如何做。通常做法是这样的：

```

interface Function<E> {

    public E exec(E e);

}



public static <E> Optional<E> map(Optional<E> original, Function<E> fn) {

    if(!original.isPresent()) return Optional.empty();

    E e = original.get();

    return Optional.of(fn.exec(e));

}



map(Optional.of(1), new Function<Integer>() {

    @Override public Integer exec(final Integer i) {

        return i * i;

    }

});

```

使用一个接口来承载一个函数。在Java中函数不是一等公民，是不能用来传递的。所以只能采用这种“曲线救国”的方式。Java 8则是把这种”曲线救国”拿到了台面上，并昭告天下，同时还对lambda提供了一些支持。所以上边我们看到的一些非常简短的lambda表达式，其实都是一个interface+一个抽象方法。



我们可以借助IDE（我使用的是IDEA）把上边列举的一些lambda表达式抽作变量，IDE可以帮助我们自动推导类型，我们来观察下他们的类型。

```

Runnable runnable = () -> {};

DoubleSupplier doubleSupplier = () -> 42;

Callable vCallable = () -> null;

IntToDoubleFunction intToDoubleFunction = (int x) -> x + 1;

IntBinaryOperator intBinaryOperator = (int x, int y) -> x + y;

```

像Callable，Runnable，DoubleSupplier这些接口就是Java内置的函数式接口。Java还提供了非常多的函数式接口，你可以在java.util.function下找到他们。函数式接口相比普通的接口有一个限制：只能有一个抽象方法。而且Java还提供了一个注解：@FunctionalInterface。你可以自己声明新的接口并为它加上这个注解。

```

@FunctionalInterface

interface Function<E> {

    public E exec(E e);

}

```

上边说过Java 8对lambda提供了一些额外支持，这种额外的支持就是一些已经实现的方法也能够用作lambda表达式。我们看一个例子：

```

Optional<Integer> arg = ...;

arg.ifPresent(System.out::print);

```

print是PrintStream中已经实现的方法。这种用法相当于：`x -> System.out.print(x)`。对于类的静态方法，类的实例方法，对象的实例方法都可以使用::操作符用在需要传递lambda表达式的地方。



我们接下来比较一下Java8前后，实现闭包的异同。先来看一下闭包的概念。

闭包是指可以包含自由变量的代码块。自由变量没有在当前代码块内或者任何全局上下文中定义的，而是在定义代码块的环境中定义(执行环境上下文)。所以一次闭包过程即要执行的代码块中的自由变量获得了执行环境上下文中的值。



在Java 8之前闭包可以通过内部类来实现。

```

import java.util.HashMap;

import java.util.Map;



class Cache {

    private Map<String, Object> contents = new HashMap<>();



    class Monitor {

        public Integer size() {

            return Cache.this.contents.size();

        }

    }

}



public class Library {

    public static void main(String[] args) {

        Cache cache = new Cache();

        Cache.Monitor monitor = cache.new Monitor();

        System.out.println("Cache size is: " + monitor.size());

    }

}

```

contents是自由变量，cache提供执行上下文，monitor.size()触发闭包执行。如果有其他函数式语言背景的人看到这种方式可能会感到非常的奇怪，但这就是Java的方式。再来看一下Java 8中如何实现一个闭包。

由于有了lambda表达式，创建一个闭包就相当简洁了：`(Integer x) -> x + y`。而且这种形式也非常的functional。接着创建一个执行上下文：

```

int y = 1;

Function<Integer, Integer> add = (Integer x) -> x + y;



add.apply(3); // 4

```

两种风格迥异，单从语法表达力来说肯定是lambda更胜一筹。但社区里也有人担心这种简洁性会影响Java的简单，造成代码风格不一致，降低代码可读性增加维护成本。仁者见仁智者见智吧，我肯定是支持使用lambda的。代码风格的话团队最好能有一个标准，不要好东西给用烂了。





## Stream

Stream可以说是集合处理的杀手锏。现象大家第一次听到Stream这个词汇肯定不是从Java这里，绝大多数应该是Spark。其实背后的思想都是一样的。我们先来看一个Stream的示例：

`Stream.of(0, 1, 2, 3, 4, 5, 6, 7, 8, 9).map(i -> i * i).reduce(0, (i, r) -> r + i)`

这个例子是计算0到9的平方和。我们可以想象一下如果用for循环实现同样的逻辑，代码行数至少是四五倍。



下面我们对这个程序作一个分解：第一部分是初始化Stream，加载数据`Stream.of(0, 1, 2, 3, 4, 5, 6, 7, 8, 9)`；第二部分是定义数据转化：`map(i -> i * i)`，第三部分是聚合结果：`reduce(0, (i, r) -> r + i)`。



### 创建函数

Stream类提供了提供了几个工厂方法来构建一个新的Stream。我们先来看一下of，of用来构建有限个数的Stream，比如我们构建一个包含十个元素的Stream：`Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 0)`。



Stream还有两个方法：generate, iterate，这两个函数都是用来构造无限流。



generate就收一个无参函数：`Supplier<T> s`，当需要一个常数或随机数队列的时候可以使用这个函数。` Stream.generate(() -> 1)`。



iterate会生成一个迭代的Stream，你需要指定初始值，以及迭代函数。`Stream.iterate(5, x -> x * 10)`。



除了上边的几种方式，我们还能够方便的将集合转化为Stream，比如List，Set。你只需要在集合变量上`.stream()`就可以得到一个Stream。实际场景中应用更多的还是将一个集合类转化为Stream。



### 转换函数

转化函数我们最常用的是map和filter。Stream也提供了flatMap函数，但是它的使用比较受限，所以原本flatMap的威力大大减弱了。下边具体解释一下map和filter函数。map的工作原理是：为所有的元素应用一个函数。为了增强理解举个现实生活中的例子：有一篓子苹果，我们要为它们都贴上标签。map过程就是拿出每个苹果贴上标签然后仍进去。filter就是一个过滤器，过滤出我们想要的东西。比如，我们要从上边篓子里挑选出大于500克的苹果。逐个拿出苹果称重，如果大于500克留在框中，如果小于500克，则把它扔掉。所以操作数据时，直接往这两个例子上套用就可以了。看一个例子，计算所有学生的总分，并取出总分大于450分的学生：

`students.map(s -> calculate(s.getScore())).filter(score -> score > 450);`。



### 聚合函数

聚合函数主要用来手机最终的结果。比如，求出一个数字队列的总和，求最大最小值，或将Stream收集到一个集合中。下边我们来操作一个Integer的Stream，首先求出最大值和最小值：

```

Stream<Integer> ints = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 0);



Optional<Integer> max = ints.max(Integer::compare);

Optional<Integer> min = ints.min(Integer::compare);

```

接下来我们求出这个数列的总和，求和我们可以使用reduce函数。reduce函数起到一个汇总的作用。它和hadoop中的reduce，fork/join中的join作用都是一样的。

```

Integer sum = ints.reduce(0, (x, y) -> x + y);

```

我们为reduce传递了两个参数，第一个是初始值（执行累加操作的第一个值），第二个是求和lambda。



Java 8还为基本类型提供了相应的Stream，比如IntStream，LongStream。使用IntStream我们直接使用sum就可以执行求和操作：

```

IntStream intsStream = ints.mapToInt(i -> i);

intsStream.sum();

```



下边我们看一下收集函数：collect()。collect()函数是将Stream中的元素放到一个新的容器中。我们上边的例子中求出了总分大于450分的同学，我们把它放到一个List中，看一下如何操作：

```

students.map(s -> calculate(s.getScore())).filter(score -> score > 450).collect(Collectors.toList())

```



以上Java 8的函数式特性全部讲完，这只是一个入门讲解，不能涵盖所有的特性及工具类。有兴趣的同学，自行探索java.util.stream包。