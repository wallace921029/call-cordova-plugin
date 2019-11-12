# HTML5 调用 Cordova 插件

## 插件
安装以下插件
```sh
cordova plugin add xxx
```
- 相机
> cordova-plugin-camera
- 扫描二维码
> https://github.com/wildabeast/BarcodeScanner.git
- mac 地址
> https://github.com/mohamed-salah/MacAddress.git

## 修改源码
打开 platforms/andriod/CordovaLib\src\org\apache\cordova\engine\SystemWebViewClient.java，修改以下内容：
```java
public void clearAuthenticationTokens() {
    this.authenticationTokens.clear();
}

-- 新增开始 --
private static final String INJECTION_TOKEN = "http://injection/"; 
-- 新增结束 --

@TargetApi(Build.VERSION_CODES.HONEYCOMB)
@Override
public WebResourceResponse shouldInterceptRequest(WebView view, String url) {

    ---新增开始---
    if(url != null && url.contains(INJECTION_TOKEN)) {
        String assetPath = url.substring(url.indexOf(INJECTION_TOKEN) + INJECTION_TOKEN.length(), url.length());
        try {
            WebResourceResponse response = new WebResourceResponse(
                    "application/javascript",
                    "UTF8",
                    view.getContext().getAssets().open(assetPath)
            );
            return response;
        } catch (IOException e) {
            e.printStackTrace(); // Failed to load asset file
            return new WebResourceResponse("text/plain", "UTF-8", null);
        }
    }
    ---新增结束---
    try {

        // Check the against the whitelist and lock out access to the WebView directory
        // Changing this will cause problems for your application
        if (!parentEngine.pluginManager.shouldAllowRequest(url)) {
            LOG.w(TAG, "URL blocked by whitelist: " + url);
            // Results in a 404.
            return new WebResourceResponse("text/plain", "UTF-8", null);
        }

        CordovaResourceApi resourceApi = parentEngine.resourceApi;
        Uri origUri = Uri.parse(url);
        // Allow plugins to intercept WebView requests.
        Uri remappedUri = resourceApi.remapUri(origUri);

        if (!origUri.equals(remappedUri) || needsSpecialsInAssetUrlFix(origUri) || needsKitKatContentUrlFix(origUri)) {
            CordovaResourceApi.OpenForReadResult result = resourceApi.openForRead(remappedUri, true);
            return new WebResourceResponse(result.mimeType, "UTF-8", result.inputStream);
        }
        // If we don't need to special-case the request, let the browser load it.
        return null;
    } catch (IOException e) {
        if (!(e instanceof FileNotFoundException)) {
            LOG.e(TAG, "Error occurred while loading a file (returning a 404).", e);
        }
        // Results in a 404.
        return new WebResourceResponse("text/plain", "UTF-8", null);
    }
}
```
在 Vue 项目的 index.html 文件中，引入 cordova.js：
```js
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width,initial-scale=1.0">
    <link rel="icon" href="<%= BASE_URL %>favicon.ico">
    <title>iME移动平台</title>
  </head>
  <body>
    <noscript>
      <strong>We're sorry but ime-mobile-latest doesn't work properly without JavaScript enabled. Please enable it to continue.</strong>
    </noscript>
    <div id="app"></div>
    <!-- built files will be auto injected -->
    <script src="http://injection/www/cordova.js" type="text/javascript" charset="UTF-8"></script>
  </body>
</html>
```

## 配置 config.xml
添加内下配置：
```xml
<allow-navigation href="http://*/*" />
<allow-navigation href="https://*/*" />
```

## 相机调用
https://cordova.apache.org/docs/en/latest/reference/cordova-plugin-camera/index.html
```js
// cameraSuccess cameraError cameraOptions 参数具体含义请参考文档
navigator.camera.getPicture(cameraSuccess, cameraError, cameraOptions);

// 例如， 在 methods 中：
callCamera() {
  navigator.camera.getPicture(
    (imageData) => {
      alert(imageData)
    },
    (message) => {
      alert(message)
    },
    {
      quality: 50,
      destinationType: 0
    }
  )
}
```

## 扫描二维码
在 vue 文件中引入 <script type="text/javascript" src="cordova_plugins.js"></script>
```js
scanCode() {
  cordova.plugins.barcodeScanner.scan(
    (result) => {
      alert(result.text, result.format, result.cancelled)
    },
    (error) => {
      alert(error);
    }
  )
}
```

## mac 地址
```js
window.MacAddress.getMacAddress(
  (macAddress) => {
    alert(macAddress)
  },
  (fail) => {
    alert(fail)
  }
);
```
