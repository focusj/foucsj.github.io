##背景

所有一切的开始都是因为这句话：一个单子（Monad）说白了不过就是自函子范畴上的一个幺半群而已，有什么难以理解的。第一次看到这句话是在这篇文章：[程序语言简史(伪)](http://www.oschina.net/news/41233/brief-incomplete-and-mostly-wrong/)。这句话出自Haskell大神Philip Wadler，也是他提议把Monad引入Haskell。Monad是编程领域比较难理解的概念之一，大部分人都是闻"虎"而色变，更不用说把它"收入囊中"了。我曾经好几次尝试去学习Monad，Functor等这些范畴论里的概念，最终都因为它太难理解，半途而废。



这次的开始完全是个误会。几周之前我开启了重温Scala的计划。当我看到Scala类型系统和Implicit相关章节时，遇到了Scala中比较重要的设计模式：类型类（type class）。于是想找一个大量使用了type class模式的开源类库学习其源码，以加深理解type class模式。Scalaz是个不错的选择。但是有一个问题，Scalaz是一个纯函数式的类库，学习它必然又会遇到Monad那些概念。好吧，再给自己一次机会。



##概念篇

我们分析一下Philip这句话：*一个单子（Monad）说白了不过就是自函子范畴上的一个幺半群而已*。这句话涉及到了几个概念：单子（Monad），自函子（Endo-Functor），幺半群（Monoid），范畴（category）。首先，我们先来把这些概念搞清楚。



###范畴

####范畴的定义

范畴由三部分组成：

1. 一组对象。

2. 一组态射（morphisms）。每个态射会绑定两个对象，假如f是从源对象A到目标对象B的态射，记作：f：A -> B。

3. 态射组合。假如h是态射f和g的组合，记作：h = f o g。



下图展示了一个简单的范畴，该范畴由对象 A, B 和 C 组成，有三个单位态射 id_A, id_B 和 id_C ，还有另外两个态射 f : C => B 和 g : A => B 。



![Simple-cat.png](http://upload-images.jianshu.io/upload_images/78847-a7ab925a26ae1ee0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



态射我们可以简单的理解为函数，假如在某范畴中存在一个态射，它可以把范畴中一个Int对象转化为String对象。在Scala中我们可以这样定义这个态射：`f : Int => String = ...`。所以态射的组合也就是函数的组合，见代码：

```scala

scala> val f1: Int => Int = i => i + 1

f1: Int => Int = <function1>



scala> val f2: Int => Int = i => i + 2

f2: Int => Int = <function1>



scala> val f3 = f1 compose f2

f3: Int => Int = <function1>

```



####范畴公理

范畴需要满足以下三个公理。

1. 态射的组合操作要满足结合律。记作：`f o (g o h) = (f o g) o h`



2. 对任何一个范畴 C，其中任何一个对象A一定存在一个单位态射，`id_A: A => A`。这个态射是组合操作的单位元。`g o id_A = id_B o g = g`。



3. 态射在组合操作下是闭合的。所以如果存在态射` f:A => B` 和` g:B => C `，那么范畴中必定存在态射 `h:A => C` 使得 `h=f => g `。以下面这个范畴为例：



![Composition-ex.png](http://upload-images.jianshu.io/upload_images/78847-e6fd7baaf2edda7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



f 和 g 都是态射，所以我们一定能够对他们进行组合并得到范畴中的另一个态射。那么哪一个是态射 f o g 呢？唯一的选择就是 id_A 了。类似地，g o f=id_B 。





###函子

####函子定义

函子有一种能力，把两个范畴关联在一起。函子本质上是范畴之间的转换。比如对于范畴 C 和 D ，函子` F : C => D` 能够：将 C 中任意对象a 转换为 D 中的 F(A);  将 C 中的态射` f : A => B` 转换为 D 中的 `F(f) : F(A) => F(B)`



下图表示从范畴C到范畴D的函子。图中的文字描述了对象 A 和 B 被转换到了范畴 D 中同一个对象，因此，态射 g 就被转换成了一个源对象和目标对象相同的态射（不一定是单位态射），而且 id_A 和 id_B 变成了相同的态射。对象之间的转换是用浅黄色的虚线箭头表示，态射之间的转换是用蓝紫色的箭头表示。



![Functor.png](http://upload-images.jianshu.io/upload_images/78847-7360d9f9f818e25e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



####单位函子

每一个范畴C都可以定义一个单位函子：`Id： C => C`。它将对象和态射直接转换成它们自己：`Id[A] = A; f: A => B, Id[f] = f`。



####函子公理

1. 给定一个对象 A 上的单位态射Id_A ， F(Id_A) 必须也是 F(A) 上的单位态射，也就是说：F(Id_A) = Id_(F(A))

2. 函子在态射组合上必须满足分配律，也就是说：F(f o g) = F(f) o F(g)



####自函子

自函子是一类比较特殊的函子，它是一种将范畴映射到自身的函子 (A functor that maps a category to itself)。



函子这部分定义都很简单，但是理解起来会相对困难一些。如果范畴是一级抽象，那么函子就是二级抽象。起初我看函子的概念时，由于其定义简单，并且我很熟悉map这种操作，所以一带而过。当看到Monad时，发现了一些矛盾的地方。返回头再看，当初的理解是错误的。所以，在学习这部分概念时，个人有一些建议：1. 函子是最基本，也是最重要的概念，这个要首先弄明白。本文后半部分有其代码实现，结合代码去理解。如何衡量已经明白其概念呢？脑补map的工作过程+自己实现Functor。2. 自函子也是我好长时间没有弄明白的概念。理解这个概念，可以参看Haskell关于Hask的定义。然后类比到Scala，这样会容易一些。



###群

下边简单介绍群相关的概念。相比函子、范畴，群是相对容易理解的。

####群的定义

群表示一个拥有满足封闭性、结合律、有单位元、有逆元的二元运算的代数结构。我们用G表示群，a，b是群中元素，则群可以这样表示：

1. 封闭性（Closure）：对于任意a，b∈G，有a*b∈G

2. 结合律（Associativity）：对于任意a，b，c∈G，有(a\*b)\*c=a\*(b\*c)

3. 单位元或幺元 （Identity）：存在幺元e，使得对于任意a∈G，e\*a=a\*e=a

4. 逆元：对于任意a∈G，存在逆元a^-1，使得a^-1\*a=a\*a^-1=e



####半群和幺半群

半群和幺半群都是群的子集。只满足封闭性和结合律的群称为**半群**（SemiGroup）；满足封闭性，结合律同时又有一个单位元，则该群群称为**幺半群**。



概念到此全部介绍完毕。数学概念定义通常都很简单，一句两句话搞定，但是由于其抽象程度高，往往很难理解。下边我们将通过Scala来实现其中的一些概念。



##Scala和范畴论

大谈了半天理论，回到编程中来。对程序员来说，离开代码理解这些定义是困难的，没有实际意义的。

####群的代码表示

由于实际应用中不会涉及到群，所以我们来看半群的代码表示。从上边的概念我们知道，半群是一组对象的集合，满应足封闭性和结合性。代码如下：

```scala

trait SemiGroup[A] {

    def op(a1: A, a2: A): A    

}

```

A表示群的构成对象，op表示两个对象的结合，它的封闭性由抽象类型A保证。接着来看Monoid的定义，Monoid是SemiGroup的子集，并且存在一个幺元。代码如下：

```scala

trait Monoid[A] extends SemiGroup[A]{

    def zero: A

}

```

下边给出了三个例子，分别是string、list和option的幺半群实现。对于不同的幺半群群，它们的结合行为，和幺元是不一样的。当自己实现一个群时一定要注意这点。比如对于Int的幺半群，在加法和乘法的情况下幺元分别是0和1。

```scala

val stringMonoid = new Monoid[String] {

 def op(a1: String, a2: String) = a1 + a2



 def zero = ""

}



def listMonoid[A] = new Monoid[List[A]] {

 def op(a1: List[A], a2: List[A]) = a1 ++ a2



 def zero = Nil

}



def optionMonoid[A] = new Monoid[Option[A]] {

 def op(a1: Option[A], a2: Option[A]) = a1 orElse a2



 def zero = None

}

```



####Functor的代码表示

```scala

trait Functor[F[_]] {

 def map[A, B](a: F[A])(f: A => B): F[B]

}



//list Functor的实现

def listFunctor = new Functor[List] {

 def map[A, B](a: List[A])(f: (A) => B) = a.map(f)

}

```

Functor代码是很简单的，但是，也不是特别容易理解（和其概念一样）。我在理解这段代码的时候又遇到了问题。第一个问题：A -> F[A]这个映射在哪里？第二个问题：A => B => F[A] => F[B]这个映射又体现在哪里？以下是我的理解：

1. Functor的定义带有一个高阶类型F[ \_ ]。在Scala里，像List[T]，Option[T]，Either[A, B]等这些高阶类型在实例化时必须要确定类型参数（把T，A，B这些类型称为类型参数）。所以，A->F[A]这条映射产生在F[ \_ ]类型实例化的时候。List[Int]隐含了这样一条映射：Int => List[Int]。

2. 要理解这个映射关系：A => B => F[A] => F[B]，首先来看listFunctor.map的使用。`map[Int, Int](List(1, 2, 3))(_ + 1)`，对于map它的入参是`List(1， 2， 3)`，执行过程是List中的每一个元素被映射该函数`_: Int + 1`，得到的结果List(2, 3, 4)。所以，对于List这个范畴来说，这个过程其实就是：List[Int] => List[Int]。放眼到Int和List范畴，就是Int => Int => List[Int] => List[Int]



###Monad

OK，该介绍的背景知识都说的差不多了。我们接下来看Monad。Monad的定义是这样的：Monad（单子）是从一类范畴映射到其自身的函子（天呐，和自函子的定义一模一样啊）。我们来看详细的定义：

> Monad是一个函子：M: C -> C，并且对C中的每一个对象x以下两个态射：

> 1. unit: x -> M[x]

> 2. join/bind: M[M[x]] -> M[x]



第一个态射非常容易理解，但是第二个是什么意思呢？在解释它之前我们先来看一个例子：



```scala

scala> val s = Some(1) //1

s: Some[Int] = Some(1)



scala> val ss = s.map(i => Some(i + 1)) //2

ss: Option[Some[Int]] = Some(Some(2))



scala> ss.flatten //3

res6: Option[Int] = Some(2)



scala> val sf = s.flatMap(i => Some(i + 1)) //4

sf: Option[Int] = Some(2)

```



程序第二步，把Monad当做一个普通的函子执行map操作，我们得到了Some(Some(2))，然后执行flatten操作，得到了最终的Some(2)。也就是说，join就是map + flatten。接着看第四步，flatMap一次操作我们就得到了期望的结果。join其实就是flatMap。



接下来我们用Scala实现Monad的定义：



```scala

trait Monad[M[_]] {

 def unit[A](a: A): M[A]   //identity

 def join[A](mma: M[M[A]]): M[A]

}

```

还有一种更为常见的定义方式，在Scala中Monad也是以这种方式出现：

```

trait Monad[M[_]] {

 def unit[A](a: A): M[A]

 def flatMap[A, B](fa: M[A])(f: A => M[B]): M[B]

}

```

其实这两种定义方式是等价的，join方法是可以通过flatMap推导出来的：```def join[A](mma: M[M[A]]): M[A] = flatMap(mma)(ma => ma)```



##结尾

不知道大家对Monad的概念有没有一个大概的了解了？其实它就是一个自函子。所以，当理解了函子的概念时，Monad已经掌握了百分之八九十。剩下的百分之十就是不断的练习和强化了。

那我们再回到Philip的这句话：`一个单子（Monad）说白了不过就是自函子范畴上的一个幺半群而已`。该如何理解这句话？我就不再费劲去解释了，如果上边的概念都弄明白了，这句话自然也就明白了。另外，受限于个人的能力，及语言表达水平，文中难免有错误。为不影响大家追求真理，给出我学习时所参看的一些资源。



参看文档：

《Functional programming in scala》

http://stackoverflow.com/questions/3870088/a-monad-is-just-a-monoid-in-the-category-of-endofunctors-whats-the-problem

http://www.zhihu.com/question/24972880

http://jiyinyiyong.github.io/monads-in-pictures/

http://hongjiang.info/scala/

http://yi-programmer.com/2010-04-06_haskell_and_category_translate.html#id5

http://www.jdon.com/idea/monad.html    

