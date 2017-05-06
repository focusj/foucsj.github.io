    Scala运行在JVM上，在JVM上有一种机制叫做类型擦除（type eraser）。类型擦除是说：在语言的编译阶段程序中所携带的泛型信息都会被擦除，最终生成的class文件中是不包含类型信息的。所以scala 2.8中引入了Manifests类来解决构建Array时遇到的一些问题。然而引入的Manifests并不完美，对于路径依赖类型，它拿到的信息是不精确的。所以后续引入了TypeTags。



展示一下各自的用法。看一个例子，利用manifest获取类型信息。

```scala

    scala> class A[T]

    

    scala> manifest[List[A[Int]]]

    res1: Manifest[List[A[Int]]] = scala.collection.immutable.List[A[Int]]

```



manifest定义在Predef中，直接使用即可。与manifest功能类似的一个方法是classManifest，

```scala    

    scala> classManifest[List[A[Int]]]

    warning: there was one deprecation warning; re-run with -deprecation for details

    res2: ClassManifest[List[A[Int]]] = scala.collection.immutable.List[A[Int]]

```



两个方法都可以拿到相应传入的类型信息，但是观察返回结果：res1的类型是Manifest[List[A[Int]]]，res2的类型是ClassManifest[List[A[Int]]]，两个东西还是有一些区别。接着再看一个例子：

```scala

    scala> manifest[List[A[_]]]

    res3: scala.reflect.Manifest[List[A[_]]] = scala.collection.immutable.List[A[_ <: Any]]



    scala> classManifest[List[A[_]]]

    warning: there was one deprecation warning; re-run with -deprecation for details

    res4: scala.reflect.ClassTag[List[A[_]]] = scala.collection.immutable.List[A[<?>]]

```



在这个例子中manifest会返回所有的类型信息，而classManifest返回了List[A]两层类型，丢失最内部的泛型信息。通过看源码注释对ClassManifest的解释，针对高阶类型它可能不会给出精确的类型信息。所以如果想要得到精确地类型信息使用manifest。



通常Manifest会以隐式参数和上下文绑定的形式使用。

```scala

    def method[T](arg: T)(implicit m: Manifest[T]) = m     //implicit parameter



    def method[T: Manifest](arg: T) = m   //context bound style

```



那么，有了Manifest，还要TypeTag干吗？

```scala

    scala> class Foo { class Bar}

    defined class Foo



    scala> def m(f: Foo)(b: f.Bar)(implicit t: Manifest[f.Bar]) = t

    m: (f: Foo)(b: f.Bar)(implicit t: scala.reflect.Manifest[f.Bar])scala.reflect.Manifest[f.Bar]



    scala> val f = new Foo; val b = new f.Bar

    f: Foo = Foo@6fe29d36

    b: f.Bar = Foo$Bar@6bf7d9d



    scala> val ff = new Foo; val bb = new ff.Bar

    ff: Foo = Foo@2fdb85b8

    bb: ff.Bar = Foo$Bar@75280b93



    scala> m(f)(b)

    res20: scala.reflect.Manifest[f.Bar] = Foo@6fe29d36.type#Foo$Bar



    scala> m(ff)(bb)

    res21: scala.reflect.Manifest[ff.Bar] = Foo@2fdb85b8.type#Foo$Bar



    scala> res20 == res21   //not expected

    res23: Boolean = true

```    

很明显f.Bar和ff.Bar是不相同的类型。所以，scala2.10 引入了TypeTags来解决上述问题，TypeTag可以看成是Manifest的升级版。

```scala

def m2(f: Foo)(b: f.Bar)(implicit ev: TypeTag[f.Bar]) = ev

val ev3 = m2(f1)(b1)

val ev4 = m2(f2)(b2)

ev3 == ev4 //false

```

和TypeTag一同引入的还有ClassTag和WeakTypeTag。ClassTag和ClassManifest一样，也不会返回完整的类型信息。但是它好classManifiest还是有区别的。

```scala

scala> classTag[List[_]]

res9: scala.reflect.ClassTag[List[_]] = scala.collection.immutable.List



scala> classManifest[List[_]]

warning: there was one deprecation warning; re-run with -deprecation for details

res10: scala.reflect.ClassTag[List[_]] = scala.collection.immutable.List[<?>]



scala> classTag[List[List[_]]]

res11: scala.reflect.ClassTag[List[List[_]]] = scala.collection.immutable.List



scala> classManifest[List[List[_]]]

warning: there was one deprecation warning; re-run with -deprecation for details

res12: scala.reflect.ClassTag[List[List[_]]] = scala.collection.immutable.List[scala.collection.immutable.List[<?>]]

```

classTag只拿到了基础类型信息，没有尝试去拿任何泛型信息，而classManifest拿到了所有已知的类型信息。WeakTypeTag相较TypeTag可以处理带有泛型的参数。看下边的例子：

```scala

scala> :pas

// Entering paste mode (ctrl-D to finish)



def weakParamInfo[T](x: T)(implicit tag: WeakTypeTag[T]): Unit = {

  val targs = tag.tpe match { case TypeRef(_, _, args) => args }

  println(s"type of $x has type arguments $targs")

}



// Exiting paste mode, now interpreting.



weakParamInfo: [T](x: T)(implicit tag: reflect.runtime.universe.WeakTypeTag[T])Unit



scala> def foo[T] = weakParamInfo(List[T]())

foo: [T]=> Unit



scala> foo[Int]

type of List() has type arguments List(T)

```



参考文档：

[typetags-manifests](http://docs.scala-lang.org/overviews/reflection/typetags-manifests.html)

[scala what is a typetag and how do i use it](http://stackoverflow.com/questions/12218641/scala-what-is-a-typetag-and-how-do-i-use-it#)

[scala-type-system-manifest-vs-typetag](http://hongjiang.info/scala-type-system-manifest-vs-typetag/)

[What-s-the-difference-between-ClassManifest-and-Manifest-td](http://scala-programming-language.1934581.n4.nabble.com/What-s-the-difference-between-ClassManifest-and-Manifest-td2125122.html)















