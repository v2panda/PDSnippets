# KVO


### 基本概念
KVO (Key-Value Observing) 是Cocoa提供的一种基于KVC的机制，允许一个对象去监听另一个对象的某个属性，当该属性改变时系统会去通知监听的对象。

添加方法：

```
- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context;
```

接受方法：

```
- (void)observeValueForKeyPath:(nullable NSString *)keyPath ofObject:(nullable id)object change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change context:(nullable void *)context;
```

移除方法：

```
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath context:(nullable void *)context
或者
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath;
```

本文相关 [Demo]()

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

可见苹果是实现了一种叫**isa-swizzling**的机制。

其实，当某个类的对象第一次被观察时，系统就会在运行期动态地创建该类的一个派生类（类名就是在该类的前面加上NSKVONotifying_ 前缀），在这个派生类中重写基类中任何被观察属性的 setter 方法。

派生类在被重写的 setter 方法实现真正的通知机制，就如前面手动实现键值观察那样，调用willChangeValueForKey:和didChangeValueForKey:方法。这么做是基于设置属性会调用 setter 方法，而通过重写就获得了 KVO 需要的通知机制。当然前提是要通过遵循 KVO 的属性设置方式来变更属性值，如果仅是直接修改属性对应的成员变量，是无法实现 KVO 的。

同时派生类还重写了 class 方法以“欺骗”外部调用者它就是起初的那个类。然后系统将这个对象的 isa 指针指向这个新诞生的派生类，因此这个对象就成为该派生类的对象了，因而在该对象上对 setter 的调用就会调用重写的 setter，从而激活键值通知机制。此外，派生类还重写了 dealloc 方法来释放资源。

### 自实现 KVO
