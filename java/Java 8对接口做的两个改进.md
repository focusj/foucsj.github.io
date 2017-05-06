Java 8对接口做了两个改进: 一个是Default Method(默认方法), 另一个是Functional Interface(函数式接口). 在Java 8之前, 接口中只能定义抽象方法(不能有方法体). 而默认方法允许接口方法可以有默认的实现. 看一下ConsoleLog这个接口, 它包含了一个默认方法log.
```
interface ConsoleLog {
    default void log(String message) {
        System.out.println(message);
    }
}
```

我们知道接口是面向抽象编程的基础, 默认方法则在此基础上又增添了一定的代码重用能力. 来看一下引入默认方法的背景. 为了更方便的遍历集合, JDK开发人员想要在Iterable接口中加入一个新的方法foreach. foreach在好多语言(比如Scala, Python, Js, Ruby等等)中都已支持, 你定义一个函数, foreach可以把你定义的函数应用到集合中所有元素. Iterable是Collection的父接口, 集合类库中大多数实现都派生自Collection. 这意味着很多类都要作出相应修改. 这绝对是不现实的. 于是默认方法应运而生. 下面的方法即是Iterable中的forEach实现:
```
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```
有了默认方法, 接口用起来就方便多了. 不用在向之前那样, 声明一个接口, 再弄一个抽象类作默认实现. 现在需要默认实现的方法直接在接口里实现就行了. 默认方法确实是方便了, 但是我们想一个场景, 假如一个接口中的默认方法和另外一个接口或父类冲突怎么办? Java制定了以下规则来解决这个问题:
- 如果与父类中冲突了, 选择父类中的实现.
- 如果两个接口方法冲突了, 必须手动覆盖该方法.

另一个改进就是Functional Interface(函数式接口). 函数式接口是为了配合lambda而对接口提出的一个规范. 如果一个接口只包含一个抽象方法, 那么这个接口就是函数式接口, 可以使用@FunctionalInterface标记, 也可以不用. 函数式接口最大的好处是可以使用lambda创建其对象.
```
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}

//implementing
final Runnable r = () -> System.out.println("Running...");
```