# Property 关键字

```
@interface ViewController ()
@property (copy)  NSMutableString* myTest;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    NSMutableString* name=[[NSMutableString alloc] initWithString:@"fuck"];
    [self initWithMyTestString:name];
    NSLog(@"%ld",CFGetRetainCount((__bridge CFTypeRef)name)); //打印ARC引用计数
    NSLog(@"%@",self.myTest);
    [name appendString:@"hehe"];
    NSLog(@"%@",self.myTest);
}
-(void)initWithMyTestString:(NSString*) aString
{
//    _myTest=aString;                      //情况1:输出 2 fuck and fuckhehe
//    _myTest=[aString copy];               //情况2:输出 1 fuck and fuck
    self.myTest=aString;                    //情况3:输出 1 fuck and fuck
}
```

情况1:直接使用instance variable进行赋值操作，这样使得myTest Property对aString进行了Strong引用，因此调用了initWithMyTestString后，对name的引用计数为2，myTest直接指向的是name，所以当name改变时，myTest也改变了；**直接引用实例变量速度会更快，因为不需要调用消息，但会忽略property中对内存管理属性的要求！**

情况2和情况3:通过这两种方式，都可以正确的对copy属性的变量进行赋值，得到预期的行为。也可以看出针对不同属性的Property，编译器生成的setter和getter是不同的。

### tips
1.copy会使属性的引用计数加一。

2.不可变对象使用 copy 和 strong 是一样的。

3.可变对象用 strong

4.copy和strong 实际在的差异就在于setter的生成的方式不一样

```
- (instancetype)setStr:(NSMutableString *)str {
  _str = [str copy]; //copy
  _str = str; //strong
  return _str;
}
```

5.copy 会开辟新内存，strong 只是引用计数+1，还是指向同一块内存

