

# Retrofit源码分析

#### 工作流程:

##### 1.通过构造者模式,生成一个Retrofit对象

```
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://www.mxnzp.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .build();
```

Retrofit里会配置几个重要的参数:

DefaultCallAdapterFactory ,网络请求适配器工厂,最后发起网络请求会调用这个类的方法

GsonConverterFactory ,数据转换的适配器工厂,它会将返回的json数据序列号

##### 2.通过动态代理生成一个代理对象

```
public <T> T create(final Class<T> service) {
  validateServiceInterface(service);
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();
        private final Object[] emptyArgs = new Object[0];

        @Override public @Nullable Object invoke(Object proxy, Method method,
            @Nullable Object[] args) throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
        }
      });
}
```

通过动态代理,客户端在调用service方法时,实际调用Retrofit里面的loadServiceMethod().invoke()方法

在loadServiceMethod()里面干了几件事

```
ServiceMethod<?> loadServiceMethod(Method method) {
  ServiceMethod<?> result = serviceMethodCache.get(method);
  if (result != null) return result;

  synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
      result = ServiceMethod.parseAnnotations(this, method);
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```

```
abstract class ServiceMethod<T> {
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(method,
          "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }

  abstract @Nullable T invoke(Object[] args);
}
```

1.首先会解析注解信息,并将此信息保存在RequestFactory类中

2.调用`HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);`

```
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
  
  CallAdapter<ResponseT, ReturnT> callAdapter =
      createCallAdapter(retrofit, method, adapterType, annotations);
  
  Converter<ResponseBody, ResponseT> responseConverter =
      createResponseConverter(retrofit, method, responseType);

  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  if (!isKotlinSuspendFunction) {
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  } else if (continuationWantsResponse) {
    //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
    return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForResponse<>(requestFactory,
        callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
  } else {
    //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
    return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForBody<>(requestFactory,
        callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
        continuationBodyNullable);
  }
```



我们只看关键的代码,这一步会通过返回值类型去生成一个CallAdapter,并且将Retrofit初始化传入的converter和callAdapter传入,到这一步,loadServiceMethod方法执行完毕,返回一个CallAdapter对象,然后执行invoke()方法,invoke方法最终会调用到CallAdapter的adapte()方法里

```
@Override final @Nullable ReturnT invoke(Object[] args) {
  Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
  return adapt(call, args);
}
```

```
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
  private final CallAdapter<ResponseT, ReturnT> callAdapter;

  CallAdapted(RequestFactory requestFactory, okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter,
      CallAdapter<ResponseT, ReturnT> callAdapter) {
    super(requestFactory, callFactory, responseConverter);
    this.callAdapter = callAdapter;
  }

  @Override protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
    return callAdapter.adapt(call);
  }
}
```

Adapt()方法里面又会调用callAdapter.adapter()方法,这个callAdapter就是初始化Retrofit时的DefaultCallAdapterFactory,看它又做了些什么

```
@Override public @Nullable CallAdapter<?, ?> get(


  final Executor executor = Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
      ? null
      : callbackExecutor;

  return new CallAdapter<Object, Call<?>>() {
    @Override public Type responseType() {
      return responseType;
    }

    @Override public Call<Object> adapt(Call<Object> call) {
      return executor == null
          ? call
          : new ExecutorCallbackCall<>(executor, call);
    }
  };
}
```

实际上是返回了一个ExecutorCallbackCall对象,最后拿到这个对象会调用enqueue()方法发起异步请求

```
@Override public void enqueue(final Callback<T> callback) {
  Objects.requireNonNull(callback, "callback == null");

  delegate.enqueue(new Callback<T>() {
    @Override public void onResponse(Call<T> call, final Response<T> response) {
      callbackExecutor.execute(() -> {
        if (delegate.isCanceled()) {
          // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
          callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
        } else {
          callback.onResponse(ExecutorCallbackCall.this, response);
        }
      });
    }

    @Override public void onFailure(Call<T> call, final Throwable t) {
      callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
    }
  });
}
```

它会调用delegate.enqueue()方法,这个delegate就是OKHttpCall,而它的enqueue()方法实际上就是调用的Okhttp3.Call的方法,最终完成一次网络请求的实现





