---
layout: post
title: View 触摸事件的分发
tags:
  - Android
  - View
---

# 触摸事件 MotionEvent
### 常见的触摸事件
Android View 系统中，通过 `MotionEvent` 类来表示在屏幕上的触摸事件。常见的触摸事件有以下几种：

|事件|说明
|--|--
|`ACTION_DOWN`	| 表示一个按下手势的开始，发生在刚触碰到屏幕时
|`ACTION_UP`	|表示一个按下手势的结束，发生在屏幕上松开时
|`ACTION_MOVE`	|表示屏幕上触摸的移动，是 **ACTION_DOWN** 和 **ACTION_UP** 之间的一系列行为
|`ACTION_CANCEL`	|表示当前触摸手势被中断，是在接收到 **ACTION_DOWN** 事件但后续事件被拦截掉时触发

如果只是一个快速的点击，那么产生的触摸事件序列只会有 ``ACTION_DOWN`` 和 ``ACTION_UP``，如果是触碰到屏幕后，产生了滑动再离开，将会额外产生一系列的 ``ACTION_MOVE`` 事件：
![](/img/posts/post-view-touch.png)

### 多点触摸
目前，一般的移动设备屏幕都支持了多点触摸，当有超过一个的触摸点同时出现时，触摸事件的类型和内容会有所不同。

**MotionEvent** 中另有 `ACTION_POINTER_DOWN` 和 `ACTION_POINTER_UP` 两个 Action 分别表示额外的触摸点带来的 **DOWN** 和 **UP** 事件，可通过以下方法获取到相关的信息：

|方法|说明
|--|--
|`getActionMasked()`	| 获取当前触摸事件的类型
|`getActionIndex()`	| 获取当前触摸事件触摸点的索引，如第一个触摸点为 0，第二个为 1

> 此外，多点触摸时，**滑动**行为统一通过 ``ACTION_MOVE`` 事件回调

### 最小滑动距离
``ACTION_MOVE`` 事件产生的必要条件是触摸屏幕后**移动了至少一定的距离**，这个系统所能识别的最小滑动距离 ``TouchSlot`` 默认值为 **8dp**，定义在 *frameworks/base/core/res/res/values/config.xml* 中：
```xml
<!-- Base "touch slop" value used by ViewConfiguration as a
        movement threshold where scrolling should begin. -->
<dimen name="config_viewConfigurationTouchSlop">8dp</dimen>
```

另外可通过以下方法在代码中获取：
```java
ViewConfiguration.get(context).getScaledTouchSlop()
```

### 触摸坐标值
有时，对于触摸事件，我们想知道具体发生触摸行为的坐标点值，对此，MotionEvent 提供了以下两组方法：

|方法|说明
|--|--
|getX / getY| 返回相对于当前 View 左上角的坐标值
|getRawX / getRawY| 返回相对于屏幕左上角的坐标值

# 触摸事件的分发
在 Android 系统中，子 View 会覆盖在父 View 的区域上显示，越底层的父 View 越显示在底部。而在一次触摸行为中，视觉上看起来是触摸点是直接与最上层的 View 接触产生触摸事件，然后再实际处理中却是“**相反的**”。**触摸事件的分发是从底层的 View 开始，一层层分发到其子 View 中去**。

![](/img/posts/post-view-touch3.png)

### 基本流程
触摸事件在 View 树中的分发流向，主要是通过以下两个个返回 **boolean** 值结果的函数来决定：

|方法|说明
|--|--
|(boolean)`dispatchTouchEvent`|表示传递触摸事件到当前 View 上，返回 true 表示该 View 消费了此触摸事件
|(boolean)`onInterceptTouchEvent`|表示是否拦截到来的触摸事件，返回 true 则触摸事件将不再向子 View 传递，ViewGroup 特有

> 以上方法定义在：<br>frameworks/base/core/java/android/view/View.java<br>frameworks/base/core/java/android/view/ViewGroup.java

默认情况下，触摸事件会从底层的 RootView 开始，通过调用逐级调用子 View 的`dispatchTouchEvent` 方法，把触摸事件分发下去，直到符合触摸点坐标最上层的 View 来处理，并把处理结果递归返回：

![](/img/posts/post-view-touch4.png)

当分发链中，某个父 ViewGroup 决定要拦截该触摸事件自行处理时，可重写 `onInterceptTouchEvent` 并返回为 **true**，此时触摸事件并不会再往下一级传递，示意如：

![](/img/posts/post-view-touch5.png)

### 事件消费的连续性 & TouchTarget
由于父 ViewGroup 可以决定是否拦截事件分发到其子 View 上，试想有这么一种情况：对于 `ACTION_DOWN` 父 ViewGroup 做拦截，而对于后续的 `ACTION_MOVE` 和 `ACTION_UP` 不做拦截，那对应位置的子 View 是否可以只接受和消费后面的 MOVE 和 UP 事件呢？

答案是不能的。在 Android 系统的触摸事件分发体系中，一次触摸行为会产生一系列的触摸事件，`ACTION_DOWN` 会被视为该系列触摸事件当中的**首个事件**，并以此增加了以下的额外机制：
- **当 `ACTION_DOWN` 事件没有被消费时，后续的事件就会在底层的 RootView 处做拦截不继续分发下去**
- **当 `ACTION_DOWN` 事件中途被拦截掉并被消费时，后续的事件将在该拦截处截止，不能再继续往下分发**
- **当 `ACTION_DOWN` 事件被某个子 View 消费后，后续事件中即便触摸点移动超出了该 View 的范围也会继续分发到该 View 来处理**

以上的机制一定程度上确保了事件分发的**连续性**，保障了目标 View 视图能完整处理到来的一系列事件。而这样的机制是通过额外引入的 `TouchTarget` 类来完成。

`TouchTarget` 顾名思义表示的是当前的触摸目标，**其记录了当前 ViewGroup 中事件分发出去后消费了 `ACTION_DOWN` 事件的子 View**。

其为 **ViewGroup** 的内部类，设计成链表结构，代码示例如下：
```java
public abstract class ViewGroup extends View {
    ...
    private static final class TouchTarget {
        // The touched child view.
        public View child;
        // The combined bit mask of pointer ids for all pointers captured by the target.
        public int pointerIdBits;
        // The next target in the target list.
        public TouchTarget next;
        ...
    }
    ...
}
```

其中有三个关键的成员变量，含义如下：

- `child`：消费事件的目标视图。

- `pointerIdBits`：目标视图多个触摸点 id 的组合，计算规则为：如第一个触摸点的 pointerId 为 0 对应 0000 0001，第三个触摸点 pointerId 为 2 对应 0000 0100，合在一起的 pointerIdBits 则是 0000 0101。

- `next`：
记录下一个 TouchTarget 对象，由此组成链表。

对于单点触控，链表结构中只有一个 TouchTarget 对象，pointerIdBits 同样只包含了单个触摸点 id；而在多点触控下，当有多个触摸目标时，TouchTarget 多个对象组成链表，pointerIdBits 保存了多个 pointerId 信息。

记下来看下 `TouchTarget` 对象如何作用在 View 树的触摸事件分发中：

在 ViewGroup 中，会有一个 TouchTarget 类型的 `mFirstTouchTarget` 成员变量来记录当前的触摸目标。每当有新的 `ACTION_DOWN` 事件传递过来时，mFirstTouchTarget 会被重置为 **null**，如果有子 View 消费了该 `ACTION_DOWN` 事件，便会创建一个新的 TouchTarget 对象记录该子 View 为触摸目标并赋值到 `mFirstTouchTarget`。

而后会以 **mFirstTouchTarget 是否为 null 判定是否有目标子 View 消费了 `ACTION_DOWN` 事件**，否则会拦截掉后续其他的事件。该逻辑核心实现在 ViewGroup 的 dispatchTouchEvent 方法中：

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final int action = ev.getAction();
    final int actionMasked = action & MotionEvent.ACTION_MASK;
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        cancelAndClearTouchTargets(ev); // mFirstTouchTarget 重设为 null
        resetTouchState();
    }
    ...
    final boolean intercepted;
    if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        if (!disallowIntercept) {
            intercepted = onInterceptTouchEvent(ev);
            ev.setAction(action);
        } else {
            intercepted = false;
        }
    } else {
        // 此时 mFirstTouchTarget 为空，表示未有子 View 消费 ACTION_DOWN 事件
        // 后续的事件都直接在此处拦截
        intercepted = true;
    }
    ...
    // 如果当前为 ACTION_DOWN 事件且有子 View 消费了该事件
    // 创建新的 TouchTarget 对象为 mFirstTouchTarget
    ...
}
```

正是以上对 `mFirstTouchTarget` 变量的使用，实现了上文提及的事件消费连续性的机制。

### OnTouch 和 OnClick 回调

`OnTouchListener#onTouch` 和 `OnClickListener#onClick` 是日常开发中最常用到的触摸事件的监听，两者的含义分别是：

|接口|说明
|--|--
|OnTouchListener#onTouch|通过 `setOnTouchListener` 设置触摸事件的监听，`onTouch` 方法会多次回调一系列 **MotionEvent**。
|OnClickListener#onClick|通过 `setOnClickListener` 设置的一个 View 的“**点击**“行为监听，`onClick` 方法会在手势离开屏幕（即 **ACTION_UP**）时触发

当这两者监听器都设置时，其生效关系如下：

`OnTouchListener#onTouch` 方法会返回一个 **boolean** 值，当其返回 false 时，表示该触摸事件未被处理，会触发 `View#onTouchEvent` 方法，其中会进一步在 **ACTION_UP** 事件中触发 `OnClickListener#onClick` 来处理点击事件。若 `onTouch` 返回 true，则点击行为监听 `onClick` 方法将不会触发：

![](/img/posts/post-view-touch6.png) 

