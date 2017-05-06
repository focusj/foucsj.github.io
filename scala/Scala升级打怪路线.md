Continuous Updating...



###第一部分，入门

1. 变量声明：var，val，lazy val

    - var声明可变变量；val声明不可变变量。推荐使用val。

    - lazy val惰性赋值：在变量调用时执行赋值，赋值后值不可变。

    - 编译器自动推断类型，无需显示指明变量类型。



2. 函数定义：val，def

    - val定义函数变量。

    - 无需显示return，默认最后一行为返回结果



3. 字符串

    - "hello world"    

    - 保存字符串格式

""" {

        attr1: hello,

        attr2: world

     } """



3. 控制结构: if else, for, while, try ... catch

    - 没有？：，只有if else

    - if else有返回值

    - for 

for {

    i ← list

    if(i % 2 == 0)

} println(i)



    - try ... catch

try {

    ...

} catch {

    case e1: Exception =>

    case e2: Exception => 

}



5. 模式匹配：match...case

    - 表达式，有返回值；

    - 匹配上即跳出，全部未匹配throw error

    - 搭配case class



6. 类

    - class C(arg: Int)

    - class C(var arg: Int)

    - class C(val arg: Int)

    - class C(private val arg: Int)

    - abstract class

使用val声明的变量不能通过`=`重新赋值



7. object

    - 单例

    - 伴生对象

        - 对象的构建



8. 样本类：case class

    - 自带apply，unapply

    - sealed武装

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

    ﻿- message-driven

    ﻿- stateful

    ﻿- 最小并发单元



4. promise和future

    ﻿



5. 第一等函数

    - 闭包

        - 函数字面量在运行时创建的函数值称为闭包。

       - 自由变量，绑定变量。

其实就是根据上下文对自由变量赋值的过程。



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

       



8. 尾递归

    ﻿- 递归不再可怕

尾递归和递归不一样的是，Scala编译器检测到尾递归就用新值更新函数参数，然后把它替换成一个回到函数开头的跳转。相当与一次新的函数调用。



9. 柯里化

    - 柯里化是把多参函数转化为单个参数逐一调用的方式。



10. 类型参数



11. 型变



12. 抽取器



13. 依赖注入



14. trait



###三，进阶

1. 隐式转换



2. 类型系统



3. 类型类



4. duck type 



5. type lambada



6. TypeTag



7. Functor，Monad



8. Combinator parsers

    

