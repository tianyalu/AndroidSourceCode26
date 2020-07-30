# `Android View`的绘制流程之测量、布局、绘制源码(API 26)分析

[TOC]

`Android`为开发者提供了一个`GUI`库，里面有很多控件，但这些控件不能满足需求时，就需要开发者自定义`View`了，此时需要了解`View`的三大绘制流程：测量（`measure`）、布局（`layout`）和绘制（`draw`）过程。  

## 一、预备知识

首先需要了解一下顶层视图`DecorView`以及`ViewRootImpl`对象的创建过程，以及其分发的三大绘制流程。

* `DecorView`：应用程序窗口内部所包含的视图对象的实际类型，它集成了`View`类，作为容器`ViewGroup`使用；  

* `ViewRoot`：相当于是`MVC`模型中的`Controller`，具有以下职责：
  
  > 1. 负责为应用程序窗口视图创建`Surface`；  
  > 2. 配合`WindowManagerService`来管理系统的应用程序窗口；  
  > 3. 负责管理、布局和渲染应用程序窗口视图的UI。  

### 1.1 顶层视图`DecorView`以及`ViewRootImpl`对象的创建过程

`Activity`组件在启动的过程中，会调用`ActivityThread`类的成员函数`handleLaunchActivity`，用来创建以及首激活`Activity`组件，以此从这个函数开始分析应用程序窗口的视图对象以及其所管理的`ViewRoot`对象的创建过程，如下图所示：  

![image](https://github.com/tianyalu/AndroidSourceCode26/raw/master/show/decorview_viewroot_create_sequence.png)  

上图第11步获取到的就是顶层视图`decor`，第13、14、15和16步就是将`decor`传递给`ViewRoot`，这样`ViewRoot`和`DecoView`就建立了关联。

### 1.2 顶层视图`DecorView`分发的三大绘制流程

在1.1图的第15步中，`ViewRoot`类的成员函数`setView`会调用`ViewRoot`类的另一个成员函数`requestLayout`来请求对顶层视图`decorView`作第一次布局及显示。下面从`ViewRoot`类的成员函数`requestLayout`开始，分析顶层视图`decorView`的绘制三大流程，如下图所示：  

![image](https://github.com/tianyalu/AndroidSourceCode26/raw/master/show/decorview_3draw_sequence.png)  

上图中的第6步会调用`ViewRootImpl`类的`performTraversals`方法，`performTraversals`方法会依次调用`performMeasure`方法、`performLayout`方法和`performDraw`方法来完成顶层视图`decorView`的测量过程（`measure`）、布局过程（`layout`）和绘制过程（`draw`）。

## 二、三大绘制流程

### 2.1 `View`的测量过程（`measure`）

1.2图第10步会遍历每一个子`View`，然后调用子`View`的`measure`方法（即第11步），继而开始子`View`的测量过程（`measure`）。`ViewGroup`类型的`View`和非`ViewGroup`类型的`View`的测量过程是不同的：非`ViewGroup`类型的`View`通过`onMeasure`方法就完成了其测量过程；而`ViewGroup`类型的`View`除了通过`onMeasure`方法完成自身的测量过程之外，还要再该该方法中完成遍历子`View`的`measure`方法，各个子`View`再去递归执行这个流程。

#### 2.1.1 非`ViewGroup`类型的`View`的测量过程

先通过如下的时序图，整体看一下测量流程：  

![image](https://github.com/tianyalu/AndroidSourceCode26/raw/master/show/view_measure_sequence.png)  

对于上面的步骤详细分析一下：第一步执行`View`类中的`measure`方法，该方法是一个`final`方法，则意味着子类不能重写该方法。`measure`方法会调用`View`类的`onMeasure`方法，`onMeasure`方法源码如下：  

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                       getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

此代码对应上图的第3、4、5、6、7步，先但第三步对应的`View`类的`getSuggestedMinimumWidth()`方法源码：  

```java
protected int getSuggestedMinimumWidth() {
  return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

从该方法中可以看到，当`View`没有设置背景时，此方法的返回值为`mMinWidth`，该值对应于`android:minWidth`属性所指定的值，如果未设置此属性，`mMinWidth`默认为0；当`View`设置了背景时，该方法返回`max(mMinWidth, mBackground.getMinimumWidth())`，下面看一下`Drawable`类中`getMinimumWidth`方法的源码：  

```java
public int getMinimumWidth() {
  final int intrinsicWidth = getIntrinsicWidth();
  return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
```

由上面的代码可以看出`getMinimumWidth`返回的是`View`的背景的原始宽度，如果没有原始宽度则返回0.

**`getSuggestedMinimumWidth()`方法小结：**  

> `View`未设置背景时，返回`android:minWidth`属性指定的值（可以为0）；  
> `View`设置背景时，返回`android:minWidth`属性指定的值与`View`背景的最小宽度中的最大值。  

下面看一下**最关键**的`View`类的`getDefaultSize`方法的源码（对应第4步）：  

```java
public static int getDefaultSize(int size, int measureSpec) {
  int result = size;
  int specMode = MeasureSpec.getMode(measureSpec);
  int specSize = MeasureSpec.getSize(measureSpec);

  switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
      result = size;
      break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
      result = specSize;
      break;
  }
  return result;
}
```

对于`MeasureSpec.UNSPECIFIED`测量模式，一般用于系统内部测量过程，该方法返回`getSuggestedMinimumWidth`方法得到的值；对于`MeasureSpec.AT_MOST`和`MeasureSpec.EXACTLY`这两种测量模式，该方法直接返回测量后的值（即父`View`通过`measure`方法传递过来的测量值，**也说明了下面注意事项的第一条**）。  

对于第5、6步，与3、4步类似，不再赘述。  

第7步`View`类的`setMeasuredDimension`方法调用了第8步中`View`类的`setMeasuredDimensionRaw`方法，该方法源码如下：  

```java
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
  mMeasuredWidth = measuredWidth;
  mMeasuredHeight = measuredHeight;

  mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

由上面的代码可知，`View`测量后的宽高被保存到`View`类的成员变量`mMeasuredWidth`和`mMeasuredHeight`中了，通过`getMeasuredWidth`和`getMeasuredHeight`方法获取到的就是这两个值。需要注意的是，在某些极端情况下，系统可能需要多次`measure`才能确定最终的测量宽高，此时，在`onMeasure`方法中拿到的测量宽高很肯能是不准确的，**一个好的习惯是在`onLayout`方法中去获取`View`最终的测量宽高**。

上面只是说在自定义`View`中什么时候获取最终的测量宽高，那么在`Activity`中什么时机可以获取`View`的测量宽高呢？《`Android`开发艺术探索》`P190`介绍如下：  

>1. 在`Activity/View#onWindowFocusChanged`方法中调用；
>2. 在`Activity`中的`onStart`方法中执行`View.pos`获取；
>3. 通过`ViewTreeObserver`获取；
>4. 通过手动执行`View.measure`获取。  

**有如下几点需要注意：**  

> 1. 直接继承`View`的自定义控件需要重写`onMeasure`方法，并且需要设置`wrap_content`时自身的大小，否则在布局中使用`wrap_content`就相当于使用`match_parent`；
> 2. 在自定义`View`时可以通过重写`onMeasure`方法设置`View`测量大小，这样的话你就抛弃了父容器通过`measure`方法传进来的建议测量值`MeasureSpec`。

#### 2.1.2 `ViewGroup`类型的`View`的测量过程

先通过如下的时序图，整体看一下测量流程：  

![image](https://github.com/tianyalu/AndroidSourceCode26/raw/master/show/viewgroup_measure_sequence.png)  

`ViewGroup`并没有定义其自身测量的具体过程（**即没有`onMeasure`方法**），这是因为`ViewGroup`是一个抽象类，其测量过程的`onMeasure`方法需要各个子类去具体实现，所以上图展示了`LinearLayout`的测量过程图。  

对于上图的步骤详细解析一下：第1步执行`View`类的`measure`方法，该方法是一个`final`方法，则意味着子类不能重写该方法，`measure`方法会调用`LinearLayout`类的`onMeasure`方法，该方法源码如下：  

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  if (mOrientation == VERTICAL) {
    measureVertical(widthMeasureSpec, heightMeasureSpec);
  } else {
    measureHorizontal(widthMeasureSpec, heightMeasureSpec);
  }
}
```

现只分析当`LinearLayout`的方向是垂直方向的情况，此时会执行该类的`measureVertical`方法，代码如下（仅展示我们关心的的逻辑代码）：  

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
	//...
  // See how tall everyone is. Also remember max width.
  for (int i = 0; i < count; ++i) {
    final View child = getVirtualChildAt(i);
			//...
      // Determine how big this child would like to be. If this or
      // previous children have given a weight, then we allow it to
      // use all available space (and we will shrink things later
      // if needed).
      //...
      measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                               heightMeasureSpec, usedHeight);

      final int childHeight = child.getMeasuredHeight();
			//...

      final int totalLength = mTotalLength;
      mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                              lp.bottomMargin + getNextLocationOffset(child));
			//...
    }
	//...
  
  // Add in our padding
  mTotalLength += mPaddingTop + mPaddingBottom;
  int heightSize = mTotalLength;
  // Check against our minimum height
  heightSize = Math.max(heightSize, getSuggestedMinimumHeight());
  // Reconcile our calculated size with the heightMeasureSpec
  int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
  heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
  //...
  setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                       heightSizeAndState);
	//...
}
```

由上面的代码可知`LinearLayout`类的`measureVertical`方法会遍历每一个子元素并且执行`LinearLayout`类的`measureChildBeforeLayout`方法对子元素进行测量，该方法内部会执行子元素的`measure`方法。源码中变量`mTotalLength`用来存放`LinearLayout`竖直方向上的当前高度，每遍历一个子元素，`mTotalLength`都会增加，增加的部分主要包括子元素自身的高度以及其在竖直方向上的`margin`。当测量完所有子元素后，`LinearLayout`会根据子元素的情况测量自身的大小，针对竖直的`LinearLayout`来说，它在水平方向的测量过程遵循`View`的测量过程，在竖直方向上有所不同：如果它的布局高度采取`match_parent`或具体的数值，那么其测量过程与`View`一致；如果它的布局中高度采用的是`wrap_content`，则**其高度是所有子元素所占用的高度总和，但是仍然不能超过父容器的剩余空间**，这个过程对应于`resolveSizeAndState`的源码：  

```java
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

下面来看一下`LinearLayout`类的`measureChildBeforeLayout`方法是如何测量子元素的？该方法的第4个和第6个参数分别代表在水平方向和垂直方向上`LinearLayout`已经被其它子元素占据的长度，该方法源码如下：  

```java
void measureChildBeforeLayout(View child, int childIndex,
                              int widthMeasureSpec, int totalWidth, int heightMeasureSpec,
                              int totalHeight) {
  measureChildWithMargins(child, widthMeasureSpec, totalWidth,
                          heightMeasureSpec, totalHeight);
}
```

`measureChildBeforeLayout`方法会调用`ViewGroup`类的`measureChildWidthMargins`方法，该方法源码如下：  

```java
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

`ViewGroup`类的`measureChildWithMargins`方法会调用子元素的`measure`方法对子元素进行测量。在对子元素测量之前会通过调用`ViewGroup`类的`getChildMeasureSpec`方法得到子元素宽高的`MeasureSpec`，从该方法的前两个参数可知，子元素`MeasureSpec`的创建与父容器的`MeasureSpec`、父容器的`padding`、子元素的`margin`和兄弟元素所占用的长度有关。该方法源码如下：  

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
  int specMode = MeasureSpec.getMode(spec);
  int specSize = MeasureSpec.getSize(spec);

  int size = Math.max(0, specSize - padding);

  int resultSize = 0;
  int resultMode = 0;

  switch (specMode) {
      // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
      if (childDimension >= 0) {
        resultSize = childDimension;
        resultMode = MeasureSpec.EXACTLY;
      } else if (childDimension == LayoutParams.MATCH_PARENT) {
        // Child wants to be our size. So be it.
        resultSize = size;
        resultMode = MeasureSpec.EXACTLY;
      } else if (childDimension == LayoutParams.WRAP_CONTENT) {
        // Child wants to determine its own size. It can't be
        // bigger than us.
        resultSize = size;
        resultMode = MeasureSpec.AT_MOST;
      }
      break;

      // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
      if (childDimension >= 0) {
        // Child wants a specific size... so be it
        resultSize = childDimension;
        resultMode = MeasureSpec.EXACTLY;
      } else if (childDimension == LayoutParams.MATCH_PARENT) {
        // Child wants to be our size, but our size is not fixed.
        // Constrain child to not be bigger than us.
        resultSize = size;
        resultMode = MeasureSpec.AT_MOST;
      } else if (childDimension == LayoutParams.WRAP_CONTENT) {
        // Child wants to determine its own size. It can't be
        // bigger than us.
        resultSize = size;
        resultMode = MeasureSpec.AT_MOST;
      }
      break;

      // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
      if (childDimension >= 0) {
        // Child wants a specific size... let him have it
        resultSize = childDimension;
        resultMode = MeasureSpec.EXACTLY;
      } else if (childDimension == LayoutParams.MATCH_PARENT) {
        // Child wants to be our size... find out how big it should
        // be
        resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
        resultMode = MeasureSpec.UNSPECIFIED;
      } else if (childDimension == LayoutParams.WRAP_CONTENT) {
        // Child wants to determine its own size.... find out how
        // big it should be
        resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
        resultMode = MeasureSpec.UNSPECIFIED;
      }
      break;
  }
  //noinspection ResourceType
  return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

`ViewGroup`类的`getChildMeasureSpec`方法的逻辑可以通过下表来说明（表中的`parentSize`是指父容器目前可使用的大小，参考《`Android`开发艺术探索》182页）：  

| childLayoutParams\parentSpecMode |       EXACTLY        |       AT_MOST        |     UNSPECIFIED     |
| :------------------------------: | :------------------: | :------------------: | :-----------------: |
|              dp/px               | EXACTLY (childSize)  | EXACTLY (childSize)  | EXACTLY (childSize) |
|           match_parent           | EXACTLY (parentSize) | AT_MOST (parentSize) |   UNSPECIFIED (0)   |
|           wrap_content           | AT_MOST (parentSize) | AT_MOST (parentSize) |   UNSPECIFIED (0)   |

`ViewGroup`类的`getChildMeasureSpec`方法返回子元素宽高的`MeasureSpec`，然后将子元素宽高的`MeasureSpec`作为`measure`方法的参数。

**小结：**

> 1.  父`View`会遍历测量每一个子`View`(通常会使用`ViewGroup`类的`measureChildWidthMargins`方法)，然后调用子`View`的`measure`方法并且将测量后的宽高作为`measure`方法的参数，但这只是父`View`的建议值，子`View`可以通过重写`onMeasure`方法来改变测量值；
> 2. 非`ViewGroup`类型的`View`自身的测量是在非`ViewGroup`类型的`View`的`onMeasure`方法中进行测量的；
> 3. `ViewGroup`类型的`View`自身的测量是在`ViewGroup`类型的`View`的`onMeasure`方法中进行测量的；
> 4. 直接继承`ViewGroup`的自定义控件需要重写`onMeasure`方法并且设置`wrap_content`时自身的大小，否则布局中使用`wrap_content`就相当于使用`match_parent`，具体原因可以通过上面的表格说明。

### 2.2 `View`的布局过程（`layout`）

1.2图的第17步会遍历每一个子元素，并且调用子元素的`layout`方法，继而开始进行子元素的布局过程。`layout`过程相较于`measure`过程比较简单，`layout`方法用来确定`View`本身的位置，而`onLayout`方法用来确定所有子元素的位置。`ViewGroup`类型的`View`和非`ViewGroup`类型的`View`的布局过程是不同的：非`ViewGroup`类型的`View`通过`layout`方法就完成了其布局过程；而`ViewGroup`类型的`View`除了通过`layout`方法完成自身布局过程外，还需要调用`onLayout`方法去遍历子元素并调用子元素的`layout`方法，各个子`View`再递归执行这个流程。

#### 2.2.1 非`ViewGroup`类型的`View`的布局过程

先通过如下的时序图，整体看一下布局过程：

![image](https://github.com/tianyalu/AndroidSourceCode26/raw/master/show/view_layout_sequence.png)  

对上面的时序图分析一下，第一步执行`View`类的`layout`方法，源码如下：

```java
public void layout(int l, int t, int r, int b) {
  if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
    onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
    mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
  }

  int oldL = mLeft;
  int oldT = mTop;
  int oldB = mBottom;
  int oldR = mRight;

  boolean changed = isLayoutModeOptical(mParent) ?
    setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

  if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
    onLayout(changed, l, t, r, b);

    if (shouldDrawRoundScrollbar()) {
      if(mRoundScrollbarRenderer == null) {
        mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
      }
    } else {
      mRoundScrollbarRenderer = null;
    }

    mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnLayoutChangeListeners != null) {
      ArrayList<OnLayoutChangeListener> listenersCopy =
        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
      int numListeners = listenersCopy.size();
      for (int i = 0; i < numListeners; ++i) {
        listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
      }
    }
  }

  mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
  mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

  if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
    mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
    notifyEnterOrExitForAutoFillIfNeeded(true);
  }
}
```

如果`isLayoutModeOptical(mParent)`方法返回`true`，那么就会执行`setOpticalFrame`方法，否则会执行`setFrame`方法，并且`setOpticalFrame`方法内部会执行`setFrame`方法，所以无论如何`setFrame`方法都会执行的。

第二步`layout`方法会调用`View`类的`setFrame`方法，部分关键代码如下：

```java
protected boolean setFrame(int left, int top, int right, int bottom) {
  boolean changed = false;
	//...
  if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
    changed = true;
		//...
    int oldWidth = mRight - mLeft;
    int oldHeight = mBottom - mTop;
    int newWidth = right - left;
    int newHeight = bottom - top;
    boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

    // Invalidate our old position
    invalidate(sizeChanged);

    mLeft = left;
    mTop = top;
    mRight = right;
    mBottom = bottom;
		//...

    if (sizeChanged) {
      sizeChange(newWidth, newHeight, oldWidth, oldHeight);
    }
    //...
  }
  return changed;
}
```

由上面代码可知，`setFrame`方法是用来设置`View`的四个顶点的位置的，即初始化`mLeft`、`mTop`、`mRight`和`mBottom`这四个值。`View`的顶点一旦确定，那么`View`在父容器中的位置也就确定了。

第三步`layout`方法接着调用`View`类的`onLayout`方法，这个方法的作用是来确定子元素的位置的。由于非`ViewGroup`类型的`View`没有子元素，所以`View`类的`onLayout`方法为空。

#### 2.2.2 `ViewGroup`类型的`View`的布局过程

先通过如下的时序图，整体看一下布局过程：  

![image](https://github.com/tianyalu/AndroidSourceCode26/raw/master/show/viewgroup_layout_sequence.png)  

上图为`LinearLayout`布局的时序图，因为`ViewGroup`的`onLayout`方法是抽象方法，所以选择其子类`LinearLayout`进行分析。

对于上面的时序图，第一步执行`ViewGroup`的`layout`方法，该方法是一个`final`方法，即子类无法重写该方法。其源码如下：

```java
@Override
public final void layout(int l, int t, int r, int b) {
  if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
    if (mTransition != null) {
      mTransition.layoutChange(this);
    }
    super.layout(l, t, r, b);
  } else {
    // record the fact that we noop'd it; request layout when transition finishes
    mLayoutCalledWhileSuppressed = true;
  }
}
```

第二步`ViewGroup`类的`layout`方法会调用`View`类的`layout`方法，第三步`View`类的`layout`方法调用`View`类的`setFrame`方法，这两步与上面讨论的**非`ViewGroup`类型的`View`的布局过程**的第一、二步相同，不再赘述。直接看第四步`View`类的`layout`方法调用`LinearLayout`类的`onLayout`方法，其源码如下：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
  if (mOrientation == VERTICAL) {
    layoutVertical(l, t, r, b);
  } else {
    layoutHorizontal(l, t, r, b);
  }
}
```

我们只分析当`LinearLayout`的方向是垂直方向的情况，此时会执行`LinearLayout`类的`layoutVertical`方法，源码如下：

```java
void layoutVertical(int left, int top, int right, int bottom) {
  final int paddingLeft = mPaddingLeft;

  int childTop;
  int childLeft;

  // Where right end of child should go
  final int width = right - left;
  int childRight = width - mPaddingRight;

  // Space available for child
  int childSpace = width - paddingLeft - mPaddingRight;

  final int count = getVirtualChildCount();

  final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
  final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

  switch (majorGravity) {
    case Gravity.BOTTOM:
      // mTotalLength contains the padding already
      childTop = mPaddingTop + bottom - top - mTotalLength;
      break;

      // mTotalLength contains the padding already
    case Gravity.CENTER_VERTICAL:
      childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
      break;

    case Gravity.TOP:
    default:
      childTop = mPaddingTop;
      break;
  }

  for (int i = 0; i < count; i++) {
    final View child = getVirtualChildAt(i);
    if (child == null) {
      childTop += measureNullChild(i);
    } else if (child.getVisibility() != GONE) {
      final int childWidth = child.getMeasuredWidth();
      final int childHeight = child.getMeasuredHeight();

      final LinearLayout.LayoutParams lp =
        (LinearLayout.LayoutParams) child.getLayoutParams();

      int gravity = lp.gravity;
      if (gravity < 0) {
        gravity = minorGravity;
      }
      final int layoutDirection = getLayoutDirection();
      final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
      switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
        case Gravity.CENTER_HORIZONTAL:
          childLeft = paddingLeft + ((childSpace - childWidth) / 2)
            + lp.leftMargin - lp.rightMargin;
          break;

        case Gravity.RIGHT:
          childLeft = childRight - childWidth - lp.rightMargin;
          break;

        case Gravity.LEFT:
        default:
          childLeft = paddingLeft + lp.leftMargin;
          break;
      }

      if (hasDividerBeforeChildAt(i)) {
        childTop += mDividerHeight;
      }

      childTop += lp.topMargin;
      setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
      childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

      i += getChildrenSkipCount(child, i);
    }
  }
}
```

可以看到`LinearLayout`类的`onLayout`方法会遍历每一个子元素，然后调用该类的`setChildFrame`方法，`setChildFrame`方法又会调用子元素的`layout`方法来对子元素进行布局，其源码如下：

```java
private void setChildFrame(View child, int left, int top, int width, int height) {
  child.layout(left, top, left + width, top + height);
}
```

### 2.3 `View`的绘制过程（`draw`）

1.2图的第26步会遍历每一个子`View`，并且调用子元素的`draw`方法，继而开始进行子`View`的绘制过程。

先通过如下时序图，整体看一下绘制过程：

![image](https://github.com/tianyalu/AndroidSourceCode26/raw/master/show/view_draw_sequence.png)  

上图是`LinearLayout`的绘制时序图，因为`View`的`onDraw`方法是空方法，所以选择`ViewGroup`的子类`LinearLayout`进行分析。

`LinearLayout`的绘制过程遵循如下步骤：

> 1. 绘制背景；  
> 2. 绘制自己（绘制分割线）；  
> 3. 绘制子`View`（`dispatchDraw`）;  
> 4. 绘制前景。

`Android`中是通过`View`类的`draw`方法来实现上述的4步的，其源码如下：  

```java
/**
 * Manually render this view (and all of its children) to the given Canvas.
 * The view must have already done a full layout before this function is
 * called.  When implementing a view, implement
 * {@link #onDraw(android.graphics.Canvas)} instead of overriding this method.
 * If you do need to override this method, call the superclass version.
 *
 * @param canvas The Canvas to which the View is rendered.
 */
@CallSuper
public void draw(Canvas canvas) {
  final int privateFlags = mPrivateFlags;
  final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
    (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
  mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

  /*
   * Draw traversal performs several drawing steps which must be executed
   * in the appropriate order:
   *
   *      1. Draw the background
   *      2. If necessary, save the canvas' layers to prepare for fading
   *      3. Draw view's content
   *      4. Draw children
   *      5. If necessary, draw the fading edges and restore layers
   *      6. Draw decorations (scrollbars for instance)
   */

  // Step 1, draw the background, if needed
  int saveCount;

  if (!dirtyOpaque) {
    drawBackground(canvas);
  }

  // skip step 2 & 5 if possible (common case)
  final int viewFlags = mViewFlags;
  boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
  boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
  if (!verticalEdges && !horizontalEdges) {
    // Step 3, draw the content
    if (!dirtyOpaque) onDraw(canvas);

    // Step 4, draw the children
    dispatchDraw(canvas);

    drawAutofilledHighlight(canvas);

    // Overlay is part of the content and draws beneath Foreground
    if (mOverlay != null && !mOverlay.isEmpty()) {
      mOverlay.getOverlayView().dispatchDraw(canvas);
    }

    // Step 6, draw decorations (foreground, scrollbars)
    onDrawForeground(canvas);

    // Step 7, draw the default focus highlight
    drawDefaultFocusHighlight(canvas);

    if (debugDraw()) {
      debugDrawFocus(canvas);
    }

    // we're done...
    return;
  }
  //...
}
```

从这个方法的注释可知：当自定义`View`需要绘制时，应该实现`View`类的`onDraw`方法而不是重新该类的`draw`方法；如果的确需要重新`draw`方法，必须在重写时调用父类的`draw`方法。

上面的代码很明显地说明了`View`绘制过程的4步。由于`View`类无法确定自己是否有子元素，所以`View`类的`dispatchDraw`方法是空的，我们来看`ViewGroup`类的`dispatchDraw`方法的源码（我们截取部分感兴趣的代码）：

```java
@Override
protected void dispatchDraw(Canvas canvas) {
  boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
  final int childrenCount = mChildrenCount;
  final View[] children = mChildren;
	//...

  boolean more = false;
  final long drawingTime = getDrawingTime();

  if (usingRenderNodeProperties) canvas.insertReorderBarrier();
  final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
  int transientIndex = transientCount != 0 ? 0 : -1;
  // Only use the preordered list if not HW accelerated, since the HW pipeline will do the
  // draw reordering internally
  final ArrayList<View> preorderedList = usingRenderNodeProperties
    ? null : buildOrderedChildList();
  final boolean customOrder = preorderedList == null
    && isChildrenDrawingOrderEnabled();
  for (int i = 0; i < childrenCount; i++) {
    while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
      final View transientChild = mTransientViews.get(transientIndex);
      if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
          transientChild.getAnimation() != null) {
        more |= drawChild(canvas, transientChild, drawingTime);
      }
      transientIndex++;
      if (transientIndex >= transientCount) {
        transientIndex = -1;
      }
    }

    final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
    final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
    if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
      more |= drawChild(canvas, child, drawingTime);
    }
  }
 	//...
}
```

该方法会遍历每一个子元素，然后调用`ViewGroup`类的`drawChild`方法对子元素进行绘制，其源码如下：

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
  return child.draw(canvas, this, drawingTime);
}
```

## 三、与`View`生命周期相关的常用的回调方法

* `onFocusChanged(boolean, int, android.graphics.Rect)`：该方法在当前`View`获得或失去焦点时被回调；
* `onWindowFocusChanged(boolean)`：该方法在包含当前`View`的`window`获得或失去焦点时被回调；
* `onAttachedToWindow()`：该方法在当前`View`被附到一个`window`上时被调用；
* `onDetachedFromWindow()`：该方法在当前`View`从一个`window`上分离时被调用；
* `onVisibilityChanged(View, int)`：该方法在当前`View`或其祖先的可见性改变时被调用；
* `onWindowVisibilityChanged(int)`：该方法在包含当前`View`的`window`可见性改变时被回调。

## 四、参考资料

1. [Android应用程序窗口（Activity）的视图对象（View）的创建过程分析](https://blog.csdn.net/luoshengyang/article/details/8245546)  
2. [Android View的测量、布局、绘制流程源码分析及自定义View实例演示](https://www.jianshu.com/p/fd25d552c500)  

