# 关于 UIWebView 在 iOS13 系统上移除问题的调研

#### 送审App，收到邮件警告

```
ITMS-90809: Deprecated API Usage - 
Apple will stop accepting submissions of apps that use UIWebView APIs . 
See https://developer.apple.com/documentation/uikit/uiwebview 
for more information.

UIWebView was removed in iOS 13 so its likely that 
apple will only start rejecting apps for this 
when they also start requiring apps to be build with ios 13. 
That will happen sometime in 2020.
```

查看UIWebView.h的文档说明

```
UIKIT_EXTERN API_DEPRECATED
("No longer supported; please adopt WKWebView."
, ios(2.0, 12.0))
```

经测试 UIWebView 在 iOS13 Beta 上可以正常使用，并且没有问题；

目前来看，不会在iOS 13 Release 上进行移除，会在未来版本进行 UIWebView 类的移除；

官方没有给出具体截止时间，在接下来的版本中将逐渐移除对 UIWebView 的使用，转向 WKWebView；

#### 查看项目，总结

项目中使用 UIWebView 的地方一共有 3 处 ：

	1. YXWebVC 封装了 UIWebView 和 WKWebView ，由标志位控制，可随时替换

	2. YXWebVC 中获取 UserAgent 过程中可能用到，使用 WkWebView 执行相同 JS 代码代替即可

	3. RCTWebView 封装了 UIWebView，目前 react-native 版本为 0.56.1
		0.59 版本中加入了 RCTWKWebView 组件， 
		0.60 版本以后完全移除了 webview 组件，
		可通过升级 rn 版本或者 替换 rn 来解决

#### PS: UIWebView VS WKWebView 的一些优缺点

- UIWebView占用过多内存，且内存峰值更是夸张
- UIWebView不支持后台 js 代码的执行 （iOS 10 之前 Crash 发生较多）
- WKWebView 加载速度更快
- WKWebView 更多的支持HTML5的特性
- WKWebView 官方宣称的高达60fps的滚动刷新率以及内置手势
- WKWebView Safari相同的JavaScript引擎
- WebKit 框架将UIWebViewDelegate与UIWebView拆分成了14类与3个协议 [官方文档](https://developer.apple.com/documentation/webkit?language=objc)
- WKWebView 拥有更直观的 JS 交互

#### 参考链接：

[Apple Developer Forums](https://forums.developer.apple.com/thread/118207)

[github/facebook/react-native](https://github.com/facebook/react-native)