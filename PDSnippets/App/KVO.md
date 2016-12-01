# KVO


### 基本概念
KVO (Key-Value Observing) 是Cocoa提供的一种基于KVC的机制，允许一个对象去监听另一个对象的某个属性，当该属性改变时系统会去通知监听的对象。

添加方法：

```
- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)opions context:(nullable void *)context;
```

接收方法：

```
- (void)observeValueForKeyPath:(nullable NSString *)keyPath ofObject:(nullable id)object change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change context:(nullable void *)context;
```

移除方法：

```
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath context:(nullable void *)context
或者
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath;
```

本文相关 [Demo](https://github.com/v2panda/PDPractice/tree/master/Demo_KVO)

### 自动 KVO
调用上面三个方法实现即自动 KVO，不细说。
### 手动 KVO
以 Demo 中的 Boy 类为例，Boy 有 `name` 和 `age` 两个属性，要实现手动发送 KVO 得先禁用 KVO 的自动发送机制，再在需要发送的地方手动发送。

1.在 Boy 类中重写以下方法：

```
/**
 *  重写此方法，设置对该 key 不自动发送通知
 */
+ (BOOL) automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"name"]) {
        return NO;
    }else if ([key isEqualToString:@"age"]) {
        return NO;
    }
    return [super automaticallyNotifiesObserversForKey:key];
}
```

2.手动发送 KVO

```
/**
 *  手动发送通知
 */
- (void)setName:(NSString *)name {
    if (_name != name) {
        [self willChangeValueForKey:@"name"];
        _name = name;
        [self didChangeValueForKey:@"name"];
    }
}
- (void)setAge:(int)age {
    if (_age != age) {
        [self willChangeValueForKey:@"age"];
        _age = age;
        [self didChangeValueForKey:@"age"];
    }
}
```

### KVO 注册依赖键
有一些属性的值取决于一个或者多个其他对象的属性值，一旦某个被依赖的属性值变了，依赖它的属性的变化也需要被通知。
#### To-one 依赖键
以 Demo 中的 Person 类为例，其 `information` 属性同时依赖 `name` 和 `age` 属性：

```
/**
 *  依赖键 information 依赖 name 和 age
 */
- (NSString *)information {
    return [NSString stringWithFormat:@"%@-%d",_name,_age];
}
```
这样的话修改 `name` 和 `age` 都会改变 `information` 的值，此时想实现对 `information` 的 KVO，需要重新确认依赖关系，这里有两种方法。

1.实现`+ (NSSet *)keyPathsForValuesAffecting<Key>:(NSString *)key`


```
+ (NSSet *)keyPathsForValuesAffectingInformation {
    NSSet * keyPaths = [NSSet setWithObjects:@"age", @"name", nil];
    return keyPaths;
}
```

2.重写`+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key`

```
+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key {
    NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];

    if ([key isEqualToString:@"information"]) {
        keyPaths = [keyPaths setByAddingObjectsFromArray:@[@"name", @"age"]];
    }
    return keyPaths;
}
```

#### To-many 依赖键
以 Demo 中的 Person 类为例，其 `totalAges` 属性依赖 `girls` 这个集合属性，`totalAges`的值为`girls`数组里所有`Girl`对象的 `age` 之和。
那么为实现对`totalAges`的 KVO，需要在 Person 类里监听 `girls` 属性，然后每次更新`totalAges`的值：

```
- (instancetype)init {
    self = [super init];
    if (self) {
        [self addObserver:self forKeyPath:@"girls" options:NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld context:totalAgesContext];
    }
    return self;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if (context == totalAgesContext) {
        NSLog(@"totalAgesContext:%@,%@",change[@"new"],change[@"old"]);
        [self updateTotalAges];
    }else {
        // Any unrecognized context must belong to super
        [super observeValueForKeyPath:keyPath
                             ofObject:object
                               change:change
                              context:context];
    }
}

- (void)updateTotalAges {
    NSString *sum = (NSString *)[self valueForKeyPath:@"girls.@sum.age"];
    [self setTotalAges:sum.intValue];
}
```

### KVO 原理
先来看看苹果怎么说

>Key-Value Observing Implementation Details

>Automatic key-value observing is implemented using a technique called **isa-swizzling**.

>The isa pointer, as the name suggests, points to the object's class which maintains a dispatch table. This dispatch table essentially contains pointers to the methods the class implements, among other data.

>When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance.

>You should never rely on the isa pointer to determine class membership. Instead, you should use the class method to determine the class of an object instance.

可见苹果是实现了一种叫 **isa-swizzling** 的机制，那么具体怎么做呢，大概有这么几步：

- 1.在运行期动态地创建被观察类的派生类（类名就是在该类的前面加上NSKVONotifying_ 前缀）

- 2.在这个派生类中重写基类中被观察属性的 setter 方法

- 3.将 isa 指向这个新建的派生类(欺骗外部调用者它就是起初的那个类)

注意这里是在派生类里被重写的 setter 方法里实现真正的通知机制，在对象上对 setter 的调用就会调用重写的 setter，从而激活 KVO。

### 自实现 KVO
根据上面的原理，来自实现一个 KVO 机制，首先创建一个 `NSObject` 的分类:

```
#import <Foundation/Foundation.h>

typedef void(^PDObservingBlock)(id observedObject, NSString *observedKey, id oldValue, id newValue);

@interface NSObject (PDKVO)

- (void)pd_addObserver:(NSObject *)observer
                forKey:(NSString *)key
             withBlock:(PDObservingBlock)block;

- (void)pd_removeObserver:(NSObject *)observer forKey:(NSString *)key;

@end
```

#### ObservationInfo
添加对 block 的支持，block info 如下：
```
@interface PDObservationInfo : NSObject

@property (nonatomic, weak) NSObject *observer;
@property (nonatomic, copy) NSString *key;
@property (nonatomic, copy) PDObservingBlock block;

@end

@implementation PDObservationInfo

- (instancetype)initWithObserver:(NSObject *)observer
                             Key:(NSString *)key
                           block:(PDObservingBlock)block {
    self = [super init];
    if (self) {
        _observer = observer;
        _key = key;
        _block = block;
    }
    return self;
}
@end
```

#### addObserver
在 `addObserver` 方法里首先得检查对象是否存在该属性的setter方法，若没有则抛出异常：
```
SEL setterSelector = NSSelectorFromString(setterForGetter(key));
Method setterMethod = class_getInstanceMethod([self class], setterSelector);
if (!setterMethod) {
    NSString *reason = [NSString stringWithFormat:@"Object %@ does not have a setter for key %@", self, key];
    @throw [NSException exceptionWithName:NSInvalidArgumentException
                                   reason:reason
                                 userInfo:nil];
    return;
}
```

然后检查自身(类)是否是 KVO 类，如果不是，新建一个继承原来类的子类，并把 isa 指向这个新建的子类：

```
Class clazz = object_getClass(self);
NSString *clazzName = NSStringFromClass(clazz);
if (![clazzName hasPrefix:kPDKVOClassPrefix]) {
    clazz = [self createKvoClassWithOriginalClassName:clazzName];
    // 改变 isa 指向刚创建的 clazz 类
    object_setClass(self, clazz);
}
```

再添加重写的 setter 方法，并将 block 信息加到数组中：

```
if (![self hasSelector:setterSelector]) {
    const char *types = method_getTypeEncoding(setterMethod);
    class_addMethod(clazz, setterSelector, (IMP)kvo_setter, types);
}

// 创建观察者的信息
PDObservationInfo *info = [[PDObservationInfo alloc] initWithObserver:observer Key:key block:block];

@synchronized (info) {
    NSMutableArray *observers = objc_getAssociatedObject(self, (__bridge const void *)(kPDKVOAssociatedObservers));
    if (!observers) {
        observers = [NSMutableArray array];
        objc_setAssociatedObject(self, (__bridge const void *)(kPDKVOAssociatedObservers), observers, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    [observers addObject:info];
}
```

需要注意的是这里 `@synchronized()` 传入的是 info 而不是 self，这是因为 synchronized 中传入的 object 的内存地址，被用作 key，通过hash map对应的一个系统维护的递归锁。所以不管是传入什么类型的object，只要是有内存地址，就能启动同步代码块的效果。因此避免传入 self，以免导致死锁，例如：

```
//class A
@synchronized (self) {
    [_sharedLock lock];
    NSLog(@"code in class A");
    [_sharedLock unlock];
}

//class B
[_sharedLock lock];
@synchronized (objectA) {
    NSLog(@"code in class B");
}
[_sharedLock unlock];
```

原因是因为self很可能会被外部对象访问，被用作key来生成一锁，类似上述代码中的@synchronized (objectA)。两个公共锁交替使用的场景就容易出现死锁。

所以正确的做法是传入一个类内部维护的NSObject对象，而且这个对象是对外不可见的。

#### 调用
一句代码就搞定，不用再到 `- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context` 方法里去嵌套 `if else`

```
[self.message pd_addObserver:self forKey:@"info" withBlock:^(id observedObject, NSString *observedKey, id oldValue, id newValue) {
        self.label.text = newValue;
}];
```

具体实现可见 [Demo](https://github.com/v2panda/PDPractice/tree/master/Demo_KVO) 的 `NSObject+PDKVO` 类。

### Reference
[如何自己动手实现 KVO](http://tech.glowing.com/cn/implement-kvo/)
