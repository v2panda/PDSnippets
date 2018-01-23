# React Native 与原生交互

##  iOS —> React Native
1. 初始化 view generator
	1. initWithBundleURL
	2. appProperties (在appProperties设置之后，React Native应用将会根据新的属性重新渲染。当然，只有在新属性和之前的属性有区别时更新才会被触发。)
	
    ``` 
    // 在 RN 中 更新 props
    componentWillReceiveProps(nextProps) {
        this.iseCallback(nextProps);
    }	
    ```

2. RCTEventEmitter
	> 类似于iOS中的通知NSNotificationCenter

	1. iOS 端 RCTEventEmitter —> sendEventWithName
	2. RN 端 NativeEventEmitter —> addListener

 ```
const bridgeModuleEmitter = new NativeEventEmitter(bridgeModule);  //创建自定义事件接口 

    iseCallback(data) {//接受原生传过来的数据    
        console.log("iseCallback" + ' ' + data)       
    }  

    componentWillMount() {  
        this.listener = bridgeModuleEmitter.addListener('sendActivitiesToJS', this.iseCallback.bind(this));  //对应了原生的名字  
    }

    componentWillUnmount() {  
        this.listener && this.listener.remove();  //记得remove
        this.listener = null;  
    }  
```

3. 导出常量

 ` — (NSDictionary<NSString *, id> *)constantsToExport;`


## React Native —> iOS 

> Callback ，JS调用一次，Native返回一次	| CallBack为异步操作，返回时机不确定

1. iOS 端通过宏(RCT_EXPORT_METHOD)配置方法供 RN 端调用, RN 端通过 NativeModules 调用 iOS 端方法
(可以添加 callback 回调 RCTResponseSenderBlock)

	> Promise 方式 优点 | 缺点 ---|--- js调用一次，Native返回一次|每次使用需要js调用一次

2. iOS 端通过宏(RCT_REMAP_METHOD) 
    
```
###Native 端
RCT_REMAP_METHOD(testPromisesEvent,
             resolver:(RCTPromiseResolveBlock)resolve
             rejecter:(RCTPromiseRejectBlock)reject)
{
  NSString *PromisesData = @"Promises数据"; //准备回调回去的数据
  if (PromisesData) {
    resolve(PromisesData);
  } else {
    NSError *error=[NSError errorWithDomain:@"我是Promise回调错误信息..." code:101 userInfo:nil];
    reject(@"no_events", @"There were no events", error);
  }
}
###React Native 端
<Text style={styles.welcome} onPress={()=>this.promisesEvent()}>Promises</Text>

//Promise回调
async promisesEvent(){
    MyModule.testPromisesEvent().then((events)=>{
        alert(events+1111)
    }).catch((e)=>{
        console.error(e);
    })
}
```




