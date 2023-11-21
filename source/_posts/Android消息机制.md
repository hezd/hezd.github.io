---
title: Android消息机制
date: 2021-11-08 20:22:37
tags: Handler
categories: 
- Android应用层
- 消息机制
#description: " "
---

### 消息机制
主线程和子线程通信
消息机制涉及到三个角色，Handler、MessageQueue、Looper

##### 基本实现

这里只介绍主线程handler创建方式，子线程后续源码部分在介绍

- 创建Handler，重写handleMessage方法
- Handler发送消息
- handleMessage接收消息并处理

##### 基本原理
Handler发送消息并最终添加到MessageQueue，Looper调用loop方法后开始轮询从MessageQueue中获取消息并调用对应的handler的dispatchMessage方法。

##### 疑问
- Handler如何发送消息的
- Looper是如何初始化，如何获取消息的
- 消息机制如何实现线程间如何切换的
- Looper死循环为什么不会导致anr
- Linux的epoll机制

带着疑问我们通过下面源码部分去解惑

##### 源码解析
- Handler到底是如何发送消息的
首先来看Hander创建
```java
    public Handler(Callback callback, boolean async) {
       //... 省略部分代码

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
创建时会调用Looper.myLooper()获取当前线程的Looper对象
```java
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
}
```
这里用到了ThreadLocal线程局部变量,内部维护着一个ThreadLocalMap以map形式存储局部变量，key是当前线程value是局部变量。这样就实现了线程之间数据隔离。避免数据冲突。

	创建完Handler以后继续看发送消息SendMessage
```java
 public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```
```java
 public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```
```java
  public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
sendMessage会调用sendMessageDelayed紧接着又调用SendMessageAtTime，最终会调用enqueueMessage，将消息添加到消息队列。这里消息队列是哪里来的呢，在我们最开始Handler初始化时有一行代码mQueue = mLooper.mQueue;原来是来自Looper的MessageQueue，具体它的创建我们在后面Looper初始化在详解。
接下来在看enqueueMessage如何将消息添加到消息队列
```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
Handler的enqueueMessage做了两件事情，第一是将当前handler引用赋值给message的target建立绑定关系以便于后面消息分发处理的时候知道要处理的消息属于哪个Handler。第二件事情就是调用MessageQueue的enqueueMessage
```java
 boolean enqueueMessage(Message msg, long when) {
        //...忽略部分代码

        synchronized (this) {
           //... 忽略部分代码

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
MessageQueue的enqueueMessage中会判断当前message是否需要立即执行如果需要就添加到表头，否则就根据时间轮询比对添加到中间位置。
到这里Handler创建和消息发送以及添加到消息队列过程就很清晰了，接下来要看Looper的创建和消息如何轮询分发了。
- Looper是如何初始化，如何获取消息的
这里又要回到Handler的初始化，在Handler初始化的时候回获取当前线程的Looper对象
```java
public Handler(Callback callback, boolean async) {
      //...忽略部分代码
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
这里可以看到如果获取到的Looper对象为空就会抛出异常，所以Handler初始化之前必须要先初始化Looper，但是我们平时主线程并没有手动创建为什么不会报错呢？这是因为在app启动入口ActivityThread的main方法中系统已经为我们做好了主线程的Looper初始化
```java
public static void main(String[] args) {
     //...忽略部分代码
        Looper.prepareMainLooper();
	//...忽略部分代码
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
那Looper到底是如何初始化的呢，接着看Looper的prepareMainLooper
```java
  public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
prepareMainLooper又调用了prepare方法
```java
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
原来Looper的初始化是调用prepare方法,这里会创建looper对象并添加到ThreadLocal中，所以一个线程只能有一个Looper，并且prepare只能调用一次否则会报错，
那Looper是如何获取消息进行分发的呢，在上面ActivityThread代码中我们也可以看到Looper初始化完以后会调用Looper.loop()真相就在这里
```java
public static void loop() {
        final Looper me = myLooper();
     	//... 忽略部分代码
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

			//...忽略部分代码
            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
          //...忽略部分代码
            msg.recycleUnchecked();
        }
    }
```
我们可以看到loop方法轮询从MessageQueue中取出消息，判断如果需要立即执行就会调用meesage的target的dispatchtMessage方法进行消息分发处理。message的target就是上面提到的message所属的handler，因此会回调到Handler的dispatchMessage方法
```java
   public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
dispatchMessage会判断message的callback是否有设置，查看源码可以知道setCallBack是hide方法只能通过反射调用它，如果它为空继续判断mCallBack是否为空，mCallBack哪儿来的呢，在Handler初始化时候有一个参数就是是否设置callback
```java
public Handler(Callback callback, boolean async) {
        //...忽略部分代码
        mCallback = callback;
	//...忽略部分代码
    }
```
如果mCallBack有设置会直接调用CallBack的handleMessge方法否则继续调用handler的handlemessage方法，因为我们在初始化Handler的时候回重写handlemessage方法所以最终会回调到我们重写的handlemessage方法里。

- 消息机制是如何实现线程间切换的？
我们通常会使用handler在子线程发送消息，然后回调到主线程处理消息。那到底是如何实现线程切换的呢，我们通过前面的分析可以知道，一个完整的消息分发流程，应该是先调用Looper.prepare()初始化Looper和消息队列，然后调用Looper.loop()开启轮询。然后创建Handler在指定线程发送消息接受回调处理消息。那回调是在哪个线程呢，重点来了：因为消息分发是Looper通过轮询拿到Message的target也就是发送者Handler在调用Handler的dispatchMessage方法完成分发的，这个调用是在Looper中，所以调用者在哪个线程当前回调就是在哪个线程，所以Looper初始化时绑定的线程就是回调的线程。

- Looper死循环为什么不会导致anr
App本质上也是一个java程序，入口就是ActivityThread的main方法，如果main方法执行完程序就退出了，如何保证不退出就是写一个死循环，ActivityThread中初始化了Looper并调用了loop在loop方法中开启了一个死循环阻塞了主线程这样程序可以保证程序一直执行不会退出。几乎所有的GUI程序都是这么实现的。既然是死循环那么其他代码怎么运行，页面交互怎么处理呢？Android是基于事件驱动的，不管是页面刷新还是交互本质上都是事件，都会被封装成Message发送到MessageQueue由Looper进行分发处理的。ANR是什么，Application no responding应用无响应，为什么没响应，因为主线程做了好事操作，loop方法死循环也会阻塞主线程为什么不会anr，什么是响应，响应就是页面刷新，交互处理等，谁来响应，其实就是looper的loop方法，，主线程做了耗时操作会阻塞loop方法导致无法处理其他message所以导致anr。

- Linux的epoll机制
我们先来看下Message的next方法
```java
Message next() {
       //...省略部分代码
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            //重点关注这一行
            nativePollOnce(ptr, nextPollTimeoutMillis);

           //...省略部分代码
        }
    }
```
在MessageQueue的next方法中会调用本地方法nativePollOnce，它是以阻塞的方式从native层的MessageQueue中获取可用消息，也就是会进行休眠。当有可用消息时进行唤醒。Looper的休眠和唤醒的实现原理是Linux的epoll机制，Looper的休眠和唤醒是通过对文件可读事件监听来实现唤醒。
什么是Linux的epoll机制？
Linux的epoll机制可以简单理解为事件监听，当空闲的时候让出cpu进行休眠，当事件触发的时候会被唤醒开始处理任务。

参考：[看完这篇还不明白Handler你砍我](https://mp.weixin.qq.com/s/H8mHoYHyTe6oOaUwfYh6_g "看完这篇还不明白Handler你砍我")
