---
title: ScrollView嵌套LitView显示问题探究
date: 2021-12-22 19:36:53
tags:
- 源码
categories: 
- 疑难杂症
#description: " "
---

#### ScrollView嵌套ListView显示不全问题及解决

开发过程中如果ScrollView嵌套ListView并且layout_height都为match_parent，会发现ListView显示不全只显示第一个条目，在上面测量过程和父View对子View测量分析中我们可以知道View的高度是父VIew和子View配合协作的结果，下面就从源码角度去分析原因。

##### ScrollView高度怎么确定的

ScrollView高度是怎么确定的，为什么在布局里指定match_parent就填充满屏幕呢

首先来看ScrollView的onMeasure方法

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    
    // 忽略部分代码……
}
```

调用了父View的onMeasure也就是FrameLayout的onMeasure

```java
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        
        // 忽略部分代码……
        
        // 保存测量尺寸
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
}
```

可以看到ScrollView的高度由heightMeasureSpec确定，heightMeasureSpec哪里来的呢？当然是它的父View也就是DecorView的子View就是mContentParent传递过来的约束（为什么是mContentParent可查看Activiey的setContentView源码），mContentParent的MeasureSpec来自它的父View就是DecorView，它测量是在ViewRootImpl的performTraversals中

```java
private void performTraversals() {
	// 忽略部分代码……
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
     // Ask host how big it wants to be
     performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

可以看到DecorView的MeasureSpec是由getRootMeasureSpec来的，根据DecorView的布局信息LayoutParms类型指定，DecorView的布局是在PhoneWindow的generateLayout中指定的

```java
protected ViewGroup generateLayout(DecorView decor) {
	// 忽略部分代码……
    
    // 根据窗口features确定layout以其中一个为例
    int layoutResource;
	int features = getLocalFeatures();
    layoutResource = R.layout.screen_simple;
    
}
```

screen_simple.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

可以看到DecorView的布局信息LayoutParams指定的match_parent,在回头看getRootMeasureSpec

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

由于布局信息LayoutParams是match_parent，所以DecorView的MeasureSpec尺寸是窗体可用空间，模式是EXACTLY。

mContentParent的MeasureSpec从哪儿来呢，来自DecorView的onMeasure对子view测量过程中传递而来，DecorView继承Framelayout最终会调用ViewGroup的getChildMeasureSpec

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    // 父view的MeasureSpec约束
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;
	
    // 通过父View的MeasureSpec结合子View的布局信息
    // 计算出子View的MeasureSpec约束
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

因为DecorView的specMode是EXACTLY，size是窗体可用空间，从screen_simple布局文件中我们可以看到mContentParent的布局信息LayoutParams为match_parent可以得出mContentParent的specSize为窗体可用空间specMode为EXACTLY，而mContentParent又是一个FrameLayout同DecorView测量过程一样进而可以得出ScrollView的specSize为窗体可用空间specMode为EXACTLY

而我们知道View测量尺寸有两个过程一个是父View获取子VIew的约束MeasureSpec并传递给子View，另一个是setMeasureDimension最终确定测量尺寸。ScrollView的setMeasureSpec是由onMeasure->Framelayout的onMeasure->setMeasureDimension

FrameLayout.java

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
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

    // Account for padding too
    maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
    maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

    // Check against our minimum height and width
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    // Check against our foreground's minimum height and width
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
        maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
    }
	
    // 确定最终测量尺寸
    // maxHeight是子View高度这里就是ListView高度
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));
}
```

setMeasureDimension的measureHeight又调用了resolveSizeAndState

```java
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    // size在这里就是ListView高度
    // specMode有上面分析知道specMode是EXACTLY
    final int specMode = MeasureSpec.getMode(measureSpec);
    // specSize由上面代码分析就是窗体高度
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    // ScrollView的specMode跟layout_height对应关系
    // 可根据上面FrameLayout的measureChildWithMargins代码分析得出
    switch (specMode) {
        // ScrollView的layout_height指定wrap_content
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
         // ScrollView的layout_height指定match_parent
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



结论：通过源码分析可以得出，ScrollView嵌套ListView当ScrollView高度layout_height为match_parent时高度为窗体可用高度，layout_height为wrap_content时高度为ListView的高度

##### ListView高度怎么确定

旧话重提View的高度是父View和子View配合协作的结果，先看ListView的onMeasure方法

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // Sets up mListPadding
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    int childWidth = 0;
    int childHeight = 0;
    int childState = 0;

    mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
    if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED
            || heightMode == MeasureSpec.UNSPECIFIED)) {
        final View child = obtainView(0, mIsScrap);

        // Lay out child directly against the parent measure spec so that
        // we can obtain exected minimum width and height.
        // 如果父View传递的specMode约束为UNSPECIFIED测量第一个条目高度childHeight
        measureScrapChild(child, 0, widthMeasureSpec, heightSize);

        childWidth = child.getMeasuredWidth();
        childHeight = child.getMeasuredHeight();
        childState = combineMeasuredStates(childState, child.getMeasuredState());

        if (recycleOnMeasure() && mRecycler.shouldRecycleViewType(
                ((LayoutParams) child.getLayoutParams()).viewType)) {
            mRecycler.addScrapView(child, 0);
        }
    }

    if (widthMode == MeasureSpec.UNSPECIFIED) {
        widthSize = mListPadding.left + mListPadding.right + childWidth +
                getVerticalScrollbarWidth();
    } else {
        widthSize |= (childState & MEASURED_STATE_MASK);
    }

    if (heightMode == MeasureSpec.UNSPECIFIED) {
        // 如果父View传递的约束specMode为UNSPECIFIED
        // ListView高度就为第一个条目高度childHeight（另外还有padding，edge）
        heightSize = mListPadding.top + mListPadding.bottom + childHeight +
                getVerticalFadingEdgeLength() * 2;
    }
	
    if (heightMode == MeasureSpec.AT_MOST) {
        // TODO: after first layout we should maybe start at the first visible position, not 0
        // 如果父View传递的约束specMode是AT_MOST
        // ListView高度为所有子条目高度累加结果
        heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
    }
	// 保存测量尺寸 heightSize就是ListView测量高度
    setMeasuredDimension(widthSize, heightSize);

    mWidthMeasureSpec = widthMeasureSpec;
}
```

从代码来看如果父View传递的约束specMode为UNSPECIFIED时ListView高度heightSize为第一个条目高度，跟我们开头说的现象一致，那么父View->ScrollView传递的约束specMode是不是UNSPECIFIED呢，来看ScrollView的onMeasure

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    // 如果ScrollView的fillViewport为false测量结束
    if (!mFillViewport) {
        return;
    }
    
	// 如果fillViewport为true，重新测量
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    if (heightMode == MeasureSpec.UNSPECIFIED) {
        return;
    }

    if (getChildCount() > 0) {
        // 获取第一个条目child，ScrollView也只能有一个子View
        final View child = getChildAt(0);
        final int widthPadding;
        final int heightPadding;
        final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;
        final FrameLayout.LayoutParams lp = (LayoutParams) child.getLayoutParams();
        if (targetSdkVersion >= VERSION_CODES.M) {
            widthPadding = mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin;
            heightPadding = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin;
        } else {
            widthPadding = mPaddingLeft + mPaddingRight;
            heightPadding = mPaddingTop + mPaddingBottom;
        }

        final int desiredHeight = getMeasuredHeight() - heightPadding;
        // 如果child测量高度小于ScrollView高度，重新测量
        if (child.getMeasuredHeight() < desiredHeight) {
            final int childWidthMeasureSpec = getChildMeasureSpec(
                    widthMeasureSpec, widthPadding, lp.width);
            // 设置约束specMode为EXACTLY
            final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                    desiredHeight, MeasureSpec.EXACTLY);
            // 重新测量
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

从源码可以看到ScrollView子View可能会进行两次测量，第一次测量调用父类的Measure方法进行测量，如果fillViewport为true会进行二次测量，下面对两次测量进行分析

第一次测量

调用父类onMeasure方法，ScrollView的父类是FrameLayout来看它的onMeasure方法

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            // 对子View进行测量
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
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

   // 忽略部分代码……

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));
  
}
```

FrameLayout的onMeasure会调用measureChildWithMargins方法对子View进行测量，需要注意的是ScrollView重写了measureChildWithMargins方法，所以看ScrollView的measureChildWithMargins

```java
@Override
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int usedTotal = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin +
            heightUsed;
    
    // 将子View约束specMode设置为UNSPECIFIED
    final int childHeightMeasureSpec = MeasureSpec.makeSafeMeasureSpec(
            Math.max(0, MeasureSpec.getSize(parentHeightMeasureSpec) - usedTotal),
            MeasureSpec.UNSPECIFIED);
    
	// 调用子View的measure方法进行测量
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

所以到这里真相大白了，ScrollView传递给子View的约束specMode为UNSPECIFIED，所以导致了显示不全。另外上面提到了如果ScrollView的fillViewport为true子View会进行二次测量会对显示结果有什么影响呢

Go ahead

二次测量

在继续回看ScrollView的onMeasure

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    // 如果ScrollView的fillViewport为false测量结束
    if (!mFillViewport) {
        return;
    }
    
	// 如果fillViewport为true，重新测量
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    if (heightMode == MeasureSpec.UNSPECIFIED) {
        return;
    }

    if (getChildCount() > 0) {
        // 获取第一个条目child，ScrollView也只能有一个子View
        final View child = getChildAt(0);
        final int widthPadding;
        final int heightPadding;
        final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;
        final FrameLayout.LayoutParams lp = (LayoutParams) child.getLayoutParams();
        if (targetSdkVersion >= VERSION_CODES.M) {
            widthPadding = mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin;
            heightPadding = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin;
        } else {
            widthPadding = mPaddingLeft + mPaddingRight;
            heightPadding = mPaddingTop + mPaddingBottom;
        }

        final int desiredHeight = getMeasuredHeight() - heightPadding;
        // 如果child测量高度小于ScrollView高度，重新测量
        if (child.getMeasuredHeight() < desiredHeight) {
            final int childWidthMeasureSpec = getChildMeasureSpec(
                    widthMeasureSpec, widthPadding, lp.width);
            // 设置约束specMode为EXACTLY
            final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                    desiredHeight, MeasureSpec.EXACTLY);
            // 重新测量
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

现在在来看这段代码就很清晰了，如果fillViewport为true进行二次测量时ScrollView传递给子View的specMode为EXACTLY结合上面ListView的onMeasure代码分析可以知道如果fillViewport为true的情况下ListView的高度为子条目高度之和也会是我们预期结果。

##### 结论

默认情况下ScrollView嵌套ListView只会显示第一个条目，如果ScrollView的fillViewport为true会进行二次测量结果ListView高度就是预期的所有子条目高度之和。

##### 解决方案

通过上面源码分析很容易找到解决方案

1.重写ListView修正父View传递的约束specMode为AT_MOST或者EXACTLY

```java
/**
 *
 *@author hezd
 *Create on 2021/12/22 16:51
 */
class MyListView(context: Context?, attrs: AttributeSet?) : ListView(context, attrs) {
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val size = MeasureSpec.getSize(heightMeasureSpec)
        val newHeightSpec = MeasureSpec.makeMeasureSpec(size,MeasureSpec.EXACTLY)
		// val newHeightSpec = MeasureSpec.makeMeasureSpec(size,MeasureSpec.AT_MOST)
        super.onMeasure(widthMeasureSpec, newHeightSpec)
    }
}
```

2.将ScrollVIew的fillVIewport属性设置为true

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

    </data>

    <ScrollView
        android:fillViewport="true"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity">

        <ListView
            android:id="@+id/listView"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />

    </ScrollView>
</layout>
```

两种方案对比：第一种方式最优因为只测量一次，第二种方式会两次测量性能有损耗
