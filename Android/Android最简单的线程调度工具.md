### 1、背景介绍。

Android是多线程的系统，线程分为UI线程与工作线程。

UI线程：app的主要线程：主要负责UI的操作。工作线程：主要负责除了UI线程操作的一切线程。

#### 问题所在：

但是很多开发者觉得切换线程麻烦并且容易内存泄漏，所以就不怎么去切换线程操作，能在主线称进行的操作就不在工作线程中执行。但是这样会导致 一、UI线程工作繁重。二、并且系统流畅性差。三、不定时的anr。

#### 不使用工作线程来做非UI线程的工作的原因：

使用工作线程来做非UI线程的工作，需要开启线程，还要考虑到类被销毁后内存泄漏的风险。因此工作量增加，导致很多人不愿意这么做。

### 2、解决方案

站着巨人的肩膀上可以看的更远。

因为RxJava可以很好的实现线程间的调度。autodispose可以很好的维护生命周期变动时线程的停止，防止内存泄漏。但是RxJava用起来需要编写很多方法，用起来比较麻烦，因此，我在RxJava与autodispose的基础上封装了一些线程切换的方法。

1、接口类：需要用户实现并且实现自己代码逻辑。

2、线程池：提供线程。

3、线程工厂：负责调度线程，并且提供线程调度方法。



```java
//1、接口类，实现各种操作。
public interface TaskRunnable {
    public abstract void run() throws Exception;
}

```

```java
//2、线程池工具类
import java.util.concurrent.Executor;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPool {
    //参数初始化
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    //核心线程数量大小
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    private static final int MAX_POOL_SIZE = 1024;
    private static final int KEEP_ALIVE_TIME = 60;
    private static ThreadPoolExecutor executor = new ThreadPoolExecutor(
            CORE_POOL_SIZE,
            MAX_POOL_SIZE,
            KEEP_ALIVE_TIME,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<Runnable>()
    );

    public static void execute(Runnable command) {
        executor.execute(command);
    }
    public static Executor getExecutor() {
        return executor;
    }
}
```

```java
//3、线程调度工厂。
import android.os.Looper;
import androidx.lifecycle.LifecycleOwner;
import com.uber.autodispose.AutoDispose;
import com.uber.autodispose.android.lifecycle.AndroidLifecycleScopeProvider;
import java.util.concurrent.TimeUnit;
import io.reactivex.Observable;
import io.reactivex.ObservableEmitter;
import io.reactivex.ObservableOnSubscribe;
import io.reactivex.android.schedulers.AndroidSchedulers;
import io.reactivex.functions.Consumer;
import io.reactivex.schedulers.Schedulers;

public enum ThreadFactory {
    /**
     * 单例
     */
    INSTANCE;

    /**
     * 工作线程执行
     * 自动维护生命周期，建议主线称使用
     * @param runWork
     */
    public void postWorker(final LifecycleOwner owner, final TaskRunnable runWork){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runWork.run();
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(Schedulers.from(ThreadPool.getExecutor()))
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe();
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postWorker(owner, runWork);
                        }
                    });
        }
    }
    /**
     * UI线程执行
     * 自动维护生命周期，建议主线称使用
     * @param runUi
     */
    public void postUi(final LifecycleOwner owner, final TaskRunnable runUi){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runUi.run();
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(AndroidSchedulers.mainThread())
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe();
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postUi(owner, runUi);
                        }
                    });
        }
    }

    /**
     * single线程执行。
     * 自动维护生命周期，建议主线称使用
     * @param  runSingle
     */
    public void postSingle(final LifecycleOwner owner, final TaskRunnable runSingle){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runSingle.run();
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(Schedulers.single())
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe();
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postSingle(owner, runSingle);
                        }
                    });
        }
    }


    /**
     * 工作线程执行，UI线程执行。
     * runUi不会绑定生命周期不要执行耗时操作。
     * 自动维护runwork生命周期，建议主线称使用
     * @param runWork, runUi
     */
    public void postWorkerAndUi(final LifecycleOwner owner, final TaskRunnable runWork, final TaskRunnable runUi){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runWork.run();
                        emitter.onNext("");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(Schedulers.from(ThreadPool.getExecutor())).observeOn(AndroidSchedulers.mainThread())
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe(new Consumer<Object>() {
                        @Override
                        public void accept(Object integer) throws Exception {
                            runUi.run();
                        }
                    });
        }else {
            Observable.just(1)
                .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postWorkerAndUi(owner, runWork, runUi);
                        }
                    });
        }
    }

    /**
     * 工作线程执行，UI线程执行。
     * runUi不会绑定生命周期不要执行耗时操作。
     * 自动维护生命周期，建议主线称使用
     * @param runWork, runUi
     */
    public void postUiAndWorker(final LifecycleOwner owner, final TaskRunnable runUi, final TaskRunnable runWork){
        if (Looper.getMainLooper() == Looper.myLooper()){
            try {
                runUi.run();
            } catch (Exception e) {
                e.printStackTrace();
            }
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runWork.run();
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(Schedulers.from(ThreadPool.getExecutor()))
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe();
        }else {
            Observable.just(1)
                .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postUiAndWorker(owner, runUi, runWork);
                        }
                    });
        }
    }

    /**
     * 工作线程执行，single线程执行。
     * 自动维护生命周期，建议主线称使用
     * @param runWork, runSingle
     */
    public void postWorkerAndSingle(final LifecycleOwner owner, final TaskRunnable runWork, final TaskRunnable runSingle){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runWork.run();
                        postSingle(owner, runSingle);
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(Schedulers.from(ThreadPool.getExecutor()))
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe();
        }else {
            Observable.just(1)
                .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postWorkerAndSingle(owner, runWork, runSingle);
                        }
                    });
        }
    }

    /**
     * 工作线程执行，single线程执行。
     * 自动维护生命周期，建议主线称使用
     * @param runWork, runUi
     */
    public void postSingleAndWorker(final LifecycleOwner owner, final TaskRunnable runSingle, final TaskRunnable runWork){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runSingle.run();
                        postWorker(owner, runWork);
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(Schedulers.single())
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe();
        }else {
            Observable.just(1)
                .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postSingleAndWorker(owner, runSingle, runWork);
                        }
                    });
        }
    }

    /**
     * single线程执行，UI线程执行。
     * runUi不会绑定生命周期不要执行耗时操作。
     * 自动维护生命周期，建议主线称使用
     * @param runSingle, runUi
     */
    public void postSingleAndUi(final LifecycleOwner owner, final TaskRunnable runSingle, final TaskRunnable runUi){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runSingle.run();
                        emitter.onNext("");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(Schedulers.single())
                    .observeOn(AndroidSchedulers.mainThread())
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe(new Consumer<Object>() {
                        @Override
                        public void accept(Object integer) throws Exception {
                            runUi.run();
                        }
                    });
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postSingleAndUi(owner, runSingle, runUi);
                        }
                    });
        }
    }

    /**
     * UI线程执行, single线程执行。
     * runUi不会绑定生命周期不要执行耗时操作。
     * 自动维护生命周期，建议主线称使用
     * @param runSingle, runUi
     */
    public void postUiAndSingle(final LifecycleOwner owner, final TaskRunnable runUi, final TaskRunnable runSingle){
        if (Looper.getMainLooper() == Looper.myLooper()){
            try {
                runUi.run();
            } catch (Exception e) {
                e.printStackTrace();
            }
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runSingle.run();
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).subscribeOn(Schedulers.single())
                    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe();
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postUiAndSingle(owner, runUi, runSingle);
                        }
                    });
        }
    }

    /**
     * UI线程中执行
     * runUi不会绑定生命周期不要执行耗时操作。
     * 自动维护生命周期，建议主线称使用
     * @param runUi
     */
    public void postDelayedUI(final LifecycleOwner owner, final TaskRunnable runUi, final long delayMillis){
        if (Looper.getMainLooper() == Looper.myLooper()) {
            Observable.timer(delayMillis, TimeUnit.MILLISECONDS)
                    .observeOn(AndroidSchedulers.mainThread())
                    .as(AutoDispose.<Long>autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe(new Consumer<Long>() {
                        @Override
                        public void accept(Long aLong) throws Exception {
                            runUi.run();
                        }
                    });
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postDelayedUI(owner, runUi, delayMillis);
                        }
                    });
        }
    }

    /**
     * Worker线程中执行
     * 自动维护生命周期，建议主线称使用
     * @param runWork
     */
    public void postDelayedWorker(final LifecycleOwner owner, final TaskRunnable runWork, final long delayMillis){
        if (Looper.getMainLooper() == Looper.myLooper()) {
            Observable.timer(delayMillis, TimeUnit.MICROSECONDS)
                    .observeOn(Schedulers.from(ThreadPool.getExecutor()))
                    .as(AutoDispose.<Long>autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe(new Consumer<Long>() {
                        @Override
                        public void accept(Long aLong) throws Exception {
                            runWork.run();
                        }
                    });
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postDelayedWorker(owner, runWork, delayMillis);
                        }
                    });
        }
    }

    /**
     * Single线程中执行
     * 自动维护生命周期，建议主线称使用
     * @param runSingle
     */
    public void postDelayedSingle(final LifecycleOwner owner, final TaskRunnable runSingle, final long delayMillis){
        if (Looper.getMainLooper() == Looper.myLooper()) {
            Observable.timer(delayMillis, TimeUnit.MICROSECONDS)
                    .observeOn(Schedulers.single())
                    .as(AutoDispose.<Long>autoDisposable(AndroidLifecycleScopeProvider.from(owner)))
                    .subscribe(new Consumer<Long>() {
                        @Override
                        public void accept(Long aLong) throws Exception {
                            runSingle.run();
                        }
                    });
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postDelayedSingle(owner, runSingle, delayMillis);
                        }
                    });
        }
    }



    /**
     * 工作线程执行
     * 注意不要在single线程使用。
     * @param runWork
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postWorker(final TaskRunnable runWork){
        if (Looper.getMainLooper() != Looper.myLooper()){
            try {
                runWork.run();
            }catch (Exception e){
                e.printStackTrace();
            }
        }else {
            Observable.just(1)
                    .observeOn(Schedulers.from(ThreadPool.getExecutor()))
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postWorker(runWork);
                        }
                    });
        }
    }

    /**
     * UI线程执行
     * @param runUi
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postUi(final TaskRunnable runUi){
        if (Looper.getMainLooper() == Looper.myLooper()){
            try {
                runUi.run();
            }catch (Exception e){
                e.printStackTrace();
            }
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postUi(runUi);
                        }
                    });
        }
    }

    /**
     * single线程执行。
     * @param  runSingle
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postSingle(final TaskRunnable runSingle){
        Observable.just(1)
                .observeOn(Schedulers.single())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        runSingle.run();
                    }
                });
    }

    /**
     * 工作线程执行，UI线程执行。
     * 注意不要在single线程使用。
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runWork, runUi
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postWorkerAndUi(final TaskRunnable runWork, final TaskRunnable runUi){
        if (Looper.getMainLooper() != Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runWork.run();
                        emitter.onNext("");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Object>() {
                        @Override
                        public void accept(Object integer) throws Exception {
                            runUi.run();
                        }
                    });
        }else {
            Observable.just(1)
                    .observeOn(Schedulers.from(ThreadPool.getExecutor()))
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postWorkerAndUi(runWork, runUi);
                        }
                    });
        }
    }

    /**
     * 工作线程执行，UI线程执行。
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runWork, runUi
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postUiAndWorker(final TaskRunnable runUi, final TaskRunnable runWork){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runUi.run();
                        emitter.onNext("");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).observeOn(Schedulers.from(ThreadPool.getExecutor()))
                    .subscribe(new Consumer<Object>() {
                        @Override
                        public void accept(Object integer) throws Exception {
                            runWork.run();
                        }
                    });
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postUiAndWorker(runUi, runWork);
                        }
                    });
        }
    }

    /**
     * 工作线程执行，single线程执行。
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runWork, runUi
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postWorkerAndSingle(final TaskRunnable runWork, final TaskRunnable runSingle){
        Observable.create(new ObservableOnSubscribe<Object>() {
            @Override
            public void subscribe(ObservableEmitter<Object> emitter){
                try {
                    runWork.run();
                    emitter.onNext("");
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).subscribeOn(Schedulers.from(ThreadPool.getExecutor()))
                .observeOn(Schedulers.single())
                .subscribe(new Consumer<Object>() {
                    @Override
                    public void accept(Object integer) throws Exception {
                        runSingle.run();
                    }
                });
    }

    /**
     * 工作线程执行，single线程执行。
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runWork, runUi
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postSingleAndWorker(final TaskRunnable runSingle, final TaskRunnable runWork){
        Observable.create(new ObservableOnSubscribe<Object>() {
            @Override
            public void subscribe(ObservableEmitter<Object> emitter){
                try {
                    runSingle.run();
                    emitter.onNext("");
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).subscribeOn(Schedulers.single())
                .observeOn(Schedulers.from(ThreadPool.getExecutor()))
                .subscribe(new Consumer<Object>() {
                    @Override
                    public void accept(Object integer) throws Exception {
                        runWork.run();
                    }
                });
    }

    /**
     * single线程执行，UI线程执行。
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runSingle, runUi
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postSingleAndUi(final TaskRunnable runSingle, final TaskRunnable runUi){
        Observable.create(new ObservableOnSubscribe<Object>() {
            @Override
            public void subscribe(ObservableEmitter<Object> emitter){
                try {
                    runSingle.run();
                    emitter.onNext("");
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        }).subscribeOn(Schedulers.single())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<Object>() {
                    @Override
                    public void accept(Object integer) throws Exception {
                        runUi.run();
                    }
                });
    }

    /**
     * UI线程执行, single线程执行。
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runSingle, runUi
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postUiAndSingle(final TaskRunnable runUi, final TaskRunnable runSingle){
        if (Looper.getMainLooper() == Looper.myLooper()){
            Observable.create(new ObservableOnSubscribe<Object>() {
                @Override
                public void subscribe(ObservableEmitter<Object> emitter){
                    try {
                        runUi.run();
                        emitter.onNext("");
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }).observeOn(Schedulers.single())
                    .subscribe(new Consumer<Object>() {
                        @Override
                        public void accept(Object integer) throws Exception {
                            runSingle.run();
                        }
                    });
        }else {
            Observable.just(1)
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            postUiAndSingle(runUi, runSingle);
                        }
                    });
        }
    }

    /**
     * UI线程中执行
     * runUi期不要执行耗时操作。
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runUi
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postDelayedUI(final TaskRunnable runUi, final long delayMillis){
        Observable.timer(delayMillis, TimeUnit.MILLISECONDS)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        runUi.run();
                    }
                });
    }

    /**
     * Worker线程中执行
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runWork
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postDelayedWorker(final TaskRunnable runWork, final long delayMillis){
        Observable.timer(delayMillis, TimeUnit.MILLISECONDS)
                .observeOn(Schedulers.from(ThreadPool.getExecutor()))
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        runWork.run();
                    }
                });
    }

    /**
     * Single线程中执行
     * 不推荐使用，因为需要自己需要自己维护生命周期。
     * @param runSingle
     */
    @SuppressWarnings("不能绑定生命周期，不推荐使用")
    public void postDelayedSingle(final TaskRunnable runSingle, final long delayMillis){
        Observable.timer(delayMillis, TimeUnit.MICROSECONDS)
                .observeOn(Schedulers.single())
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        runSingle.run();
                    }
                });
    }
}
```

### 3、用法

```java
ThreadFactory.INSTANCE.postWorkerAndUi(ThreadMainActivity.this, new TaskRunnable() {
    @Override
    public void run() throws Exception{
        Log.d(TAG, "work4 run name : "+Thread.currentThread().getName());
        workSing();
    }
}, new TaskRunnable() {
    @Override
    public void run()throws Exception {
        Log.d(TAG, "UI4 run i = "+i+" j = "+j+" name : "+Thread.currentThread().getName());
        uiTask();

    }
});
```