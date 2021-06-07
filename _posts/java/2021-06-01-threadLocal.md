---
layout: post
title: Java ThreadLocal 类的使用和原理
tags:
  - Java
---

## 关于 ThreadLocal
`ThreadLocal`，顾名思义，此类是用来提供**线程本地变量**。ThreadLocal 类设计成了带泛型，一个 ThreadLocal 对象可以在不同的线程中，分别存入一个特定类型的变量。每个线程仅可以访问各自存入的变量，从而达到**线程隔离**。

## 使用与举例
ThreadLocal 类使用的接口设计地非常简单，仅需要通过 `set(T value)` 方法设置线程本地变量，通过 `get()` 获取该变量即可：
```java
final ThreadLocal<String> threadLocal = new ThreadLocal<>();
threadLocal.set("value1");

System.out.println(threadLocal.get());
// 输出结果为 “value1”

new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println(threadLocal.get());
        // 输出结果为 “null”
    }
}).start();
```

举个具体的例子将很好的说明 ThreadLocal 的使用场景：在 Android 的线程消息机制中，每个线程需要拥有“自己”的 Looper 对象来分开处理线程内的队列消息，这其中便是通过 ThreadLocal 类来实现的：
```java
public final class Looper {
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    // 通过 Looper.myLooper() 便可以返回当前线程特定的 Looper 对象
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
}
```

## 基本原理
为了达到每次对 ThreadLocal 对象进行 `set(T value)` 操作时，存入的对象“仅属于”当前的线程，引入了一个可以**存储多个 key-value 的 ThreadLocaMap 类**。

每个线程的 **Thread** 类对象中持有一个该 ThreadLocalMap 对象：

```java
public class Thread implements Runnable {
    ...
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ...
}
```

先来看下 `set()` 方法实现：

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

在上面的代码中，若当前线程 Thread 对象的 `threadLocals` 为空，则先创建一个新的 ThreadLocalMap。否则，把目标 value 存储进此ThreadLocalMap 中，整体结构如下：

![](/img/posts/post-threadlocal-set2.png)

同样的，`get()`  方法实现也是通过从线程专属的 ThreadLocalMap 对象中根据目标 key（ThreadLocal 对象自身）来获取到目标 value，若目标 key 不存在，则默认返回 null：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue(); // 默认值为 null
}
```

换言之，**ThreadLocal 对象本身不会直接存储 key-value**，而是对外提供一套操作的 `get()/set()` 接口，以自身为 key，存放在每个线程 Thread 对象特有的一个 **ThreadLocalMap** 数据结构里，从而实现**线程隔离**的对象存储。

## ThreadLocalMap 
ThreadLocal 的运作机制核心实现在 ThreadLocalMap 中，下面来一探其究竟。

### 基本结构
**ThreadLocalMap** 为 ThreadLocal 的静态内部类，其效果类似于一个 Map，可以存放多个 key-value。ThreadLocalMap 内部也有一个“特殊”的 **Entry** 类来表示每个 key-value:

```java
static class ThreadLocalMap {
    
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** ThreadLocal 关联的变量 */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
}
```

之所以说是特殊的，是因为该 Entry 继承自弱应用类 WeakReference，它的 **key 为 ThreadLocal 对象并将其设置为自身的弱引用对象**，value 则为 ThreadLocal 目标存入的变量。

![](/img/posts/post-threadlocal-entry.png)

由于一个 ThreadLocal 对象可能会在不同的线程中调用设置，每个线程又会各自持有一个 ThreadLocalMap 对象，因此通过对 ThreadLocal 设置为弱引用的方式，在 ThreadLocal 对象不再被外部持用使用时自动释放。

接着看 ThreadLocalMap 内部的 Entry 数组结构，这里和 **HashMap** 等 Map 实现类结构类似：

```java
static class ThreadLocalMap {
    /** 数组初始大小 */
    private static final int INITIAL_CAPACITY = 16;
    /** Entry 数组 */
    private Entry[] table;
    /** 当前存入 table 的 Entry 个数 */
    private int size = 0;
    /** 数组需要拓展大小的临界值 */
    private int threshold;

    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }
    ...

    /** 构造方法 */
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY]; // 初始化 Entry 数组
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
}
```

### set() 方法实现
接着来看 ThreadLocalMap 的 `set()` 方法实现：

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1); // 取哈希值合适的低位为节点 index

    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) { // 新旧 key 相同，更新 value 变量
            e.value = value;
            return;
        }

        if (k == null) { // key 为空（弱引用已释放），替换该节点为新的 Entry
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value); // 在目标节点插入新的 Entry
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

在 `set()` 方法的实现中，目标是把传入的 key-value 组成一个 Entry 存在进数组中。为此和 HashMap 的实现类似，需要先通过一定的哈希码离散算出存放的节点 index，这里哈希码 `threadLocalHashCode` 的算法实现比较简单，示例如下：

```java
public class ThreadLocal<T> {
    // 此 ThreadLocal 实例的 hashcode
    private final int threadLocalHashCode = nextHashCode();

    /**
     * 全局静态变量，自动更新
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * 返回下一个哈希码
     */
    private static int nextHashCode() {
        // 这里使用了「斐波那契散列法」，来保证离散度
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
}
```

在算出要存在的目标节点后，判断当前目标节点是否有已存在的 Entry，整个描述如下：

<!-- - Entry 不存在，在目标节点插入新的 Entry
- Entry 存在，获取已有 Entry 的弱引用 key(ThreadLocal)：
   - 若与目标 key 相同，更新 value 变量
   - key 为空（弱引用已释放），向前遍历节点，查找与目标 key 相同的 Entry（`replaceStaleEntry()` 方法实现）：
     - 若有，交换该 Entry 到目标节点，并替换 Entry 的旧 value 为目标 value
     - 若没有，在目标节点插入新的 Entry
     - 擦除上述过程中发现的 key 已被回收的 Entry（`expungeStaleEntry() 方法实现`）
   - key 不同且不为空，依次获取下一个节点并重复上面判断逻辑 -->

![](/img/posts/post-threadlocal-set.png)

### getEntry() 方法实现
基于上面 `set()` 方法的实现原理，可知目标 Entry 优先存放在 key(ThreadLocal) 的哈希码低位算出来的数组节点，否则再考虑存放在其他可用的节点。那么 `getEntry()` 方法的实现则比较简单了：

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        // 目标节点存放的 Entry 存在且 key 相同，直接返回该 Entry
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key) // 在其他节点位置找到目标 key
            return e;
        if (k == null) // 当前 key 已释放，擦除该 Entry
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len); // 下一节点，遍历数组
        e = tab[i];
    }
    return null; // 否则目标 key 不存在，返回 null
}
```

## 总结
1. 使用 **ThreadLocal** 类可以存储“专属”当前线程的变量，从而实现在不同线程中操作不同变量等特定需求。
2. ThreadLocal 类不直接储存变量数据，而仅仅是提供对外的接口，数据真正存放在线程 Thread 对象中的 **ThreadLocalMap** 数据结构中。
3. ThreadLocalMap 内通过以 ThreadLocal 对象为 key，以数组的结构实现了存储多个 key-value 的 Map 能力。

