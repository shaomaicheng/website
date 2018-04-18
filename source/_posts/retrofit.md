---
title: retrofit源码解析
date: 2017-04-10 23:02:08
tags:
---
在Android客户端的项目网络请求实践中，对retrofit进行了实践和源码的阅读。从retrofit的用法入手，对retrofit进行解析。
首先看一下retrofit的基本用法：
第一步创建retrofit对象：
```
Retrofit retrofit = new Retrofit.Builder()
          .baseUrl(baseUrl)
          .build();
```
第二步创建一个service接口，接口的组成如下
```
public interface NetWorkService {
    @FormUrlEncoded
    @Post("/reativePath")
    Call<ResponseBody> getResponse(@Field("param") String param);
}
```
第三进行网络请求
```
NetWorkService service = retrofit.create(NetWorkService.class);
Call<ResponseBody> call = service.getResponse(param);
```

查看Retrofit的类结构，使用了经典的Builder设计模式，通过Builder的方法我们通常可以设置OkHttpClient、HttpUrl、ConverterFactory、CallAdapterFactory等。
查看Retrofit的create方法
```
return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
```
这里使用了Java的动态代理模式，关于代理模式和动态代理，可以查看[这里](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)。
在create中，创建了一个最终暴露给我们的代理类A，类型为T，也就是xxService，内部的委托类B为service，也是我们的类xxService。其中起到2个类的桥接作用的是一个实现了InvocationHandler接口的匿名类C，C为A的委托类，为B的代理类。最终A在外部对接口方法的调用会传递到C中，C再将调用传递给B。
在匿名类中
```
ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
```
这一段代码就是在代理类中进行的调用。我们进入这几个方法可以看到retrofit对于xxService的整个处理。
  查看loadService的代码
 ```
ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;
synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
```
在Retrofit类中维护了一个Map用来存放ServiceMethod，如果method对象已经存在就会使用已经存在的缓存对象，避免每次对象在代理类里面运行的开销。
进入ServiceMethod的build方法，我们可以看到retrofit对service的处理。我将service接口的处理过程梳理了一下
1. 创建CallAdapter
2. 创建处理返回数据格式的Converter
3. 解析方法上方的注解。例如@FormUrlEncoded、@Post
4. 新建参数处理对象ParamterHandler
5. 对service接口中的方法参数进行读取和解析，在这里，会对参数注解进行解析。

接下来我们对每一个步骤进行解析

###CallAdapter
service的build中会调用retrofit的callAdapter方法，继而调用nextCallAdapter方法。
```
int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
```
我们会取出retrofit的存放adapter的list中第一个不为空的adapter作为最后使用的adapter，并且根据returntype去除CallAdapter，结合动态代理部分可以看出，retrofit会把Call根据我们的CallAdapter转换为想要的类型。例如retrofit+rxjava使用的时候把Call转化为一个Observable。

### Converter
接下来会执行createResponseConverter方法，这个主要是解析返回的数据。在里面会调用retrofit的responseBodyConverter方法，类似callAdapter，接下来会调用nextResponseBodyConverter方法
```
int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
```
我们会拿到retrofit中存放converter的list中第一个获取到ConVerter对象不为空的作为方法的返回值。例如GsonResponseBodyConverter中就会把返回的response数据利用Gson解析为对应的实体类对象。
而且在retrofit的Build类的build方法中，在存adapter的list里面加入了默认的对象
```
adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
```
默认添加的是一个DefaultCallAdapterFactory对象
而converter的list则会默认添加一个BuiltInConverters对象
```
Builder(Platform platform) {
      this.platform = platform;
      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
    }
```

###parseMethodAnnotation
接下来会对retrofit的xxService接口进行方法注解的解析
截取一小段代码进行分析：
```
if (annotation instanceof POST) {
   parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
 }
```
其中最后一个参数的true和false是methodService中的hasBody变量
根据源码我们可以将方法注解分类如下：
1. Http方法 hasBody为false
  * delete
  * get
  * head
  * options
2. Http方法 hasBody为true
  * patch
  * post
  * put
3. 非http请求
  * Http
  * Headers
  * FormUrlEncoded
  * Multipart
其中FormUrlEncoded表示参数编码为表单，multipart为文件上传。2者互斥不得同时存在。
接下来会对方法参数的路径进行解析以及验证
如果方法注解为headers，则表示里面为http头信息，会单独进行解析。

###ParameterHandler
创建ParamterHandler对象，这个对象是用来处理参数的。在后面的request构建会使用到。其中，每一个method的注解都有它对应的ParamterHandler对象。

###parseParameter
在serviceMethod对象build前的最后一步是解析method的参数注解，会将读取到的数据传入ParamterHandler对象中，最终拼成http请求的参数


最后，在Retrofit的create中，我们会返回一个Call对象，这个对象实际在默认情况下是一个ExecutorCallbackCall对象，ExecutorCallbackCall对象内部包裹了一个OkHttpCall对象。

###OkHttpCall
OkHttpCall对象是实际上执行网络请求的类，这个对象实际是包装了OkHttp里面的Call对象。最终在enqueue等方法中会调用OkHttp的对应的方法。


####小结
陆陆续续看了一段时间的retrofit源码，虽然这个库的代码不多，但是设计思路非常的牛逼。我们应该多吸收这种优秀三方库的思想到我们自己的代码中