---
title: 使用 WKWebView 的一些经验
date: 2018-03-02 10:14:10
categories: 
-webview
tags:
- webview
---


> 白屏问题 UIWebView 上当内存占用过大时，App 会 crash；WKWebView 上当内存占用过大时，WebContent process 会 crash，导致白屏。

[](#白屏问题 "白屏问题")白屏问题
--------------------

UIWebView 上当内存占用过大时，App 会 crash；  
WKWebView 上当内存占用过大时，WebContent process 会 crash，导致白屏。  
此时，wkWebView 的 url 变为 nil，reload 操作已无效。

所以，白屏的根本原因是由内存占用过大引发的 App crash 问题，转换成了 WebContent process 的 crash 问题。

### [](#解决方法 "解决方法")解决方法

#### [](#WKNavigationDelegate的回调方法webViewWebContentProcessDidTerminate "WKNavigationDelegate的回调方法webViewWebContentProcessDidTerminate")WKNavigationDelegate 的回调方法 webViewWebContentProcessDidTerminate

在 webViewWebContentProcessDidTerminate 回调中执行 webView 的 reload 操作。  
当 WebContent process 即将白屏时会触发该回调。  
此时，wkWebView 的 url 尚未变成 nil，所以 reload 可以生效。  
而在高内存消耗的界面可能会频繁 reload。

添加一个 web content process 崩溃的标记位  
在崩溃的时候 webViewWebContentProcessDidTerminate 设置该标记位，在 viewWillAppear 中根据该标记位来判断，然后 reload 操作。

#### [](#通过webView-url来判断 "通过webView.url来判断")通过 webView.url 来判断

在 viewWillAppear 的时候对 url 进行判断，若为空，则 reload。

同时使用 KVO 来监听 URL 为空的情况，

```
if ([self.webView.scrollView.superview isKindOfClass:[WKWebView class]]) {
    WKWebView *wkWebView = (WKWebView *)(self.webView.scrollView.superview);
    [wkWebView addObserver:self forKeyPath:@"URL" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if ([keyPath isEqualToString:@"URL"]) {
        NSURL *newUrl = change[@"new"];
        NSURL *oldUrl = change[@"old"];
        if ([newUrl isEqual:[NSNull null]] && ![oldUrl isEqual:[NSNull null]]) {
            [self.webView reload];
        }
    }
}
```

#### [](#通过webView-title来判断 "通过webView.title来判断")通过 webView.title 来判断

在 viewWillAppear 的时候对 title 进行判断，若为空，则 reload。

并非所有白屏的情况都会调用上边的回调方法。  
如在 H5 界面 present 出来一个相机拍照，导致 web content 挂掉，则返回 H5 界面的时候，wkwebview 的 title 会为空。  
若拍照耗了太多内存，导致内存紧张，WebContent process 挂掉，  
但上边的回调方法 webViewWebContentProcessDidTerminate 并未被调用。  
白屏时 webView 的 title 会被置空，所以可以在 viewWillAppear 中检测 webView.title 是否为空来 reload 界面。  
不懂？

#### [](#通过innerHTML来判断 "通过innerHTML来判断")通过 innerHTML 来判断

在 viewWillAppear 的时候对 innerHTML 进行判断，若为空，则 reload。

```
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    __weak typeof(self) weakSelf = self;
    [self.webView evaluateJavaScript:@"document.querySelector('body').innerHTML" completionHandler:^(id result, NSError *error) {
        __strong __typeof(weakSelf) strongSelf = weakSelf;
        if (!result || ([result isKindOfClass:[NSString class]] && [((NSString *)result) length] == 0)) {
            NSLog(@"--- H5页面加载异常，重新加载中 -----");
            // reload your page
            [strongSelf.webView reload];
        } else {
            NSLog(@"--- H5页面加载正常 -----");
        }
    }];
}
```

[](#Cookie问题 "Cookie问题")Cookie 问题
---------------------------------

问题在于：

WKWebView 发起的请求不会自动带上存储于 NSHTTPCookieStorage 中的 cookie。  
而 UIWebView 会自动带上。

Cookie 格式：

```
name=Nicholas;value=test;domain=y.qq.com;expires=Sat, 02 May 2019 23:38:25 GMT;
```

### [](#UIWebView设置cookie "UIWebView设置cookie")UIWebView 设置 cookie

```
var props = Dictionary<HTTPCookiePropertyKey, Any>()
props[HTTPCookiePropertyKey.name] = "tigerobocookie"
props[HTTPCookiePropertyKey.value] = cookieValue // 一串key-value，以;间隔。
props[HTTPCookiePropertyKey.path] = "/"
props[HTTPCookiePropertyKey.domain] = API.tigerResearchWebDomain // 必须设置souyanbao.tigerobo.com

if let cookie = HTTPCookie(properties: props) {
  NSHTTPCookieStorage.shared.setCookie(cookie)
}

xxx
let request = URLRequest(url: url)
webView.loadRequest(request)
```

### [](#WKWebView设置cookie "WKWebView设置cookie")WKWebView 设置 cookie

而 WKWebView 使用 NSHTTPCookieStorage 则不行。

#### [](#使用WKWebView-configuration-websiteDataStore-httpCookieStore "使用WKWebView.configuration.websiteDataStore.httpCookieStore")使用 WKWebView.configuration.websiteDataStore.httpCookieStore

```
guard let cookies = HTTPCookie(properties: props) else { return }
let httpCookieStore = wkWebView.configuration.websiteDataStore.httpCookieStore
httpCookieStore.setCookie(cookie) {
  DispatchQueue.main.async {
    self.wkWebView.load(request)
  }
}
```

但是 httpCookieStore 只在 iOS 11 之后才能用。

#### [](#在loadRequest之前，在request-header中设置cookie，解决首个cookie带不上的问题 "在loadRequest之前，在request header中设置cookie，解决首个cookie带不上的问题")在 loadRequest 之前，在 request header 中设置 cookie，解决首个 cookie 带不上的问题

```
NSURL *url = [NSURL URLWithString:@"http://github.com"];
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
[request addValue:@"skey=skeyValue" forHTTPHeaderField:@"Cookie"];

WKWebView *webView = [WKWebView new];
[webView loadRequest:request];
```

#### [](#通过document-cookie设置cookie解决后续页面（同域）Ajax，iframe请求的cookie问题 "通过document.cookie设置cookie解决后续页面（同域）Ajax，iframe请求的cookie问题")通过 document.cookie 设置 cookie 解决后续页面（同域）Ajax，iframe 请求的 cookie 问题

注意：document.cookie() 无法跨域设置 cookie。

```
WKUserContentController *userContentController = [WKUserContentController new];
WKUserScript *cookieScript = [[WKUserScript alloc] initWithSource:@"document.cookie='skey=skeyValue';" injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
[userContentController addUserScript:cookieScript];
```

无法解决 302 请求的 cookie 问题。  
可以在 webView:decidePolicyForNavigationAction:decisionHandler: 回调方法中拦截 302 请求，copy request，  
在 request header 中带上 cookie 并重新 loadRequest。不过依然解决不了页面 iframe 跨越请求的 cookie 问题。  
因为 loadRequest 只适合加载 mainFrame 请求。

[](#NSURLProtocol可以用于中间人攻击 "NSURLProtocol可以用于中间人攻击")NSURLProtocol 可以用于中间人攻击
---------------------------------------------------------------------------

WKWebView 独立于 app 之外的进程（webContent process）执行网络请求，请求数据不经过主进程。  
所以 WKWebView 中无法直接使用 NSURLProtocol 来拦截请求，

[](#loadRequest问题 "loadRequest问题")loadRequest 问题
------------------------------------------------

WKWebView 通过 loadRequest 发起的 post 请求 body 数据会丢失。  
因为进程间通信性能问题，HTTPBody 字段被丢弃

```
[request setHTTPMethod:@"POST"];
[request setHTTPBody:[@"bodyData" dataUsingEncoding:NSUTF8StringEncoding]];
[wkWebView loadRequest:request];
```

[](#由此想到的 "由此想到的")由此想到的
-----------------------

小程序开发中通过的性能优化关键在于 setData 方法的调用。为啥会这么影响性能呢？

小程序的视图层目前使用 WebView 作为渲染载体，而逻辑层是由独立的 JavascriptCore 作为运行环境。在架构上，WebView 和 JavascriptCore 都是独立的模块，并不具备数据直接共享的通道。

当前，视图层和逻辑层的数据传输，实际上通过两边提供的 evaluateJavascript 所实现。即用户传输的数据 (由视图层去往逻辑层)，需要将其转换为字符串形式传递，同时把转换后的数据内容拼接成一份 JS 脚本，再通过执行 JS 脚本的形式传递到两边独立环境。

而 evaluateJavascript 的执行会受很多方面的影响，数据到达视图层并不是实时的。同一进程内的 WebView 实际上会共享一个 JS VM，如果 WebView 内 JS 线程正在执行渲染或其他逻辑，会影响 evaluateJavascript 脚本的实际执行时间，

另外多个 WebView 也会抢占 JS VM 的执行权限；另外还有 JS 本身的编译执行耗时，都是影响数据传输速度的因素。

常见的 setData 错误操作：

1.  频繁的去执行 setData。 Android 下用户在滑动时会感觉到卡顿，操作反馈延迟严重；渲染有出现延时。
2.  每次 setData 都传递大量新数据。
3.  后台态页面进行 setData。

为何一定要使用 setData 这种方式：

因为 js 语言的限制, 不能检测到对象属性的添加或删除.  
不能检测到单独对 data 做修改的操作.

[](#参考 "参考")参考
--------------

*   [WKWebView 那些坑](https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA)
*   [70% 以上业务由 H5 开发，手机 QQ Hybrid 的架构如何优化演进？](https://mp.weixin.qq.com/s/evzDnTsHrAr2b9jcevwBzA)

坚持原创技术分享，您的支持将鼓励我继续创作！ So，来杯咖啡？
