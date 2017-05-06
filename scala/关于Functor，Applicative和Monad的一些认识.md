>学习函数式编程时，每个学习到这一章节的人都会写一篇博客尝试把自己的理解记录下来



这好像是一个诅咒。从另一方面讲也证明了这一章节知识的难度之高。什么Functor，什么Monad在之前的工作和学习中压根就没有听过这些东西。即便是看了一些讲解的博客和书籍，感觉还是半知半解。这些概念的抽象程度实在太高了，涉及到了数学领域的范畴论。



###Functor



函子（Functor）：Functor表示范畴A与范畴B之间的映射。映射的是一种过程；范畴对应Scala中的高阶类型T[_]。在Scala中常见的高阶类型有Option，集合类型，Function等。Functor的典型招式就是map，比如Some(5).map(_ + 1); List(1, 2, 3).map(_ + 1)。Function这个稍微有点区别，不是调用map函数，而是compose函数：



```scala

scala> val f1 = (x: Int) => x + 1

f1: Int => Int = <function1>



scala> val f2 = (x: Int) => x + 1

f2: Int => Int = <function1>



scala> f1.compose(f2)

res2: Int => Int = <function1>



scala> res2(1)

res3: Int = 3

```



举个例子：Some(3).map(_ + 3)，Functor在背后做的事情是这样的：

![functor](http://7xjjxu.com1.z0.glb.clouddn.com/w1024.png)



###Applicative 



我们把类似Option这一类值当作封装过的值，Functor可以把这类封装过的值拆包，并执行函数映射。但是，当Function也是被包装过时，Functor还能发挥作用吗？看一个实验：



```scala

scala> val fop = Some((x: Int) => x * x)

fop: Some[Int => Int] = Some(<function1>)



scala> val op1 = Some(1)

op1: Some[Int] = Some(1)



scala> op1.map(fop _)

<console>:10: error: type mismatch;

 found   : () => Some[Int => Int]

 required: Int => ?

              op1.map(fop _)

                      ^



scala> op1.map(o => fop.map(f => o))

res14: Option[Option[Int]] = Some(Some(1))

```



通过看上边的log发现是不可以的。其实通过代码也可以看出这是不可行的，op1是一个Option类型，如果执行内部函数应该使用map方法。通过map函数虽然可以执行到函数但是返回值却不是我们期望的：`Option[Option[Int]]`。OK，Applicative出场。



Applicative的作用是: 应用封装过的函数到封装过的值。正好使用我们的适用场景。看代码：



```scala

scala> val op1 = Some(1)

op1: Some[Int] = Some(1)



scala> val op2 = Some(2)

op2: Some[Int] = Some(2)



scala> for(o1 <- op1; o2 <- op2) yield o1 + o2

res7: Option[Int] = Some(3)



scala> val fop = Some((x: Int) => x * x)

fop: Some[Int => Int] = Some(<function1>)



scala> for(f <- fop; o <- op1) yield f(o)

res9: Option[Int] = Some(1)

```



接下来看一个看一个有意思的case；上边说了Collection类型也可以应用Functor。假如我们把一组函数映射到一组值会发生什么呢？



```scala

scala> val f1 = (x: Int) => x + 1

f1: Int => Int = <function1>



scala> val f2 = (x: Int) => x + 2

f2: Int => Int = <function1>



scala> val fs = List(f1, f2)

fs: List[Int => Int] = List(<function1>, <function1>)



scala> for(f <- fs; x <- List(1, 2, 3)) yield f(x)

res15: List[Int] = List(2, 3, 4, 3, 4, 5)

```

List(1, 2, 3)映射List(f1, f2)之后得到了List(2, 3, 4, 3, 4, 5)，非常神奇。



###Monad

"说白了，Monad不就是自函子范畴上的幺半群嘛"。话是这么说，但是究竟能有多少人能够理解这个定义呢。光是自函子这个东西就足够让人抓狂了。从现实使用场景出发，Monad的招牌动作就是flatMap，表示一类能够被压缩的东西。比如List(List(1), List(2)) flat 之后是 List(1, 2) , Some(Some(2)) flat之后是 Some(2)

