---
title: 细说Android下的多线程
data: 2020-02-21
tags:
	- Android
---

# 一、多线程是什么？我为什么要用多线程？

## 1.1 线程和进程的概念
**按照操作系统中的描述，线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用。一个进程可以包含多个线程。**

简单点理解，一个Android APP就是一个进程，一个APP里面有多个线程，我们多线程编程的意义就是实现“一个APP多个线程”。

有杠精可能会问，那我可不可以一个APP多个进程？又可不可以一个进程只有一个线程？

我告诉你，可以，都可以。

单线程的APP只包括Android的UI线程也是能运行的；一个APP多个进程也是可以达到的，实现方式涉及到Android的IPC机制，这里不细说。

## 1.2 为什么要使用多线程？

这里杠精可能会说，那你单线程也能跑，我为啥还要整多线程？

我告诉你，首先这句话从Android开发的角度来讲，近似于一个假命题。因为谷歌爸爸现在强制规定了不能在UI线程进行耗时操作，必须放到子线程里面去，除非你的程序不涉及耗时操作。究其原因，是因为在UI线程进行耗时操作的话，给用户的使用体验就是界面“卡顿”。同时，如果UI线程被阻塞超过一定时间会触发ANR(Application Not Responding)错误。

从底层的角度来讲，多线程可以使得整个环境能够异步执行，这有助于防止浪费CPU时钟周期从而提高效率。换言之，多线程能更充分的利用CPU资源，从而提高程序的运行效率。

# 二、那我怎么进行多线程编程？
## 2.1 Thread类和Runnable接口
要想定义一个线程只需要新建一个类继承自Thread，然后重写父类的run方法即可

```
class MyThread extends Thread {
    @Override
    public void run() {
        doSomething();
    }
}

//在需要的时候启动线程
new MyThread().start();
```

优化一下？

我们可以没必要继承整个Thread类，只实现Runnable接口就好了
```
class MyThread implements Runnable {
    @Override
    public void run() {
        doSomething()
    }
}

//启动线程
MyThread myThread = new MyThread();
new Thread(myThread).start();
```
那我不想专门再写一个线程类怎么办？可以使用匿名类
```
new Thread(new Runnable() {
    @Override
    public void run() {
        doSomething();
    }
}).start();
```

## 2.2 线程池
### 2.2.1 线程池的意义
既然我都会用Runnable接口来创建线程了，还要线程池干啥？其实不然，随意创建线程的操作在实际开发中是极为不推荐的。为啥？因为线程也是一种资源，反复的创建和销毁线程会带来一定性能上的额外开销。与其相比，线程池主要有以下几个优点：

- **重用线程池中的线程，避免因为线程的创建和销毁所带来的性能开销**
- **能有效控制线程池的最大并发数，避免大量的线程之间因相互抢占系统资源而导致的阻塞现象**
- **能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行等功能**

### 2.2.2 线程池的结构和原理
一个完整的线程池应该有这么几个组成部分
- **核心线程**
- **任务队列**
- **非核心线程**

当我们通过线程池执行异步任务的时候，其实是依次进行了下面的流程
1. 检查核心线程数是否到达最大值，否则创建新的核心线程执行任务，是则进行下一步
2. 检查任务队列是否已满，否则将任务添加到任务队列中，是则进行下一步
3. 检查非核心线程数是否到达最大值，否则创建新的非核心线程执行任务，是则说明这个线程池已经饱和了，执行饱和策略。默认的饱和策略是抛出RejectedExecutionException异常

下面手搓一个线程池的实现
```
//CPU核心数
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
//核心线程数
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
//最大线程数
private static final int MAX_POOL_SIZE = CPU_COUNT * 2 + 1;
//非核心线程闲置的超时时间
private static final int KEEP_ALIVE_TIME = 1;
//任务队列
private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);
//线程池
private ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,
        MAX_POOL_SIZE, KEEP_ALIVE_TIME, TimeUnit.SECONDS, sPoolWorkQueue);

private void fun(){
    Runnable runnable = new Runnable() {
        @Override
        public void run() {
            //子线程处理耗时操作
            doSomething();
        }
    };
    poolExecutor.execute(runnable);
}
```

这样我们就实现了一个简单的线程池，核心线程数为CPU数量+1，非核心线程数为CPU数量*2+1，非核心线程的闲置时间为1秒，任务队列的大小为128。

线程池还有具体的好几种分类和相应不同的实现方式，这里不再细说。

## 2.3 Handler

有朋友可能会说，你讲的这些都是Java多线程里面的东西，能不能整点咱Android特有的？OK，现在进入专业时间。

Handler是Android提供的一种异步消息处理机制，要学会使用Handler我们首先来了解下消息处理四兄弟：

- **Message**
- **Handler**
- **MessageQueue**
- **Looper**

Handler可以帮助我们实现在不同的线程之间传递消息，这里的Message就是消息本体，也就是我们想要传递的那个东西。

Handler在这里扮演的角色是消息处理者，它的主要作用是发送和处理消息。

MessageQueue是一个消息队列，Handler发送过来的消息会放在这个队列里面，每个线程只会有一个MessageQueue对象。

Looper是线程中消息队列的管家，它会无限循环运行，每发现MessageQueue中存在一条消息，它就会把消息取出然后发送给Handler。每一个线程也只能有一个Looper对象。

好了，基本原理已经了解，现在我们来反手搓一个Handler

```
private static final int FLAG = 1;

private Handler mHandler = new Handler(){
    @Override
    public void handleMessage(@NonNull Message msg) {
        if (FLAG == msg.what){
            //这里已经回到主线程了
            doSomething();
        }
    }
};

private void fun(){
    new Thread(new Runnable() {
        @Override
        public void run() {
            //子线程发送消息
            Message message = new Message();
            message.what = FLAG;
            mHandler.sendMessage(message);
        }
    }).start();
}
```

## 2.4 AsyncTask

除了Handler以外，谷歌爸爸还给我们提供AsyncTask来进行线程的切换。AsyncTask是一种轻量级的异步任务，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递给主线程。从实现原理上来讲，AsyncTask是对Thread和Handle的再次封装。

AsyncTask本身是一个抽象的泛型类，有四个亲儿子：

- **onPreExecute()**
- **doInBackground(Params...params)**
- **onProgressUpdate(Progress...values)**
- **onPostExecute(Result result)**

最先执行的是方法是onPreExecute()方法，位于主线程中，一般用来做一些准备工作。

然后执行doInBackground()方法，位于线程池中，用来执行异步任务，params表示异步任务的输入参数。这个方法需要返回结果给onPostExecute()方法。

onProgressUpdate()方法在主线程中执行，当后台任务的执行进度发生变化时这个方法会被调用。

onPostExecute()方法在最后异步任务完成之后会被调用，位于主线程中，result参数是后台任务的返回值，即doInBackground()的返回值。

OK，基本原理已经了解了，现在我们来手搓一个AsyncTask

```
class DownloadTask extends AsyncTask<Void,Integer,Boolean> {

    @Override
    protected void onPreExecute() {
        //这里我们使用了一个显示进度的Dialog，具体实现不表
        progressDialog.show();
    }

    @Override
    protected Boolean doInBackground(Void... voids) {
        try {
            while (true){
                //调用我们的doDownload下载方法，具体实现不表
                int downloadPercent = doDownload();
                //使用publishProgress方法来更新执行的进度
                publishProgress(downloadPercent);
                if (downloadPercent >= 100)
                    break;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return true;
    }

    @Override
    protected void onProgressUpdate(Integer... values) {
        //更新下载进度
        progressDialog.setMessage("Download "+values[0]+"%");
    }

    @Override
    protected void onPostExecute(Boolean aBoolean) {
        //下载完成
        progressDialog.dismiss();
    }
}
```

这里我们创建了一个Download类继承自AsyncTask，有三个泛型，void表示不需要给后台任务传入参数，Integer表示用整数类型来作为进度显示的单位，Boolean表示用布尔类型来反馈后台任务的执行结果。

要让我们的这个AsyncTask跑起来也很简单，只需要执行：

`new DownloadTask().execute();`

## 2.5 IntentService

IntentService是一种特殊的Service，它继承了Service并且是一个抽象类，我们可以创建它的子类来使用。IntentService也可以用于执行后台的耗时任务，并且当任务执行完毕之后它会自动停止。

IntentService因为是服务的原因，所以和单纯的线程相比它的优先级要高很多，从而更不容易被系统杀死。

IntentService的内部实现是封装了HandlerThread和Handler，使用的话要遵循Service的使用方法，这里先略过后面有机会在Service的专栏里面再详细介绍。

## 2.6 RxJava

**有杠精可能会说，你讲的这些方法，一个比一个长，一个比一个复杂，就不能整个简单又粗暴的东西？**

这个时候就需要祭出神兵利器RxJava了。

### 2.6.1 RxJava又是个啥？

其实网络上RxJava的入门文章多如过江之鲫，这里不打算过多的深入介绍。RxJava是一种响应式编程，大家不是很明白的话可以粗暴的理解为更优雅的多线程实现即可。

### 2.6.2 那怎么操作RxJava？

先手搓一个RxJava的普通实现方式

```
private void fun(){
    Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
        @Override
        public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
            emitter.onNext(1);
        }
    });

    observable.subscribeOn(Schedulers.io())     //表示在io线程执行订阅
            .observeOn(AndroidSchedulers.mainThread())  //表示在主线程接收订阅
            .subscribe(new Observer<Integer>() {
                @Override
                public void onSubscribe(Disposable d) {
                    //接收订阅之前调用
                }

                @Override
                public void onNext(Integer integer) {
                    //接收订阅成功调用
                    doSomething();
                }

                @Override
                public void onError(Throwable e) {
                    //接收订阅出错调用
                }

                @Override
                public void onComplete() {
                    //接收订阅完成调用
                }
            });
}
```

emmmmm看起来好像还是挺复杂的啊，能不能再整简单点？

OK，链式调用加lambda安排上

```
private void fun() {
    Observable.create(emitter -> emitter.onNext(1))
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(integer -> {
                //接收订阅成功
                doSomething();
            }, throwable -> {});
}
```

嗯......有内味了。

这串代码我们是发送了一个Integer类型的数据；

subscribeOn()指定了我们发送的线程是在后台的io线程，就可以理解为一个子线程；

observeOn指定了我们接收的线程为主线程；

subscribe只接收成功的消息，相当于上面的OnNext()方法，本质上是我们在这里创建了一个Comsumer对象来接收；

throwable在接收失败的时候调用，相当于上面的onError()方法。

**RxJava有多达几十种的操作符，灵活运用能实现各种不同的异步任务，这里就不再花大量的篇幅详细介绍了，有兴趣的朋友可以去查看[ReactiveX中文文档](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/From.html)**

## 2.7 RxKotlin

RxKotlin可以理解为RxJava在Kotlin上的一个变种，原理都是一样的，只是操作语言变成了Kotlin，然后封装了一下使得可以更优雅的调用，这里给大家一个具体的实现案例，不再过多讲解。

```
private fun test() {
    Observable.create<Int> { 1 }
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribeBy(
                    onNext = {},
                    onError = {}
            )
}
```

## 2.8 Kotlin协程

协程其实和上面所说的线程并不是一个概念，协程是什么？根据官方文档的描述，协程本质上是轻量级的线程。既然是轻量，那说明协程的资源消耗和性能等方面和线程比起来应该是有优势的。那这样看来我们以前使用多线程实现的异步功能，现在基本上都可以用协程来替代了。

协程是一个全新的东西，介于篇幅这里就不展开讲解了，后面会专门写介绍协程的文章。

# 三、总结
今天总结了Android平台上实现多线程的几种方式，希望能给到需要的朋友一些帮助。

有不对之处还望各位同僚雅正。

## 参考资料：

__1. 《Java完全参考手册 第八版》__

__2. 《第一行代码 第二版》__

__3. 《Android进阶之光》__

__4. 《Android开发艺术探究》__

__5. [ReactiveX中文文档](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/From.html)__