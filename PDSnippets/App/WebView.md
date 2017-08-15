# WebView 相关




## 移动 H5 首屏打开时间优化
- 1.当App首次打开时，默认是并不初始化浏览器内核的；只有当创建WebView实例的时候，才会创建WebView的基础框架。
- 2.所以与浏览器不同，App中打开WebView的第一步并不是建立连接，而是启动浏览器内核。
- 3.预先初始化一个 webview 然后释放，启动浏览器内核
    
```
UIWebView *tempWebView = [[UIWebView alloc]init];
tempWebView = nil;
```


## 相关文章
[移动 H5 首屏秒开优化方案探讨](http://blog.cnbang.net/tech/3477/)

[WebView性能、体验分析与优化](https://tech.meituan.com/WebViewPerf.html)

[STMURLCache](https://github.com/ming1016/STMURLCache)