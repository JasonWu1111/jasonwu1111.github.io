---
layout: post
title: Android 线程消息机制（一）—— Java 层消息模型
tags:
  - Android
  - Thread
---

## Android 线程消息机制模型
Android 的线程消息模型是系统中一个非常重要且核心的机制，它重新定义了 Android 内线程运作以及线程间通信的机制，它贯通了整个 framework 层（包括 java 和 native）里很多重要的线程运作机理。要想透彻的理解这套线程消息机制的原理，笔者认为可以先思考以下两个问题：

- 对于普通的线程来说，当线程中的可执行代码被执行完时，线程的生命周期便该终止退出了。而对于如应用进程的主线程等，我们希望它能一直持续性的运作下去。最直接的方式是一直死循环地执行代码，这样便能确保线程一直运作不会退出，但简单的死循环会特别消耗 CPU 资源，显然是不可取的，因此需要引入合适的机制来让线程在不需要工作时进入“休眠”。
   
- 线程中的代码执行是串行的，当别的线程希望通知到当前线程来处理“额外到来”的任务，比如 Android 系统常常子线程常常需要通知主线程来更新 UI 等操作，这就需要线程具备一个机制能缓存从别的线程过来的“消息”，在合适的时机来处理。

基于上面的一些考虑，Android 系统设计了一套巧妙的线程消息通信模型：
1. 在线程启动运行之后，创建一个线程特有的 **Looper** 对象来一直不停循环的执行，同时在 **Looper** 对象中维护一个消息队列 **MessageQueue**;
2. 线程通过 **Handler** 把每个需要处理的任务封装成了一个个的 **Message**，并把 **Message** 插入到目标线程的 **MessageQueue** 中；
3. **Looper** 则会一直循环地从 **MessageQueue** 中获取下一个要处理的 **Message**，若当前无需要处理的消息，则触发阻塞线程，释放 CPU 资源；
4. 当有新的 Message 需要处理时，则唤醒当前线程，**Looper** 继续运行，回调到目标 **Handler** 中完成消息的处理。

整体模型示例如下：
![](/img/posts/post-handler.png)

**Looper、MessageQueue、Message、Handler** 这四个核心的角色各自承担自己的职责，构成了 Android 线程消息机制模型。

**Looper、MessageQueue、Message、Handler** 这四个核心的角色各自承担自己的职责，构成了 Android 线程消息机制模型。

**这一套线程消息机制在 Android Framework 中不仅是在 java 层，在 native 层同样可以应用**。本文优先介绍在线程消息机制 java 层的实现与应用（这同时也是日常开发最常接触的部分）。[**Native 层的实现与原理**]() 将在下一篇中做介绍。

涉及到的源码文件有：

```light
frameworks/base/core/java/android/os/
    - Message.java
    - MessageQueue.java
    - Handler.java
    - Looper.java
```

## Message
我们先来看下 **Message** 类。

### 基本结构
Message 对象作为单个消息的载体，记录了一个消息的一些核心信息，示例如下：

```java
public final class Message implements Parcelable {
    public int what;
    public int arg1;
    public int arg2;
    ...
    public long when;
    ...
    Handler target;
    Runnable callback;
    ...
}
```

|成员变量|类型|含义
|--|--|--
|`what`|int|用于标识消息的 code，自定义
|`when`|long|此消息的目标传递时间
|`target`|Handler|消息传递的目标 Handler
|`callback`|Runnable|消息传递的回调（具体执行的任务）
|...|...|...

> 涉及到的源码路径为：
> - frameworks/base/core/java/android/os/Message.java

每个 Message 都会有一个预期的传递时间，默认为 0，也就是希望当下立即被传递。在某些情况下，我们希望这个 Message **在一段时间之后再被处理（延迟处理）**，那么就可以通过设置 `when` 的值来实现。

此外，Message 类设计成了**单向链表**的结构，示例如下：
```java
public final class Message {
    ...
    Message next;
    ...
}
```

### 异步消息
Message 还有另一个重要的特性为是否是“**异步**”消息。一个 Message 对象创建时默认为“**同步**”的消息，可以通过 `setAsynchronous(boolean)` 方法来设置当前的消息是否为“异步”消息，通过 `isAsynchronous()` 方法来获取当前消息是否为“异步的”：
```java
public final class Message {

    static final int FLAG_ASYNCHRONOUS = 1 << 1;
    ...
    int flags;
    ...

    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
    }

    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
}
```

根据消息的“异步”性质，可以将消息分为以下：
- **同步消息**
- **异步消息**
- **同步屏障消息**

这里所谓的同步屏障消息，其实是一个 `target` 为空的消息，这类型消息不会携带具体要处理的任务，仅充当一个标记来使用。在 **MessageQueue** 中会判断如果当前的消息 `target` 为 null 即为同步屏障消息，以此为标记来处理其他的逻辑。

这一部分涉及到消息的异步性以及同步屏障的概念，将会在下面的 MessageQueue 的[同步屏障机制](#同步屏障机制)中做详细介绍。

### 缓存池
虽然 Message 对象可以通过 `new` 的方式直接创建，但在实际的使用中，Message 对象可能会被大量不断地被创建，因此，在 Message 类中引入了一个**全局的缓存池**，可以用来缓存最多 **50 个**的 Message 对象：

```java
public final class Message {

    public static final Object sPoolSync = new Object();
    private static Message sPool; // Message 缓存池，指向 Message 链表中的表头
    private static int sPoolSize = 0; // 当前缓存池的 size

    private static final int MAX_POOL_SIZE = 50; // 缓存池最大个数为 50
}
```

可以通过 `obtain()` 方法来优先从缓存池中获取消息：
```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool; // 取出缓存池 Message 链表的表头，并返回
            sPool = m.next; 
            m.next = null;
            m.flags = 0;
            sPoolSize--;
            return m;
        }
    }
    return new Message(); // 如果缓存池满了，则直接创建新的 Message 对象
}
```

同样，在 Message 使用之后，应该调用 recycle 方法来回收进入缓存池：
```java
void recycleUnchecked() {
    // 清空消息内其他的信息
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    ...
    when = 0;
    target = null;
    callback = null;
    ...
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this; // 把当前空消息插入到缓存池链表表头
            sPoolSize++;
        }
    }
}
```

## Looper
### 线程唯一性
**Looper** 类可以在一个线程中启动 `loop` 循环来不停得获取消息。一个新启动的线程，默认是没有启动 Looper 的，如果需要，只需要在当前线程中，调用以下静态方法即可：
```java
Looper.prepare(); // 为当前线程创建 Looper 对象
...
Looper.loop(); // 启动 loop 循环
```

由于 Looper 是每个线程特用的，因此在 Looper 类 `prepare()` 方法实现中，通过 **`ThreadLocal`** 对象来实现 Looper 对象的线程唯一性：
```java
public final class Looper {
    ...
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
    ...
}
```

而在 Looper 的实际构造方法中，会创建一个关键的 **MessageQueue** 对象用于获取消息（下面再介绍）：
```java
public final class Looper {
    final MessageQueue mQueue;
    final Thread mThread;
    ...
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
    ...
}
```

**Looper** 和 **MessageQueue** 是一一对应的，因此 **MessageQueue** 对象也可以认为是线程唯一的。

### loop 实现
下面接着来看关键的 `loop()` 方法实现：

```java
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) { // 不停地循环处理消息
        // 从当前的消息队列中获取下一个要处理的消息，如果暂时没有，此方法会产生线程阻塞
        Message msg = queue.next();
        if (msg == null) {
            // 如果获取到的消息为空，说明 Looper & MessageQueue 已经退出，这里 return 结束循环
            return;
        }
        ...
        // 分发消息至目标 Handler 中处理
        msg.target.dispatchMessage(msg);
        ...
        // 消息分发完之后，回收至复用缓存池
        msg.recycleUnchecked();
    }
}
```

可以看到，核心 `loop()` 方法的实现其实是比较简单的，会一直循环地调用消息队列的 `next()` 方法来获取下一个要处理的消息，消息分发之后做回收处理。其中有以下两点关键：
- 当消息队列 MessageQueue 中无需要立即处理的 Message 时，`next()` 方法会**阻塞线程，释放 CPU，直至有消息需要处理时再唤醒**，避免造成 CPU 浪费的情况；
- 当 Looper 退出时，`next()` 才会返回一个 null 的 Message（其他情况都认为不会返回 null），此时 `loop()` 循环才会中断。

Looper 的退出是只需要调用 `quit()` 方法即可：
```java
public void quit() {
    mQueue.quit(false);
}
```

## MessageQueue
### 基本结构
顾名思义，**MessageQueue** 表示着消息队列，MessageQueue 对象创建时会同时通过 JNI 创建一个 **native** 对象：
```java
public final class MessageQueue {

    private long mPtr; // 指向 native 层的 MessageQueue
    ...
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
}
```

实际上，MessageQueue 类是作为整个线程消息模型中 java 层和 native 层的通信交互入口，其内包含了一系列的 native 方法实现：

```java
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

关于 native 层的消息通信实现，将在系列文章的下一篇 [Android 线程消息机制（二）native 消息处理]() 中再做详细介绍。

由于 [Message](#Message) 类的自身已经是设计为单向链表的结构，所以实际上，**MessageQueue** 对象中不需要再额外引入队列数据结构，只需要引用着消息链表的表头即可：

```java
public final class MessageQueue {
    Message mMessages;
    ...
}
```

消息在链表中的顺序是**按照消息的预期分发时间从早到晚排序**的，这个特性将会在下面的[消息入列](#消息入列)中体现。

### 获取下一个消息
在上面的 **Looper** 的 [`loop()`](#loop-实现) 方法介绍中提及到，其会循环地调用 MesssageQueue 的 `next()` 方法来获取下一条要分发的消息，`next()` 方法的实现代码如下所示：
```java
Message next() {
    ...
    int nextPollTimeoutMillis = 0;
    for (;;) {
        ...
        // 阻塞操作，核心在 native 层实现，在等待 nextPollTimeoutMillis 时长或者消息队列被唤醒时才返回
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages; // 链表表头的消息
            if (msg != null && msg.target == null) { // 若当前消息的 target 为空，说明这是一个同步屏障消息
                do {
                    prevMsg = msg;
                    msg = msg.next; // 找到链表后面的首个异步消息
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 如果当前的消息分发时间还没到，则计算出下一次唤醒线程的 nextPollTimeoutMillis
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取一个需要分发的消息，从链表中取出并返回该消息
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // msg 为空，说明暂无消息需要分发，nextPollTimeoutMillis = -1，继续阻塞线程
                nextPollTimeoutMillis = -1;
            }
            if (mQuitting) { // 如果消息队列正在退出，直接返回 null
                dispose();
                return null;
            }
            ...
        }
        ...
        nextPollTimeoutMillis = 0;
    }
}
```

总结一下，`next()` 方法获取一下条消息的处理过程中的逻辑如下：

![](/img/posts/post-messagequeue-next.png)

### 新消息入列
当有一个新的 Message 想要插入到 MessageQueue 时，可以调用其 `enqueueMessage()` 方法插入消息链表中，代码示例如下：
```java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    synchronized (this) {
        ...
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // 若队列中的消息为空或者新消息的时间更早，把新消息插入到链表首位
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; // 当阻塞时需要唤醒
        } else {
            // 一般不需要唤醒线程，除非消息队头为同步屏障消息，且新消息为队列中最早的异步消息
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            // 否则按照时间排序，找到新消息应该所在的位置
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
            // 插入消息在链表中间位置
            msg.next = p;
            prev.next = msg;
        }

        if (needWake) {
            nativeWake(mPtr); // 唤醒线程
        }
    }
    return true;
}
```

新消息插入到消息链表中的操作比较简单，归总如下：
- 若队列中无消息或者新消息为最早的消息，则插入到队列链表的首位；
- 否则根据时间顺序，插入到链表中间合适的位置。

除了常规的消息进入队列外，MessageQueue 还提供了插入**同步屏障消息**的方法（有关[同步屏障机制](#同步屏障机制)将在下一小节详细介绍），代码示例如下：
```java
@hide
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            // 根据消息的分发时间（也就是当下），找到消息队列中合适插入的位置
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) {
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

同步屏障消息的插入实现比较简单，就是新获取一个空的消息，插入到消息队列合适的位置当中去。

### 同步屏障机制
上文多次提到的同步屏障消息以及异步消息，其实都是为了本小节的**同步屏障机制**服务。同步屏障机制设立的初衷和原由其实是显而易见的：

对于应用进程的主线程来说，开发者是可以随意的推送多个自定义消息到主线程的消息队列。然而对于系统 Framework 来说，**UI 的测量绘制任务是希望主线程能够能够快速响应处理的**，若 UI 的测量绘制任务消息推送到主线程消息队列上，而该消息的前面还有很多其他的消息等待着被处理时，那么 UI 的绘制渲染工作可能就会被大大的延迟，在用户层面就会出现明显的掉帧等糟糕的体验。

因此，为了满足上述的需要，定义其他普通的消息默认为**同步消息**，意思是只能在消息队列中默认按照队列顺序，逐个被取出分发。同时定义一种特殊的消息为**同步屏障消息**，它的 `target` 为 null 即不需要被分发，只是用来特定标识当前消息为“同步屏障”。**当队列最先的消息为同步屏障消息时，消息队列会把后面的同步消息延缓处理，而是找到最早的一条“异步消息”来分发**。

当没有同步屏障时，消息的取出都是按照队列的顺序依次来：
![](/img/posts/post-messagequeue-next2.png)

当取到同步屏障消息时，延缓处理队列后面的同步消息，直接取最早的一条异步消息：
![](/img/posts/post-messagequeue-next3.png)

因此，这里异步消息的“**异步**”定义就很清晰了。当一个消息设置为异步时，意味此消息可以在某些情况下（遇到同步屏障消息时），跳出正常的逐个排序的同步流程，“异步”的分发此消息。

#### 使用示例

诚如上文所说，同步屏障机制的设计目的就是为了避免 UI 的测量绘制任务被“堵在”主线程的消息队列中，因此可以在 **ViewRootImpl** 的 `scheduleTraversals()` 方法实现中找此机制的使用位置：
```java
public final class ViewRootImpl... {
    ...
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // 插入一个同步屏障消息
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 推送一个 UI 测量绘制任消息
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
    ...
}
```

### IdleHandler
此外，MessageQueue 还提供了一个额外的接口 **IdleHandler**，其会在消息队列“空闲”时回调：
```java
public static interface IdleHandler {
    /**
     * 当消息队列用完消息并且现在将等待更多消息时调用，
     * 该方法调用在队列的消息已用完，或者是等待中的消息当前无需分发时。
     *
     * 返回 true 表示保留此 IdleHandler，
     * 返回 false 以将其删除。
     */
    boolean queueIdle();
}
```

消息队列所谓的空闲，即是：**当前消息队列消息为空（已分发完）或者等待中的消息当前无需分发（未到预期时间）**。

相应的，MessageQueue 内维护着一个 **IdleHandler** 的 List：
```java
public final class MessageQueue {
    ...
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();

    public void addIdleHandler(@NonNull IdleHandler handler) {
        ...
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }

    public void removeIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);
        }
    }
}
```

在关键的 `next()` 方法，当未获取到下一个要处理的消息时，便会走到 IdleHandler 的相关逻辑：

```java
private IdleHandler[] mPendingIdleHandlers;

Message next() {
    ...
    int pendingIdleHandlerCount = -1;
    ...
    for (;;) {
        ....
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // 获取下一个要分发的消息

            ...
            // for 循环的首次 pendingIdleHandlerCount = -1
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // 如果当前无需要处理 IdleHandler，continue 进入下一次循环，从而进入线程阻塞
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null;

            boolean keep = false;
            try {
                // queueIdle() 的返回值表示是否继续保留此 IdleHandler
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                // 若不保留则从 mIdleHandlers 中移除
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // 重设 pendingIdleHandlerCount 为 0，确保上面的代码只会执行一次
        pendingIdleHandlerCount = 0;
        // 当有至少一个 IdleHanlder 处理时，可能会产生新消息，设置 nextPollTimeoutMillis = 0，
        // 此时下一次循环的 nativePollOnce() 会直接返回（不阻塞），再走一次获取下一个消息的逻辑
        nextPollTimeoutMillis = 0;
    }
}
```

总结为，当未能获取到下一个要处理的消息返回时，查看当前已添加的 IdleHandler：
   - 若个数 <= 0，开始阻塞线程；
   - 若个数 > 0，逐个 IdleHandler 取出回调 `queueIdle()`，并再走一次处理消息的循环（新循环将不再处理 IdleHandler）。

#### 使用示例
比如，在 **ActivityThread** 类中就有一个 **`GcIdler`** 的内部类，用来在消息队列空闲时执行 gc 相关操作：

```java
public final class ActivityThread ... {

    ...
    final GcIdler mGcIdler = new GcIdler();
    
    void scheduleGcIdler() {
        if (!mGcIdlerScheduled) {
            mGcIdlerScheduled = true;
            Looper.myQueue().addIdleHandler(mGcIdler);
        }
        mH.removeMessages(H.GC_WHEN_IDLE);
    }

    final class GcIdler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            doGcIfNeeded();
            purgePendingResources();
            return false;
        }
    }
}

```

## Handler
**Handler** 作为消息的发送者和处理者，可以说是开发者使用最直接和频繁的类。使用 Handler 主要有以下两个用途：

1. **计划一个消息任务延迟在之后某个特定时间再执行**。
2. **在其他线程中插入一个要执行的消息任务**。

### 基本结构
每个 Handler 对象在创建时需要绑定一个线程的 **Looper** 对象，换言之，**一个 Handler 对象在创建时就决定了它负责向哪个线程推送以及分发消息**。

```java
public class Handler {
    ...
    final Looper mLooper; // 绑定对应线程的 Looper
    final MessageQueue mQueue; // 绑定对应线程的 MessageQueue
    final Callback mCallback;
    final boolean mAsynchronous;
}
```

为此，Handler 提供了多个对象创建的构造方法：
```java
public Handler()
public Handler(@Nullable Callback callback)
public Handler(@NonNull Looper looper)
public Handler(@NonNull Looper looper, @Nullable Callback callback)
```

无参的构造方法 `Handler()` 默认会使用当前线程的 Looper（通过调用 `Looper.myLooper()` 方法）。如果当前的线程未[启动 Looper](#looper)，则会直接抛出 `RuntimeException`。**因此，创建 Handler 对象前，请先确保目标线程已启动 Looper**。

### 推送新消息到队列
对于新消息的推送，Handler 提供了多个方便使用的接口，即可以简单地传入一个 `Runnable` 对象，也可以构建一个完整的 Message 对象来发送信息：
```java
public final boolean post(@NonNull Runnable r);
public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis);
public final boolean postDelayed(@NonNull Runnable r, long delayMillis);
public final boolean postAtFrontOfQueue(@NonNull Runnable r);
...
```

对于上面一系列的方法，如果参数传入的是 Runnable 对象，则会进一步封装为 **Message**，如果传入的是延迟的时间间隔，则会加上当前的时间戳计算出期望分发消息的时间戳，最终都是调用到 `enqueueMessage()` 方法中：
```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

最终，调用 MessageQueue 的 [`enqueueMessage()`](#新消息入列) 方法完成[新消息入列](#新消息入列)。

### 消息的处理
前文有提及到，在 MessageQueue 获取到要处理的消息给 Looper 后，会回调 Handler 的 `dispatchMessage()` 方法，完成消息的分发：
```java
public class Handler {
    final Callback mCallback;

    ...
    public void dispatchMessage(@NonNull Message msg) {
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

    private static void handleCallback(Message message) {
        message.callback.run();
    }

    // 默认的实现为空，子类需要做重写完善这里的逻辑
    public void handleMessage(@NonNull Message msg) {
    }
}
```

从 `dispatchMessage()` 方法的实现可知，消息的分发处理有以下优先级：
1. 若 Message 自身的 callback（`Runnable` 对象）不为空，回调此 callback，结束消息处理；
2. 若 Handler 自身的 mCallback（构造方法传入）不为空，回调此 mCallback，若 Callback 内的 `handleMessage()` 方法返回 true，结束消息处理；
3. 回调 Handler 自身的 `handleMessage()` 方法实现。

### 移除消息
一个延迟执行的消息通过 Handler 插入到 MessageQueue 内时，如果当前消息还没到时间分发，那么此消息是可以被提前移除的。

同样，Handler 提供了多个方法来**移除一个或者全部的消息**：
```java
public final void removeCallbacks(@NonNull Runnable r);
public final void removeCallbacks(@NonNull Runnable r, @Nullable Object token)
public final void removeMessages(int what)
public final void removeCallbacksAndMessages(@Nullable Object token);
...
```

相关的 remove 操作最终也是调用到 MessageQueue 提供的移除方法中做移除，消息的移除实现比较简单，只需要遍历消息链表找到匹配的 Message 做移除回收即可，下面是其中 MessageQueue 中的 `removeMessages()` 方法示例：
```java
void removeMessages(Handler h, int what, Object object) {
    ...
    synchronized (this) {
        Message p = mMessages;

        // 移除队列头部匹配的消息
        while (p != null && p.target == h && p.what == what
                && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n;
            p.recycleUnchecked();
            p = n;
        }

        // 移除队列后面匹配的消息
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                    && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    n.recycleUnchecked();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
    }
}
```

**一个 Handler 只能移除通过自身推送的消息**。