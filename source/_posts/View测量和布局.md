---
title: View测量和布局
date: 2021-12-22 19:12:46
tags:
- View测量
- View布局
categories: 
- Android应用层
- View
---

#### 测量

从根View开始递归调用每一级子View的measure方法进行测量，实际测量工作是在onMeasure中完成，最终将测量尺寸保存以便后面布局时使用。

简单来说分两个过程

①父View获取子View的MeasureSpec约束并传递给子View

②子View调用setMeasureDimension最终确定测量尺寸并保存，保存之前通常还会进行测量尺寸修正

View.java


```java
/**
  * This is called to find out how big a view should be. The parent
  * supplies constraint information in the width and height parameters.
  * 从文档注释可以看出：widthMeasureSpec和heightMeasureSpec是父View对子View尺寸测量的一个约束信息
 */
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
     //忽略部分代码...
     if (forceLayout || needsLayout) {
            // 忽略部分代码...
         
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
     }
 }

/**
* 实际测量工作是onMeasure方法
*/
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 保存测量结果
  	setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

ViewGroup并没有实现onMeasure因为它是一个抽象类，每一个实现类的具体onMeasure行为不同，都是对子View遍历测量，RelativeLayout中是调用measureChild而LinearLayout是mearsureChildWithMargins以RelativeLayout为例

RelativeLayout.java

```java
 	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
         // 忽略部分代码...
        
         int width = 0;
         int height = 0;
         for (int i = 0; i < count; i++) {
             // ignore some code...
             // 遍历测量每一个子View
         	measureChild(child, params, myWidth, myHeight);
         }
        
        // 保存测量结果
        setMeasuredDimension(width, height);
    }

	private void measureChild(View child, LayoutParams params, int myWidth, int myHeight) {
        // 获取子View的MeasureSpec约束
        int childWidthMeasureSpec = getChildMeasureSpec(params.mLeft,
                params.mRight, params.width,
                params.leftMargin, params.rightMargin,
                mPaddingLeft, mPaddingRight,
                myWidth);
        int childHeightMeasureSpec = getChildMeasureSpec(params.mTop,
                params.mBottom, params.height,
                params.topMargin, params.bottomMargin,
                mPaddingTop, mPaddingBottom,
                myHeight);
        //调用子View的measure测量并将约束传递给子View
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

measureChild最终调用getChildMeasureSpec获取子View的MeasureSpec约束，此方法在后面父View如何对子View进行测量部分在做详细解析

#### 布局

从根布局开始递归调用每一级子View的layout方法进行布局，如果有子View还会调用onLayout，将测量过程中得到的尺寸和位置传递给子View并保存，以便确定View最终在屏幕中显示的位置

View.java

```java
 public void layout(int l, int t, int r, int b) {
        // 保存位置坐标
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            // 如果是ViewGroup需要对子View进行布局
            onLayout(changed, l, t, r, b);
        }
     	// 调用setFrame保存位置坐标
     	boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
 }

 /**
 	* View的onLayout是空实现,ViewGroup需要重写此方法
 	* 对子View进行布局
     * Called from layout when this view should
     * assign a size and position to each of its children.
     * 
     */
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```

如果是自定义View我们需要根据需求自定义layout规则，例如流布局，具体实现会在[《自定义ViewGroup》](https://hezd.github.io/2021/12/24/%E8%87%AA%E5%AE%9A%E4%B9%89ViewGroup/ "《自定义ViewGroup》")章节讲解，文章链接：自定义ViewGroup

#### MeasureSpec

```java
/**
 * A MeasureSpec encapsulates the layout requirements passed from parent to child.
 * Each MeasureSpec represents a requirement for either the width or the height.
 * A MeasureSpec is comprised of a size and a mode...
 */
public static class MeasureSpec 
```

从文档注释可以看到MeasureSpec包含了父View对子View的布局要求也就是约束信息

MeasureSpec由32位组成的整型数，前两位代表测量模式mode后面30位代表specSize，mode有三种分为Exactly，At_most，unspecified

Exactly：父View指定了精确值

AT_MOST：父View指定了最大值

UNSPECIFIED:未指定，子View想要多大就多大



#### 父View如何对子View进行测量

尺寸测量是一个父View和子View配合协作的过程，父View在测量过程中会计算子View的MeasureSpec约束并传递给子View，子View在根据MeasureSpec约束和自己的布局信息计算出实际尺寸。因此计算子View的MeasureSpec约束和子View最终测量决定了子View最终的测量尺寸下面分别来看：

###### 子View的约束

在上面测量过程中介绍到ViewGroup没有onMeasure的具体实现因为不同的ViewGroup有不同行为要求，LinearLayout和ListView都是由ViewGroup的getChildMeasureSpec类获取子View的MeasureSpec约束，RelativeLayout是自己内部的getChildMeasure来获取约束，我们把ViewGroup的getChildMeasureSpec作为一个通用实现来看

ViewGroup.java

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    // 父View的MeasureSpec
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
	
    // 可用空间
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // 如果父View的specMode是EXACTLY
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

    // 如果父View的specMode是AT_MOST
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

    // 如果父View的specMode是UNSPECIFIED
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

结论：

如果父View的测量模式是EXACTLY

- 如果子View布局参数指定了确切值那么测量尺寸就取确切值测量模式为EXACTLY
- 如果子View布局参数LayoutParams是MATCH_PARENT那么测量尺寸为剩余空间测量模式为EXACTLY
- 如果子View的布局参数LayoutParams是WRAP_CONTENT那么测量尺寸为剩余空间测量模式为AT_MOST

如果父View的测量模式是AT_MOST

- 如果子View布局参数指定了精确值那么测量尺寸为精确值测量模式为EXACTLY
- 如果子View布局参数LayoutParams为MATCH_PARENT那么测量尺寸为剩余空间测量模式为AT_MOST
- 如果子View布局参数LayoutParams为WRAP_CONTENT那么测量尺寸为剩余空间测量模式为AT_MOST

如果父View的测量模式是UNSPECIFIED

- 如果子View布局参数指定了精确值那么测量尺寸为精确值测量模式为EXACTLY
- 如果子View布局参数LayoutParams为MATCH_PARENT那么测量尺寸为0或剩余空间测量模式为UNSPECIFIED
- 如果子View布局参数LayoutParams为WRAP_CONTENT那么测量尺寸为0或剩余空间测量模式为WRAP_CONTENT

###### 子View最终测量

在上面测量过程中我们知道View实际测量工作是在onMeasure中，来查看View的onMeasure

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;

        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}
```

```java
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

最终会调用setMeasuredDimensionRaw方法保存测量尺寸，而测量尺寸又由getDefaultSize方法最终确定

```java
public static int getDefaultSize(int size, int measureSpec) {
    // 建议的最小尺寸
    int result = size;
    // 子View的MeasureSpec约束
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
	
    // 根据View的MeasureSpec约束确定测量尺寸
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

建议的最小尺寸如果有背景的话就是背景drawable的尺寸否则为0

```java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

结论：

如果View的specMode约束是UNSPECIFIED测量尺寸为建议最小尺寸，否则就以specSize作为最终测量尺寸

#### 案例分析				

我们在开发中有时候需要使用ScrollView嵌套ListView，xml指定layout_height都为match_parent，但是会发现ListView只显示第一个条目，这是为什么呢，我们可以从源码角度解析问题的原因，因为篇幅问题放到另一篇文章介绍请戳此链接：

[ScrollView嵌套LitView显示问题探究](https://hezd.github.io/2021/12/22/ScrollView%E5%B5%8C%E5%A5%97LitView%E6%98%BE%E7%A4%BA%E9%97%AE%E9%A2%98%E6%8E%A2%E7%A9%B6/ "ScrollView嵌套LitView显示问题探究")
