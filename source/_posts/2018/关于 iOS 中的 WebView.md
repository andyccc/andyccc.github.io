---
title: 关于 iOS 中的 WebView
date: 2018-03-10 11:14:10
categories: 
-webview
tags:
- webview
---

> UIWebView 使用方式与普通 View 一样。

使用方式与普通 View 一样。内置一个 UIScrollView。

需要设置 UIWebViewDelegate。delegate 只有四个回调方法：是否开始，load 开始，load 完成，load 失败。

[](#HTML "HTML")HTML
--------------------

```
[webView loadHTMLString:@"<html><body><p>this is baidu!</p></body></html>" baseURL:];
```

[](#HTML-File "HTML File")HTML File
-----------------------------------

```
NSString *str = [[NSBundle mainBundle] pathForResource:@"index" ofType:@"html"];
NSURL *url = [NSURL fileURLWithPath:str];
[webView loadFileURL:url allowingReadAccessToURL:url];
```

[](#NSData "NSData")NSData
--------------------------

```
NSString *path = [[NSBundle mainBundle] pathForResource:@"demo" ofType:@"gif"];
NSData *data = [NSData dataWithContentsOfFile:path];
[webView loadData:data MIMEType:@"image/gif" textEncodingName:@"UTF-8" baseURL:url];
webView.userInteractionEnabled = NO;
```

设置 MIMEType 为指定类型

```
@"image/jpg"
@"image/gif"
@"application/pdf"
```

[](#URLRequest "URLRequest")URLRequest
--------------------------------------

```
NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:urlString]];
[webView loadRequest:request];
```

[](#UIWebViewDelegate "UIWebViewDelegate")UIWebViewDelegate
-----------------------------------------------------------

```
// 是否运行load
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
    
    return YES;
}

// 开始load
- (void)webViewDidStartLoad:(UIWebView *)webView {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

// load完成
- (void)webViewDidFinishLoad:(UIWebView *)webView {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

// load失败
- (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}
```

[](#关键词 "关键词")关键词
-----------------

1.  属于 App 占用内存，内存消耗过大可能导致 app 崩溃。
2.  不如 WKWebView 灵活，提供的 delegate 回调方法很少。

加载 HTML，NSData 与 UIWebView 一致。

独立的进程，network loading 和 ui rendering 多进程同时允许，类似 Chrome。所以 App 的内存消耗大幅下降，而其他进程的内存消耗增加。

内存消耗过大，UIWebView 会导致 app 被杀死，而 WKWebView 的 web content process 被杀死则会出现白屏。

有 WKNavigationDelegate 和 WKUIDelegate 两个。

alert 弹窗：实现 WKUIDelegate 协议的 runJavaScriptAlertPanelWithMessage 方法。

confirm 弹窗：实现 WKUIDelegate 协议的 runJavaScriptConfirmPanelWithMessage 方法。

[](#WKWebViewConfiguration "WKWebViewConfiguration")WKWebViewConfiguration
--------------------------------------------------------------------------

使用 WKWebView 的默认初始化方法：

```
- (instancetype)initWithFrame:(CGRect)frame configuration:(WKWebViewConfiguration *)configuration;
```

会将 configuration 对象复制一份，之后再去修改 configuration 则不生效。

[](#WKNavigationDelegate "WKNavigationDelegate")WKNavigationDelegate
--------------------------------------------------------------------

自定义 webView 接受、加载、完成请求等过程的行为。

*   WKNavigation 对象包含了跟踪页面加载过程的信息。
*   WKNavigationAction 对象包含了可能导致一次加载的操作的信息，用于制定策略决策。
*   WKNavigationResponse 对象包含用于制定策略决策的浏览响应信息

关键的，与 UIWebView 不同的是：

```
/*! A class conforming to the WKNavigationDelegate protocol can provide
 methods for tracking progress for main frame navigations and for deciding
 policy for main frame and subframe navigations.
 */

// MARK: 拦截URL

/*! @abstract Decides whether to allow or cancel a navigation.
 @param webView The web view invoking the delegate method.
 @param navigationAction Descriptive information about the action
 triggering the navigation request.
 @param decisionHandler The decision handler to call to allow or cancel the
 navigation. The argument is one of the constants of the enumerated type WKNavigationActionPolicy.
 @discussion If you do not implement this method, the web view will load the request or, if appropriate, forward it to another application.
 */
// 接收到响应后, 如何处理(是否跳转, 是否拦截URL等)
// 调用decisionHandler(WKNavigationActionPolicyAllow);则允许跳转
// 调用decisionHandler(WKNavigationActionPolicyCancel);即不允许跳转
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
    // 该方法实现了, 则必须调用decisionHandler.
    
    NSURL *url = navigationAction.request.URL;
    
    if ([url.absoluteString isEqualToString:@"about:blank"]) {
        decisionHandler(WKNavigationActionPolicyAllow);
        return;
    }
    
    NSString *scheme = [url scheme];
    
    NSURL *docUrl = navigationAction.request.mainDocumentURL;
    NSURL *urlOfBundleFilePrefix = [docUrl URLByDeletingLastPathComponent];
    NSString *route = [url.absoluteString stringByReplacingOccurrencesOfString:urlOfBundleFilePrefix.absoluteString withString:@""];
    
    
    if ([route containsString:@"actionOC_JS://"]) {
        [self routeJS_OC_Action:route];
        
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    
    NSString *host = [url host];
    
    NSLog(@"comes from %@ %@", scheme, host);
    
    if ([host containsString:@"baidu"]) {
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    
    decisionHandler(WKNavigationActionPolicyAllow);
}

/*! @abstract Decides whether to allow or cancel a navigation after its
 response is known.
 @param webView The web view invoking the delegate method.
 @param navigationResponse Descriptive information about the navigation
 response.
 @param decisionHandler The decision handler to call to allow or cancel the
 navigation. The argument is one of the constants of the enumerated type WKNavigationResponsePolicy.
 @discussion If you do not implement this method, the web view will allow the response, if the web view can show it.
 */
//- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler {
//    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
//}

/*! @abstract Invoked when a main frame navigation starts.
 @param webView The web view invoking the delegate method.
 @param navigation The navigation.
 */
// 页面开始加载
- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(null_unspecified WKNavigation *)navigation {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

/*! @abstract Invoked when a server redirect is received for the main
 frame.
 @param webView The web view invoking the delegate method.
 @param navigation The navigation.
 */
// 接收到服务器的重定向
- (void)webView:(WKWebView *)webView didReceiveServerRedirectForProvisionalNavigation:(null_unspecified WKNavigation *)navigation {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

/*! @abstract Invoked when an error occurs while starting to load data for
 the main frame.
 @param webView The web view invoking the delegate method.
 @param navigation The navigation.
 @param error The error that occurred.
 */
// 页面加载失败
- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
    NSLog(@"error: %@", error);
}

/*! @abstract Invoked when content starts arriving for the main frame.
 @param webView The web view invoking the delegate method.
 @param navigation The navigation.
 */
// 跳转已经完成, 页面内容开始到达并展示
- (void)webView:(WKWebView *)webView didCommitNavigation:(null_unspecified WKNavigation *)navigation {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

/*! @abstract Invoked when a main frame navigation completes.
 @param webView The web view invoking the delegate method.
 @param navigation The navigation.
 */
// 页面跳转完成
- (void)webView:(WKWebView *)webView didFinishNavigation:(null_unspecified WKNavigation *)navigation {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

/*! @abstract Invoked when an error occurs during a committed main frame
 navigation.
 @param webView The web view invoking the delegate method.
 @param navigation The navigation.
 @param error The error that occurred.
 */
// 页面跳转失败
- (void)webView:(WKWebView *)webView didFailNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

/*! @abstract Invoked when the web view needs to respond to an authentication challenge.
 @param webView The web view that received the authentication challenge.
 @param challenge The authentication challenge.
 @param completionHandler The completion handler you must invoke to respond to the challenge. The
 disposition argument is one of the constants of the enumerated type
 NSURLSessionAuthChallengeDisposition. When disposition is NSURLSessionAuthChallengeUseCredential,
 the credential argument is the credential to use, or nil to indicate continuing without a
 credential.
 @discussion If you do not implement this method, the web view will respond to the authentication challenge with the NSURLSessionAuthChallengeRejectProtectionSpace disposition.
 */
//- (void)webView:(WKWebView *)webView didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * _Nullable credential))completionHandler {
//    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
//}

/*
 这个方法比较关键，web content process因内存消耗过大被杀死，则此回调会触发。在其中可以对webView进行reload操作。
*/
/*! @abstract Invoked when the web view's web content process is terminated.
 @param webView The web view whose underlying web content process was terminated.
 */
- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}
```

其中：

```
/**
 在decidePolicyForNavigationAction方法中,
 根据navigationAction的一些参数, 解析出对应的action和参数, 执行即可.
 
 对于这些action:
 若调用iOS系统的一些接口, 则over.
 若准备一些参数, 再传递给JS中的函数, 则依然需要调用evaluateJavaScript方法.
 */
- (void)routeJS_OC_Action:(NSString *)route {
    NSString *action = [route stringByReplacingOccurrencesOfString:@"actionOC_JS://" withString:@""];
    if ([action containsString:@"setColor"]) {
        NSString *paramStr = [action stringByReplacingOccurrencesOfString:@"setColor?" withString:@""];
        NSArray *params = [paramStr componentsSeparatedByString:@"&"];
        
        NSMutableDictionary *colorDic = [NSMutableDictionary dictionary];
        for (NSString *colorStr in params) {
            NSArray *key_value = [colorStr componentsSeparatedByString:@"="];
            [colorDic setObject:key_value[1] forKey:key_value[0]];
        }
        CGFloat r = [[colorDic objectForKey:@"r"] floatValue];
        CGFloat g = [[colorDic objectForKey:@"g"] floatValue];
        CGFloat b = [[colorDic objectForKey:@"b"] floatValue];
        CGFloat a = [[colorDic objectForKey:@"a"] floatValue];

        UIColor *color = [UIColor colorWithRed:r/255.0 green:g/255.0 blue:b/255.0 alpha:a];
        [self p_actionSetColor:color];
    } else if ([action containsString:@"getLocation"]) {
        [self p_actionGetLocation];
    } else if ([action containsString:@"share"]) {
        [self p_actionShare];
    } else if ([action containsString:@"scan"]) {
        
    } else if ([action containsString:@"shake"]) {
        
    } else if ([action containsString:@"pay"]) {
    
    }
}
```

[](#WKUIDelegate "WKUIDelegate")WKUIDelegate
--------------------------------------------

提供为网页展示 native 用户界面的方法。WebView 用户界面通过实现这个协议来控制新窗口的打开，增强用户单击元素时显示的默认菜单项的表现，并执行其他用户界面相关的任务。这些方法可以通过处理 JavaScript 或其他插件内容来调用。默认每个 WebView 一个窗口，如果需要实现一个非常规用户界面，需要依靠 WKUIDelegate 来实现。

关键的，与 UIWebView 不同的是：

```
/*! A class conforming to the WKUIDelegate protocol provides methods for
 presenting native UI on behalf of a webpage.
 */

/*! @abstract Creates a new web view.
 @param webView The web view invoking the delegate method.
 @param configuration The configuration to use when creating the new web
 view.
 @param navigationAction The navigation action causing the new web view to
 be created.
 @param windowFeatures Window features requested by the webpage.
 @result A new web view or nil.
 @discussion The web view returned must be created with the specified configuration. WebKit will load the request in the returned web view.
 
 If you do not implement this method, the web view will cancel the navigation.
 */
//- (nullable WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures;

/*! @abstract Notifies your app that the DOM window object's close() method completed successfully.
 @param webView The web view invoking the delegate method.
 @discussion Your app should remove the web view from the view hierarchy and update
 the UI as needed, such as by closing the containing browser tab or window.
 */
- (void)webViewDidClose:(WKWebView *)webView {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

/*! @abstract Displays a JavaScript alert panel.
 @param webView The web view invoking the delegate method.
 @param message The message to display.
 @param frame Information about the frame whose JavaScript initiated this
 call.
 @param completionHandler The completion handler to call after the alert
 panel has been dismissed.
 @discussion For user security, your app should call attention to the fact
 that a specific website controls the content in this panel. A simple forumla
 for identifying the controlling website is frame.request.URL.host.
 The panel should have a single OK button.
 
 If you do not implement this method, the web view will behave as if the user selected the OK button.
 */
// 弹出警告框
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
    
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Alert" message:message preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
        completionHandler();
    }]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

/*! @abstract Displays a JavaScript confirm panel.
 @param webView The web view invoking the delegate method.
 @param message The message to display.
 @param frame Information about the frame whose JavaScript initiated this call.
 @param completionHandler The completion handler to call after the confirm
 panel has been dismissed. Pass YES if the user chose OK, NO if the user
 chose Cancel.
 @discussion For user security, your app should call attention to the fact
 that a specific website controls the content in this panel. A simple forumla
 for identifying the controlling website is frame.request.URL.host.
 The panel should have two buttons, such as OK and Cancel.
 
 If you do not implement this method, the web view will behave as if the user selected the Cancel button.
 */
// 弹出确认框
- (void)webView:(WKWebView *)webView runJavaScriptConfirmPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(BOOL result))completionHandler {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
    
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"Confirm" message:message preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(YES);
    }]];
    
    [self presentViewController:alert animated:YES completion:nil];
}

/*! @abstract Displays a JavaScript text input panel.
 @param webView The web view invoking the delegate method.
 @param message The message to display.
 @param defaultText The initial text to display in the text entry field.
 @param frame Information about the frame whose JavaScript initiated this call.
 @param completionHandler The completion handler to call after the text
 input panel has been dismissed. Pass the entered text if the user chose
 OK, otherwise nil.
 @discussion For user security, your app should call attention to the fact
 that a specific website controls the content in this panel. A simple forumla
 for identifying the controlling website is frame.request.URL.host.
 The panel should have two buttons, such as OK and Cancel, and a field in
 which to enter text.
 
 If you do not implement this method, the web view will behave as if the user selected the Cancel button.
 */
// 弹出输入框
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(nullable NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * _Nullable result))completionHandler {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

#if TARGET_OS_IPHONE

/*! @abstract Allows your app to determine whether or not the given element should show a preview.
 @param webView The web view invoking the delegate method.
 @param elementInfo The elementInfo for the element the user has started touching.
 @discussion To disable previews entirely for the given element, return NO. Returning NO will prevent
 webView:previewingViewControllerForElement:defaultActions: and webView:commitPreviewingViewController:
 from being invoked.
 
 This method will only be invoked for elements that have default preview in WebKit, which is
 limited to links. In the future, it could be invoked for additional elements.
 */
// 是否预览
- (BOOL)webView:(WKWebView *)webView shouldPreviewElement:(WKPreviewElementInfo *)elementInfo {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
    return YES;
}

/*! @abstract Allows your app to provide a custom view controller to show when the given element is peeked.
 @param webView The web view invoking the delegate method.
 @param elementInfo The elementInfo for the element the user is peeking.
 @param defaultActions An array of the actions that WebKit would use as previewActionItems for this element by
 default. These actions would be used if allowsLinkPreview is YES but these delegate methods have not been
 implemented, or if this delegate method returns nil.
 @discussion Returning a view controller will result in that view controller being displayed as a peek preview.
 To use the defaultActions, your app is responsible for returning whichever of those actions it wants in your
 view controller's implementation of -previewActionItems.
 
 Returning nil will result in WebKit's default preview behavior. webView:commitPreviewingViewController: will only be invoked
 if a non-nil view controller was returned.
 */
//- (nullable UIViewController *)webView:(WKWebView *)webView previewingViewControllerForElement:(WKPreviewElementInfo *)elementInfo defaultActions:(NSArray<id <WKPreviewActionItem>> *)previewActions API_AVAILABLE(ios(10.0))

/*! @abstract Allows your app to pop to the view controller it created.
 @param webView The web view invoking the delegate method.
 @param previewingViewController The view controller that is being popped.
 */
- (void)webView:(WKWebView *)webView commitPreviewingViewController:(UIViewController *)previewingViewController {
    NSLog(@">>>>>>>>>> %s", __FUNCTION__);
}

#endif // TARGET_OS_IPHONE

#if !TARGET_OS_IPHONE

/*! @abstract Displays a file upload panel.
 @param webView The web view invoking the delegate method.
 @param parameters Parameters describing the file upload control.
 @param frame Information about the frame whose file upload control initiated this call.
 @param completionHandler The completion handler to call after open panel has been dismissed. Pass the selected URLs if the user chose OK, otherwise nil.
 
 If you do not implement this method, the web view will behave as if the user selected the Cancel button.
 */
//- (void)webView:(WKWebView *)webView runOpenPanelWithParameters:(WKOpenPanelParameters *)parameters initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSArray<NSURL *> * _Nullable URLs))completionHandler API_AVAILABLE(macosx(10.12));

#endif
```

[](#关键词-1 "关键词")关键词
-------------------

1.  相比 UIWebView 的控制粒度更细。在服务端请求响应之后，会询问是否载入内容到当前 WKWebView。而 UIWebView 是直接载入。URL 的 content 下载完成，会询问是否允许下载的内容载入。
2.  多了一个重定向的通知，WKWebView 进程退出时有个 terminate 的回调方法。
3.  不占用 App 自身内存。但 web content process 的内存带来了白屏等问题。
4.  alert、conform 等视图需要通过 WKUIDelegate 协议来接收通知，iOS 原生执行。
5.  Native->JS 通信，WKWebView 是异步的，而 UIWebView 是同步的。
6.  cookie 机制比较复杂。

支持 KVO 的一些属性：

1.  title
2.  URL
3.  estimatedProgress
4.  hasOnlySecureContent，页面中的所有资源是否都是通过安全链接加载的？
5.  loading

其中，title、URL、loading 通常可以用来检测是否加载成功，例如是否出现 webView 白屏等。

iOS 模拟器进入对应的 WebView 界面，打开 Safari，打开开发菜单即可。注意，调试时候，App 不需要处于 debugging 中。

iOS 真机调试 WebView，则要先 **_设置 ->Safari 浏览器 -> 高级 -> 打开 JavaScript 和 Web 检查器_** , 同时使用 Debug 证书。

管理与特定的 WKWebsiteDataStore 关联的 HTTP cookie 的对象:

```
- (void)getAllCookies:(void (^)(NSArray<NSHTTPCookie *> *))completionHandler;
- (void)setCookie:(NSHTTPCookie *)cookie completionHandler:(void (^)(void))completionHandler;
- (void)deleteCookie:(NSHTTPCookie *)cookie completionHandler:(void (^)(void))completionHandler;
- (void)addObserver:(id<WKHTTPCookieStoreObserver>)observer;
- (void)removeObserver:(id<WKHTTPCookieStoreObserver>)observer;
- (void)cookiesDidChangeInCookieStore:(WKHTTPCookieStore *)cookieStore;
```

注意，cookie 可以用来监控。

JSCore 跟 Google 的 V8 引擎类似，都是用于解释执行 JavaScript 代码的核心引擎。

JSVirtualMachine，JSContext，JSValue，JSExport，evaluateJavaScript。

以下都是一对多：

```
JS VM ---> JSContext ---> JSValue
```

[](#JSVirtualMachine "JSVirtualMachine")JSVirtualMachine
--------------------------------------------------------

为 JS 代码的运行提供一个虚拟机环境。一个 JS VM 只能执行一个线程，若要多线程，则需要创建多个 JS VM。跟 JS 的单线程模型是什么关系？

每个 JS VM 都有自己的 GC，多个 JS VM 之间的对象无法传递。

[](#JSContext "JSContext")JSContext
-----------------------------------

是 JS 运行环境的上下文，负责 Native 和 JS 的数据传递

[](#JSValue "JSValue")JSValue
-----------------------------

JS 的值对象，提供 JS 的原始值对象与 Native 对象的转换方式。

[](#Native-gt-JS "Native->JS")Native->JS
----------------------------------------

```
[webView evaluateJavaScript:jsString]
```

或者

```
[jsContext evaluateScript:jsString];
```

使用 toString，toNumber，toDictionary 从 JSValue 中取出对应的 Native 对象。

若要使用 JS 的函数对象，则使用 callWithArguments 方法:

```
[context evaluateString:@"function addition(x,y) {return x+ y}"];
JSValue *addition = context[@"addition"];
JSValue *resultValue = [addition callWithArguments:@[@(4), @(8)]];
[resultValue toNumber];
```

Weex 中类似：

```
- (JSValue *)callJSMethod:(NSString *)method args:(NSArray *)args {
    return [[_jsContext globalObject] invokeMethod:method withArguments:args];
}
```

[](#JS-gt-Native "JS->Native")JS->Native
----------------------------------------

使用 block：

```
context[@"subtraction"] = ^(int x, int y) {
    return x + y;
}
JSValue *subValue = [context evaluateScript:@"subtraction(4,8);"];
[subValue toNumber];
```

JS 调用 Native 代码，是通过 block 或 closure 来实现的。

[](#JSExport "JSExport")JSExport
--------------------------------

JSExport 协议，可以将 Native 对象 export 到 JSContext 中。

自定义一个 protocol 继承 JSExport 协议，将 OC 对象传入 JS 中。

```
@protocol PersonJSExport <JSExport>

@end

jsContext[@"Person"] = [Person class];
[jsContext evaluateScript:@"var personWithName = function(name) { var person = Person.persionWithName(name); return person; }"];
[[jsContext objectForKeyedSubscript:@"persionWithName"] callWithArguments:@[@"Chris"]];
```

这里，将 OC 的类 Person 继承 JSExport 协议，然后封装一个函数 personWithName 来调用该 OC 类的初始化代码。  
通过 objectForKeyedSubscript 来取出 JS 中的函数，使用 callWithArguments: 来执行。

[](#JSPatch原理 "JSPatch原理")JSPatch 原理
------------------------------------

除了 block 和 JSExport 之外，还可以使用 JSContext 和 JSValue 的类型转换和 OC 的 runtime 消息转发机制实现动态调用，巧妙避开 JSExport 协议，即 JSPatch 的实现原理。

[](#JavaScriptCore的组成 "JavaScriptCore的组成")JavaScriptCore 的组成
------------------------------------------------------------

真实的 JavaScript 虚拟机，包含了解释器和运行时。

解释器将高级的脚本语言编译成字节码，runtime 用来管理 runtime 的内存空间。

JSCore 内部由 Parse，Interpreter，Compiler，GC 组成。

[](#解释执行 "解释执行")解释执行
--------------------

由 Parser 进行词法分析，语法分析，生成字节码。

Interpreter 进行解释执行。先有 LLInt（Low Level Interpreter）来执行 Parser 生成的字节码，JSCore 会对运行频次高的函数或循环进行优化。

优化器有 Baseline JIT，DFG JIT，FTL JIT 等。

[](#执行js代码 "执行js代码")执行 js 代码
----------------------------

```
- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^)(id, NSError *error))completionHandler;
```

注意，其 completionHandler 是在主线程执行的。

[](#WKProcessPool "WKProcessPool")WKProcessPool
-----------------------------------------------

Web Content 的进程池。WKWebView 对象初始化的时候，会从该进程池中获取一个 Web Content 进程对象。

[](#截图 "截图")截图
--------------

```
- (void)takeSnapshotWithConfiguration:(WKSnapshotConfiguration *)snapshotConfiguration completionHandler:(void (^)(UIImage *snapshotImage, NSError *error))completionHandler;
```

可使用 WKSnapshotConfiguration 来自定义 webView 截图区域。

[](#WKUserContentController "WKUserContentController")WKUserContentController
-----------------------------------------------------------------------------

提供了向 WKWebView 发送 JS 消息或者注入 JS 脚本的方法。

一个 WKUserScript 对象代表了一个可以被注入网页中的脚本。脚本注入的时机，只能通过 WKUserScriptInjectionTime 枚举来设置。其包含 WKUserScriptInjectionTimeAtDocumentStart 和 WKUserScriptInjectionTimeAtDocumentEnd。

[](#WKScriptMessageHandler "WKScriptMessageHandler")WKScriptMessageHandler
--------------------------------------------------------------------------

实现该协议的类，可以接收来自网页的 JS 调用方法。

WKScriptMessage 对象即包含了一个 JS 消息的相关信息。其属性 id body 只能是对象类型。

[](#WKWebsiteDataStore "WKWebsiteDataStore")WKWebsiteDataStore
--------------------------------------------------------------

如果一个 WebView 关联了一个非持久化的 WKWebsiteDataStore，将不会有数据被写入到文件系统  
该特性可以用来实现隐私浏览。  
同样可以通过设置 configuration 来设置无痕模式。

一个 WKWebsiteDataStore 对象代表了被网页使用的各种类型的数据。包括 cookies，磁盘文件，内存缓存以及持久化数据如 WebSQL，IndexedDB 数据库，local storage。

[](#WKURLSchemeHandler "WKURLSchemeHandler")WKURLSchemeHandler
--------------------------------------------------------------

用来处理 WebKit 无法处理的 URL Scheme 类型的资源

加载特定资源的时候使用。

```
- (void)webView:(WKWebView *)webView startURLSchemeTask:(id<WKURLSchemeTask>)urlSchemeTask;
- (void)webView:(WKWebView *)webView stopURLSchemeTask:(id<WKURLSchemeTask>)urlSchemeTask;
```

WKURLSchemeTask 即用来加载资源的任务。

[](#使用KVO来监控load进度 "使用KVO来监控load进度")使用 KVO 来监控 load 进度
------------------------------------------------------

```
[webView addObserver:self forKeyPath:@"loading" options:NSKeyValueObservingOptionNew context:nil];
[webView addObserver:self forKeyPath:@"estimatedProgress" options:NSKeyValueObservingOptionNew context:nil];
```

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if ([keyPath isEqualToString:@"loading"]) {
        NSLog(@"loading: %d", _webView.isLoading);
    } else if ([keyPath isEqualToString:@"estimatedProgress"]) {
        NSLog(@"estimatedProgress: %f", _webView.estimatedProgress);
        self.progressView.progress = _webView.estimatedProgress;
        if (self.progressView.progress >= 1.f) {
            self.progressView.progress = 0.f;
        }
    }
}
```

[](#查看所有cookie "查看所有cookie")查看所有 cookie
---------------------------------------

```
webView.configuration.websiteDataStore.httpCookieStore.getAllCookies { cookies in
    for cookie in cookies {
        if cookie.name == "authentication" {
            self.webView.configuration.websiteDataStore.httpCookieStore.delete(cookie)
        } else {
            print("\(cookie.name) is set to \(cookie.value)")
        }
    }
}
```

[](#白屏原因及处理措施 "白屏原因及处理措施")白屏原因及处理措施
-----------------------------------

详细内容见之前一篇博客 [WKWebView 的一些问题汇总](https://juejin.im/post/5df213f56fb9a01613801a4f) 。

*   [教你使用 WKWebView 的正确姿势](https://juejin.im/entry/5975916e518825594d23d777)
*   [WKWebView 详解](https://cloud.tencent.com/developer/article/1033743)
*   [iOS 开发高手课](https://time.geekbang.org/column/intro/161)
