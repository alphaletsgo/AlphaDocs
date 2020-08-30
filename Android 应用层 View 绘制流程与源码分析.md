## **1 背景**

还记得前面[《Android 应用 setContentView 与 LayoutInflater 加载解析机制源码分析》](http://blog.csdn.net/yanbober/article/details/45970721)这篇文章吗？我们有分析到 Activity 中界面加载显示的基本流程原理，记不记得最终分析结果就是下面的关系：

![img](.assets/20150528211309106)

看见没有，如上图中 id 为 content 的内容就是整个 View 树的结构，所以对每个具体 View 对象的操作，其实就是个递归的实现。

前面[《Android 触摸屏事件派发机制详解与源码分析一 (View 篇)》](http://blog.csdn.net/yanbober/article/details/45887547)文章的 3-1 小节说过 Android 中的任何一个布局、任何一个控件其实都是直接或间接继承自 View 实现的，当然也包括我们后面一步一步引出的自定义控件也不例外，所以说这些 View 应该都具有相同的绘制流程与机制才能显示到屏幕上（因为他们都具备相同的父类 View，可能每个控件的具体绘制逻辑有差异，但是主流程都是一样的）。经过总结发现每一个 View 的绘制过程都必须经历三个最主要的过程，也就是 `measure`、`layout`和`draw`。

既然一个 View 的绘制主要流程是这三步，那一定有一个开始地方呀，就像一个类从 main 函数执行一样呀。对于 View 的绘制开始调运地方这里先给出结论，本文后面会反过来分析原因的，先往下看就行。具体结论如下：

整个 View 树的绘图流程是在 ViewRootImpl 类的 `performTraversals()` 方法（这个方法巨长）开始的，该函数做的执行过程主要是根据之前设置的状态，判断是否重新计算视图大小 (measure)、是否重新放置视图的位置 (layout)、以及是否重绘 (draw)，其核心也就是通过判断来选择顺序执行这三个方法中的哪个，如下：

```java
private void performTraversals() {
        ......
        //最外层的根视图的widthMeasureSpec和heightMeasureSpec由来
        //lp.width和lp.height在创建ViewGroup实例时等于MATCH_PARENT
        int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
        int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
        ......
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        ......
        mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
        ......
        mView.draw(canvas);
        ......
    }
/**
     * Figures out the measure spec for the root view in a window based on it's
     * layout params.
     *
     * @param windowSize
     *            The available width or height of the window
     *
     * @param rootDimension
     *            The layout params for one dimension (width or height) of the
     *            window.
     *
     * @return The measure spec to use to measure the root view.
     */
    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        ......
        }
        return measureSpec;
    }
```

可以看见这个方法的注释说是用来测 Root View 的。上面传入参数后这个函数走的是 `MATCH_PARENT`，使用 `MeasureSpec.makeMeasureSpec` 方法组装一个 `MeasureSpec`，`MeasureSpec` 的 `specMode` 等于 `EXACTLY`，`specSize` 等于 `windowSize`，也就是为何根视图总是全屏的原因。

其中的 mView 就是 View 对象。如下就是整个流程的大致流程图：



![img](.assets/20150529090922419)



如下我们就依据 View 绘制的这三个主要流程进行详细剖析（基于 Android5.1.1 API 22 源码进行分析）。

## **2 View 绘制流程第一步：递归 measure 源码分析**

整个 View 树的源码 measure 流程图如下：



![img](.assets/20150529163050000)



### **2-1 measure 源码分析**

先看下 View 的 measure 方法源码，如下：

```java
/**
     * <p>
     * This is called to find out how big a view should be. The parent
     * supplies constraint information in the width and height parameters.
     * </p>
     *
     * <p>
     * The actual measurement work of a view is performed in
     * {@link #onMeasure(int, int)}, called by this method. Therefore, only
     * {@link #onMeasure(int, int)} can and must be overridden by subclasses.
     * </p>
     *
     *
     * @param widthMeasureSpec Horizontal space requirements as imposed by the
     *        parent
     * @param heightMeasureSpec Vertical space requirements as imposed by the
     *        parent
     *
     * @see #onMeasure(int, int)
     */
     //final方法，子类不可重写
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ......
        //回调onMeasure()方法
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        ......
    }
```

看见注释信息没有，他告诉你了很多重要信息。为整个 View 树计算实际的大小，然后设置实际的高和宽，**每个 View 控件的实际宽高都是由父视图和自身决定的**。实际的测量是在 `onMeasure` 方法进行，所以在 View 的子类需要重写 `onMeasure` 方法，这是因为 measure 方法是 final 的，不允许重写，所以 View 子类只能通过重写 `onMeasure` 来实现自己的测量逻辑。

这个方法的两个参数都是父 View 传递过来的，也就是代表了父 view 的规格。他由两部分组成，高 2 位表示 MODE，定义在 MeasureSpec 类（View 的内部类）中，有三种类型，`MeasureSpec.EXACTLY` 表示确定大小， `MeasureSpec.AT_MOST` 表示最大大小，`MeasureSpec.UNSPECIFIED` 不确定。低 30 位表示 size，也就是父 View 的大小。**对于系统 Window 类的DecorVIew对象Mode一般都为 `MeasureSpec.EXACTLY`，而 size 分别对应屏幕宽高。对于子 View 来说大小是由父 View 和子 View 共同决定的。**

在这里可以看出 measure 方法最终回调了 View 的 onMeasure 方法，我们来看下 View 的 onMeasure 源码，如下：

```java
/**
     * <p>
     * Measure the view and its content to determine the measured width and the
     * measured height. This method is invoked by {@link #measure(int, int)} and
     * should be overriden by subclasses to provide accurate and efficient
     * measurement of their contents.
     * </p>
     *
     * <p>
     * <strong>CONTRACT:</strong> When overriding this method, you
     * <em>must</em> call {@link #setMeasuredDimension(int, int)} to store the
     * measured width and height of this view. Failure to do so will trigger an
     * <code>IllegalStateException</code>, thrown by
     * {@link #measure(int, int)}. Calling the superclass'
     * {@link #onMeasure(int, int)} is a valid use.
     * </p>
     *
     * <p>
     * The base class implementation of measure defaults to the background size,
     * unless a larger size is allowed by the MeasureSpec. Subclasses should
     * override {@link #onMeasure(int, int)} to provide better measurements of
     * their content.
     * </p>
     *
     * <p>
     * If this method is overridden, it is the subclass's responsibility to make
     * sure the measured height and width are at least the view's minimum height
     * and width ({@link #getSuggestedMinimumHeight()} and
     * {@link #getSuggestedMinimumWidth()}).
     * </p>
     *
     * @param widthMeasureSpec horizontal space requirements as imposed by the parent.
     *                         The requirements are encoded with
     *                         {@link android.view.View.MeasureSpec}.
     * @param heightMeasureSpec vertical space requirements as imposed by the parent.
     *                         The requirements are encoded with
     *                         {@link android.view.View.MeasureSpec}.
     *
     * @see #getMeasuredWidth()
     * @see #getMeasuredHeight()
     * @see #setMeasuredDimension(int, int)
     * @see #getSuggestedMinimumHeight()
     * @see #getSuggestedMinimumWidth()
     * @see android.view.View.MeasureSpec#getMode(int)
     * @see android.view.View.MeasureSpec#getSize(int)
     */
     //View的onMeasure默认实现方法
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

看见没有，其实注释已经很详细了（自定义 View 重写该方法的指导操作注释都有说明），不做过多解释。

对于非 ViewGroup 的 View 而言，通过调用上面默认的`onMeasure`即可完成 View 的测量，当然你也可以重载`onMeasure`并调用`setMeasuredDimension`来设置任意大小的布局，但一般不这么做，因为这种做法不太好，至于为何不好，后面分析完你就明白了。

我们可以看见`onMeasure`默认的实现仅仅调用了`setMeasuredDimension`，`setMeasuredDimension`函数是一个很关键的函数，它对View的成员变量`mMeasuredWidth`和`mMeasuredHeight`变量赋值，measure 的主要目的就是对View树中的每个View的`mMeasuredWidth`和`mMeasuredHeight`进行赋值，所以一旦这两个变量被赋值意味着该 View 的测量工作结束。既然这样那我们就看看设置的默认尺寸大小吧，可以看见 setMeasuredDimension 传入的参数都是通过 getDefaultSize 返回的，所以再来看下 getDefaultSize 方法源码，如下：

```java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        //通过MeasureSpec解析获取mode与size
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

看见没有，如果 specMode 等于 AT_MOST 或 EXACTLY 就返回 specSize，这就是系统默认的规格。

回过头继续看上面 onMeasure 方法，其中 getDefaultSize 参数的 widthMeasureSpec 和 heightMeasureSpec 都是由父 View 传递进来的。getSuggestedMinimumWidth 与 getSuggestedMinimumHeight 都是 View 的方法，具体如下：

```java
protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }

    protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());

    }
```

看见没有，建议的最小宽度和高度都是由 View 的 Background 尺寸与通过设置 View 的 miniXXX 属性共同决定的。

到此一次最基础的元素 View 的 measure 过程就完成了。上面说了 View 实际是嵌套的，而且 measure 是递归传递的，所以每个 View 都需要 measure。实际能够嵌套的 View 一般都是 ViewGroup 的子类，所以在 ViewGroup 中定义了 measureChildren, measureChild, measureChildWithMargins 方法来对子视图进行测量，measureChildren 内部实质只是循环调用 measureChild，measureChild 和 measureChildWithMargins 的区别就是是否把 margin 和 padding 也作为子视图的大小。如下我们以 ViewGroup 中稍微复杂的 measureChildWithMargins 方法来分析：

```java
/**
     * Ask one of the children of this view to measure itself, taking into
     * account both the MeasureSpec requirements for this view and its padding
     * and margins. The child must have MarginLayoutParams The heavy lifting is
     * done in getChildMeasureSpec.
     *
     * @param child The child to measure
     * @param parentWidthMeasureSpec The width requirements for this view
     * @param widthUsed Extra space that has been used up by the parent
     *        horizontally (possibly by other children of the parent)
     * @param parentHeightMeasureSpec The height requirements for this view
     * @param heightUsed Extra space that has been used up by the parent
     *        vertically (possibly by other children of the parent)
     */
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        //获取子视图的LayoutParams
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
        //调整MeasureSpec
        //通过这两个参数以及子视图本身的LayoutParams来共同决定子视图的测量规格
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);
        //调用子View的measure方法，子View的measure中会回调子View的onMeasure方法
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

关于该方法的参数等说明注释已经描述的够清楚了。该方法就是对父视图提供的 measureSpec 参数结合自身的 LayoutParams 参数进行了调整，然后再来调用 child.measure() 方法，具体通过方法 getChildMeasureSpec 来进行参数调整。所以我们继续看下 getChildMeasureSpec 方法代码，如下：

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        //获取当前Parent View的Mode和Size
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        //获取Parent size与padding差值（也就是Parent剩余大小），若差值小于0直接返回0
        int size = Math.max(0, specSize - padding);
        //定义返回值存储变量
        int resultSize = 0;
        int resultMode = 0;
        //依据当前Parent的Mode进行switch分支逻辑
        switch (specMode) {
        // Parent has imposed an exact size on us
        //默认Root View的Mode就是EXACTLY
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                //如果child的layout_wOrh属性在xml或者java中给予具体大于等于0的数值
                //设置child的size为真实layout_wOrh属性值，mode为EXACTLY
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //如果child的layout_wOrh属性在xml或者java中给予MATCH_PARENT
                // Child wants to be our size. So be it.
                //设置child的size为size，mode为EXACTLY
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //如果child的layout_wOrh属性在xml或者java中给予WRAP_CONTENT
                //设置child的size为size，mode为AT_MOST
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        ......
        //其他Mode分支类似
        }
        //将mode与size通过MeasureSpec方法整合为32位整数返回
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

可以看见，getChildMeasureSpec 的逻辑是通过其父 View 提供的 MeasureSpec 参数得到 specMode 和 specSize，然后根据计算出来的 specMode 以及子 View 的 childDimension（layout_width 或 layout_height）来计算自身的 measureSpec，如果其本身包含子视图，则计算出来的 measureSpec 将作为调用其子视图 measure 函数的参数，同时也作为自身调用 setMeasuredDimension 的参数，如果其不包含子视图则默认情况下最终会调用 onMeasure 的默认实现，并最终调用到 setMeasuredDimension。

所以可以看见 onMeasure 的参数其实就是这么计算出来的。同时从上面的分析可以看出来，最终决定 View 的 measure 大小是 View 的 setMeasuredDimension 方法，所以我们可以通过 setMeasuredDimension 设定死值来设置 View 的 mMeasuredWidth 和 mMeasuredHeight 的大小，但是一个好的自定义 View 应该会根据子视图的 measureSpec 来设置 mMeasuredWidth 和 mMeasuredHeight 的大小，这样的灵活性更大，所以这也就是上面分析 onMeasure 时说 View 的 onMeasure 最好不要重写死值的原因。

可以看见当通过 setMeasuredDimension 方法最终设置完成 View 的 measure 之后 View 的 mMeasuredWidth 和 mMeasuredHeight 成员才会有具体的数值，所以如果我们自定义的 View 或者使用现成的 View 想通过 getMeasuredWidth() 和 getMeasuredHeight() 方法来获取 View 测量的宽高，必须保证这两个方法在 onMeasure 流程之后被调用才能返回有效值。

还记得前面[《Android 应用 setContentView 与 LayoutInflater 加载解析机制源码分析》](http://blog.csdn.net/yanbober/article/details/45970721)文章 3-3 小节探讨的 inflate 方法加载一些布局显示时指定的大小失效问题吗？当时只给出了结论，现在给出了详细原因分析，我想不需要再做过多解释了吧。

至此整个 View 绘制流程的第一步就分析完成了，可以看见，相对来说还是比较复杂的，接下来进行小结。

### **2-2 measure 原理总结**

通过上面分析可以看出 measure 过程主要就是从顶层父 View 向子 View 递归调用 view.measure 方法（measure 中又回调 onMeasure 方法）的过程。具体 measure 核心主要有如下几点：

- MeasureSpec（View 的内部类）测量规格为 int 型，值由高 2 位规格模式 specMode 和低 30 位具体尺寸 specSize 组成。其中 specMode 只有三种值：

```java
MeasureSpec.EXACTLY //确定模式，父View希望子View的大小是确定的，由specSize决定；
MeasureSpec.AT_MOST //最多模式，父View希望子View的大小最多是specSize指定的值；
MeasureSpec.UNSPECIFIED //未指定模式，父View完全依据子View的设计值来决定；
```

- View 的 measure 方法是 final 的，不允许重写，View 子类只能重写onMeasure 来完成自己的测量逻辑。
- 最顶层 DecorView 测量时的 MeasureSpec 是由 ViewRootImpl 中 getRootMeasureSpec 方法确定的（LayoutParams 宽高参数均为 MATCH_PARENT，specMode 是 EXACTLY，specSize 为物理屏幕大小）。
- ViewGroup 类提供了 measureChild，measureChild 和 measureChildWithMargins 方法，简化了父子 View 的尺寸计算。
- 只要是 ViewGroup 的子类就必须要求 LayoutParams 继承子 MarginLayoutParams，否则无法使用 layout_margin 参数。
- View 的布局大小由父 View 和子 View 共同决定。
- 使用 View 的 getMeasuredWidth() 和 getMeasuredHeight() 方法来获取 View 测量的宽高，必须保证这两个方法在 onMeasure 流程之后被调用才能返回有效值。

【工匠若水 http://blog.csdn.net/yanbober 转载烦请注明出处，尊重分享成果】

## **3 View 绘制流程第二步：递归 layout 源码分析**

在上面的背景介绍就说过，当 ViewRootImpl 的 performTraversals 中 measure 执行完成以后会接着执行 mView.layout，具体如下：

```java
private void performTraversals() {
    ......
    mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
    ......
}
```

可以看见 layout 方法接收四个参数，这四个参数分别代表相对 Parent 的左、上、右、下坐标。而且还可以看见左上都为 0，右下分别为上面刚刚测量的 width 和 height。

至此又回归到 View 的 layout(int l, int t, int r, int b) 方法中去实现具体逻辑了，所以接下来我们开始分析 View 的 layout 过程。

整个 View 树的 layout 递归流程图如下：



![img](.assets/20150529163153998)



### **3-1 layout 源码分析**

layout 既然也是递归结构，那我们先看下 ViewGroup 的 layout 方法，如下：

```java
@Override
    public final void layout(int l, int t, int r, int b) {
        ......
        super.layout(l, t, r, b);
        ......
    }
```

看着没有？ViewGroup 的 layout 方法实质还是调运了 View 父类的 layout 方法，所以我们看下 View 的 layout 源码，如下：

```java
public void layout(int l, int t, int r, int b) {
        ......
        //实质都是调用setFrame方法把参数分别赋值给mLeft、mTop、mRight和mBottom这几个变量
        //判断View的位置是否发生过变化，以确定有没有必要对当前的View进行重新layout
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        //需要重新layout
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            //回调onLayout
            onLayout(changed, l, t, r, b);
            ......
        }
        ......
    }
```

看见没有，类似 measure 过程，lauout 调运了 onLayout 方法。

对比上面 View 的 layout 和 ViewGroup 的 layout 方法可以发现，View 的 layout 方法是可以在子类重写的，而 ViewGroup 的 layout 是不能在子类重写的，言外之意就是说 ViewGroup 中只能通过重写 onLayout 方法。那我们接下来看下 ViewGroup 的 onLayout 方法，如下：

```java
@Override
    protected abstract void onLayout(boolean changed,
            int l, int t, int r, int b);
```

看见没有？ViewGroup 的 onLayout() 方法竟然是一个抽象方法，这就是说所有 ViewGroup 的子类都必须重写这个方法。所以在自定义 ViewGroup 控件中，onLayout 配合 onMeasure 方法一起使用可以实现自定义 View 的复杂布局。自定义 View 首先调用 onMeasure 进行测量，然后调用 onLayout 方法动态获取子 View 和子 View 的测量大小，然后进行 layout 布局。重载 onLayout 的目的就是安排其 children 在父 View 的具体位置，重载 onLayout 通常做法就是写一个 for 循环调用每一个子视图的 layout(l, t, r, b) 函数，传入不同的参数 l, t, r, b 来确定每个子视图在父视图中的显示位置。

再看下 View 的 onLayout 方法源码，如下：

```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }
```

我勒个去！是一个空方法，没啥可看的。

既然这样那我们只能分析一个现有的继承 ViewGroup 的控件了，就拿 LinearLayout 来说吧，如下是 LinearLayout 中 onLayout 的一些代码：

```java
public class LinearLayout extends ViewGroup {
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
}
```

看见没有，LinearLayout 的 layout 过程是分 Vertical 和 Horizontal 的，这个就是 xml 布局的 orientation 属性设置的，我们为例说明 ViewGroup 的 onLayout 重写一般步骤就拿这里的 VERTICAL 模式来解释吧，如下是 layoutVertical 方法源码：

```java
void layoutVertical(int left, int top, int right, int bottom) {
        final int paddingLeft = mPaddingLeft;

        int childTop;
        int childLeft;

        // Where right end of child should go
        //计算父窗口推荐的子View宽度
        final int width = right - left;
        //计算父窗口推荐的子View右侧位置
        int childRight = width - mPaddingRight;

        // Space available for child
        //child可使用空间大小
        int childSpace = width - paddingLeft - mPaddingRight;
        //通过ViewGroup的getChildCount方法获取ViewGroup的子View个数
        final int count = getVirtualChildCount();
        //获取Gravity属性设置
        final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
        final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
        //依据majorGravity计算childTop的位置值
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
        //重点！！！开始遍历
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                //LinearLayout中其子视图显示的宽和高由measure过程来决定的，因此measure过程的意义就是为layout过程提供视图显示范围的参考值
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();
                //获取子View的LayoutParams
                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();

                int gravity = lp.gravity;
                if (gravity < 0) {
                    gravity = minorGravity;
                }
                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                //依据不同的absoluteGravity计算childLeft位置
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
                //通过垂直排列计算调运child的layout设置child的位置
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

                i += getChildrenSkipCount(child, i);
            }
        }
    }
```

从上面分析的 ViewGroup 子类 LinearLayout 的 onLayout 实现代码可以看出，一般情况下 layout 过程会参考 measure 过程中计算得到的 mMeasuredWidth 和 mMeasuredHeight 来安排子 View 在父 View 中显示的位置，但这不是必须的，measure 过程得到的结果可能完全没有实际用处，特别是对于一些自定义的 ViewGroup，其子 View 的个数、位置和大小都是固定的，这时候我们可以忽略整个 measure 过程，只在 layout 函数中传入的 4 个参数来安排每个子 View 的具体位置。

**到这里就不得不提 getWidth()、getHeight() 和 getMeasuredWidth()、getMeasuredHeight() 这两对方法之间的区别（上面分析 measure 过程已经说过 getMeasuredWidth()、getMeasuredHeight() 必须在 onMeasure 之后使用才有效）。可以看出来 getWidth() 与 getHeight() 方法必须在 layout(int l, int t, int r, int b) 执行之后才有效。**那我们看下 View 源码中这些方法的实现吧，如下：

```java
public final int getMeasuredWidth() {
        return mMeasuredWidth & MEASURED_SIZE_MASK;
    }

    public final int getMeasuredHeight() {
        return mMeasuredHeight & MEASURED_SIZE_MASK;
    }

    public final int getWidth() {
        return mRight - mLeft;
    }

    public final int getHeight() {
        return mBottom - mTop;
    }

    public final int getLeft() {
        return mLeft;
    }

    public final int getRight() {
        return mRight;
    }

    public final int getTop() {
        return mTop;
    }

    public final int getBottom() {
        return mBottom;
    }
```

这也解释了为什么有些情况下 getWidth() 和 getMeasuredWidth() 以及 getHeight() 和 getMeasuredHeight() 会得到不同的值，所以这里不做过多解释。

到此整个 View 的 layout 过程分析就算结束了，接下来进行一些总结工作。

### **3-2 layout 原理总结**

整个 layout 过程比较容易理解，从上面分析可以看出 layout 也是从顶层父 View 向子 View 的递归调用 view.layout 方法的过程，即父 View 根据上一步 measure 子 View 所得到的布局大小和布局参数，将子 View 放在合适的位置上。具体 layout 核心主要有以下几点：

- View.layout 方法可被重载，ViewGroup.layout 为 final 的不可重载，ViewGroup.onLayout 为 abstract 的，子类必须重载实现自己的位置逻辑。
- measure 操作完成后得到的是对每个 View 经测量过的 measuredWidth 和 measuredHeight，layout 操作完成之后得到的是对每个 View 进行位置分配后的 mLeft、mTop、mRight、mBottom，这些值都是相对于父 View 来说的。
- 凡是 layout_XXX 的布局属性基本都针对的是包含子 View 的 ViewGroup 的，当对一个没有父容器的 View 设置相关 layout_XXX 属性是没有任何意义的（前面[《Android 应用 setContentView 与 LayoutInflater 加载解析机制源码分析》](http://blog.csdn.net/yanbober/article/details/45970721)也有提到过）。
- 使用 View 的 getWidth() 和 getHeight() 方法来获取 View 测量的宽高，必须保证这两个方法在 onLayout 流程之后被调用才能返回有效值。

【工匠若水 http://blog.csdn.net/yanbober 转载烦请注明出处，尊重分享成果】

## **4 View 绘制流程第三步：递归 draw 源码分析**

在上面的背景介绍就说过，当 ViewRootImpl 的 performTraversals 中 measure 和 layout 执行完成以后会接着执行 mView.draw，具体如下：

```java
private void performTraversals() {
    ......
    final Rect dirty = mDirty;
    ......
    canvas = mSurface.lockCanvas(dirty);
    ......
    mView.draw(canvas);
    ......
}
```

draw 过程也是在 ViewRootImpl 的 performTraversals() 内部调运的，其调用顺序在 measure() 和 layout() 之后，这里的 mView 对于 Actiity 来说就是 PhoneWindow.DecorView，ViewRootImpl 中的代码会创建一个 Canvas 对象，然后调用 View 的 draw() 方法来执行具体的绘制工。所以又回归到了 ViewGroup 与 View 的树状递归 draw 过程。

先来看下 View 树的递归 draw 流程图，如下：



![img](.assets/20150530154328068)



如下我们详细分析这一过程。

### **4-1 draw 源码分析**

由于 ViewGroup 没有重写 View 的 draw 方法，所以如下直接从 View 的 draw 方法开始分析：

```java
public void draw(Canvas canvas) {
        ......
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
        ......
        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        ......

        // Step 2, save the canvas' layers
        ......
            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }
        ......

        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        ......
        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }
        ......

        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);
        ......
    }
```

看见整个 View 的 draw 方法很复杂，但是源码注释也很明显。从注释可以看出整个 draw 过程分为了 6 步。源码注释说（”skip step 2 & 5 if possible (common case)”）第 2 和 5 步可以跳过，所以我们接下来重点剩余四步。如下：

**第一步，对 View 的背景进行绘制。**

可以看见，draw 方法通过调运 drawBackground(canvas); 方法实现了背景绘制。我们来看下这个方法源码，如下：

```java
private void drawBackground(Canvas canvas) {
        //获取xml中通过android:background属性或者代码中setBackgroundColor()、setBackgroundResource()等方法进行赋值的背景Drawable
        final Drawable background = mBackground;
        ......
        //根据layout过程确定的View位置来设置背景的绘制区域
        if (mBackgroundSizeChanged) {
            background.setBounds(0, 0,  mRight - mLeft, mBottom - mTop);
            mBackgroundSizeChanged = false;
            rebuildOutline();
        }
        ......
            //调用Drawable的draw()方法来完成背景的绘制工作
            background.draw(canvas);
        ......
    }
```

**第三步，对 View 的内容进行绘制。**

可以看到，这里去调用了一下 View 的 onDraw() 方法，所以我们看下 View 的 onDraw 方法（ViewGroup 也没有重写该方法），如下：

```java
/**
     * Implement this to do your drawing.
     *
     * @param canvas the canvas on which the background will be drawn
     */
    protected void onDraw(Canvas canvas) {
    }
```

可以看见，这是一个空方法。因为每个 View 的内容部分是各不相同的，所以需要由子类去实现具体逻辑。

**第四步，对当前 View 的所有子 View 进行绘制，如果当前的 View 没有子 View 就不需要进行绘制。**

我们来看下 View 的 draw 方法中的 dispatchDraw(canvas); 方法源码，可以看见如下：

```java
/**
     * Called by draw to draw the child views. This may be overridden
     * by derived classes to gain control just before its children are drawn
     * (but after its own view has been drawn).
     * @param canvas the canvas on which to draw the view
     */
    protected void dispatchDraw(Canvas canvas) {

    }
```

看见没有，View 的 dispatchDraw() 方法是一个空方法，而且注释说明了如果 View 包含子类需要重写他，所以我们有必要看下 ViewGroup 的 dispatchDraw 方法源码（这也就是刚刚说的对当前 View 的所有子 View 进行绘制，如果当前的 View 没有子 View 就不需要进行绘制的原因，因为如果是 View 调运该方法是空的，而 ViewGroup 才有实现），如下：

```java
@Override
    protected void dispatchDraw(Canvas canvas) {
        ......
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        ......
        for (int i = 0; i < childrenCount; i++) {
            ......
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        ......
        // Draw any disappearing views that have animations
        if (mDisappearingChildren != null) {
            ......
            for (int i = disappearingCount; i >= 0; i--) {
                ......
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        ......
    }
```

可以看见，ViewGroup 确实重写了 View 的 dispatchDraw() 方法，该方法内部会遍历每个子 View，然后调用 drawChild() 方法，我们可以看下 ViewGroup 的 drawChild 方法，如下：

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }
```

可以看见 drawChild() 方法调运了子 View 的 draw() 方法。所以说 ViewGroup 类已经为我们重写了 dispatchDraw() 的功能实现，我们一般不需要重写该方法，但可以重载父类函数实现具体的功能。

**第六步，对 View 的滚动条进行绘制。**

可以看到，这里去调用了一下 View 的 onDrawScrollBars() 方法，所以我们看下 View 的 onDrawScrollBars(canvas); 方法，如下：

```java
/**
     * <p>Request the drawing of the horizontal and the vertical scrollbar. The
     * scrollbars are painted only if they have been awakened first.</p>
     *
     * @param canvas the canvas on which to draw the scrollbars
     *
     * @see #awakenScrollBars(int)
     */
    protected final void onDrawScrollBars(Canvas canvas) {
        //绘制ScrollBars分析不是我们这篇的重点，所以暂时不做分析
        ......
    }
```

可以看见其实任何一个 View 都是有（水平垂直）滚动条的，只是一般情况下没让它显示而已。

到此，View 的 draw 绘制部分源码分析完毕，我们接下来进行一些总结。

### **4-2 draw 原理总结**

可以看见，绘制过程就是把 View 对象绘制到屏幕上，整个 draw 过程需要注意如下细节：

- 如果该 View 是一个 ViewGroup，则需要递归绘制其所包含的所有子 View。
- View 默认不会绘制任何内容，真正的绘制都需要自己在子类中实现。
- View 的绘制是借助 onDraw 方法传入的 Canvas 类来进行的。
- 区分 View 动画和 ViewGroup 布局动画，前者指的是 View 自身的动画，可以通过 setAnimation 添加，后者是专门针对 ViewGroup 显示内部子视图时设置的动画，可以在 xml 布局文件中对 ViewGroup 设置 layoutAnimation 属性（譬如对 LinearLayout 设置子 View 在显示时出现逐行、随机、下等显示等不同动画效果）。
- 在获取画布剪切区（每个 View 的 draw 中传入的 Canvas）时会自动处理掉 padding，子 View 获取 Canvas 不用关注这些逻辑，只用关心如何绘制即可。
- 默认情况下子 View 的 ViewGroup.drawChild 绘制顺序和子 View 被添加的顺序一致，但是你也可以重载 ViewGroup.getChildDrawingOrder() 方法提供不同顺序。

【工匠若水 http://blog.csdn.net/yanbober 转载烦请注明出处，尊重分享成果】

## **5 View 的 invalidate 和 postInvalidate 方法源码分析**

你可能已经看见了，在上面分析 View 的三步绘制流程中最后都有调运一个叫 invalidate 的方法，这个方法是啥玩意？为何出现频率这么高？很简单，我们拿出来分析分析不就得了。

### **5-1 invalidate 方法源码分析**

来看一下 View 类中的一些 invalidate 方法（ViewGroup 没有重写这些方法），如下：

```java
/**
     * Mark the area defined by dirty as needing to be drawn. If the view is
     * visible, {@link #onDraw(android.graphics.Canvas)} will be called at some
     * point in the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     * <p>
     * <b>WARNING:</b> In API 19 and below, this method may be destructive to
     * {@code dirty}.
     *
     * @param dirty the rectangle representing the bounds of the dirty region
     */
     //看见上面注释没有？public，只能在UI Thread中使用，别的Thread用postInvalidate方法，View是可见的才有效，回调onDraw方法,针对局部View
    public void invalidate(Rect dirty) {
        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        //实质还是调运invalidateInternal方法
        invalidateInternal(dirty.left - scrollX, dirty.top - scrollY,
                dirty.right - scrollX, dirty.bottom - scrollY, true, false);
    }

    /**
     * Mark the area defined by the rect (l,t,r,b) as needing to be drawn. The
     * coordinates of the dirty rect are relative to the view. If the view is
     * visible, {@link #onDraw(android.graphics.Canvas)} will be called at some
     * point in the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     *
     * @param l the left position of the dirty region
     * @param t the top position of the dirty region
     * @param r the right position of the dirty region
     * @param b the bottom position of the dirty region
     */
     //看见上面注释没有？public，只能在UI Thread中使用，别的Thread用postInvalidate方法，View是可见的才有效，回调onDraw方法，针对局部View
    public void invalidate(int l, int t, int r, int b) {
        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        //实质还是调运invalidateInternal方法
        invalidateInternal(l - scrollX, t - scrollY, r - scrollX, b - scrollY, true, false);
    }

    /**
     * Invalidate the whole view. If the view is visible,
     * {@link #onDraw(android.graphics.Canvas)} will be called at some point in
     * the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     */
     //看见上面注释没有？public，只能在UI Thread中使用，别的Thread用postInvalidate方法，View是可见的才有效，回调onDraw方法，针对整个View
    public void invalidate() {
        //invalidate的实质还是调运invalidateInternal方法
        invalidate(true);
    }

    /**
     * This is where the invalidate() work actually happens. A full invalidate()
     * causes the drawing cache to be invalidated, but this function can be
     * called with invalidateCache set to false to skip that invalidation step
     * for cases that do not need it (for example, a component that remains at
     * the same dimensions with the same content).
     *
     * @param invalidateCache Whether the drawing cache for this view should be
     *            invalidated as well. This is usually true for a full
     *            invalidate, but may be set to false if the View's contents or
     *            dimensions have not changed.
     */
     //看见上面注释没有？default的权限，只能在UI Thread中使用，别的Thread用postInvalidate方法，View是可见的才有效，回调onDraw方法，针对整个View
    void invalidate(boolean invalidateCache) {
    //实质还是调运invalidateInternal方法
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }

    //！！！！！！看见没有，这是所有invalidate的终极调运方法！！！！！！
    void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        ......
            // Propagate the damage rectangle to the parent view.
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                //设置刷新区域
                damage.set(l, t, r, b);
                //传递调运Parent ViewGroup的invalidateChild方法
                p.invalidateChild(this, damage);
            }
            ......
    }
```

看见没有，View 的 invalidate（invalidateInternal）方法实质是将要刷新区域直接传递给了父 ViewGroup 的 invalidateChild 方法，在 invalidate 中，调用父 View 的 invalidateChild，这是一个从当前向上级父 View 回溯的过程，每一层的父 View 都将自己的显示区域与传入的刷新 Rect 做交集 。所以我们看下 ViewGroup 的 invalidateChild 方法，源码如下：

```java
public final void invalidateChild(View child, final Rect dirty) {
        ViewParent parent = this;
        final AttachInfo attachInfo = mAttachInfo;
        ......
        do {
            ......
            //循环层层上级调运，直到ViewRootImpl会返回null
            parent = parent.invalidateChildInParent(location, dirty);
            ......
        } while (parent != null);
    }
```

这个过程最后传递到 ViewRootImpl 的 invalidateChildInParent 方法结束，所以我们看下 ViewRootImpl 的 invalidateChildInParent 方法，如下：

```java
@Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        ......
        //View调运invalidate最终层层上传到ViewRootImpl后最终触发了该方法
        scheduleTraversals();
        ......
        return null;
    }
```

看见没有？这个 ViewRootImpl 类的 invalidateChildInParent 方法直接返回了 null，也就是上面 ViewGroup 中说的，层层上级传递到 ViewRootImpl 的 invalidateChildInParent 方法结束了那个 do while 循环。看见这里调运的 scheduleTraversals 这个方法吗？scheduleTraversals 会通过 Handler 的 Runnable 发送一个异步消息，调运 doTraversal 方法，然后最终调用 performTraversals() 执行重绘。开头背景知识介绍说过的，performTraversals 就是整个 View 数开始绘制的起始调运地方，所以说 View 调运 invalidate 方法的实质是层层上传到父级，直到传递到 ViewRootImpl 后触发了 scheduleTraversals 方法，然后整个 View 树开始重新按照上面分析的 View 绘制流程进行重绘任务。

到此 View 的 invalidate 方法原理就分析完成了。

### **5-2 postInvalidate 方法源码分析**

上面分析 invalidate 方法时注释中说该方法只能在 UI Thread 中执行，其他线程中需要使用 postInvalidate 方法，所以我们来分析分析 postInvalidate 这个方法源码。如下：

```java
public void postInvalidate() {
        postInvalidateDelayed(0);
    }
```

继续看下他的调运方法 postInvalidateDelayed，如下：

```java
public void postInvalidateDelayed(long delayMilliseconds) {
        // We try only with the AttachInfo because there's no point in invalidating
        // if we are not attached to our window
        final AttachInfo attachInfo = mAttachInfo;
        //核心，实质就是调运了ViewRootImpl.dispatchInvalidateDelayed方法
        if (attachInfo != null) {
            attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
        }
    }
```

我们继续看他调运的 ViewRootImpl 类的 dispatchInvalidateDelayed 方法，如下源码：

```java
public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
        Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
        mHandler.sendMessageDelayed(msg, delayMilliseconds);
    }
```

看见没有，通过 ViewRootImpl 类的 Handler 发送了一条 MSG_INVALIDATE 消息，继续追踪这条消息的处理可以发现：

```java
public void handleMessage(Message msg) {
    ......
    switch (msg.what) {
    case MSG_INVALIDATE:
        ((View) msg.obj).invalidate();
        break;
    ......
    }
    ......
}
```

看见没有，实质就是又在 UI Thread 中调运了 View 的 invalidate(); 方法，那接下来 View 的 invalidate(); 方法我们就不说了，上名已经分析过了。

到此整个 View 的 postInvalidate 方法就分析完成了。

### **5-3 invalidate 与 postInvalidate 方法总结**

依据上面对 View 的 invalidate 分析我总结绘制如下流程图：



![img](.assets/20150531111928069)



依据上面对 View 的 postInvalidate 分析我总结绘制如下流程图：



![img](.assets/20150531121123441)



关于这两个方法的具体流程和原理上面也分析过了，流程图也给出了，相信已经很明确了，没啥需要解释的了。所以我们对其做一个整体总结，归纳出重点如下：

invalidate 系列方法请求重绘 View 树（也就是 draw 方法），如果 View 大小没有发生变化就不会调用 layout 过程，并且只绘制那些 “需要重绘的”View，也就是哪个 View(View 只绘制该 View，ViewGroup 绘制整个 ViewGroup) 请求 invalidate 系列方法，就绘制该 View。

**常见的引起 invalidate 方法操作的原因主要有：**

- 直接调用 invalidate 方法. 请求重新 draw，但只会绘制调用者本身。
- 触发 setSelection 方法。请求重新 draw，但只会绘制调用者本身。
- 触发 setVisibility 方法。 当 View 可视状态在 INVISIBLE 转换 VISIBLE 时会间接调用 invalidate 方法，继而绘制该 View。当 View 的可视状态在 INVISIBLE\VISIBLE 转换为 GONE 状态时会间接调用 requestLayout 和 invalidate 方法，同时由于 View 树大小发生了变化，所以会请求 measure 过程以及 draw 过程，同样只绘制需要 “重新绘制” 的视图。
- 触发 setEnabled 方法。请求重新 draw，但不会重新绘制任何 View 包括该调用者本身。
- 触发 requestFocus 方法。请求 View 树的 draw 过程，只绘制 “需要重绘” 的 View。

### **5-4 通过 invalidate 方法分析结果回过头去解决一个背景介绍中的疑惑**

分析完 invalidate 后需要你回过头去想一个问题。还记不记得这篇文章的开头背景介绍，我们说整个 View 绘制流程的最初代码是在 ViewRootImpl 类的 performTraversals() 方法中开始的。上面当时只是告诉你了这个结论，至于这个 ViewRootImpl 类的 performTraversals() 方法为何会被触发没有说明原因。现在我们就来分析一下这个触发的源头。

让我们先把大脑思考暂时挪回到[《Android 应用 setContentView 与 LayoutInflater 加载解析机制源码分析》](http://blog.csdn.net/yanbober/article/details/45970721)这篇博文的 setContentView 机制分析中（不清楚的请点击先看这篇文章再回过头来继续看）。我们先来看下那篇博文分析的 PhoneWindow 的 setContentView 方法源码，如下：

```java
@Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        ......
        //如果mContentParent为空进行一些初始化，实质mContentParent是通过findViewById(ID_ANDROID_CONTENT);获取的id为content的FrameLayout的布局（不清楚的请先看《Android应用setContentView与LayoutInflater加载解析机制源码分析》文章）
        if (mContentParent == null) {
            installDecor();
        } 
        ......
        //把我们的view追加到mContentParent
        mContentParent.addView(view, params);
        ......
    }
```

这个方法是 Activity 中 setContentView 的实现，我们继续看下这个方法里调运的 addView 方法，也就是 ViewGroup 的 addView 方法，如下：

```java
public void addView(View child) {
        addView(child, -1);
    }

    public void addView(View child, int index) {
        ......
        addView(child, index, params);
    }

    public void addView(View child, int index, LayoutParams params) {
        ......
        //该方法稍后后面会详细分析
        requestLayout();
        //重点关注！！！
        invalidate(true);
        ......
    }
```

看见 addView 调运 invalidate 方法没有？这不就真相大白了。当我们写一个 Activity 时，我们一定会通过 setContentView 方法将我们要展示的界面传入该方法，该方法会讲我们界面通过 addView 追加到 id 为 content 的一个 FrameLayout（ViewGroup）中，然后 addView 方法中通过调运 invalidate(true) 去通知触发 ViewRootImpl 类的 performTraversals() 方法，至此递归绘制我们自定义的所有布局。

【工匠若水 http://blog.csdn.net/yanbober 转载烦请注明出处，尊重分享成果】

## **6 View 的 requestLayout 方法源码分析**

### **6-1 requestLayout 方法分析**

和 invalidate 类似，其实在上面分析 View 绘制流程时或多或少都调运到了这个方法，而且这个方法对于 View 来说也比较重要，所以我们接下来分析一下他。如下 View 的 requestLayout 源码：

```java
public void requestLayout() {
        ......
        if (mParent != null && !mParent.isLayoutRequested()) {
            //由此向ViewParent请求布局
            //从这个View开始向上一直requestLayout，最终到达ViewRootImpl的requestLayout
            mParent.requestLayout();
        }
        ......
    }
```

看见没有，当我们触发 View 的 requestLayout 时其实质就是层层向上传递，直到 ViewRootImpl 为止，然后触发 ViewRootImpl 的 requestLayout 方法，如下就是 ViewRootImpl 的 requestLayout 方法：

```java
@Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            //View调运requestLayout最终层层上传到ViewRootImpl后最终触发了该方法
            scheduleTraversals();
        }
    }
```

看见没有，类似于上面分析的 invalidate 过程，只是设置的标记不同，导致对于 View 的绘制流程中触发的方法不同而已。

### **6-2 requestLayout 方法总结**

可以看见，这些方法都是大同小异。对于 requestLayout 方法来说总结如下：

requestLayout() 方法会调用 measure 过程和 layout 过程，不会调用 draw 过程，也不会重新绘制任何 View 包括该调用者本身。

## **7 View 绘制流程总结**

至此整个关于 Android 应用程序开发中的 View 绘制机制及相关重要方法都已经分析完毕。关于各个方法的总结这里不再重复，直接通过该文章前面的目录索引到相应方法的总结小节进行查阅即可。