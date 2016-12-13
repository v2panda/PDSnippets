# Block

## weakSelf&strongSelf

```
__weak typeof(self) weakSelf = self;
[self doSomeBlockJob:^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf) {
        ...
    }
}];
```

### 什么时候在 block 里面用 self，不需要使用 weakSelf？
当 block 本身不被 self 持有，而被别的对象持有，同时不产生循环引用的时候，就不需要使用 weak self 了。
最常见的代码就是 UIView 的动画代码，我们在使用 UIView 的 animateWithDuration:animations 方法 做动画的时候，并不需要使用 weak self，因为引用持有关系是：

- UIView 的某个负责动画的对象持有了 block
- block 持有了 self

因为 self 并不持有 block，所以就没有循环引用产生，因为就不需要使用 weak self 了。
当动画结束时，UIView 会结束持有这个 block，如果没有别的对象持有 block 的话，block 对象就会释放掉，从而 block 会释放掉对于 self 的持有。整个内存引用关系被解除。

### 什么时候需要 weakSelf
如果block被一个单例或者其他的常驻内存对象持有，且不知道何时被释放，需要weakSelf的

### 什么时候需要 strongSelf  
在block里使用strongSelf是防止在block执行过程中self被释放，这样很容易出现一些奇怪的逻辑，甚至闪退。

在block里用strong引用，保证了持有引用的周期只在block被执行时，闭包函数返回后就释放了。

### block会产生循环引用，但是业务又需要你不能使用 weak self? 举一个例子并且解释这种情况下如何解决循环引用问题
可以通过在执行完block代码后手动把block置为nil来打破引用循环，AFNetworking就是这样处理的，避免使用者不了解引用循环造成内存泄露。