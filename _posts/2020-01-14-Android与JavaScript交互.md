### 一、Android與JavaScript交互的先決條件
1. 設置WebView可以執行JavaScript代碼
```
WebSettings settings = webview.getSettings();
settings.setJavaScriptEnabled(true);        // 啟用JS
```

2. 給WebView綁定一個接口對象
```
mJSInterface = new WebViewJSInterface(webview, mCallback);
webview.addJavascriptInterface(mJSInterface, "pkb");
```
addJavascriptInterface方法把Java對象綁定給JavaScript使用，同時指明JavaScript側別名"pkb"

### 二、JavaScript調用Android
要想使JavaScript调用Android方法，此方法必须满足两个条件：
1. 此方法是public
2. 必须加上@JavascriptInterface标签

假設WebViewJSInterface類如下:
```
public class WebViewJSInterface {
    private WebView mWebView;
    private final int mCallback;

    public WebViewJSInterface(WebView webview, int cb) {
        mWebView = webview;
        mCallback = cb;
    }
    
    @JavascriptInterface
    public void printMessage(String msg) {
        Log.d(msg);
    }
}
```
在JavaScript側代碼調用Android方法printMessage:
```
pkb.printMessage("This is JS code.")
```

### 三、Android調用JavaScript
Android調用JavaScript方法，需要以字符串的方式進行調用，字符串前需要加"javascript:"，並且需要在UI線程（主線程）上調用
```
final String evaluateStr = "javascript:callJSFunc('This is a Android message')"
AppActivity.instance.runOnUiThread(new Runnable() {
    @Override
    public void run() {
        mWebView.evaluateJavascript(evaluateStr, new ValueCallback<String>() {
            @Override
            public void onReceiveValue(String s) {

            }
        });
    }
});
```
要在Android側執行JavaScript的回調（Callback），需要在JavaScript側進行一層適配

```
window.pk = {}
let pk = window.pk

let __uuid = 0;
let __callbackDict = [];
let addCallback = function(funcName, callbackId, callback) {
    __callbackDict[funcName] = {
        callbackId: callbackId,
        callback: callback
    }
}

pk.callback = function(callbackId, data) {
    let callback;
    for (let funcName in __callbackDict) {
        if (__callbackDict[funcName].callbackId == callbackId) {
            callback = __callbackDict[funcName].callback;
            if (callback) {
                callback(data);
            }
            return;
        }
    }
}

// 獲取音樂是否可用
pk.getMusicEnabled = function(params) {
    let callbackId = ++__uuid;
    addCallback("getMusicEnabled", callbackId, params.callback);
    pkb.getMusicEnabled(callbackId);
}

```
然後在Android側調用時，把callbackId回傳給JavaScript，修改後的Android調用JavaScript代碼如下：

```
final String resultStr = String.format("callJSFunc(%d,%s)", callbackId, 'This is a Android message');
final String evaluateStr = "javascript:" + resultStr;
AppActivity.instance.runOnUiThread(new Runnable() {
    @Override
    public void run() {
        mWebView.evaluateJavascript(evaluateStr, new ValueCallback<String>() {
            @Override
            public void onReceiveValue(String s) {

            }
        });
    }
});
```
### 四、release包混淆問題
* 需要把WebViewJSInterface幾其內部的public方法，聲明為不混淆
* 假設WebViewJSInterface類在包com.poker.webview內
```
-keepclassmembers class com.poker.webview.** {
  public *;
}
-keepattributes *Annotation*
-keepattributes *JavascriptInterface*
```

