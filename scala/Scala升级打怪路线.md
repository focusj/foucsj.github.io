Continuous Updating...



###第一部分，入门

1. 变量声明方式：var，val，lazy val

    - var声明可变变量；val声明不可变变量。推荐使用val。

    - lazy val惰性赋值：在变量使用时进行赋值，赋值后值将不可变。

    - 编译器自动推断类型，无需显示指明变量类型。


2. 函数一等公民

    - 函数赋值给变量；

    - 函数当做参数传递；
    
    - 将函数作为返回值返回； 


3. 字符串

    - "hello world"    
    - string interpolation: s"$a, ${b.c}"

    - 格式化的字符串
```
""" {

        attr1: hello,

        attr2: world

     } """
```


3. 程序控制: if else, for, while, try ... catch

    - 没有三木表达式，只有if...else，它也是表达式

    - for & for...yield

```
for {

    i ← list

    if(i % 2 == 0)

} println(i)
```


    - try ... catch
  
```
try {

    ...

} catch {

    case e1: Exception =>

    case e2: Exception => 

}
```


1. 模式匹配：match...case

    - 表达式，有返回值；

    - 匹配上即跳出，全部未匹配throw error

    - 搭配case class



2. 类

    - class C(arg: Int)

    - class C(var arg: Int)

    - class C(val arg: Int)

    - class C(private val arg: Int)

    - abstract class

使用val声明的变量不能通过`=`重新赋值



7. object

    - 单例

    - 伴生对象

        - 伴生对象用于对象的构建



8. 样本类：case class

    - 自带apply，unapply

    - sealed 武装

    - 搭配模式匹配



9. trait

    - 功能更强大的接口

特质相较与接口和抽象类功能更为强大，而且特质可以保证抽象的粒度更小一下。当明确了继承关系时使用抽象类，但只是抽象某一特性时要使用特质。



10. 集合

    - List

        - head，tail

        - Nil

        - 提取：val List(head, tail) = List(1, 2, 3)    

    - Seq        

    - Map

    - Array

    - Set



11. 集合操作

    - map

    - foreach

    - flatMap

    - fold

    - reduce

    - filter



12. 数据结构

    - Option

         - say good-bye to null

        - 支持集合操作



    - Tuple

        - (t1, t2), t1 -> t2

        - Map(t1 -> t2)

        - 两元素集合



    - Either

        - 左error，右result



13. Template String

    - s"${hello},  ${firstName + lastName}"



14. import 

    - 位置：文件头，文件中，方法中

    - 全部引用 import package._，单个引用import package.c1



15. App特质

    - 启动程序



16. 重复参数

    - def fn(i: Int*)

    - fn(1, 2, 3)

    - fn(1, 2, 3:_*)



###二，进阶

1. 并发集合

    - List(1, 2, 3, 4).par



2. 懒集合、视图

    - Stream    

        - 递归数据结构

        - 延迟计算

        - 拿多少给多少

    - View

        - 延迟计算

        - 合并复杂计算



3. map/reduce/fold/flatMap

    - 假设我们有一筐苹果，fold操作就好像是我们要把这个框子中的苹果打成果酱，我们从框中把苹果一个一个取出来扔到机器中，最后得到了一滩果酱；

    - map操作就好像我们要给每个苹果套一个包装袋，每套完一个扔到另外的框中，最后得到了一筐带包装的苹果。



3. Actor

    - message-driven

    - stateful

    - 最小并发单元



4. [promise和future](https://docs.scala-lang.org/overviews/core/futures.html)
- future是提交一段可异步执行的代码，创建过程需要上下文中提供一个可用的ExecutionContext，创建完成后便不可修改.
- promise是一个容器，我们可以通过success和failure来填充这个promise，同时promise通过future接口，将结果转化为一个future。




1. 第一等函数

    - 闭包

        - 闭包是一个函数，这个函数包含自由变量，绑定变量。对自由变量赋值的过程就称作闭包。


    - 偏应用

        - *偏函数*只对其定义域的一部分做处理。

        - 对于一个函数，只提供其部分（或者不）参数，就构成一个偏应用函数。

        - 偏应用函数是函数



    - 赋值/传递

        - 函数可以随意用来赋值和传递



    - 高阶函数

        -  def f2(f: ()=>Unit) { f() }



    - 匿名函数

        - f2(() => println("hello")) //匿名函数

       

2. 尾递归

    - 递归不再可怕

如果一个递归函数在最后一步调用自己（这里调用自己只是函数调用），那么这个函数的栈是可以被复用的。



7. 柯里化

    - 柯里化是把多参函数转化为单个参数逐一调用的方式。



8.  类型参数



9.  型变



10. [抽取器](https://docs.scala-lang.org/tour/extractor-objects.html)



11. 依赖注入



12. trait



###三，进阶

1. 隐式转换



2. 类型系统



3. [类型类](https://scalac.io/typeclasses-in-scala/)
类型类由三部分构成：行为定义，由trait定义某一类行为；行为触发器，通过隐式参数查找特定类型的行为；类型行为定义。

```
  trait Show[A] {
    def show(A: A): String
  }

  object Shows {
    def show[A](a: A)(implicit show: Show[A]) = show.show(a)

    implicit val showInt: Show[Int] = new Show[Int] {
      override def show(i: Int): String = {
        i.toString
      }
    }
  }

  import Shows._
  show(10)
```


4. duck type 



5. type lambada



6. TypeTag



7. Functor，Monad



8. Combinator parsers

    

