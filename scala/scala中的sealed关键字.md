sealed关键字可以修饰类和特质（特质）。密封类提供了一种约束：不能在类定义的文件之外定义任何新的子类。然而，这样做有什么用呢？

在scala源码中[List](
https://github.com/scala/scala/blob/v2.11.6/src/library/scala/collection/immutable/List.scala#L76)的实现用到了sealed关键字。



抽象类List有sealed关键字修饰，接下来Nil和::分别继承了该List抽象类。那我们来尝试实现基于List实现一个自己的List：

```scala

scala> class NewList extends List

<console>:7: error: illegal inheritance from sealed class List

       class NewList extends List



```

编译出错了，我们是不能在外部文件继承List类。所以，如果子类都明确的情况下，为了防止继承滥用，为抽象类添加sealed。



模式匹配是scala非常好用的一个语法特性。但是，如果当条件经常改变，我们可能会忘了修改相应的case，那么这种情况就可能会出现错误。看下边例子：

```scala

scala> :pas

// Entering paste mode (ctrl-D to finish)



  abstract class People

  case object American extends People

  case object Japanese extends People

  case object Chinese extends People

  case object Russia extends People



  def people(p: People) = p match {

    case American ⇒ println("American person")

    case Japanese ⇒ println("Japanese person")

    case Chinese ⇒ println("Chinese person")

  }



// Exiting paste mode, now interpreting.



defined class People

defined object American

defined object Japanese

defined object Chinese

defined object Russia

people: (p: People)Unit



scala> people(American)

American person



scala> people(Russia)

scala.MatchError: Russia (of class Russia$)

  at .people(<console>:13)

  ... 33 elided

```

这时我们为People加上sealed，当我们编译代码时得到了一个警告：

```scala

<console>:18: warning: match may not be exhaustive.

It would fail on the following input: Russia

         def people(p: People) = p match {

```

多温馨的一个提示啊。事情都不是绝对的，如果确定people只处理美国，日本，中国人，编译时总是给这么个警告也挺闹心。能破吗？可定能，看代码：

```scala

  def people(p: People) = (p: @unchecked) match {

    case American ⇒ println("American person")

    case Japanese ⇒ println("Japanese person")

    case Chinese ⇒ println("Chinese person")

  }

```