---
title: "flutter webview android h5 上传文件失败解决办法"
date: 2019-09-15T16:01:23+08:00
lastmod: 2019-09-15T18:01:23+08:00
draft: false
tags: ["bugFixed"]
categories: ["flutter"]

---


在项目中遇到的第二个坑： android webview 里点击\<input type='file'>会秒退。

出现这个问题的原因是，我使用的官方的webview_flutter插件没有针对这一功能做适配。而我在网上找到另一款flutter webview插件flutter webview plugin.这个插件针对android端webview上传文件的功能做了适配。

这里第一种思路是直接摒弃原版的webview_flutter插件。发现行不通，因为后者没有实现js channel和一些其他的功能。

第二种思路是把flutter webview plugin插件的文件上传部分的代码迁移到webview_flutter中。



#### 插件是怎么调起来的

在代码迁移之前，首先说一下插件的代码是从哪开始走的。

在flutter工程的android/app/src/main/java/../mainActivity里边有这一行代码

```
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //注册插件
    GeneratedPluginRegistrant.registerWith(this);
  }
  
```

在下面目录的GeneratedPluginRegistrant类里

```
FlutterWebviewPlugin.registerWith(registry.registrarFor("com.flutter_webview_plugin.FlutterWebviewPlugin"));
    WebViewFlutterPlugin.registerWith(registry.registrarFor("io.flutter.plugins.webviewflutter.WebViewFlutterPlugin"));
```

然后去插件文件夹里找registerWith这个静态方法。

```
  public static void registerWith(Registrar registrar) {
    factory = new WebViewFactory(registrar.messenger(), registrar.view());
    registrar
        .platformViewRegistry()
        .registerViewFactory(
            "plugins.flutter.io/webview",
            factory);
    
    final WebViewFlutterPlugin instance = new WebViewFlutterPlugin(registrar.activity(),registrar.activeContext());
    registrar.addActivityResultListener(instance);
    FlutterCookieManager.registerWith(registrar.messenger());
  }
```

然后就是插件的内部逻辑了。



#### android webview 文件上传的思路

不论代码怎么变，Android webview文件上传始终是那个思路。

点击选择文件的时候，会调用 webviewClient 下的`openFileChooser()`（5.0及以上系统回调`onShowFileChooser()`），这个时候我们在`openFileChooser`方法中通过`Intent`打开系统相册或者支持该`Intent`的第三方应用来拿到选择文件的uri，最后在onActivityResult()中将选择的文件uri通过`ValueCallback`的`onReceiveValue`方法返回给`WebView`，然后通过js上传。



#### 代码迁移

我在这里简单介绍一下代码迁移的思路方法。我贴出的代码直接拷贝可能不能用，因为在其他细节的地方我还做了一些类型的适配,在这里不一一介绍。在文章尾我会贴出我修改后的webview_flutter插件的代码，可以拿来直接用。

1. 实现openFileChooer()方法：

在webview_flutter插件目录/plugins/webviewflutter/flutterWebview.java中。找到

```
webView = new InputAwareWebView(activityContext, containerView);
```

webview的定义，然后在后面实现并根据各android版本重写onShowFileChooser();

```
webView.setWebChromeClient(new WebChromeClient()
        {
            //The undocumented magic method override
            //Eclipse will swear at you if you try to put @Override here
            // For Android 3.0+
            public void openFileChooser(ValueCallback<Uri> uploadMsg) {

                mUploadMessage = uploadMsg;
                Intent i = new Intent(Intent.ACTION_GET_CONTENT);
                i.addCategory(Intent.CATEGORY_OPENABLE);
                i.setType("image/*");
                activity.startActivityForResult(Intent.createChooser(i,"File Chooser"), FILECHOOSER_RESULTCODE);

            }

            // For Android 3.0+
            public void openFileChooser( ValueCallback uploadMsg, String acceptType ) {
                mUploadMessage = uploadMsg;
                Intent i = new Intent(Intent.ACTION_GET_CONTENT);
                i.addCategory(Intent.CATEGORY_OPENABLE);
                i.setType("*/*");
               activity.startActivityForResult(
                        Intent.createChooser(i, "File Browser"),
                        FILECHOOSER_RESULTCODE);
            }

            //For Android 4.1
            public void openFileChooser(ValueCallback<Uri> uploadMsg, String acceptType, String capture){
                mUploadMessage = uploadMsg;
                Intent i = new Intent(Intent.ACTION_GET_CONTENT);
                i.addCategory(Intent.CATEGORY_OPENABLE);
                i.setType("image/*");
                activity.startActivityForResult( Intent.createChooser( i, "File Chooser" ), FILECHOOSER_RESULTCODE );

            }

            //For Android 5.0+
            public boolean onShowFileChooser(
                    WebView webView, ValueCallback<Uri[]> filePathCallback,
                    FileChooserParams fileChooserParams){
                if(mUploadMessageArray != null){
                    mUploadMessageArray.onReceiveValue(null);
                }
                mUploadMessageArray = filePathCallback;

                final String[] acceptTypes = getSafeAcceptedTypes(fileChooserParams);
                List<Intent> intentList = new ArrayList<Intent>();
                fileUri = null;
                videoUri = null;
                if (acceptsImages(acceptTypes)) {
                    Intent takePhotoIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
                    fileUri = getOutputFilename(MediaStore.ACTION_IMAGE_CAPTURE);
                    takePhotoIntent.putExtra(MediaStore.EXTRA_OUTPUT, fileUri);
                    intentList.add(takePhotoIntent);
                }
                if (acceptsVideo(acceptTypes)) {
                    Intent takeVideoIntent = new Intent(MediaStore.ACTION_VIDEO_CAPTURE);
                    videoUri = getOutputFilename(MediaStore.ACTION_VIDEO_CAPTURE);
                    takeVideoIntent.putExtra(MediaStore.EXTRA_OUTPUT, videoUri);
                    intentList.add(takeVideoIntent);
                }
                Intent contentSelectionIntent;
                if (Build.VERSION.SDK_INT >= 21) {
                    final boolean allowMultiple = fileChooserParams.getMode() == FileChooserParams.MODE_OPEN_MULTIPLE;
                    contentSelectionIntent = fileChooserParams.createIntent();
                    contentSelectionIntent.putExtra(Intent.EXTRA_ALLOW_MULTIPLE, allowMultiple);
                } else {
                    contentSelectionIntent = new Intent(Intent.ACTION_GET_CONTENT);
                    contentSelectionIntent.addCategory(Intent.CATEGORY_OPENABLE);
                    contentSelectionIntent.setType("*/*");
                }
                Intent[] intentArray = intentList.toArray(new Intent[intentList.size()]);

                Intent chooserIntent = new Intent(Intent.ACTION_CHOOSER);
                chooserIntent.putExtra(Intent.EXTRA_INTENT, contentSelectionIntent);
                chooserIntent.putExtra(Intent.EXTRA_INITIAL_INTENTS, intentArray);
                activity.startActivityForResult(chooserIntent, FILECHOOSER_RESULTCODE);
                return true;
            }

            @Override
            public void onProgressChanged(WebView view, int progress) {
                Map<String, Object> args = new HashMap<>();
                args.put("progress", progress / 100.0);
                FlutterWebviewPlugin.channel.invokeMethod("onProgressChanged", args);
            }

            public void onGeolocationPermissionsShowPrompt(String origin, GeolocationPermissions.Callback callback) {
                callback.invoke(origin, true, false);
            }
```

这样，就打通了第一个环节，能调用原生的选择文件功能了。

接下来参考flutter_webview_plugin的思路，写一个内部类拿到这个选择文件的uri。

```
@TargetApi(7)
    class ResultHandler {
        public boolean handleResult(int requestCode, int resultCode, Intent intent){
            boolean handled = false;
            if(Build.VERSION.SDK_INT >= 21){
                if(requestCode == FILECHOOSER_RESULTCODE){
                    Uri[] results = null;
                    if (resultCode == Activity.RESULT_OK) {
                        if (fileUri != null && getFileSize(fileUri) > 0) {
                            results = new Uri[] { fileUri };
                        } else if (videoUri != null && getFileSize(videoUri) > 0) {
                            results = new Uri[] { videoUri };
                        } else if (intent != null) {
                            results = getSelectedFiles(intent);
                        }
                    }
                    if(mUploadMessageArray != null){
                        mUploadMessageArray.onReceiveValue(results);
                        mUploadMessageArray = null;
                    }
                    handled = true;
                }
            }else {
                if (requestCode == FILECHOOSER_RESULTCODE) {
                	Uri result = null;
                    if (resultCode == RESULT_OK && intent != null) {
                        result = intent.getData();
                    }
                    if (mUploadMessage != null) {
                        mUploadMessage.onReceiveValue(result);
                        mUploadMessage = null;
                    }
                    handled = true;
                }
            }
            return handled;
        }
    }
```

这样实现了第二步，拿到uri，

然后就是通过onActivityResult()中将选择的文件uri通过`ValueCallback`的`onReceiveValue`方法返回给`WebView`，然后通过js上传。

onActivityResult()方法需要在插件的注册类中实现。

webview_flutter的注册类是WebViewFlutterPlugin，在这个类中实现PluginRegistry.ActivityResultListener接口，然后重写onActivityResult()方法。

```
/** WebViewFlutterPlugin */
public class WebViewFlutterPlugin implements PluginRegistry.ActivityResultListener{
  /** Plugin registration. */
  private Activity activity;
  private Context context;

  //add for file
  // private ValueCallback<Uri> mUploadMessage;
  // private ValueCallback<Uri[]> mUploadMessageArray;
  private final static int FILECHOOSER_RESULTCODE=1;
  // private Uri fileUri;
  // private Uri videoUri;
  private static WebViewFactory factory;

  private WebViewFactory factory2;
  public static void registerWith(Registrar registrar) {
    factory = new WebViewFactory(registrar.messenger(), registrar.view());
    registrar
        .platformViewRegistry()
        .registerViewFactory(
            "plugins.flutter.io/webview",
            factory);
    
    final WebViewFlutterPlugin instance = new WebViewFlutterPlugin(registrar.activity(),registrar.activeContext());
    registrar.addActivityResultListener(instance);
    FlutterCookieManager.registerWith(registrar.messenger());
  }

  private WebViewFlutterPlugin(Activity activity, Context context){
    this.activity = activity;
    this.context = context;
  }

  @Override
    public boolean onActivityResult(int i, int i1, Intent intent) {
      
        if(factory!=null){
          factory2 = factory;
          System.out.println(factory2.getFlutterWebView() != null);
          if (factory2.getFlutterWebView() != null) {
            return factory2.getFlutterWebView().resultHandler.handleResult(i, i1, intent);
          }
          return false;
        }else{
          WebViewFactory flutterWebViewFactory = new WebViewFactory(null,null);
          if(flutterWebViewFactory!=null){
            return flutterWebViewFactory.getFlutterWebView().resultHandler.handleResult(i, i1, intent);
          }
          return false;
        }
        
    }
}


```

第三步完成，写web的小伙伴终于可以拿到你选的文件了，喜大普奔。



完整的webview_flutter插件目录：[百度云](https://pan.baidu.com/s/1wPo4No914uK1Cgu9-QZt4g)