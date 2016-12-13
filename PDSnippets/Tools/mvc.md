# 基于 MVC 的项目重构

大家好，今天跟大家分享的主题是 基于 MVC 的项目重构，主要是在这次 iOS 客户端重构改版上的一些研究。当然在架构方面我经验还是非常浅薄的，
这次也借这个机会来讲讲我在项目重构上面的一些实践，请大家随便批评，我多多学习，谢谢！
好的，现在开始！

## 背景
最近公司的项目要更新所有界面的 UI 风格，简单的说就是 UI 部分要重构了。而由于需求不断变化和接手项目人员不断更迭等等历史原因，导致现在的项目十分臃肿，代码中存在过多的重复、过度的耦合、过长的方法和类等等。虽然在迭代时我也对项目做了很多缝缝补补的工作，但还是有很多部分是牵一发而动全身的，简单的说就是顽疾，必须动刀才能治。所以趁这个机会，决定把整个项目重构一遍，本文主要记录重构时的一些选择和解决的问题。

## 架构
首先是架构的选择：还是 MVC，
虽然现在有很多热门的架构思想，如 MVC、MVVM、MVCS、VIPER等等。
对此我的观点是：架构的选择要结合具体的情况，比如业务的复杂度、开发人员的接受程度，以及重构的时间周期等等，选择一个对的架构比选择一个复杂的架构更为重要。
在老项目里选择的是 MVC，对于项目中的绝大部分场景来说，MVC 没有成为制约业务发展的瓶颈，同时考虑到新项目的学习成本以及重构时间周期的问题，所以在新项目上还是选择了 MVC。


## 规范
确定了架构之后，就是代码风格,编码规范

在老项目里由于流程的缺失和开发人员的更迭，一直没有一个统一的编码规范。而在多人开发时，保持统一的编码规范是很有必要的，这样易于保持代码一致性和 Code Review，下面是一些特别需要注意的点：

具体的我们看 gitlab 上

Git Commit message

OC 编码规范

## 分层
主要分为5层，由下至上分为:

网络层：负责和服务器通讯获取数据。
数据层：存储用户的数据，包括内存cache。
业务层：包含各种业务逻辑。
UI数据层：负责提供UI层所需要的数据，UI只和这层打交道。
UI层：包括ViewController和View，处理用户的输入。

## AutoLayout + Xib/Storyboard
老项目是纯 Frame 布局，代码里有一大堆 Magic Number，一大堆屏幕尺寸判断，而这些造成了项目里有过多的布局代码，对于刚上手项目的人来说也不太容易看懂。所以这次重构整体采用 AutoLayout 布局，摒弃老项目里的纯 Frame 布局方式，精简代码。

## 精简 ViewController

这一部分我认为是本次重构的重头戏，也是本次重构的主要目的所在，精简 ViewController 里的代码，解决老项目中 Massive ViewController 的问题。
主要工作分为以下几步：

1.所有的网络请求都拆分到 Model 里。
2.针对 TableView 做一层封装，把 DataSource 和 Delegate 从 ViewController 里剥离出来。
3.能不放在 Controller 做的事情就尽量不要放在 Controller 里面去做。


这里举一个例子：
对比首页：
[RHNewHomeViewController](http://gitlab.1001.cn/dx_ios/rhredhorse/blob/master/RHRedHorse/Classes/Home/Controller/RHNewHomeViewController.m )    946行

[RHRecommendController](http://gitlab.1001.cn/dx_ios/rhfinance/blob/master/RHFinance/RHFinance/Sections/Recommend/Controllers/RHRecommendController.m)    227行

精简了700+行

## 胖 Model 的问题
上文精简 ViewController 把代码都加入到 Model 里去，所以会造成胖 Model 的问题。对此我的理解是，除了优化外，代码的总量是确定的，一方的精简必然会造成一方的增加，而 Model 在这里是用来帮 ViewController 处理业务逻辑的，胖 Model 也是可以接受的。

## TODO
URLRoute

组件化

TestCase

## 最后
本文只是我在这次重构中总结的一些比较重要的点，并不意味着都是最优解。而我在项目的架构和代码的组织上经验尚浅，若本文有什么错误或是有更好的方法请直接指出，欢迎交流讨论。

----
写程序那么久发现一个写出易维护代码的规律，大家看看有没有道理，即：Controller 里的代码尽可能的少，Model 的功能尽可能的完整，View 尽可能独立。就能构建一个容易维护，便于协同的项目
----


