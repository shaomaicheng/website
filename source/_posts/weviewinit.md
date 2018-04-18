---
title: WebView深究之Android是如何实现webview初始化的
date: 2018-01-12 00:42:53
tags:
---
### webview初始化
关注Android加载webview内核的过程。我们从WebView的init过程中切入。
WebView的构造方法,最终都会调用
```
WebView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes,
            Map<String, Object> javaScriptInterfaces, boolean privateBrowsing)
```
这个方法，这个方法做了如下几个事情：

1. 线程检查
2. Provider类的选择
3. Provider类的初始化
4. Cookie相关的处理


线程检查是为了保证WebView在同一个线程中进行创建，暂时不深究。把主要的经历放在Provider相关的逻辑。

mProvider是WebView中一个成员变量，类型是WebViewProvider.实际上，我们可以发现WebView所有的逻辑处理都是通过WebViewProvider来实现的，例如：

load(String url)方法的源码实现是
```java
public void loadUrl(String url, Map<String, String> additionalHttpHeaders) {
        checkThread();
        mProvider.loadUrl(url, additionalHttpHeaders);
}
```

WebViewProvider实际是一个interface
进入源码，他包括了大部分WebView中的同名方法。包括loadUrl,reload,goBack等。所以他是WebView的真正能力提供者。

查看源码ensureProviderCreated()方法
```java
mProvider = getFactory().createWebView(this, new PrivateAccess());
```

getFactory的具体实现是
```java
private static synchronized WebViewFactoryProvider getFactory() {
        return WebViewFactory.getProvider();
}
```

这一步是为了确定WebViewProvider的子类,Provider的构造根据WebViewFactory类确定。Android在5.0后将webview内核从webkit换成了chromium，这里使用工厂模式可以将内核实现反射和上层初始化的代码解耦

我们选取getProvider()方法中的核心逻辑进行分析

```java
try {
    Class<WebViewFactoryProvider> providerClass = getProviderClass();

    Trace.traceBegin(Trace.TRACE_TAG_WEBVIEW, "providerClass.newInstance()");
        try {
            sProviderInstance = providerClass.getConstructor(WebViewDelegate.class)
            .newInstance(new WebViewDelegate());
            return sProviderInstance;
        } catch (Exception e) {
        } finally {
        }
    finally {
    }
}
```

有些版本的实现为
```java
try {
    Class<WebViewFactoryProvider> providerClass = getProviderClass();
    Method staticFactory = null;
    try {
        staticFactory = providerClass.getMethod(CHROMIUM_WEBVIEW_FACTORY_METHOD, WebViewDelegate.class);
    } catch (Exception e) {
    }
    
    try {
        sProviderInstance = (WebViewFactoryProvider)staticFactory.invoke(null, new WebViewDelegate());
        return sProviderInstance;
    } catch (Exception e) {
    }
    
} catch (Exception e) {
}
```

这里在获取了实际的FactoryProvider class之后，利用反射，创建了真实的FactoryProvider对象

那么到底是如何确认应该获取那个FactoryProvider对象呢
继续看getProviderClass()方法的代码：
```java
Context webViewContext = null;
Application initialApplication = AppGlobals.getInitialApplication();

try {

    try {
        webViewContext = getWebViewContextAndSetProvider();
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_WEBVIEW);
    }
    
    
    try {
        ClassLoader clazzLoader = webViewContext.getClassLoader();

        loadNativeLibrary(clazzLoader);
        
        try {
            return (Class<WebViewFactoryProvider>) Class.forName(CHROMIUM_WEBVIEW_FACTORY,true, clazzLoader);
        } finally {
        }
        
    } catch () {
        
    }

} catch (MissingWebViewPackageException e) {
    try {
        return (Class<WebViewFactoryProvider>) Class.forName(NULL_WEBVIEW_FACTORY);
    } catch (ClassNotFoundException e2) {
        // Ignore.
    }
}


```

可以看到
Provider类型结果分为两种
* CHROMIUM_WEBVIEW_FACTORY com.android.webview.chromium.WebViewChromiumFactoryProvider
* NULL_WEBVIEW_FACTORY  com.android.webview.nullwebview.NullWebViewFactoryProvider


正常的加载结果大致分为2步
1. 确定webview的PackageInfo和Context
2. 根据此结果load library


分析第一步，我们查看方法getWebViewContextAndSetProvider()

里面的核心逻辑为
```java
WebViewProviderResponse response = null;
response = getUpdateService().waitForAndGetProvider();
PackageInfo newPackageInfo = null;

newPackageInfo = initialApplication.getPackageManager().getPackageInfo(
                    response.packageInfo.packageName,
                    PackageManager.GET_SHARED_LIBRARY_FILES
                    | PackageManager.MATCH_DEBUG_TRIAGED_MISSING
                    // Make sure that we fetch the current provider even if its not
                    // installed for the current user
                    | PackageManager.MATCH_UNINSTALLED_PACKAGES
                    // Fetch signatures for verification
                    | PackageManager.GET_SIGNATURES
                    // Get meta-data for meta data flag verification
                    | PackageManager.GET_META_DATA);
                    
                    
verifyPackageInfo(response.packageInfo, newPackageInfo);


Context webViewContext = initialApplication.createApplicationContext(
                        newPackageInfo.applicationInfo,
                        Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY);
sPackageInfo = newPackageInfo;
return webViewContext;


```

我们重点追踪waitForAndGetProvider()这个方法。这个方法会调用
 WebViewUpdateServiceImpl类的waitForAndGetProvider()方法，继而调用WebViewUpdate的waitForAndGetProvider()
 
 waitForAndGetProvider()中会根据mCurrentWebViewPackage创建WebViewProviderResponse对象

在onWebViewProviderChanged()中，我们获取到了Package对象。在findPreferredWebViewPackage()方法中，我们会获取到最新的package对象

findPreferredWebViewPackage()的核心逻辑为
```java
String userChosenProvider = mSystemInterface.getUserChosenWebViewProvider(mContext);
```

如果用户已经选择了WebView，那么使用用户选择的
```java
for (ProviderAndPackageInfo providerAndPackage : providers) {
    if (providerAndPackage.provider.packageName.equals(userChosenProvider)) {
    // userPackages can contain null objects.
    List<UserPackage> userPackages =
    mSystemInterface.getPackageInfoForProviderAllUsers(mContext,providerAndPackage.provider);
        if (isInstalledAndEnabledForAllUsers(userPackages)) {
            return providerAndPackage.packageInfo;
        }
    }
}

```

如果选择失败或者没有选择，就使用最稳定的
```java
for (ProviderAndPackageInfo providerAndPackage : providers) {
    if (providerAndPackage.provider.availableByDefault) {
        // userPackages can contain null objects.
        List<UserPackage> userPackages =
        mSystemInterface.getPackageInfoForProviderAllUsers(mContext,providerAndPackage.provider);
        if (isInstalledAndEnabledForAllUsers(userPackages)) {
            return providerAndPackage.packageInfo;
        }
    }
}
```

那么Android是如何获取所有的package呢？
在com.android.server.webkit.SystemImpl中有答案，
```java
parser = AppGlobals.getInitialApplication().getResources().getXml(
                    com.android.internal.R.xml.config_webview_packages);
```

在SystemImpl中调用了如下逻辑。原来，webview相关的package信息是存放在一个xml文件里面的。

config_webview_packages.xml的内容如下：
```xml
<webviewproviders>
    <!-- The default WebView implementation -->
    <webviewprovider description="Android WebView" packageName="com.android.webview" />
</webviewproviders>
```

由此可见，Android默认的system webview的package name就是com.android.webview，在Android 7.0后，用户可以在settings里面选择webview实现。如果用户手机安装了Chrome android app，那么可以将webview实现选择为chrome


这里我们就跟踪完了第一步获取WebView的Context和Provider。这里会返回Context。

接下来分析第二步骤，真正的加载webview的chromium动态库

loadNativeLibrary方法的核心逻辑如下
```java
String[] args = getWebViewNativeLibraryPaths(sPackageInfo);
int result = nativeLoadWithRelroFile(args[0] /* path32 */,
                                             args[1] /* path64 */,
                                             CHROMIUM_WEBVIEW_NATIVE_RELRO_32,
                                             CHROMIUM_WEBVIEW_NATIVE_RELRO_64,
                                             clazzLoader);
return result;
```

走到getWebViewNativeLibraryPaths，这个是获取webview动态库的path，分别还对32位和64位系统进行了处理。而这个动态库的形式，是一个APK文件。
在getLoadFromApkPath方法中有如下调用
```java
try (ZipFile z = new ZipFile(apkPath)) {
    for (String abi : abiList) {
        final String entry = "lib/" + abi + "/" + nativeLibFileName;
        ZipEntry e = z.getEntry(entry);
        if (e != null && e.getMethod() == ZipEntry.STORED) {
            // Return a path formatted for dlopen() load from APK.
            return apkPath + "!/" + entry;
        }
    }
} catch (IOException e) {
    throw new MissingWebViewPackageException(e);
}
```

loadWithRelroFile方法则调用了一个jni方法
nativeLoadWithRelroFile()

我们可以找到源码framework/base/native/webview里面找到loader.cpp
里面有jint LoadWithRelroFile(JNIEnv* env, jclass, jstring lib, jstring relro32,
                       jstring relro64, jobject clazzLoader)方法，这个方法调用了jint DoLoadWithRelroFile(JNIEnv* env, const char* lib, const char* relro,
                         jobject clazzLoader)方法，这个方法的内部逻辑如下

```cpp
int relro_fd = TEMP_FAILURE_RETRY(open(relro, O_RDONLY));
android_namespace_t* ns =
      android::FindNamespaceByClassLoader(env, clazzLoader);
  android_dlextinfo extinfo;
  extinfo.flags = ANDROID_DLEXT_RESERVED_ADDRESS | ANDROID_DLEXT_USE_RELRO |
                  ANDROID_DLEXT_USE_NAMESPACE;
  extinfo.reserved_addr = gReservedAddress;
  extinfo.reserved_size = gReservedSize;
  extinfo.relro_fd = relro_fd;
  extinfo.library_namespace = ns;
  void* handle = android_dlopen_ext(lib, RTLD_NOW, &extinfo);
  close(relro_fd);
  return LIBLOAD_SUCCESS;
```
这部分cpp代码不是很熟，参考了网上的解释
"
函数DoLoadWithRelroFile会将告诉函数android_dlopen_ext在加载Chromium动态库的时候，将参数relro描述的Chromium GNU_RELRO Section文件内存映射到内存来，并且代替掉已经加载的Chromium动态库的GNU_RELRO Section。这是通过将指定一个ANDROID_DLEXT_USE_RELRO标志实现的。

之所以可以这样做，是因为参数relro描述的Chromium GNU_RELRO Section文件对应的Chromium动态库的加载地址与当前App进程加载的Chromium动态库的地址一致。只要两个相同的动态库在两个不同的进程中的加载地址一致，它们的链接和重定位信息就是完全一致的，因此就可以通过文件内存映射的方式进行共享。共享之后，就可以达到节省内存的目的了
"

这里就完成了Chromium动态库的加载。

回到webview初始化的地方，继续会调用WebViewFactoryProvider的createWebView方法,如果加载的chromium，那么这个接口的实现类是WebViewChromiumFactoryProvider，它在createWebView中会返回一个WebViewChromium对象
```java
WebViewChromium wvc = new WebViewChromium(this, webView, privateAccess);

synchronized (mLock) {
    if (mWebViewsToStart != null) {
        mWebViewsToStart.add(new WeakReference<WebViewChromium>(wvc));
    }
}

return wvc;
```

我们实际的Provider类为WebViewChromium,接下来会调用WebViewChromium的init方法，这里会初始化真正的WebView引擎，包括渲染等模块的加载

WebView的初始化过程就分析到这里。后面会继续分析Provider的工作流程，以及WebView是如何加载动态库的细节~

参考文章地址：http://blog.csdn.net/luoshengyang/article/details/53209199