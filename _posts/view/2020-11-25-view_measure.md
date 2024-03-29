---
layout: post
title: View 的测量与布局
author: JasonWu
tags:
  - Android
  - View
---

# View 的测量与布局

## View 树的构建
Android 系统包括应用内的 UI 界面都是可以划分为一个个的 View 视图，View 是一个树形结构，多层级的 View 树示例如下：
![](/img/posts/post-view-tree.png)

其中，需要注意的是，包含子 View 的父 View 必然是 `ViewGroup` 的实现类，其实现了 `ViewParent` 接口。

而顶层的 `ViewRooltImpl` 由[系列文章（二）]()可知，其并非一个真正的 View，它是整个窗口视图的管理类，同样实现了 `ViewParent` 接口，**在 View 树的结构层次上，可以看作是 RootView 的父 View**。

## 测量与布局概述
一个复杂的视图往往是由多层级的 View 嵌套而成，每个 View 本身以一个矩形的区域表示该 View 的内容。而描述一个矩形区域视图，无非是以下两点：
- 尺寸，即视图的宽和高
- 位置，即视图在其父视图当中的位置

View 的测量（`measure`）过程负责计算 View 的尺寸，布局（`layout`）则是负责计算 View 的位置关系。

而每个 View 的尺寸和位置是由自身的布局属性（一般可外部设置），父 View 的布局属性、测量结果，以及子 View 的布局属性、测量结果等等共同决定。所以一个复杂 View 的视图区域的构建流程通常是**自上而下递归**的，下面的分析将进一步体现该设计思路。

## View 的测量
先执行的是 View 的测量过程，整体流程时序图如下：

![](/img/posts/post-view-measure.png)

<!-- Title: View 的测量过程
participant ViewRootImpl
participant RootView
participant ChildView
participant ...

ViewRootImpl->ViewRootImpl: doTraversal()
ViewRootImpl->ViewRootImpl: performTraversals()

ViewRootImpl->RootView: performMeasure\n(MeasureSpec wm, MeasureSpec hm)
RootView->RootView: measure(wm, hm)
RootView->ChildView: onMeasure(wm, hm)
ChildView->ChildView: measure(wm, hm)
ChildView->...: onMeasure(wm, hm)
ChildView-\->RootView: w, h
RootView->RootView: setMeasuredDimension(w, h) -->

```light
frameworks/base/core/java/android/view
    - ViewRootImpl.java
    - View.java
    - ViewGroup.java
```

整个 View 树的测量发起点是 ViewRootImpl 的 `performTraversal` 方法，顾名思义，开始遍历整个 View 树。接着触发 `performMeasure`，开始 View 树的测量流程。

其中，对于一个 View 而言，测量过程整体可分为两步：
1. 根据父 View 的布局参数以及自身的布局参数计算生成两个 int 值 `widthMeasureSpec`、`heightMeasureSpec`，分别代表父 View 对 子 View 在宽高上的布局要求（对应上面时序图的 wm、hm）；
2. 结合父 View 的布局要求 MeasureSpec 以及子 View 的测量结果来计算自身的测量结果宽高 `MeasuredWidth`、`MeasuredHeight`。

> 用于 View 的测量结果依赖于父 View 和子 View 的测量结果，所以一个 View 的测量过程可能会触发多次才得于计算出最终的测量结果。

### MeasureSpec 的计算
MeasureSpec 表示的是**父 View 传递到子 View 的布局要求**，具体用一个 32 位的 int 值，可拆分为：

|高 2 位|低 30 位
|--|--
|测量模式 SpecMode|测量要求大小 SpecSize

SpecMode 表示的是测量模式，共三种，含义如下：

| SpecMode | 说明 |
|--|--|
| `EXACTLY` | 精确测量模式，父 View 已经确定子 View 的测量结果，测量值为 SpecSize 的值|
| `AT_MOST` | 最大值测量模式，子 View 可以根据需要的大小而定，最大不超过 SpecSize |
| `UNSPECIFIED` | 不定测量模式，父 View 对子 View 大小没有任何限制，通常用于内部多次测量过程中的临时测量结果，应用开发中较少用到 |

先来看下 RootView 的 MeasureSpec 计算逻辑。

对于窗口顶层的 RootView 来说，它的 MeasureSpec 由窗口参数和其自身的 LayoutParams 共同决定。在 ViewRootImpl 的 `getRootMeasureSpec(int windowSize, int rootDimension)` 方法中可整理到对应关系为：

| RootView - LayoutParams | MeasureSpec
|--|--
| dp / px | EXACTLY(viewSize) 
| match_parent | EXACTLY(windowSize) 
| wrap_content | AT_MOST(windowSize)

> 其中 windowSize 的默认值宽取 `com.android.internal.R.dimen.config_prefDialogWidth`，普通手机屏幕中为 320dp，高为手机屏幕高度。

对于往下层级的 View，它的 MeasureSpec 由父 View 的 MeasureSpec 和其自身的 LayoutParams 共同决定。在 ViewGroup 的 `getChildMeasureSpec` 方法中可整理到对应关系为：

| Child - LayoutParams | Parent - SpecMode<br>EXACTLY | Parent - SpecMode<br>AT_MOST | Parent - SpecMode<br>UNSPECIFIED
|--|--|--|--
| dp / px | EXACTLY(childSize) | EXACTLY(childSize) | EXACTLY(childSize)
| match_parent | EXACTLY(parentSize) | AT_MOST(parentSize) | UNSPECIFIED(parentSize)
| wrap_content | AT_MOST(parentSize) | AT_MOST(parentSize) | UNSPECIFIED(parentSize) 

### 实际测量宽高的计算
在一个 View 从父 View 那获取到宽高的 MeasureSpec 值之后，会在其 `onMeasure` 方法中完成自身实际宽高的测量。

对一个简单的 View 来说，实际的测量宽高值 `mMeasuredWidth`，`mMeasuredHeight` 直接取用 MeasureSpec 里的 SpecSize 即可，因此 `onMeasure` 方法的默认实现为（部分方法调用已合并）：
```java
// frameworks/base/core/java/android/view/View.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    mMeasuredWidth = getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec);
    mMeasuredHeight = getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec);

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}

public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED: // UNSPECIFIED 默认最小默认值
        result = size;
        break;
    case MeasureSpec.AT_MOST: // AT_MOST & EXACTLY 模式取 SpecSize
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

然而，对于一个包含子 View 的 ViewGroup 来说，**仅仅是知道父 View 对它的布局要求，是不足以完成自身宽高的计算的**。原因是当它的布局参数设置为 `wrap_conent` 时，其宽或高会需要子 View 的尺寸来适应。这时，需要逐级向下触发并完成子 View 的测量，**根据子 View 的测量结果来计算自身的测量宽高**。

以最常用的 `FrameLayout` 为例，FrameLayout 的布局特性是它的多个子 View 在其内布局是“可重叠”的，因此当 FrameLayout 自身的宽或高设置为 `wrap_conent` 时，它的测量宽或高的值理应大于等于它尺寸最大的那个子 View 的宽或高即可。按照这个思路，可以看到 FrameLayout 重写实现的 `onMeasure` 方法为：
```java
// frameworks/base/core/java/android/widget/FrameLayout.java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();
    ...
    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;
    // 先依次触发子 View 的测量
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            // 记录下子 View 的最大宽度和高度
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        }
    }
    ...
    // 根据子 View 的最大宽度 & 高度以及 MeasureSpec 的值计算自身真正的测量宽高
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));
    // 在自身的宽高完成测量后，子 View 的测量可能发现变化，因此重新触发子 View 的测量
    count = mMatchParentChildren.size();
    if (count > 1) {
        for (int i = 0; i < count; i++) {
            final View child = mMatchParentChildren.get(i);
            ...
            // 重新触发子 View 的测量
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

其中，`resolveSizeAndState` 方法的实现如下：
```java
// frameworks/base/core/java/android/view/View.java
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```

至此，View 的一次测量过程就算完成了，其成员变量 `mMeasureWidth`、`mMeasureHeigth` 对应为具体的测量结果宽高。

<!-- ### 针对多次重复测量的优化
上文提及到，由于一个 View 的 `measure` 方法可能会在一次完整的测量过程中会被多次调用，每一次调用又会递归的触发其子 View 的测量。如果每一次的测量传入的 MeasureSpec 值是相同的，那么后续的测量过程其实是无用多余的，因为只是在重复参数、结果一样的计算过程。因此，Android View 系统中，针对这样的情况，做了优化：**在每次测量过程完成后，会把传入的 MeasureSpec 值保存下来，在下一次测量时判断是否直接复用上一次的测量结果还是继续重新测量**。

优化实现在了 View 的 `measure` 方法中：
```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    // mMeasureCache 保存了由 widthMeasureSpec 和 heightMeasureSpec 拼接成 64 位的 long 型
    long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
    if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
    // 只有强制布局刷新以及新的 MeasureSpec 的值与旧的不同时，才往下执行继续测量过程
    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {
        ...
        int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            onMeasure(widthMeasureSpec, heightMeasureSpec);
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
        }
    }

    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;

    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
            (long) mMeasuredHeight & 0xffffffffL);
}
``` -->

## View 的布局
在 View 树的测量（`measure`）流程完成后，接着触发的是 View 的布局（`layout`）流程。相对于测量，布局的过程相对简单，整体时序图如下：

![](/img/posts/post-view-layout.png)

<!-- Title: View 的布局过程
participant ViewRootImpl
participant RootView
participant ChildView
participant ...

ViewRootImpl->RootView: performMeasure()
RootView->ChildView: ...
ViewRootImpl->RootView: performLayout()
RootView->RootView: layout(l,t,r,b)
RootView->RootView: setFrame(l,t,r,b)
RootView->ChildView: onLayout(l,t,r,b)
ChildView->...: layout() / ... -->

布局过程的核心是**计算一个 View 在其父 View 当中的位置**，在 Android View 系统中，是通过一个 View 相对于父 View 的 `left`、`top`、`right`、`bottom` 四个参数值来确定的（分别对应上时序图的 l、t、r、b）。其含义如下图所示：

![](/img/posts/post-view-layout2.png)

显然，在一个 View 内，会有：
```java
width = right - left
height = top - bottom
```

而后会调用 View 的 `layout` 方法，传入计算出来的 left、top、right、bottom 并记录，回调 `onLayout` 方法。代码如下（已部分省略）：
```java
public void layout(int l, int t, int r, int b) {
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b); // 

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        ...
    }
}

protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;

    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
        int oldWidth = mRight - mLeft;
        int oldHeight = mBottom - mTop;
        int newWidth = right - left;
        int newHeight = bottom - top;
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);
        ...
        // 记录下更新的后 left、top、right、bottom
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        ...            
    }
    return changed;
}
```

对于窗口顶层的 RootView 来说，由于没有父 View 做参照，故它的左上右下分别是：0，0，MeasureWidth，MeasureHegith。

对于往下层级的 View，它的左上右下由该 View 在父 ViewGroup 的布局属性决定。

在 ViewGroup 回调 `onLayout` 后，会触发到子 View 的 `layout` 方法，以此递归完成整个 View 树的布局过程。以 `RelativeLayout` 为例：

```java
// frameworks/base/core/java/android/widget/RelativeLayout.java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    final int count = getChildCount();

    for (int i = 0; i < count; i++) {
        View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            RelativeLayout.LayoutParams st =
                    (RelativeLayout.LayoutParams) child.getLayoutParams();
            child.layout(st.mLeft, st.mTop, st.mRight, st.mBottom);
        }
    }
}
```

至此，View 的树布局过程已完成。