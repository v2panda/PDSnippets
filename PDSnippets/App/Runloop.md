# Runloop


## 概念

### 基本概念
一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，让线程能随时处理事件但并不退出，这就是事件循环(event loop)机制。

- Foundation－> NSRunLoop
- Core Foundation -> CFRunLoopRef

NSRunLoop 和 CFRunLoopRef 都代表着 RunLoop 对象，NSRunLoop 是基于 CFRunLoopRef 的一层 OC 包装。

### 相关类和 Mode
Core Foundation 中相关 RunLoop 的 5 个类：

- CFRunLoopRef
- CFRunLoopModeRef
- CFRunLoopSourceRef
- CFRunLoopTimerRef
- CFRunLoopObserverRef

5 种 Mode:

- KCFRunLoopDefaultMode 默认 Mode，主线程在这个 Mode 下运行
- UITrackingRunLoopMode 用于 ScrollView 追踪触摸滑动，保证滑动时不受其他 Mode 影响
- UIInitializationRunLoopMode，在刚启动 App 时进入的第一个 Mode，启动完成后就不再使用
- GSEventReceiveRunLoopMode，接受系统事件的内部 Mode，通常用不到
- KCFRunLoopCommonModes，这是一个占位用的 Mode

在这之间还有一个 `CommonModes` 的概念，例如：主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为"Common"属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。

有时你需要一个 Timer，在两个 Mode 中都能得到回调，一种办法就是将这个 Timer 分别加入这两个 Mode。还有一种方式，就是将 Timer 加入到顶层的 RunLoop 的 "commonModeItems" 中。"commonModeItems" 被 RunLoop 自动更新到所有具有"Common"属性的 Mode 里去。


## 应用

#### 1.NSTimer
使用默认方法创建 NSTimer 时，是默认加在 NSDefaultRunLoopMode 下的。若此时滑动 ScrollView 则 Timer 会失效，因为此时从 NSDefaultRunLoopMode 切换到了 UITrackingRunLoopMode。
若停止滑动，Timer 又开始工作，因为又从 UITrackingRunLoopMode 切换到了 NSDefaultRunLoopMode。

解决这个问题只需要在开启 NSTimer 的时候设置好 RunLoop 的 Mode 设置为 NSRunLoopCommonModes 即可。

```
NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:1.0f target:self selector:@selector(testTimer) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];

```

#### 2.显示大图
有一个 tableView，每个 cell 都显示了三张图，一屏大概能显示18张图，每张图的大小是2034 × 1525 pixels，如果在一次 RunLoop 的时候绘制 18 张图到 cell 上面会其卡无比。

这个时候监听 runloop CFRunLoopObserver 的 kCFRunLoopBeforeWaiting 状态，在这个状态下添加方法绘制图片（由原来的一次 runloop 绘制全屏的图片变为一次 runloop 绘制一张图片），这样就会有tableViewCell上图片逐个显示。

详细的例子见[RunLoopWorkDistribution](https://github.com/diwu/RunLoopWorkDistribution)。

#### 3.监测卡顿
核心还是使用 CFRunLoopObserverRef 来监听两次 RunLoop 之间的时差，如果在一个新的 RunLoop 开启的时候，这两个之间的时差超过你设置的某个值就表示有卡顿了。然后将这次 RunLoop 执行的所有方法打印出来，差不多就可以定位在某个函数执行的时候发生了卡顿。

详细的例子见[PerformanceMonitor](https://github.com/suifengqjn/PerformanceMonitor)。

#### 4.一个tableview延迟加载图片的新思路
 
前提 ：每次cell在setImage时可能会比较'卡' （网络加载图片）

旧思路：tableView写一delegate，判断每次滑动时不set，停下来再set。

新思路：

```
UIImage *dowmLoadedImage = ...;
self.avatarImageView performSelector:@selector(setImage:)
								withObject:dowmLoadedImage
								afterDelay:0
								    inModes:@[NSDefaultRunLoopMode]];
```
	
## Reference
[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)
	