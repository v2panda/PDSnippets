# 渲染优化
[渲染优化](http://weibo.com/ttarticle/p/show?id=2309404058068643105152&sudaref=weibo.com)
[优化](http://ios.jobbole.com/92237/)

iOS 上视图或者动画渲染的各个阶段：

在 APP 内部的有4个阶段：

- 布局：在这个阶段，程序设置 View/Layer 的层级信息，设置 layer 的属性，如 frame，background color 等等。
- 创建 backing image：在这个阶段程序会创建 layer 的 backing image，无论是通过 setContents 将一个 image 传給 layer，还是通过 drawRect：或 drawLayer:inContext：来画出来的。所以 drawRect：等函数是在这个阶段被调用的。
- 准备：在这个阶段，Core Animation 框架准备要渲染的 layer 的各种属性数据，以及要做的动画的参数，准备传递給 render server。同时在这个阶段也会解压要渲染的 image。（除了用 imageNamed：方法从 bundle 加载的 image 会立刻解压之外，其他的比如直接从硬盘读入，或者从网络上下载的 image 不会立刻解压，只有在真正要渲染的时候才会解压）。
- 提交：在这个阶段，Core Animation 打包 layer 的信息以及需要做的动画的参数，通过 IPC（inter-Process Communication）传递給 render server。

在 APP 外部的2个阶段：

当这些数据到达 render server 后，会被反序列化成 render tree。然后 render server 会做下面的两件事：

- 根据 layer 的各种属性（如果是动画的，会计算动画 layer 的属性的中间值），用 OpenGL 准备渲染。
- 渲染这些可视的 layer 到屏幕。

如果做动画的话，最后的两个步骤会一直重复知道动画结束。


## 要做的

设置 view 的 backgroundColor 为一个固定的，不透明的 color。

如果一个 view 是不透明的，设置 opaque 属性为 YES。（直接告诉程序这个是不透明的，而不是让程序去计算）

这样会减少 blending 和 overdraw。

如果使用 image 的话，尽量避免设置 image 的 alpha 为透明的，如果一些效果需要几个图片融合而成，就让设计用一张图画好，不要让程序在运行的时候去动态的融合。