###什么是隐式转换

我们经常引入第三方库，但当我们想要扩展新功能的时候通常是很不方便的，因为我们不能直接修改其代码。scala提供了隐式转换机制和隐式参数帮我们解决诸如这样的问题。

Scala中的隐式转换是一种非常强大的代码查找机制。当函数、构造器调用缺少参数或者某一实例调用了其他类型的方法导致编译不通过时，编译器会尝试搜索一些特定的区域，尝试使编译通过。



场景一，现在我们要为Java的File类提供一个获得所有行数的方法：

```scala

  implicit class Files(file: File) {

    def lines: Array[String] = {

      val fileReader: FileReader = new FileReader(file)

      val reader = new BufferedReader(fileReader)

      try {

        var lines = Array[String]()

        var line = reader.readLine()



        while (line != null) {

          lines = lines :+ line

          line = reader.readLine()

        }

        lines

      } finally {

        fileReader.close()

        reader.close()

      }

    }

  }



  private val file: File = new File("/path/to")



  file.lines foreach println

```



场景二，我期望可以像操作集合那样来操作一个文件中的所有行。比如，对所有的行映射（map）一个指定函数。

```scala

  implicit def file2Array(file: File): Array[String] = file.lines



  def map[R](source: Array[String])(fn: String ⇒ R) = {

    source.map(fn)

  }



  map(new File("/path/to"))(println)

```



###隐式操作规则

1. 标记规则：只有标记为implicit的变量，函数或对象定义才能被编译器当做隐式操作目标。



2. 作用域规则：插入的隐式转换必须是单一标示符的形式处于作用域中，或与源/目标类型关联在一起。单一标示符是说当隐式转换作用时应该是这样的形式：file2Array(arg).map(fn)的形式，而不是foo.file2Array(arg).map的形式。假设file2Array函数定义在foo对象中，我们应该通过`import foo._`或者`import foo.file2Array`把隐式转换导入。简单来说，隐式代码应该可以被"直接"使用，不能再依赖类路径。

假如我们把隐式转换定义在源类型或者目标类型的伴生对象内，则我们可以跳过单一标示符的规则。因为编译器在编译期间会自动搜索源类型和目标类型的伴生对象，以尝试找到合适的隐式转换。



3. 无歧义规则：不能存在多于一个隐式转换使某段代码编译通过。因为这种情况下会产生迷惑，编译器不能确定到底使用哪个隐式转换。



4. 单一调用规则：不会叠加（重复嵌套）使用隐式转换。一次隐式转化调用成功之后，编译器不会再去寻找其他的隐式转换。



5. 显示操作优先规则：当前代码类型检查没有问题，编译器不会尝试查找隐式转换。



###隐式解析的搜索范围

隐式转换本身是一种代码查找机制，所以下面会介绍隐式转换的查找范围：

-当前代码作用域。最直接的就是隐式定义和当前代码处在同一作用域中。

-当第一种解析方式没有找到合适的隐式转换时，编译器会继续在隐式参数类型的隐式作用域里查找。一个类型的隐式作用域指的是与该类型相关联的所有的伴生对象。



**对于一个类型T它的隐式搜索区域包括如下：**

-假如T是这样定义的：T with A with B with C，那么A, B, C的伴生对象都是T的搜索区域。

-如果T是类型参数，那么参数类型和基础类型都是T的搜索部分。比如对于类型List[Foo]，List和Foo都是搜索区域

-如果T是一个单例类型p.T，那么p和T都是搜索区域。

-如果T是类型注入p#T，那么p和T都是搜索区域。



所以，只要在上述的任何一个区域中搜索到合适的隐式转换，编译器都可以使编译通过。



看两个例子：

1. 通过类型参数获得隐式作用域

```scala

scala> implicit val i: Int = 1

i: Int = 1

scala> implicitly[Int]

res11: Int = 1

```



2. 通过嵌套获得隐式作用域

```scala

object Foo {

  trait Bar

  implicit val bar = new Bar {

    override def toString = "Foo`s Bar"

  }

}

object Test extends App {

  import Foo.Bar

  class B extends Bar

  def m(implicit bar: Bar) = println(bar.toString)

  m

}

```



###常用法

####转换类型为期望的类型

```scala

scala> val i: Int = 3.5

<console>:7: error: type mismatch;

 found   : Double(3.5)

 required: Int

       val i: Int = 3.5

                    ^

scala> implicit def double2Int(d: Double) = d.toInt

warning: there was one feature warning; re-run with -feature for details

double2Int: (d: Double)Int



scala> val i: Int = 3.5

i: Int = 3

```

当我们尝试把一个带有精度的数字复制给Int类型时，编译器会给出编译错误，因为类型不匹配。当我们创建了一个double to int的隐式转换之后编译正常通过。还有一种情况是与新类型的操作。

```scala

  case class Rational(n: Int, d: Int) {

    def +(r: Rational) = Rational(n + r.n, d + r.d)

  }



  implicit def int2Rational(v: Int) = Rational(v, 1)



  Rational(1, 1) + Rational(1, 1)



  1 + Rational(1, 1)

```



####模拟新的语法

比如scala中的arrow（->）语法就是一个隐式转换

```scala

  implicit final class ArrowAssoc[A](private val self: A) extends AnyVal {

    @inline def -> [B](y: B): Tuple2[A, B] = Tuple2(self, y)

    def →[B](y: B): Tuple2[A, B] = ->(y)

  }

```



####类型类

类型类是一种非常灵活的设计模式，可以把类型的定义和行为进行分离，让扩展类行为变得非常方便。

```scala

 @implicitNotFound("No member of type class NumberLike in scope for ${T}")

  trait Increasable[T] {

    def inc(t: T): T

  }



  object Increasable {



    implicit object IncreasableInt extends Increasable[Int] {

      def inc(t: Int) = t + 1

    }



    implicit object IncreasableString extends Increasable[String] {

      def inc(t: String) = t + t

    }



  }



  def inc[T: Increasable](list: List[T]) = {

    val ev = implicitly[Increasable[T]]

    list.map(ev.inc)

  }



  inc(List(1, 2, 3))

  inc(List("z", "a", "b"))

```



###隐式参数

当我们在定义方法时，可以把最后一个参数列表标记为implicit，表示该组参数是隐式参数。一个方法只会有一个隐式参数列表，置于方法的最后一个参数列表。如果方法有多个隐式参数，只需一个implicit修饰即可。

当调用包含隐式参数的方法是，如果当前上下文中有合适的隐式值，则编译器会自动为改组参数填充合适的值。如果没有编译器会抛出异常。当然，标记为隐式参数的我们也可以手动为该参数添加默认值。`def foo(n: Int)(implicit t1: String, t2: Double = 3.14)`



###隐式视图

隐式视图：把一种类型转换为其他的类型，转换后的新类型称为视图类型。隐式视图会用于以下两种场景：当传递给函数的参数与函数声明的类型不匹配时；

```scala

scala> def log(msg: String) = println(msg)

log: (msg: String)Unit



scala> log("hello world")

hello world



scala> log(123)

<console>:9: error: type mismatch;

 found   : Int(123)

 required: String

              log(123)

                  ^



scala> implicit def int2String(i: Int): String = i.toString

warning: there was one feature warning; re-run with -feature for details

int2String: (i: Int)String



scala> log(123)

123

```

当调用foo.bar，并且foo中并没有bar成员时（常用于丰富已有的类库）。

```scala



scala> :pas

// Entering paste mode (ctrl-D to finish)

  class Strings(str: String) {

    def compress = str.filter(_ != ' ').mkString("")

  }



  implicit def strings(str: String): Strings = new Strings(str)



// Exiting paste mode, now interpreting.



warning: there was one feature warning; re-run with -feature for details

defined class Strings

strings: (str: String)Strings



scala> " a b c d ".compress

res0: String = abcd



```



###隐式类型

如果细心观察上边的compress的实现和文章开头lines的实现，这两段代码实现功能所采用的思路是类似的。但是，两端代码的实现形式是有区别的。lines的实现采用了scala中的`隐式类型`特性。隐式类型是scala提供的一种语法糖，隐式类型还是要转换为：类型+隐式视图的形式（也就是compress的形式）。



本文内容整理自《Scala in depth》，《Scala 编程》。

