# ReactNative 封装使用原生 UI 组件（Android）

以 ReactNative 提供的 [Webview](https://github.com/facebook/react-native/blob/ffbd3db61bce21b9332f7783086fce703706031f/ReactAndroid/src/main/java/com/facebook/react/views/webview/ReactWebViewManager.java) 来看

## 1、创建 ViewManager 的子类

一般可以选择实现 `SimpleViewManager` 类，并重写 `getName` 和 `createReactWebViewInstance`，`getName` 方法返回的名字会用于在 JavaScript 端引用这个原生视图，`createReactWebViewInstance` 方法返回的就是要导出的原生组件，还可以在 `onDropViewInstance` 在 View 被移除的时候释放资源

```java
public class ReactWebViewManager extends SimpleViewManager<WebView> {
  //..
  @Override
  public String getName() {
    return REACT_CLASS;
  }

  @Override
    protected WebView createViewInstance(ThemedReactContext reactContext) {
      ReactWebView webView = new ReactWebView(reactContext);
      webView.setWebChromeClient(new WebChromeClient() {
        //...
      });
      reactContext.addLifecycleEventListener(webView);
      mWebViewConfig.configWebView(webView);
      webView.getSettings().setBuiltInZoomControls(true);
      webView.getSettings().setDisplayZoomControls(false);
      webView.getSettings().setDomStorageEnabled(true);
      // Fixes broken full-screen modals/galleries due to body height being 0.
      webView.setLayoutParams( new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));
      if (ReactBuildConfig.DEBUG && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        WebView.setWebContentsDebuggingEnabled(true);
      }
      return webView;
    }

    @Override
    public void onDropViewInstance(WebView webView) {
      super.onDropViewInstance(webView);
      ((ThemedReactContext) webView.getContext()).removeLifecycleEventListener((ReactWebView) webView);
      ((ReactWebView) webView).cleanupCallbacksAndDestroy();
    }
}
```

其中 `ReactWebView` 就是原生 `WebView` 的一个实现类，`ReactWebViewBridge` Java 对象会注入到 `WebView` 中，名称为 `__REACT_WEB_VIEW_BRIDGE`，用来作为 WebView 和 JAVA 端通信的桥梁，公共的且带 `@JavascriptInterface` 的方法可以在 JS 端中调用，这里提供的方法是 `postMessage`，这个方法的回调流程会如下 `WebView ==> Native ==> JS` ，最终由组件的 `onMessage` 属性处理，但 `addJavascriptInterface` 方法在 4.2 一下的机器会有安全隐患，需要确保运行在 4.2 已收的机子上[了解更多](http://www.jianshu.com/p/3a345d27cd42)

```java

protected static class ReactWebView extends WebView implements LifecycleEventListener {
  protected @Nullable String injectedJS;
  protected boolean messagingEnabled = false;
  protected @Nullable ReactWebViewClient mReactWebViewClient;

  protected class ReactWebViewBridge {
    ReactWebView mContext;

    ReactWebViewBridge(ReactWebView c) {
      mContext = c;
    }

    @JavascriptInterface
    public void postMessage(String message) {
      mContext.onMessage(message);  //回调流程，WebView ==> Native ==> JS
    }
  }

  public ReactWebView(ThemedReactContext reactContext) {
    super(reactContext);
  }

  //....

  @Override
  public void onHostDestroy() {
    cleanupCallbacksAndDestroy();
  }

  @Override
  public void setWebViewClient(WebViewClient client) {
    super.setWebViewClient(client);
    mReactWebViewClient = (ReactWebViewClient)client;
  }

  public @Nullable ReactWebViewClient getReactWebViewClient() {
    return mReactWebViewClient;
  }

  public void setInjectedJavaScript(@Nullable String js) {
    injectedJS = js;
  }

  protected ReactWebViewBridge createReactWebViewBridge(ReactWebView webView) {
    return new ReactWebViewBridge(webView);
  }

  public void setMessagingEnabled(boolean enabled) {
    if (messagingEnabled == enabled) {
      return;
    }

    messagingEnabled = enabled;
    if (enabled) {
      addJavascriptInterface(createReactWebViewBridge(this), BRIDGE_NAME);
      linkBridge();
    } else {
      removeJavascriptInterface(BRIDGE_NAME);
    }
  }

  public void callInjectedJavaScript() {
    if (getSettings().getJavaScriptEnabled() &&
        injectedJS != null &&
        !TextUtils.isEmpty(injectedJS)) {
      loadUrl("javascript:(function() {\n" + injectedJS + ";\n})();");
    }
  }
  //为 webview 的 window 对象添加一个 `postMessage` 方法
  public void linkBridge() {
    if (messagingEnabled) {
      if (ReactBuildConfig.DEBUG && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        // See isNative in lodash
        String testPostMessageNative = "String(window.postMessage) === String(Object.hasOwnProperty).replace('hasOwnProperty', 'postMessage')";
        evaluateJavascript(testPostMessageNative, new ValueCallback<String>() {
          @Override
          public void onReceiveValue(String value) {
            if (value.equals("true")) {
              FLog.w(ReactConstants.TAG, "Setting onMessage on a WebView overrides existing values of window.postMessage, but a previous value was defined");
            }
          }
        });
      }

      loadUrl("javascript:(" +
        "window.originalPostMessage = window.postMessage," +
        "window.postMessage = function(data) {" +
          BRIDGE_NAME + ".postMessage(String(data));" +
        "}" +
      ")");
    }
  }

  public void onMessage(String message) {
    dispatchEvent(this, new TopMessageEvent(this.getId(), message));
  }

  protected void cleanupCallbacksAndDestroy() {
    setWebViewClient(null);
    destroy();
  }
}
```

## 3、通过 @ReactProp（或 @ReactPropGroup）注解来导出属性的设置方法

如果需要在 JS 端控制组件的某些属性，那么还需要导出属性的设置方法，如下导出部分属性

Value 支持类型为 `int`, `boolean`, `double`, `float`, `String`, `Boolean`, `ReadableArray`, `ReadableMap`

```java
public class ReactWebViewManager extends SimpleViewManager<WebView> {
  //..

  @ReactProp(name = "javaScriptEnabled")
  public void setJavaScriptEnabled(WebView view, boolean enabled) {
    view.getSettings().setJavaScriptEnabled(enabled);
  }

  @ReactProp(name = "messagingEnabled")
  public void setMessagingEnabled(WebView view, boolean enabled) {
    ((ReactWebView) view).setMessagingEnabled(enabled);
  }

  @ReactProp(name = "source")
  public void setSource(WebView view, @Nullable ReadableMap source) {
    if (source != null) {
      if (source.hasKey("html")) {
        String html = source.getString("html");
        if (source.hasKey("baseUrl")) {
          view.loadDataWithBaseURL( source.getString("baseUrl"), html, HTML_MIME_TYPE, HTML_ENCODING, null);
        } else {
          view.loadData(html, HTML_MIME_TYPE, HTML_ENCODING);
        }
        return;
      }
      if (source.hasKey("uri")) {
        String url = source.getString("uri");
        String previousUrl = view.getUrl();
        if (previousUrl != null && previousUrl.equals(url)) {
          return;
        }
        if (source.hasKey("method")) {
          String method = source.getString("method");
          if (method.equals(HTTP_METHOD_POST)) {
            byte[] postData = null;
            if (source.hasKey("body")) {
              String body = source.getString("body");
              try {
                postData = body.getBytes("UTF-8");
              } catch (UnsupportedEncodingException e) {
                postData = body.getBytes();
              }
            }
            if (postData == null) {
              postData = new byte[0];
            }
            view.postUrl(url, postData);
            return;
          }
        }
        HashMap<String, String> headerMap = new HashMap<>();
        if (source.hasKey("headers")) {
          ReadableMap headers = source.getMap("headers");
          ReadableMapKeySetIterator iter = headers.keySetIterator();
          while (iter.hasNextKey()) {
            String key = iter.nextKey();
            if ("user-agent".equals(key.toLowerCase(Locale.ENGLISH))) {
              if (view.getSettings() != null) {
                view.getSettings().setUserAgentString(headers.getString(key));
              }
            } else {
              headerMap.put(key, headers.getString(key));
            }
          }
        }
        view.loadUrl(url, headerMap);
        return;
      }
    }
    view.loadUrl(BLANK_URL);
  }
  //...
}
```

## 4、注册 ViewManager

需要新建一个 `ReactPackage` 或者使用现成的 `ReactPackage` 类，并在 `createViewManagers` 方法中返回封装的的 `ViewManager` 实例，最好记得把该 `ReactPackage` 注册到 应用的 `ReactApplication`

```java
public class DemoPackage implements ReactPackage {
    //...
    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactApplicationContext) {
        return Arrays.<ViewManager>asList(new ReactWebViewManager());
    }
    //...
}
```

## 5、实现对应的 JavaScript 模块

```javascript

import { PropTypes } from 'react';
import { requireNativeComponent, View } from 'react-native';

var RCTWebView = requireNativeComponent('RCTWebView', WebView, {
  nativeOnly: {
    messagingEnabled: PropTypes.bool,
  },
});
```

`requireNativeComponent`，通常接受两个参数，第一个参数是原生视图的名字，而第二个参数是一个描述组件接口的组件，如上则是 `WebView` 这个组件。组件接口应当声明一个友好的 name，用来在调试信息中显示；组件接口还 **必须** 声明 propTypes 字段，用来对应到原生视图上，

```javascript

class WebView extends React.Component {
  static propTypes = {
    ...ViewPropTypes,
    renderError: PropTypes.func,
    renderLoading: PropTypes.func,
    onLoad: PropTypes.func,
    onLoadEnd: PropTypes.func,
    onLoadStart: PropTypes.func,
    onError: PropTypes.func,
    automaticallyAdjustContentInsets: PropTypes.bool,
    contentInset: EdgeInsetsPropType,
    onNavigationStateChange: PropTypes.func,
    onMessage: PropTypes.func,
    onContentSizeChange: PropTypes.func,
    startInLoadingState: PropTypes.bool, // force WebView to show loadingView on first load
    style: ViewPropTypes.style,

    html: deprecatedPropType(
      PropTypes.string,
      'Use the `source` prop instead.'
    ),

    url: deprecatedPropType(
      PropTypes.string,
      'Use the `source` prop instead.'
    ),
    source: PropTypes.oneOfType([
      PropTypes.shape({

        uri: PropTypes.string,

        method: PropTypes.oneOf(['GET', 'POST']),

        headers: PropTypes.object,

        body: PropTypes.string,
      }),
      PropTypes.shape({

        html: PropTypes.string,

        baseUrl: PropTypes.string,
      }),

      PropTypes.number,
    ]),

    javaScriptEnabled: PropTypes.bool,

    domStorageEnabled: PropTypes.bool,

    injectedJavaScript: PropTypes.string,

    scalesPageToFit: PropTypes.bool,

    userAgent: PropTypes.string,

    testID: PropTypes.string,

    mediaPlaybackRequiresUserAction: PropTypes.bool,

    allowUniversalAccessFromFileURLs: PropTypes.bool,

    injectJavaScript: PropTypes.func,

    mixedContentMode: PropTypes.oneOf([
      'never',
      'always',
      'compatibility'
    ]),

    saveFormDataDisabled: PropTypes.bool,
  };
  //...
}

module.exports = WebView;
```

ReactNative 端的 `WebView` 只是一层壳，其包括了两层覆盖的 `View`，其中 `{webView}` 为 `RCTWebView` 组件，映射原生 `RCTWebView` 组件，是真正加载 web 页面的组件，`{otherView}` 主要用来渲染加载失败视图及加载中的提示视图

```javascript

render() {
  //....
  var webView =
    <RCTWebView
      ref={RCT_WEBVIEW_REF}
      key="webViewKey"
      style={webViewStyles}
      source={resolveAssetSource(source)}
      scalesPageToFit={this.props.scalesPageToFit}
      injectedJavaScript={this.props.injectedJavaScript}
      userAgent={this.props.userAgent}
      javaScriptEnabled={this.props.javaScriptEnabled}
      domStorageEnabled={this.props.domStorageEnabled}
      messagingEnabled={typeof this.props.onMessage === 'function'}
      onMessage={this.onMessage}
      contentInset={this.props.contentInset}
      automaticallyAdjustContentInsets={this.props.automaticallyAdjustContentInsets}
      onContentSizeChange={this.props.onContentSizeChange}
      onLoadingStart={this.onLoadingStart}
      onLoadingFinish={this.onLoadingFinish}
      onLoadingError={this.onLoadingError}
      testID={this.props.testID}
      mediaPlaybackRequiresUserAction={this.props.mediaPlaybackRequiresUserAction}
      allowUniversalAccessFromFileURLs={this.props.allowUniversalAccessFromFileURLs}
      mixedContentMode={this.props.mixedContentMode}
      saveFormDataDisabled={this.props.saveFormDataDisabled}
    />;

  return (
    <View style={styles.container}>
      {webView}
      {otherView}
    </View>
  );
}
```

## 6、通信

### JS Module => Native Module

ViewManager 实现类重写 `getCommandsMap` 方法和 `receiveCommand` 方法可以用来为 JS 端的控件提供可用命令并接收来自 JS 端发送过来的命令，

如下，`ReactWebViewManager` 提供了 6 种命令和其处理

```java
public class ReactWebViewManager extends SimpleViewManager<WebView> {
  //..
  @Override
  public @Nullable Map<String, Integer> getCommandsMap() {
    return MapBuilder.of(
        "goBack", COMMAND_GO_BACK,
        "goForward", COMMAND_GO_FORWARD,
        "reload", COMMAND_RELOAD,
        "stopLoading", COMMAND_STOP_LOADING,
        "postMessage", COMMAND_POST_MESSAGE,
        "injectJavaScript", COMMAND_INJECT_JAVASCRIPT
      );
  }

  @Override
  public void receiveCommand(WebView root, int commandId, @Nullable ReadableArray args) {
    switch (commandId) {
      //...
      case COMMAND_POST_MESSAGE:
        try {
          JSONObject eventInitDict = new JSONObject();
          eventInitDict.put("data", args.getString(0));
          root.loadUrl("javascript:(function () {" +
            "var event;" +
            "var data = " + eventInitDict.toString() + ";" +
            "try {" +
              "event = new MessageEvent('message', data);" +
            "} catch (e) {" +
              "event = document.createEvent('MessageEvent');" +
              "event.initMessageEvent('message', true, true, data.data, data.origin, data.lastEventId, data.source);" +
            "}" +
            "document.dispatchEvent(event);" +
          "})();");
        } catch (JSONException e) {
          throw new RuntimeException(e);
        }
        break;
      case COMMAND_INJECT_JAVASCRIPT:
        root.loadUrl("javascript:" + args.getString(0));
        break;
    }
  }
}
```

在 JS 端，通过 `UIManager.dispatchViewManagerCommand` 来进行命令发送

```javascript
class WebView extends React.Component {
  //..

  getWebViewHandle = () => {
    return ReactNative.findNodeHandle(this.refs[RCT_WEBVIEW_REF]);  //RCT_WEBVIEW_REF 为 RTCWebView 的 ref
  };

  postMessage = (data) => {
    UIManager.dispatchViewManagerCommand(
      this.getWebViewHandle(),
      UIManager.RCTWebView.Commands.postMessage,    //导出的命令类型
      [String(data)]
    );
  };

  injectJavaScript = (data) => {
    UIManager.dispatchViewManagerCommand(
      this.getWebViewHandle(),
      UIManager.RCTWebView.Commands.injectJavaScript,   //导出的命令类型
      [data]
    );
};
}
```

### Native Module ==> JS Module

主要通过 `RCTEventEmitter` 这个接口，

```java

protected static void dispatchEvent(WebView webView, Event event) {
  ReactContext reactContext = (ReactContext) webView.getContext();
  EventDispatcher eventDispatcher = reactContext.getNativeModule(UIManagerModule.class).getEventDispatcher();
  eventDispatcher.dispatchEvent(event);
}
```

或者这样

```java

public void dispatchEvent(WebView webView, String event) {
   WritableMap event = Arguments.createMap();
   event.putString("message", event);
   ReactContext reactContext = (ReactContext)webView.getContext();
   reactContext.getJSModule(RCTEventEmitter.class).receiveEvent(getId(), "topChange", event); //topChange 在 JS 端映射到 onChange 回调属性上（这个映射关系在UIManagerModuleConstants.java文件里）
 }
```

区别在是否自己封装一个 `Event`，发送事件需要提供原生视图的 id（用来和 JS 模块中的组件联系）、事件类型、事件值

如下，`ReactWebView` 在也没加载完之后发送一个 `TopLoadingFinishEvent` 的事件，事件名为 `topLoadingFinish`，映射到 JS 端的 `onLoadingFinish`，这个映射关系在 `UIManagerModuleConstants.java` 文件里

```java
public class ReactWebViewManager extends SimpleViewManager<WebView> {
//...
    protected static class ReactWebViewClient extends WebViewClient {
        //...
        @Override
        public void onPageFinished(WebView webView, String url) {
          super.onPageFinished(webView, url);

          if (!mLastLoadFailed) {
            ReactWebView reactWebView = (ReactWebView) webView;
            reactWebView.callInjectedJavaScript();
            reactWebView.linkBridge();
            emitFinishEvent(webView, url);
          }
        }
        //...
    }

    protected void emitFinishEvent(WebView webView, String url) {
      dispatchEvent(
          webView,
          new TopLoadingFinishEvent(webView.getId(), createWebViewEvent(webView, url)));
      }

    protected WritableMap createWebViewEvent(WebView webView, String url) {
      WritableMap event = Arguments.createMap();
      event.putDouble("target", webView.getId());
      event.putString("url", url);
      event.putBoolean("loading", !mLastLoadFailed && webView.getProgress() != 100);
      event.putString("title", webView.getTitle());
      event.putBoolean("canGoBack", webView.canGoBack());
      event.putBoolean("canGoForward", webView.canGoForward());
      return event;
    }

    protected static void dispatchEvent(WebView webView, Event event) {
      ReactContext reactContext = (ReactContext) webView.getContext();
      EventDispatcher eventDispatcher = reactContext.getNativeModule(UIManagerModule.class).getEventDispatcher();
      eventDispatcher.dispatchEvent(event);
    }
}
```

映射到 `onLoadingFinish` 属性

```javascript
  <RCTWebView
    ref={RCT_WEBVIEW_REF}
    //...
    onContentSizeChange={this.props.onContentSizeChange}
    onLoadingStart={this.onLoadingStart}
    onLoadingFinish={this.onLoadingFinish}
    onLoadingError={this.onLoadingError}
    //..
    />;
```

# 参考

- [你不知道的 Android WebView 使用漏洞](http://www.jianshu.com/p/3a345d27cd42)
- [源码分析之Native UI的封装和管理](http://blog.csdn.net/u013531824/article/details/54020287)
- [React Native 原生UI组件](http://www.jianshu.com/p/40e173fd5350)
