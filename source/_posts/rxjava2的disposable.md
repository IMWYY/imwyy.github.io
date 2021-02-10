---
title: rxjava2的disposable
date: 2017-12-10
tags: Android
---

## rxjava+retrofit处理网络请求
在使用rxjava+retrofit处理网络请求的时候，一般会采用对观察者进行封装，实现代码复用和拓展。可以参考我的这篇文章：[rxjava2+retrofit封装处理网络请求全解析](https://blog.csdn.net/C_J33/article/details/80231348)。一种可行的封装如下：

- 基类observer

```java
public abstract class BaseObserver<T> implements Observer<T> {

    protected String errMsg = "";
    protected Disposable disposable;

    @Override
    public void onSubscribe(Disposable d) {
        disposable = d;
    }

    @Override
    public void onNext(T t) {}

    @Override
    public void onError(Throwable e) {
        LogUtils.d("Subscriber onError", e.getMessage());
        if (!NetworkUtils.isConnected()) {
            errMsg = "网络连接出错,";
        } else if (e instanceof APIException) {
            APIException exception = (APIException) e;
            errMsg = exception.getMessage() + ", ";
        } else if (e instanceof HttpException) {
            errMsg = "网络请求出错,";
        } else if (e instanceof IOException) {
            errMsg = "网络出错,";
        }

        if (disposable != null && !disposable.isDisposed()) {
            disposable.dispose();
        }
    }

    @Override
    public void onComplete() {
        if (disposable != null && !disposable.isDisposed()) {
            disposable.dispose();
        }
    }
}
```

- 封装请求（登录为例） 这里userService是retrofit接口类

```java
    public void login(String phone, String password, BaseObserver<ResponseBean<UidBean>> observer) {
        userService.login(phone,password)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(observer);
    }

```

- 方法调用

```java
APIUser.getInstance().login(phone, password, new BaseObserver<ResponseBean<UidBean>>() {
             @Override
             public void onNext(ResponseBean<UidBean> responseBean) {
                ToastUtils.showShort("登录成功");
             }
         });
```

关于rxjava和retrofit的详细封装，可以看我的这篇文章：[rxjava2+retrofit封装处理网络请求全解析](https://blog.csdn.net/C_J33/article/details/80231348)。

## 关于disposable

rxjava虽然好用，但是总所周知，容易遭层内存泄漏。也就说在订阅了事件后没有及时取阅，导致在activity或者fragment销毁后仍然占用着内存，无法释放。而disposable便是这个订阅事件，可以用来取消订阅。但是在什么时候取消订阅呢？我知道有两种方式:

- 使用CompositeDisposable

看源码，CompositeDisposable的介绍很简单

>  A disposable container that can hold onto multiple other disposables and offers O(1) add and removal complexity.

一个disposable的容器，可以容纳多个disposable，添加和去除的复杂度为O(1)。
这里需要注意的是在该类的`addAll`方法有这么一句注释

> Atomically adds the given array of Disposables to the container or disposes them all if the container has been disposed

也就是说，如果这个CompositeDisposable容器已经是处于dispose的状态，那么所有加进来的disposable都会被自动切断。

所以说可以创建一个`BaseActivity`，用CompositeDisposable来管理订阅事件disposable，然后在acivity销毁的时候，调用`compositeDisposable.dispose()`就可以切断所有订阅事件，防止内存泄漏。

-  在oError和onComplete后调用`disposable.dispose();`，也就是上面我给的例子中的方法。

查看源码，ObservableCreate的静态类CreateEmitter就是这种方式实现的。同时也可以看到，onError和onComplete不可以同时调用的原因：每次掉用过onError或onComplete其中一个方法后，就会掉用`dispose()`方法，此时订阅取消，自然也就不能掉用另一个方法了

```java
static final class CreateEmitter<T>
implements ObservableEmitter<T>, Disposable {

    private static final long serialVersionUID = -3434801548987643227L;

    final Observer<? super T> observer;

    CreateEmitter(Observer<? super T> observer) {
        this.observer = observer;
    }

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

    @Override
    public void onError(Throwable t) {
        if (t == null) {
            t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
        }
        if (!isDisposed()) {
            try {
                observer.onError(t);
            } finally {
                dispose();
            }
        } else {
            RxJavaPlugins.onError(t);
        }
    }

    @Override
    public void onComplete() {
        if (!isDisposed()) {
            try {
                observer.onComplete();
            } finally {
                dispose();
            }
        }
    }

    @Override
    public void setDisposable(Disposable d) {
        DisposableHelper.set(this, d);
    }

    @Override
    public void setCancellable(Cancellable c) {
        setDisposable(new CancellableDisposable(c));
    }

    @Override
    public ObservableEmitter<T> serialize() {
        return new SerializedEmitter<T>(this);
    }

    @Override
    public void dispose() {
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return DisposableHelper.isDisposed(get());
    }
}
```


除此之外，在github发现一个开源库[RxLifecyclee](https://github.com/trello/RxLifecycle/tree/2.x)，粗略了解发现他实现的原理是绑定acvitvity是生命周期，在onStart中绑定就在onStop中解绑，其他onResume，onCreate同理。这个和第一种方式似乎又差不多，只不过第一种方式简单，只在ondestory的时候销毁所有事件。

所以那两种方法哪种更好，我也不是很清楚。等到踩到什么坑了可能就知道了。
如果某位大佬知道，希望不吝指教。