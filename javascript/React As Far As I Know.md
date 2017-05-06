##写在开始之前



前端从来都不缺少轮子，几乎每天都有新的轮子被创造出来。满目琳琅，数不胜数。但是当下最时兴的轮子恐怕就是React。从github上来看，目前[React](http://reactjs.org)的watch数：`1521`，start数：`21522`，fork数：`3018`。对比一下目前也相对时兴的一些技术，比如说[Go](https://golang.org)，三个指标分别为：`754`，`786`，`656`；在比如说[Docker](http://docker.com)：`1848`，`21384`，`4996`。由此可见React的火热程度。有幸去年十二月份也开始接触React，陆陆续续接触React也有小半年的时间了，是时候停下来把自己知道的，已经忘记的总结一下记录下来，万一能对刚入手React的同仁有点帮助，那更是求之不得。



---



##简单介绍一下React

官方文档的介绍是这样的：React是用来构建用户界面额Javascript库。前端的框架都已经这么多了，你们Facebook还开源React作甚？下边可能是官方给出的答案：我们和别的框架不一样，我们有以下特性：

1.我们不是一个传统的前端MVC框架，我们只关注V这一层。我们有component的理念，用户可以随便定义和重用component来构建页面。假设你的项目刚刚开始你可以使用React，如果一年之前你不认识React误选了Angular，React还是可以踏踏实实为你做好UI的工作。就是这么任性。

2.React定义了自己的虚拟DOM。我们发现了这样的一个事实：Javascript的执行速度是非常快的，但是DOM操作往往是比较耗时的。所以，我们提出了虚拟DOM的理念。通过虚拟DOM我们会过滤掉很多不必要的DOM操作。我们要做性能最高的前端框架。

3.我们的数据流是单向的，我们这样的机制可以帮助程序员快速定位bug。



看到这样的框架能不动心吗，以上我们统统代表Facebook。



提纲：

-Jsx语法入门

-React的构建基础-Component

-深入理解Component

-Flux框架入门         //TODO

-Fluxthis框架入门       //TODO

-为什么React这么快[Virtual Dom]     //TODO



---



## Jsx语法入门

Jsx是React工具链中最基础的一环。Jsx是构建React Component的有力工具。Jsx之于React犹如Swift之于iOS。Jsx把javascript和xml揉在一起，在javascript中写xml标签。前几天逛论坛，有人提出了把html写到javascript文件中，大大影响了代码的美观和可读性。虽然，React提供了使用pure javascript的方式，但是，过后你会发现Jsx才是最高效的。并且写代码时费费脑子组织一下代码，代码的可读性是非常好滴。所以那个帖子中有人这样回复：

>你可以尝试非Jsx的写法，但是迟早你会发现，其实Jsx更适合你。



言归正传，首先看一个简单的例子来熟悉一下jsx语法。

```javascript

var Ul = React.createClass({

  render: function() {

  	var lis = [1, 2, 3].map(x => <li>{x}</li>)

        return (

            <ul className="ul-class">

        	{lis}

            </ul>

        );

  }

});



//编译后

var Ul = React.createClass({displayName: "Ul", render: function() {

  	var lis = [1, 2, 3].map(function(x)  {return React.createElement("li", null, x);})

        return (

        	React.createElement("ul", {className: "ul-class"}, lis)；

        );

  }

});



```

这个示例非常简单，展示1到3的数字列表。看一下render函数，首先它定义了一个lis的变量，这个变量记录了所有的li条目，每个li都是一个`<li>`标签。然后，return中的`<ul>`标签中使用了该变量。很简短的一个例子，对比一下编译后的代码，还是可以发现Jsx的版本是比较简洁明了的。如果想在html中插入javascript代码，使用`{}`就行了。比如`{ lis }`, `{true ? ... : ... }`, `{/* comment */}`这些都是支持的。但是下边的这些情况就是不可以的了`{if(true) ... }`, 虽然Jsx支持三元表达式，但是不支持if；`{var x = 1}`这也是不可以的。那我们在{}内部在嵌套一个{}吗？对不起这样做也不行。



有时候我自己也纳闷：为什么Jsx支持的语法这么匮乏。我们来看一下Jsx不支持的几种情况：

1.`{var x = 1}`。为什么要在html elements中间声明变量呢？仔细想一下数据其实都可以放到javascript代码中处理，html中只对处理好的结果做一个引用。`{lis}`这样也就够了。



2.`{if(true)....}`。在javascript中if-else是没有返回值的，是有两个表达式组成的复杂表达式，所以不支持if-else也很合理。同样把if-else逻辑放到javascript中去。



3.`{"x"} {"y"}`。这个例子是我帮一个同事调代码是看到的。这case编译是不会有问题的，但是结果不是你想要的结果。强调一下：确保在html中的每一句javascript简单，能一句话搞定的不要搞成两句。



###我日常写Jsx的一些经验

1.**确保每个render返回的结构清晰简单**。要做到这一点要遵循下边两个要求：*不要在html中混入复杂的处理逻辑*；*提取Component*。

对于第一点，html和javascript杂糅在一起肯定会导致代码可读性降低，React专注于UI，所以每个Component除了名字要代表它的意义之外，它render的html也要一目了然。

第二点，不要搞超大Component。每一个Component都是一个函数，尽量让每一个Component只做一件事情（keep it simple）。Component有View Component和Business Component之分。这里着重指的是View Component，Business Component的复杂性由具体业务决定。Component小了，相对的html结构也就简单了。



2.处理页面逻辑时不要手动修改props。把props当做一个不可变的属性对待，不要直接就这props做数据处理。



3.还有一些code style上的建议，具体参考这篇[文章](https://reactjsnews.com/react-style-guide-patterns-i-like/)吧。



---



##React的构建基础-Component



Component是React的核心理念。在React的世界里只有Component的概念，Component是构建应用的基础。就像React文档中给出的例子，搭建这样的一个应用我们需要这几种Component的组合。这是很有意思的事情，让我们写前端的时候不再关注这个页面应该有哪些div组成，每个div往哪摆这些琐碎的问题。而是站在一个更高的level对我们的应用做设计和规划。（我是一个前端门外汉，主观感受，说错了不负责任）



每个Component都是一个状态机，也就是说你把足球，篮球和棒球传入了展示Component，那么你一定会得到三种球类的展示。页面输出依赖于传入的数据，页面和传入的数据是保持一致性的。同时这种思想也符合函数编程思想，对于一个函数（component也是）相应的输入总能得到期望的输出。这对于程序员的好处就是对自己写的代码更放心。下面介绍些Component的使用。





![thinking in react](http://facebook.github.io/react/img/blog/thinking-in-react-components.png)


###Component常用Api及生命周期



1.**render**：是Component必须要有的。render方法可以使一段html（注意，这段html必须由一个标签包围，并且标签要闭合），同时也可以引用其他的Component。render()方法必须是无副作用的，不能够修改Component的状态。



2.**getInitialState**：一些Component是拥有自己的状态的，这些状态要实现在getInitialState中声明。比如：

```javascript

render: function() {

    return {

        name: "default"

    };

}

```

只有声明过之后的state才可以通过`this.state.name`使用。`getInitialState`函数只在Component挂在之前调用一次。



3.**compomentDidMount**：这个函数非常有用。如果Component依赖于后台数据，一般ajaxcall都会放在这个函数中。



4.**propsTypes**：propsTypes不是必须的，如果你需要严格的规定某个props的类型，以及指明某个props是否是必须的，可以使用该函数。



5.**shouldComponentUpdate**：这个函数是性能相关的函数，95%的情况下你不需要管它，但是需要知道它是干什么的。当Component的状态改变的时候，需要重新render一次，你可以通过这个函数控制Component是否应该重新render。



###props和state



React是非常容易上手的，哪怕是初体验者恐怕半个小时内都能跑起一两个例子。但是，用好React就得花一些功夫，其中state和props这两个概念觉得值得你花时间去研究。不论是props还是state都是都是提供数据的，但是props和state是完全不同的两种东西。



在说props和state之前先说一下Stateful Component和Stateless Component。



**Stateless Component**：这类Component可以说是Pure UI Component。它主要用作展示数据（图1中红框标记的Component），或者是一些简单的HTML标签的组合（图1中蓝框标记的Component）。



**Stateful Component**：这类Component主要负责前端后台的沟通（比如，发送ajax call），和响应页面事件（比如表单验证Component，需要一个state记录用户的输入）。



回到props和state上来：



**props**：props的职责的为Component提供配置信息，由Component外部提供。props是public的，React提供了`propsTypes`函数来限制每个props的类型，以及是否可选的（optional）。还有props是只读的，不要在Component内部手动修改props。



比如有这样一个Component：显示一个带标题的输入框。这样的Component在表单场景中很常见，复用性非常高。那这个Component的title属性应该通过props传给Component。



**state**：state寄生在Component内部，是Component私有的属性，state使用前必须通过`getInitialState`进行声明并初始化。当状态改变的时候调需要掉用`setState`使最新状态生效。每次调用`setState`方法都会触发Component重新render，切记setState方法不能在render方法中调用。state是专门针对Stateful Component的，对于Stateless 的Component完全可以忽略该属性。



###Thinking In React



先让大家看一个[例子](http://react-china.org/t/you-mei-you-xian-cheng-de-jsxtiao-jian-yu-ju-ku/713/6)：

> 条件语句在JSX里太痛苦了，把条件提到前面的话模板太乱，用？：的话条件一多就写要崩，如果有个库可以这样用：

> ```javascript

> <div className="content-body">

>     {

>         case()

>             .when(!this.state.users, <div>Loading</div>)

>             .when(!this.state.users.length, () => <div className="mute-text">No users</div>)

>             .else(() => this.state.map(user => { ... }) )

>         .end()

>     }

> </div>

> ```

>



首先，来分析一下他想要实现什么功能：应该是通过ajax call拿到一组user，然后把这些user展示到页面上，还有ajax pending时页面显示loading。非常常见的一个例子。然后我们看一下他的实现：根据user 状态3种不同的值来控制页面不同的展示结果。这种实现方式是可以解决问题的，但是这种思路不是特别的React。先给出一个比较合理的实现，做一个对比：

```javascript

React.createClass({

  getInitialState: function() {

    return {users: [], isDataReady: false} ;//when ajax done set `isDataReady` to true

  },



  render: function() {

    var usersTemplates = this.state.users.length > 0 ? this.state.users.map(user => <User user={user} />) : "no user";



    return (

      <div>

        <Splash show={ this.state.isDataReady}> //this template`s can handle if shows depends on isDataReady state

        { usersTemplates } //show users

      </div>

    )

  }

});

```

接着再来看一下上边的实现有哪些欠妥的地方：

1.对state的使用不是特别清楚，没有弄明白到底这个Component到底应该有哪些state。首先，这个Component有三种显示：Loading，No User，Users。第一种是在ajax call pending时的显示，当load到数据之后第一种显示切换为第二或第三种显示（依据是否有users）。那么这个Component到底应该有哪几个state呢？首先，users这个状态肯定需要；其次，isDataReady状态用来记录ajax call是否完成。



2.Component的抽象不清晰。实现这个需求至少需要两个组件：Splash和User。Splash在loading时显示，并且Splash是一个非常通用的组件，可以做到全站复用。有没有必要再抽一个Users的Component？这个可抽可不抽，主要看当前这个Component的功能及复杂性。





参考：

[Props vs State](https://github.com/uberVU/react-guide/blob/master/props-vs-state.md)

[Thinking in React](http://facebook.github.io/react/docs/thinking-in-react.html)

[React Tips and Best Practices](http://aeflash.com/2015-02/react-tips-and-best-practices.html)

