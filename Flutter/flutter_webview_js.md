# Flutter webview与JS交互

## 1. 常用属性

- initialUrl：初始load的url。

- javascriptMode：JS执行模式，是否允许JS执行。

- javascriptChannels：JS和Flutter通信的Channel。

- onWebViewCreated： 假如不为空，webview创建时候调用一次。

- navigationDelegate：路由委托，可以通过在此处拦截url。

- onPageFinished：WebView加载完毕时的回调。

- onPageStarted:WebView开始加载时回调。

- onProgress:WebView loading时回调。

- onWebResourceError:Webview资源加载失败时回调。

- userAgent:指定HTTP header 的User-Agent，不配置的话使用默认的。ios9之前自定义的userAgent会被忽略，因为系统不支持自定义的userAgent。

## 2. JS调用Flutter

### 1. 使用javascriptChannels发送消息

```dart
//Flutter 端
//Flutter 与 JS共同约定JavascriptChannel的name，JS使用name.postMessage()进行回调。
JavascriptChannel _alertJavascriptChannel(BuildContext context) { 
    return JavascriptChannel( name: 'Print', 
        onMessageReceived: (JavascriptMessage message) { 
        //回调处理    
        print(message.message);
    }); 
 } 

WebView( 
//````
    javascriptMode: JavascriptMode.unrestricted,
    javascriptChannels: <JavascriptChannel>{_alertJavascriptChannel(context)}
//````
);

//JS端
//
<button onclick="callFlutter()">callFlutter</button>
function callFlutter(){
   Print.postMessage("JS调用了Flutter");
}

```

postMessage使用：https://juejin.cn/post/6844903665694687240

### 2. 使用navigationDelegate拦截url

- NavigationDecision.prevent：阻止路由替换

- NavigationDecision.navigate：允许路由替换

```js
//JS端
<button onclick="callFlutter()">callFlutter</button> 
function callFlutter(){ 
//约定的url协议为：js://webview?arg1=111&arg2=222 
    document.location = "js://webview?arg1=111&args2=222";
}

//Flutter
WebView(
//```
javascriptMode: JavascriptMode.unrestricted,
navigationDelegate: (NavigationRequest request) { 
//判断是否以约定的url开头
    if (request.url.startsWith('js://webview')) { 
    //阻止替换，进行业务处理
        return NavigationDecision.prevent; 
    } 
    //允许替换
    return NavigationDecision.navigate; 
}
//```
);
```

## 3. Flutter调用JS

- evaluateJavascript() 

- onPageFinished加载完成后调用

- Android返回数据格式是JSON

- iOS返回数据格式是String或者String数组

```dart
//Flutter
//
final Completer<WebViewController> _controller = 
    Completer<WebViewController>();
  
WebView(
//```
    initialUrl: url,
    javascriptMode: JavascriptMode.unrestricted,
    onWebViewCreated: (WebViewController webViewController) {
        _controller.complete(webViewController);
     }
//```
 );
 
 _controller.future.then((controller) {
     controller
     .evaluateJavascript('callJS("visible")')
     .then((result) {
     
     });
 });
 
 //JS端
 //将文字隐藏、显示
<p id="p1" style="visibility:hidden;"> 
    Flutter 调用了 JS. Flutter 调用了 JS. Flutter 调用了 JS. 
</p> 

function callJS(message){ 
    document.getElementById("p1").style.visibility = message; 
}
```
