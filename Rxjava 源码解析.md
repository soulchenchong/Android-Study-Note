## Rxjava 源码解析

#### 一.解析内容

1.从源码角度分析Rxjava的调用流程

2.Rxjava是如何做到线程切换的

#### 调用流程

```
Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("1");
    }
})
        .map(new Function<String, String>() {
            @Override
            public String apply(String s) throws Exception {
                return s;
            }
        })
        .subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                
            }

            @Override
            public void onNext(String s) {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onComplete() {

            }
        });
```

整个Rxjava的调用流程采用一层一层包装的装饰模式,在调用.subscribe()时开始拆包装

- **Observable.create** 

```
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

很简单,返回一个ObservableCreate对象,看看ObservableCreate构造函数

```
public ObservableCreate(ObservableOnSubscribe<T> source) {
    this.source = source;
}
```

也很简单,只是把我们方法中的source赋值给成员变量,注意这个source是ObservableOnSubscribe类型

- **.map()** 

```
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```

同样的,返回了一个ObservableMap对象,注意这个this,就是前一个ObservableCreate对象,mapper就是我们传入的Function类型

```
public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
    super(source);
    this.function = function;
}
```

也是讲source赋值给成员变量,只不过这个source是ObservableCreate对象

- **.subscribe()** 

  ```
  @SchedulerSupport(SchedulerSupport.NONE)
  @Override
  public final void subscribe(Observer<? super T> observer) {
      ObjectHelper.requireNonNull(observer, "observer is null");
      try {
          observer = RxJavaPlugins.onSubscribe(this, observer);
  
          ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");
  
          subscribeActual(observer);
      } catch (NullPointerException e) { // NOPMD
          throw e;
      } catch (Throwable e) {
          Exceptions.throwIfFatal(e);
          // can't call onError because no way to know if a Disposable has been set or not
          // can't call onSubscribe because the call might have set a Subscription already
          RxJavaPlugins.onError(e);
  
          NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
          npe.initCause(e);
          throw npe;
      }
  }
  ```

看主要的方法`subscribeActual(observer);` 这是**Observer**的抽象方法,实际是调用的**ObservableMap** 的subscribeActual()方法

```
@Override
public void subscribeActual(Observer<? super U> t) {
    source.subscribe(new MapObserver<T, U>(t, function));
}
```

注意,自从调用.subscribe()方法后,就开始拆包装和包装过程

拆包装-- 是指拆从.create()方法一路下来的包装,拆Observable的包装

包装----- 是指把Observer一层一层包装起来,送到Observale上游,有包装就有拆包装,后面再分析

还是看上面的方法,在**ObservableMap** 的`subscribeActual(observer);`方法里,首先是将Observer包装成一个MapObserver,看看这个MapObserver的构造

```
MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
    super(actual);
    this.mapper = mapper;
}
```

好,先看到这,我们先记住Observer被包装成MapObserver就行,接着看subscribeActual(observer)的方法,会调用

`source.subscribe(new MapObserver<T, U>(t, function))` 这个source如果你还记得,就是上面我们说的**ObservableCreate**对象,所以下一步,我们要看看ObservableCreate.subscribe()方法做了什么

```
@Override
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);

    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```

实际上会调到这个方法,还是一样的套路,这个方法第一步又会把传进来的MapObserver包装成CreateEmitter,然后调用`observer.onSubscribe(parent);` 从这里可以看到只从.subscribe一调用,首先就是调用的observer.onSubscribe()方法

接着又会调用source.subscribe(parent);这个source就是.create()方法传进来的ObservableOnSubscribe,

![image-20200602161453971](/Users/chochen/Library/Application Support/typora-user-images/image-20200602161453971.png)

也就是会调用到这个方法,我们可以看到我们会拿到emitter.onNext().而这个emitter就是CreateEmitter,看看他的onNext()方法,注意现在又开始一层一层的拆包装了,这次是Observer的包装

```
@Override
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    if (!isDisposed()) {
        observer.onNext(t);
    }
}
```

调用的是observer.onNext(),这个observer就是MapObserver,我们接着看MapObserver.onNext()

```
@Override
public void onNext(T t) {
    if (done) {
        return;
    }

    if (sourceMode != NONE) {
        downstream.onNext(null);
        return;
    }

    U v;

    try {
        v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
    } catch (Throwable ex) {
        fail(ex);
        return;
    }
    downstream.onNext(v);
}
```

只看最后一行,`downstream.onNext(v);` 这个downstream就是我们调用.subscribe(observer)中的最开始的observer,这样Rxjava的一个简单流程就走完了