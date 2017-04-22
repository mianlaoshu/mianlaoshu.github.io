# 可观测数据源
RxJava2.0的数据源主要有以下几种
* Flowable
* Observable
* Single
* Maybe
* Completable

它们之间的区别大家应该有所了解，比如Flowable支持背压，Observable不支持，Single只能发射一个数据或通知错误，Maybe则是最多一个，Completable只有完成通知而无数据等。他们分别都有着自己的基础接口
```Java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
interface ObservableSource<T> {
    void subscribe(Observer<? super T> observer);
}

interface SingleSource<T> {
    void subscribe(SingleObserver<? super T> observer);
}

interface CompletableSource {
    void subscribe(CompletableObserver observer);
}

interface MaybeSource<T> {
    void subscribe(MaybeObserver<? super T> observer);
}
```
XXXSource就是相应XXX数据源的接口，它们的参数也就是我们后面所要说的观察者的基础接口们。

但是，等等，还有一个Publisher看着和其他人不一样呢？
是的Publisher是Reactive-Stream库中的四个接口之一，这几个接口是Reactive规范定义的，其他的Rx库如RxJS，RxScala等都遵循这几个基础接口，便于沟通交互。

我们从源代码分析下如下最简单的RxJava代码执行过程
```Java
Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> e) throws Exception {
        e.onNext("a");
        e.onComplete();
    }
}).subscribe(new Observer<String>() {
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


## 创建数据源
我们以Observable分析创建的源代码，其他几个从代码形式和框架上都非常类似，只是具体实现不同（不是接口-实现的那种**实现**哦）

我们可以把数据源看作一个黑盒子，具体是从哪创建数据，如何创建数据我们都不关心，****我们只关心它发射的数据和发射的规则，然后做出响应。这是响应式编程的核心****。
### 1. 创建数据源的方法
创建意味着无中生有，了解Observable的同学知道，创建数据源的方法多种多样，如：
* create
* from
* just
* range
* repeat
* start
* timer
* interval
* defer
* empty/never/throw

这些是创建Observable的方法，实际上也是Operator。下面分析源代码将会了解到，创建数据源的Operator由于没有上游可观测数据源，它们在实现上与普通Operator还是有一些不同的。

另外需要注意的是，并不是所有列在这的Operator其它几种数据源必须都要有，如Single是没有range方法的，因为range对于Single毫无意义。

另外需要说明的一点是，虽然每种数据源都有诸如create的方法，但是他们的参数并不相同，也没有公共的接口，并不能再抽象一层公共的方法/层次。

### 2. 关于Observable
让我们先来分析下Observable本身。按照上面所说，它实现了ObservableSource唯一的方法`void subscribe(Observer<? super T> observer);`，其余长篇累牍的都是自己新加的方法，如刚才列出来的创建Observable的几个方法（静态工厂方法），其他RxJava定义的各种Operator也在Observable可以看到（非静态final方法，不允许修改这些操作符的行为）。此时，我们只需关注下实现的subscribe接口
```Java
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);
        ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");
        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        RxJavaPlugins.onError(e);
        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
```
抛开一些细节，如RxJavaPlugins的hook，异常处理等，其实质是简单调用subscribeActual方法
```Java
protected abstract void subscribeActual(Observer<? super T> observer);
```
上面就是Observable定义的框架核心代码，其余的部分都是各个功能函数

下面以create为例子剖析下源代码

### 3. create方法
create方法主要涉及到如下几个类
* ObservableOnSubscribe
* ObservableCreate
* CreateEmitter/ObservableEmitter/Emitter
* Disposable/DisposableHelper

首先看create代码
```Java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```
ObjectHelper.requireNonNull只是用于检查参数合法性。RxJavaPlugins是hook，在绝大多数情况下，可以忽略它。因此这个方法其实质就是创建了一个Observable:ObservableCreate对象。

这里还有一个ObservableOnSubscribe，这个接口是暴露给开发者的接口之一
```Java
public interface ObservableOnSubscribe<T> {
    void subscribe(ObservableEmitter<T> e) throws Exception;
}
```
大家应该比较熟悉，当我们调用subscribe方法监听时，就会进入到ObservableOnSubscribe的subscribe方法，开发者在这里就可以添加业务逻辑获取数据，而这里的参数ObservableEmitter我们稍后就会看到，它其实是封装了开发者的Observer（准确来说是直接下游的Observer）。那么，开发者调用subscibe是怎么触发到这里的呢？开发者的Observer是如何保存的呢？我们如何处理下游的取消事件呢？（Observable不支持背压，后面等看到这部分源代码再记录）

首先我们先回到这部分的开始，介绍下CreateEmitter/ObservableEmitter/Emitter
#### 关于ObservableEmitter和Emitter
关于Emitter接口，初一看和Observer有点类似
```Java
public interface Emitter<T> {
    void onNext(@NonNull T value);
    void onError(@NonNull Throwable error);
    void onComplete();
}
```

Observer接口
```Java
public interface Observer<T> {
    void onSubscribe(Disposable d);
    void onNext(T t);
    void onError(Throwable e);
    void onComplete();
}
```

就差个onSubscribe方法。那源码中直接使用Observer不就得了，干嘛还要多了一堆Emitter？而且确实，在1.x时代，开发者调用subscibe时的观察者（1.x是Subscriber，2.x是Observer了）是直接给了Observable。2.x出于什么样的考虑创建Emitter系列呢？
在这里[Emitter vs. Observer - what is the difference?](https://github.com/ReactiveX/RxJava/issues/4787)中回答了该问题，就是处于安全的考虑，不希望ObservableOnSubscribe能够调用到Observer的onSubscribe方法，否则这就违反了Rx的协议了。如果所有的代码都是我们自己实现，这可能不是个问题，但是如果我们使用第三方的Rx库，理论上就会存在滥用的可能性。因此，用一个和1.x Observer非常类似的Emitter可以避免滥用。

Emitter的注释写的很明白：`Base interface for emitting signals in a push-fashion in various generator-like source operators (create, generate).`，因此只有上述创建类型的operators才会见到Emitter。

而ObservableEmitter接口
```Java
public interface ObservableEmitter<T> extends Emitter<T> {
    void setDisposable(Disposable d);
    void setCancellable(Cancellable c);
    boolean isDisposed();
    ObservableEmitter<T> serialize();
}
```
加了几个关于取消的行为接口，而CreateEmitter则实现了这个接口，封装了下游Observer，同时还实现了Dispose接口，因此下游的观察者可以用该对象取消请求等。关于这块等后面看了取消订阅相关代码后再单独写这块，感觉还是有点复杂的。

现在让我们来分析开始时创建的那个ObservableCreate对象
#### ObservableCreate
```Java
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

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
    // 下面的省略
}
```
刚才讲了，它是用下游传递过来的ObservableCreate创建的Observable，根据刚才的说明，正常情况下下游订阅实际就是走到subscribeActual中。在这里，创建了只是简单创建了一个CreateEmitter封装了下游的Observer，然后调用Observer的onSubscribe方法（CreateEmitter实现了Disposable接口），然后调用ObservableOnSubscribe（开发者创建Observable时创建的）的subscribe方法，这样，就走到了开发者的业务逻辑，开发者通过ObservableEmitter（CreateEmitter实现的 接口）完成数据流的发射和通知。

CreateEmitter也比较直白
```Java
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
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
正常状态下，onNext, onComplete, onError直接转发给它封装的下游Observer了。它起到的作用主要有
* 拦截dispose后的请求，按照Rx规范，取消订阅后不应再收到/发送任何通知，否则应该抛出异常
* 处理dispose取消订阅。

所以从功能上，Emitter充当了内部的Observer的角色

所以，如果没有subscribeOn/observeOn发生线程切换的话，create订阅调用过程将会如下：
1. 开发者创建ObservableOnSubscribe实例，创建Observable对象（ObservableCreate实例，封装了ObservableOnSubscribe实例）
2. 创建Observer，调用第一步ObservableCreate实例的subscribe方法，Observer作为参数
3. 进入到ObservableCreate对象的subscribeActual方法，Observer作为参数
4. 在ObservableCreate对象的subscribeActual方法中，创建CreateEmitter对象，封装上面传进来的Observer
5. 在ObservableCreate对象的subscribeActual方法中，调用Observer的onSubscribe方法。此时开发者的第一个方法得到调用
6. 在ObservableCreate对象的subscribeActual方法中，调用ObservableOnSubscribe（ObservableCreate对象将其封装为成员变量）的onSubscribe方法，开发者第二个方法得到调用。这里进入开发者的业务逻辑，开始产生数据<br/>
以上步骤在没有subscribeOn/observeOn是同步进行的，下面的步骤同步异步视开发者业务逻辑<br/>
7. 开发者根据情况调用ObservableEmitter（ObservableOnSubscribe的参数）的onNext*, (onComplete | onError)
8. 在create创建的Observable中，第7步调用的是CreateEmitter的方法。
9. CreateEmitter依次转发调用之前封装的开发者Observer的方法，开发者收到数据

上面总结的是最简单的一个RxJava调用过程，这个demo没有任何意义，实际使用也会有一些问题。后面我们会说明。
