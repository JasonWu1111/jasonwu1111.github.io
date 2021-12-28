---
layout: post
title: Android 线程消息机制（二）—— Native 消息处理
tags:
  - Android
  - Thread
  - Native
---

## 前言
在系列文章上篇 [Android 线程消息机制 —— Java 层消息模型](/2021/06/07/thread-message-java/#android-线程消息机制模型) 中提到，Android 的线程消息模型是系统中一个非常重要且核心的机制，它重新定义了 Android 内线程运作以及线程间通信的机制，它贯通了整个 framework 层（包括 Java 和 Native）里很多重要的线程运作机理。

Java 层整体模型示例如下：
![](/img/posts/post-handler.png)

**Looper、MessageQueue、Message、Handler** 这四个核心的角色各自承担自己的职责，构成了 Android 线程消息机制模型。

**这一套线程消息机制在 Android Framework 中不仅是在 Java 层，在 Native 层同样可以应用**。本文将接着介绍在线程消息机制 Native 层的实现与原理。

涉及到的源码文件有：

```light
frameworks/base/core/java/android/os/
    - MessageQueue.java

frameworks/base/core/jni/
    - android_os_MessageQueue.cpp

system/core/include/utils/
    - Looper.h

system/core/libutils/
    - Looper.cpp
```

## MessageQueue 对象的创建
在介绍 [MessageQueue Java 类结构](/2021/06/07/thread-message-java/#基本结构-1) 中有提及到，Java 层的 **MessageQueue** 对象在创建时会进一步通过 JNI 来创建一个 Native 对象：

```java
public final class MessageQueue {
    private long mPtr;

    ...
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
}   
```

>frameworks/base/core/java/android/os/MessageQueue.java

在 `nativeInit()` 中，进一步通过 JNI 调用，创建了 Native 层消息队列 **NativeMessageQueue** 以及 **Looper** 对象，时序图示意如下：

![](/img/posts/post-messagequeue-new.png)
<!-- Title: MessageQueue 的创建
...->MessageQueue: new
MessageQueue-\->JNI: nativeInit()
JNI->JNI:android_os_MessageQueue\n_nativeInit()
JNI->NativeMessageQueue\n(native): new
NativeMessageQueue\n(native)->Looper\n(native): new
Looper\n(native)->Looper\n(native): rebuildEpollLocked()
Looper\n(native)->Looper\n(native): ... -->

在 Jave 层中，MessageQueue 是在 Looper 对象创建时创建的，而在 Native 层，则刚好相反：**Looper 对象会在 NativeMessageQueue 对象创建时创建**：

```cpp
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```
> frameworks/base/core/jni/android_os_MessageQueue.cpp

这里 Native 层的 Looper 和 Java 类的 Looper 并没有直接联系，而是在 Native 层又实现了一套类似功能的逻辑。

在 Java 层，Looper 是通过 ThreadLocal 来保证线程唯一的，而在 Natice 层，则是依靠 `pthread_setspecific()` 函数来实现的 Looper 对象的线程唯一性：

```cpp
void Looper::setForThread(const sp<Looper>& looper) {
    sp<Looper> old = getForThread(); // also has side-effect of initializing TLS
    ...
    pthread_setspecific(gTLSKey, looper.get());
    ...
}

sp<Looper> Looper::getForThread() {
    int result = pthread_once(& gTLSOnce, initTLSKey);
    ...
    return (Looper*)pthread_getspecific(gTLSKey);
}
```
> system/core/libutils/Looper.cpp

在 Native 层，**Looper** 作为整个消息流转的“中心”以及核心实现，其运作机理是依赖于 Linux 内核的 **epoll** 机制实现的。下面一节先单独介绍 [epoll 机制](#linux-epoll-机制)，然后再看下 [epoll 在 Looper 当中的应用](#epoll-在-looper-中的应用)。

## Linux epoll 机制
**epoll** 是 Linux 内核中最高效的 I/O 多路复用机制，其**使用一个文件描述符，可以同时监控多个文件描述符**。

当要监控的文件描述符就绪时，会通知对应程序进行读 / 写操作。其本质上是同步 I/O，即读 / 写都是阻塞的。

**epoll** 有三个主要的函数：

### epoll_create 
```cpp
int epoll_create(int size);
```
**epoll_create** 用来创建一个 epoll 的文件描述符（fd）。
- size：要监听的文件描述符（fd）个数。

### epoll_ctl 
```cpp
int epoll_ctl(int epoll_fd, int op, int fd, struct epoll_event* event);
```
**epoll_ctl** 用于对需要监听的文件描述符来执行一定的操作。
- epoll_fd：是 **epoll_create** 的返回值
- op：具体的操作，公用
  - EPOLL_CTL_ADD：添加
  - EPOLL_CTL_DEL：删除
  - EPOLL_CTL_MOD：修改
- fd：需要监听的文件描述符
- event：需要监听的事件

**epoll_event** 结构体如下：
```cpp
struct epoll_event {
  uint32_t events; // events 的取值请看下面表格(可以同时带有多个)
  epoll_data_t data; 
}
```

|events 取值|含义
|--|--|--
|EPOLLIN|可读
|EPOLLPRI|高优先级的可读
|EPOLLOUT|可写
|EPOLLERR|错误
|EPOLLHUP|中断
|...|...

### epoll_wait 
```cpp
int epoll_wait(int epoll_fd, struct epoll_event* events, int event_count, int timeout_ms);
```

**epoll_wait** 会阻塞线程等待事件。
- epoll_fd：是 **epoll_create** 的返回值
- events：从内核得到事件的集合
- event_count：events 的数量，不能大于 **epoll_create** 传入的 size；
- timeout_ms：超时的时间，单位为毫秒

当调用了该方法后，会进入阻塞状态，等待 epoll_fd 上的 IO 事件，若 epoll_fd 监听的某个文件描述符发生 IO 有事件产生时，就会进行回调，从而使得 epoll 被唤醒并返回需要处理的事件个数。若超过了设定的超时时间，同样也会被唤醒并返回 0 避免一直阻塞。

## epoll 在 Looper 中的应用
在了解了 **epoll** 机制之后，下面来看下 epoll 在 Looper 中的应用。

### epoll fd 的创建
如前面介绍的，epoll 是通过一个文件描述符来同时监控多个文件描述符。在 **Looper** 中，**使用 `mEpollFd` 来存储 epoll 创建的文件描述符，`mWakeEventFd` 来存储默认要监控的文件描述符（同时使用它来唤醒阻塞中的线程）**。

```cpp
private:
    android::base::unique_fd mWakeEventFd;
    android::base::unique_fd mEpollFd;
    ...
```
> 其中 **unique_fd** 类会对存入进去的 fd 做自动回收处理，具体可看：
> - system/core/base/include/android-base/unique_fd.h

在 Looper 的构造方法中，会创建新的被监控的 **eventfd** 存入到 `mWakeEventFd` 中，然后再创建 epoll 相关的 fd。

```cpp
Looper::Looper(bool allowNonCallbacks)
    : ...{
    // 创建新的被监控的 eventfd
    mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC));
    ...
    // 创建 epoll fd
    rebuildEpollLocked();
}
```

`rebuildEpollLocked()` 方法实现如下：
```cpp
void Looper::rebuildEpollLocked() {
    // 重设 mEpollFd
    if (mEpollFd >= 0) {
        mEpollFd.reset();
    }

    // 创建新的 epoll fd 保存到 mEpollFd 中
    mEpollFd.reset(epoll_create1(EPOLL_CLOEXEC));
    ...
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event));
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeEventFd.get();
    // 添加 mWakeEventFd 保存的 fd 到 epoll 中监控
    int result = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, mWakeEventFd.get(), &eventItem);
    ...
}
```

可见，其中会先调用 `epoll_create()` 方法来创建一个 **epoll fd** 保存到 `mEpollFd` 中，然后调用 `epoll_ctl()` 方法（其中 op 为 EPOLL_CTL_ADD，即添加）来添加 `mWakeEventFd` 保存的 fd 到监控中去。

### 监控新的文件描述符
在 **Looper** 中，除了会使用 `mEpollFd` 来默认监控 `mWakeEventFd` 外，还支持了外部调用，添加其他新的 fd 进来被监控。为了方便管理插入的 fd 请求和后续的响应处理，这里使用了两个新的结构体 **Request** 和 **Response：**：
```cpp
struct Request {
    int fd;
    int ident;
    int events;
    int seq;
    sp<LooperCallback> callback;
    void* data;

    void initEventItem(struct epoll_event* eventItem) const;
};

struct Response {
    int events;
    Request request;
};
```
> system/core/include/utils/Looper.h

**Looper** 中创建了两个 vector 容器，分别存储 Request 和 Response 数组：
```cpp
private:
    ...
    KeyedVector<int, Request> mRequests;
    int mNextRequestSeq;
    Vector<Response> mResponses;
    size_t mResponseIndex;
    ...
```

**Looper** 提供了 `addFd()` 方法用来添加新的 fd 来被 epoll 监控，其实现如下：
```cpp
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
    ...
    { // 获取锁
        AutoMutex _l(mLock);
        // 新一个 Request 对象
        Request request;
        request.fd = fd;
        request.ident = ident;
        request.events = events;
        request.seq = mNextRequestSeq++;
        request.callback = callback;
        request.data = data;
        if (mNextRequestSeq == -1) mNextRequestSeq = 0; // reserve sequence number -1

        struct epoll_event eventItem;
        request.initEventItem(&eventItem);

        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) { // 若目标 fd 未被添加到 epoll，做添加
            int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, fd, &eventItem);
            if (epollResult < 0) {
                return -1;
            }
            mRequests.add(fd, request);
        } else {
            // 若目标 fd 已被添加过，这里做更新
            int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_MOD, fd, &eventItem);
            if (epollResult < 0) {
                ...
            }
            mRequests.replaceValueAt(requestIndex, request);
        }
    } // 释放锁
    return 1;
}
```

同样，还有对应的 `removeFd()` 方法，来实现 fd 从 epoll 中的移除，其内就是通过 `epoll_ctl()` 来从 epoll 中移除目标fd，以及

```cpp
int Looper::removeFd(int fd, int seq) {
    { // 获取锁
        AutoMutex _l(mLock);
        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) { // mRequests 中不存在当前 fd 对应的 Request
            return 0;
        }
        // 目标 Request 和要移除的不一致
        if (seq != -1 && mRequests.valueAt(requestIndex).seq != seq) {
            return 0;
        }
        // 从 mRequests 中移除目标 Request
        mRequests.removeItemsAt(requestIndex);
        // 从 epoll 中移除目标 fd
        int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_DEL, fd, nullptr);
        if (epollResult < 0) {
            ...
        }
    } // 释放锁
    return 1;
}
```

### nativePollOnce
在系列文章上篇 [Android 线程消息机制 —— Java 层消息模型](/2021/06/07/thread-message-java/#android-线程消息机制模型) 中提到，在 Java 层 **MessageQueue** 的 `next()` 方法中，每次获取新的 Java 消息时，都会先调用 `nativePollOnce()` 方法来进入线程阻塞：
```java
Message next() {
    ...
    int nextPollTimeoutMillis = 0;
    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis);
        ...
    }
}
```
> frameworks/base/core/java/android/os/MessageQueue.java

其阻塞原理便是最终调用到 `epoll_wait()` 方法来实现，然而 `nativePollOnce()` 方法最终实现的逻辑还有更多，相面将具体展开来一探究竟：

`nativePollOnce()` 方法会进一步通过 JNI 调用到 Native 层的 **Looper** 当中去，调用流程如下：

![](/img/posts/post-messagequeue-pollonce.png)
<!-- Title: MessageQueue nativePollOnce()
MessageQueue->MessageQueue: next()
MessageQueue-\->JNI: nativePollOnce()
JNI->NativeMessageQueue\n(native):android_os_MessageQueue\n_nativePollOnce()
NativeMessageQueue\n(native)->Looper\n(native): pollOnce(timeoutMillis)
Looper\n(native)->Looper\n(native): pollOnce()
Looper\n(native)->Looper\n(native): ... -->

真正实现的机理在 **Looper** 的 `pollInner()` 函数中：

```cpp
int Looper::pollInner(int timeoutMillis) {
    ...
    // Poll 操作
    int result = POLL_WAKE;
    mResponses.clear(); // 先清空 mResponses
    mResponseIndex = 0;
    // 线程即将处于 idle 状态
    mPolling = true;
    struct epoll_event eventItems[EPOLL_MAX_EVENTS]; // EPOLL_MAX_EVENTS = 16
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    mPolling = false;

    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        // 若 epoll 事件个数小于 0，说明发生了错误，直接跳转至 Done;
        result = POLL_ERROR;
        goto Done;
    }

    if (eventCount == 0) {
        // 若 epoll 事件个数等于 0，说明发生了超时，直接跳转至 Done;
        result = POLL_TIMEOUT;
        goto Done;
    }
    // 有 epoll 事件返回，遍历每个事件
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd.get()) {
            // 由 mWakeEventFd 写入事件，读取并清空管道数据
            if (epollEvents & EPOLLIN) {
                awoken();
            } ...
        } else {
            // 找到对应事件 fd 的 Request
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0; 
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                // 将 Request 生成对应的 Reponse 对象，并放入到 mResponses 数组中
                pushResponse(events, mRequests.valueAt(requestIndex));
            } ...
        }
    }
Done: ;
    // 这部分是 Native 消息的处理
     
    ...
    // 遍历 mResponses，开始处理
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            // 触发回调监听
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            // 若回调结果为 0，移除此 fd 监控
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }

            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

`pollInner()` 方法处理逻辑整理如下：
![](/img/posts/post-looper-pollinner.png)

因此，根据实际的情况，`pollInner()` 方法共有以下四个返回值，代表不同的线程阻塞等待返回：

|返回值|说明
|--|--
|POLL_WAKE|由 mWakeEventFd 写入事件唤醒（通过调用 `Looper::wake()` 方法）
|POLL_CALLBACK|表示某个被监听 fd 被触发并触发回调
|POLL_TIMEOUT|表示阻塞等待超时返回（等待时长为 timeoutMillis）
|POLL_ERROR|表示阻塞等待过程中发生异常返回

### nativeWake
在调用了 `nativePollOnce()` 方法进入了阻塞等待后，当 Java 层有新消息传入到消息队列时，会调用 `nativeWake()` 方法来进一步唤起线程，同样是通过 **JNI** 逐步调用到 Native 层的 Looper 当中去：

![](/img/posts/post-messagequeue-wake.png)
<!-- Title: MessageQueue nativeWake()
MessageQueue->MessageQueue: enqueueMessage()
MessageQueue-\->JNI: nativeWake()
JNI->NativeMessageQueue\n(native):android_os_MessageQueue\n_nativeWake()
NativeMessageQueue\n(native)->Looper\n(native): wake(timeoutMillis)
Looper\n(native)->Looper\n(native): wake()
Looper\n(native)->Looper\n(native): ... -->

`wake()` 方法的实现就比较简单了，通过前面的介绍，只需要向管道 mWakeEventFd 中写入字符 1，触发 epoll 的阻塞等待返回即可：
```cpp
void Looper::wake() {
    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        ...
    }
}
```

## Native 消息
**Looper** 中对 epoll 的使用，可以说是 Android 线程消息机制实现最为之重要的基础，在此之上，Native 层同样提供了和 Java 层类似的消息 & 消息队列来分发处理线程消息。

### MessageEnvelope 结构体
与 Java 层的 MessageQueue 不同，Native 层的 **NativeMessageQueue** 自身并不维护实际的“消息队列”，而是一个承担连接的角色。Native 层的真正的“消息队列”实际上是维护在 `Looper` 的一个 vector 容器中：

```cpp
class Looper : public RefBase {
    ...
private:
    ...
    Vector<MessageEnvelope> mMessageEnvelopes;
}
```

容器内的元素为结构体 `MessageEnvelope`。MessageEnvelope 如其名字所意，表示的是消息的信封，记录着：
- 送信时间 uptime
- 收信人 MessageHandler
- 信件内容 Message

```cpp
struct MessageEnvelope {
    MessageEnvelope() : uptime(0) { }

    MessageEnvelope(nsecs_t u, const sp<MessageHandler> h,
            const Message& m) : uptime(u), handler(h), message(m) {
    }

    nsecs_t uptime;
    sp<MessageHandler> handler;
    Message message;
};
```

其中 **Message** 结构体表示信息的具体内容，示例如下：
```cpp
struct Message {
    Message() : what(0) { }
    Message(int w) : what(w) { }

    int what; // 消息类型
};
```

**MessageHandler** 表示信息的处理者：
```java
class MessageHandler : public virtual RefBase {
protected:
    virtual ~MessageHandler();

public:
    // 处理消息
    virtual void handleMessage(const Message& message) = 0;
};
```

看得出来，在 Native 层，**Message** 结构体仅代表消息内容本身，而包含消息分发相关的 **MessageEnvelope** 则更相当于 Java 类 android/os/Message。


### 消息发送
还是在 **Looper** 类中，提供了多个方便推送消息的方法：
```cpp
void sendMessage(const sp<MessageHandler>& handler, const Message& message);
void sendMessageDelayed(nsecs_t uptimeDelay, const sp<MessageHandler>& handler,
        const Message& message);
void sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message);
```
> system/core/libutils/Looper.cpp

其最终都是调用到 `sendMessageAtTime()` 方法，来将新消息，根据时间排序，插入到 `mMessageEnvelopes` 中（与 Java 层一致）：
```cpp
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
    size_t i = 0;
    {
        AutoMutex _l(mLock);

        size_t messageCount = mMessageEnvelopes.size();
        // 根据 uptime 时间排序，找到合适的插入数组的位置
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }

        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);

        if (mSendingMessage) {
            return;
        }
    }
    // 若新消息是插入到消息队列的队头，唤醒线程来做进一步的处理
    if (i == 0) {
        wake();
    }
}
```

### 消息处理
对于消息的处理，在前面 [nativePollOnce](#nativepollonce) 一节已有提及到，在 `Looper::pollInner()` 方法中，在 epoll 阻塞等待返回后，**在处理 epoll 事件回调前，会先处理 Native 消息：**
```cpp
int Looper::pollInner(int timeoutMillis) {
    // 传入的 timeoutMillis 为 Java 消息的超时时间
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        // 若 Native 消息的超时时间更早，则更新取更早的时间
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
    }
    ...
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    ...
Done: ;

    mNextMessageUptime = LLONG_MAX;
    // 开始循环
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        // 取出消息队列第一个消息（即分发时间最早的消息）
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) { // 如果此消息到了时间分发
            {
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();
                handler->handleMessage(message); // 分发处理消息
            }

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            // 若当前最早的消息仍未到时间分发，记录下一次分发的时间，结束循环
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    ...
    // 后面是 epoll 事件回调的处理
}
```

## 总结
1. Android 的这套线程消息处理机制，把原本线程单一的“串行执行”改为了基于消息队列的“任务消费”模型，既满足了对于线程执行任务的动态管理，更提供了一种方便易用的线程间的通信方式。
   
2. 整个线程消息处理机制，贯穿 Android Framework 的 Java 层和 Native 层。Java 层和 Native 层有各自独立的消息推送和处理的实现，**但整个消息循环遍历是一体的：Java 层负责驱动消息的遍历，Native 底层实现了阻塞等待的能力**。
![](/img/posts/post-looper-loop.png)
   
3. 此机制最底层的阻塞等待 & 唤醒机制是依赖于 Linux 高效的 I/O 多路复用机制 **epoll** 实现。
   
4. 在基于 epoll 事件之上，Android Framework 设计了 **Looper、MessageQueue、Message、Handler** 四个核心角色构建了整个线程消息模型。

5. **MessageQueue** 通过 JNI 串联起了 Java 层和 Native 层消息处理的实现，起到了桥梁的作用。

6. 对于消息的处理，顺序是 Native 消息 -> epoll 事件回调 -> Java 消息。
