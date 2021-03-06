# Runtime


## 能做什么
- 1.动态获取类名
- 2.动态获取类的成员变量
- 3.动态获取类的属性列表
- 4.动态获取类的方法列表
- 5.动态获取类所遵循的协议列表
- 6.动态添加新的方法
- 7.类的实例方法实现的交换
- 8.动态属性关联
- 9.**消息发送与消息转发机制等**

## 实例

### 消息处理和消息转发
当你调用一个类的方法时，先在本类中的方法缓存列表中进行查询，如果在缓存列表中找到了该方法的实现，就执行，如果找不到就在本类中的方法列表中进行查找。在本类方法列表中查找到相应的方法实现后就进行调用，如果没找到，就去父类中进行查找。如果在父类中的方法列表中找到了相应方法的实现，那么就执行，否则就执行下方的几步。

当调用一个方法在缓存列表，本类中的方法列表以及父类的方法列表找不到相应的实现时，到程序崩溃阶段中间还会有几步让你来挽救。接下来就来看看这几步该怎么走。

#### 1.消息处理（Resolve Method）

当在相应的类以及父类中找不到类方法实现时会执行`+resolveInstanceMethod:`这个类方法。该方法如果在类中不被重写的话，默认返回NO。如果返回NO就表明不做任何处理，走下一步。如果返回YES的话，就说明在该方法中对这个找不到实现的方法进行了处理。在该方法中，我们可以为找不到实现的SEL动态的添加一个方法实现，添加完毕后，就会执行我们添加的方法实现。这样，当一个类调用不存在的方法时，就不会崩溃了。

#### 2、消息快速转发

如果不对上述消息进行处理的话，也就是`+resolveInstanceMethod:`返回NO时，会走下一步消息转发，即`-forwardingTargetForSelector:`。该方法会返回一个类的对象，这个类的对象有SEL对应的实现，当调用这个找不到的方法时，就会被转发到SecondClass中去进行处理。这也就是所谓的消息转发。当该方法返回self或者nil, 说明不对相应的方法进行转发，那么就该走下一步了。

#### 3.消息常规转发

如果不将消息转发给其他类的对象，那么就只能自己进行处理了。如果上述方法返回self的话，会执行`-methodSignatureForSelector:`方法来获取方法的参数以及返回数据类型，也就是说该方法获取的是方法的签名并返回。如果上述方法返回nil的话，那么消息转发就结束，程序崩溃，报出找不到相应的方法实现的崩溃信息。