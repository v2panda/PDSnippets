# 基于 MVC 的项目重构

大家好，今天跟大家分享的主题是 基于 MVC 的项目重构，主要是这次 iOS 重构改版上的一些总结。
分享的过程中，若有问题，请大家随便批评，我多多学习，谢谢！
好的，现在开始！

## 背景
首先说说背景，也就是为什么要重构。

因为重构是需要成本的，一不小心修改错了，就会让原来完好的产品出各种问题，所以先总结下为什么要重构

1.没有统一的代码风格
这个就比较痛苦了，因为历史的原因，或者说这个项目初期就没有代码规范，我接手项目前，两个人的代码风格就非常随意，我完全重构的时间成本比较高，就一直没去做

2.臃肿的类文件
这个好理解，一个类几千行代码，都挤在一起

3.难以理解的逻辑代码
业务逻辑代码混乱不堪，看的头疼，还不敢乱删

4.混乱的类调用
混乱的类调用，没有明确一个类该做什么，全都堆在一起

## WTF？重构！！怎么做？
所以趁这次 客户端 重写 UI 的机会，决定把整个项目重构一遍

## 基本架构
那么首先要确定的是基本架构
这里有在客户端开发上，比较热门的架构思想，如 MVC、MVVM、MVCS、VIPER等等。

依次介绍



MVVM：MVVM是基于胖Model的架构思路建立的，然后在胖Model中拆出两部分：Model和ViewModel。
重点是 ViewModel 去处理完数据要通知让View响应ViewModel的改变，
这就要求做到数据绑定，OC 的有ReactiveCocoa，Swift 有 RxSwift

MVCS：基于MVC衍生出来的一套架构，从概念上来说，它拆分的部分是Model部分，拆出来一个Store。这个Store专门负责数据存取。但从实际操作的角度上讲，它拆开的是Controller。
（MVCS使用的前提是，它假设了你是瘦Model，同时数据的存储和处理都在Controller去做。所以对应到MVCS，它在一开始就是拆分的Controller。因为Controller做了数据存储的事情，就会变得非常庞大，那么就把Controller专门负责存取数据的那部分抽离出来，交给另一个对象去做，这个对象就是Store。这么调整之后，整个结构也就变成了真正意义上的MVCS。）


不管MVVM也好，MVCS也好，他们的共识都是Controller会随着软件的成长，变很大很难维护很难测试。只不过两种架构思路的前提不同，MVCS是认为Controller做了一部分Model的事情，要把它拆出来变成Store，MVVM是认为Controller做了太多数据加工的事情，所以MVVM把数据加工的任务从Controller中解放了出来，使得Controller只需要专注于数据调配的工作，ViewModel则去负责数据加工并通过通知机制让View响应ViewModel的改变。


那么选哪个呢，或者怎么选

对此我的观点是：架构的选择要结合具体的情况，比如业务的复杂度、开发人员的接受程度，以及重构的时间周期等等，

选择一个对的架构比选择一个复杂的架构更为重要。

在老项目里选择的是 MVC，对于项目中的绝大部分场景来说，MVC 没有成为制约业务发展的瓶颈，同时考虑到新项目的学习成本以及重构时间周期的问题，所以在新项目上还是选择了 MVC。


## 规范
无规矩不成方圆，确定了架构之后，就是代码风格问题,确定编码规范

在老项目里由于一直没有一个统一的编码规范。而在多人开发时，保持统一的编码规范是很有必要的，这样易于保持代码一致性和 Code Review，当然这个事情做得比较早，7月份整理出来的

## 整体划分
从代码逻辑上，大致分为 5 层

网络层：负责和服务器通讯获取数据。
(RHNetManager等)

数据层：存储用户的数据，包括内存cache。
(RHCacheTool，BankPlist，各种处理数据的工具类)

业务层：包含各种业务逻辑。
(Controller)

UI数据层：负责提供UI层所需要的数据，UI只和这层打交道。
（数据 Model 层）

UI层：包括ViewController和View，处理用户的输入。
(View、xib、storyboard)

分层使项目更清晰，开发时做到各个层独立，高内聚、低耦合

## 重构 - 设计
1.尽可能减少继承层级，涉及苹果原生对象的尽量不要继承
这点也是最近刚总结出来的，在现在的项目中还是用了基类，上次代码审查时也提到了。
但我最近又想了下，还是不要继承的好，要继承也只继承苹果原生对象。

为什么？首先我们继承是为了做什么，通过继承的方案来给原生对象添加功能。

我仔细看了下我的基类，很简单，也没加几个功能。但用了继承就得创建的每个类都继承基类，非常麻烦，一不小心就忘了，尤其是在引入第三方的时候，比如 IMSDK，里面的类都是继承自原生的，就各种 bug。

不用继承怎么做？AOP，面向切面，来实现重载函数，用Category来实现添加函数

2.确定布局规范
除了业务逻辑外，UI 布局算是客户端里的一个重点

老项目是纯 Frame 布局，代码里有一大堆 Magic Number，一大堆屏幕尺寸判断，而这些造成了项目里有过多的布局代码，对于刚上手项目的人来说也不太容易看懂。

所以这次重构整体采用 AutoLayout + Xib/Storyboard 布局，统一布局方式，精简代码。

3.能不放在Controller做的事情就尽量不要放在Controller里面去做
Api 请求抽到网络层，存储数据放到数据层等

## 重构 - 精简
那么具体怎么做呢，
这一部分我认为是本次重构的重头戏，也是本次重构的主要目的所在，精简代码，解决老项目中 各种类过重 的问题。

主要工作分为以下几步：

1.保留最重要的任务，拆分其它不重要的任务
- 所有的网络请求都拆分到 Model 里。
- 针对 TableView 做一层封装，把 DataSource 和 Delegate 从 ViewController 里剥离出来。
- 能不放在 Controller 做的事情就尽量不要放在 Controller 里面去做。

2.拆分后的模块要尽可能提高可复用性，尽量做到DRY
拆出来的部分最好能够归成某一类对象，然后最好能够抽象出一个通用逻辑出来，使他能够复用

3.提高拆分模块后的抽象度 (RHNetApiManager)
拆分的粒度要尽可能大一点，封装得要透明一些，对外只暴露调用方法，在模块内部处理逻辑

这里举一个例子：
对比首页：
[RHNewHomeViewController](http://gitlab.1001.cn/dx_ios/rhredhorse/blob/master/RHRedHorse/Classes/Home/Controller/RHNewHomeViewController.m )    946行

[RHRecommendController](http://gitlab.1001.cn/dx_ios/rhfinance/blob/master/RHFinance/RHFinance/Sections/Recommend/Controllers/RHRecommendController.m)    227行

精简了700+行

## 胖 Model 的问题
Fat model, skinny controller

上面我们精简了半天 Controller ，那么代码都去哪儿了？

除了优化外，代码的总量是确定的，一方的精简必然会造成一方的增加，所以在这把代码添加到 Model 层里去了。

也就是说胖 Model 包含了部分弱业务逻辑。胖 Model 要达到的目的是，Controller从胖 Model 这里拿到数据之后，不用额外做操作或者只要做非常少的操作，就能够将数据直接应用在View上。

所以 Model 在这里是用来帮 ViewController 处理业务逻辑的，胖 Model 也是可以接受的。

瘦 Model 是什么：
是专门用于表达数据，然后存储、数据处理都交给外面的来做。
瘦Model只负责业务数据的表达，所有业务无论强弱一律扔到Controller。
瘦Model要达到的目的是，尽一切可能去编写细粒度Model，然后配套各种helper类或方法来对弱业务做抽象，强业务依旧交给Controller。

## 单元测试
老项目是不写测试代码的，但这是新工程，单元测试也得补上。

当然在 iOS 里单元测试分 UITest 和 UnitTest。

在这主要写 UnitTest，目前已经完成了一部分

单元测试对于目前来说，就是为了方便测试一些功能是否正常运行，还有调试接口是否能正常使用。
有时候你可能是为了测试某一个网络接口，然后每次都重新启动并且经过很多操作之后才测试到了那个网络接口。
如果使用了单元测试，就可以直接测试那个方法，相对方便很多。
比如由于修改较多，想测试一下分享功能是否正常，这时候就有用了。

直接看一些接口返回的数据也会非常直观，不用启动整个工程。

## TODO
URLRoute：跨业务页面调用方案

模块化：

其实还有一个，就是上面提到的分层，在现有的工程中，限于时间关系与需求变更，层的划分还没完全完成。

## 总结
那么总结起来就三点：

Controller 里的代码尽可能的少
Model 的功能尽可能的完整
View 尽可能独立

就能构建一个容易维护，便于协同的项目

## Thanks

以上只是我在这次重构中总结的一些比较重要的点，并不意味着都是最优解。

而我在项目的架构和代码的组织上经验尚浅，

如果有什么错误或是有更好的方法可以一起讨论，谢谢大家！



