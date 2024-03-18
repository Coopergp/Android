---
自定义View
---
#### 目录

1. 基础知识储备
   - 坐标系
   - 颜色
2. 自定义 View 
   - 分类和流程
   - Canvas 之常用 API
   - MotionEvent
   - 手势检测
3. 常见问题汇总
   - 处理 warp_content 
   - 处理 padding
   - 处理 margin
4. 实战
   - 圆形进度条
   - 数字进度条
#### 基础知识储备

##### 坐标系

Android 中的屏幕坐标系是以屏幕的左上角为坐标原点的，向右为 x 正轴，向下是 y 正轴。

这里就要提一下 View 的坐标系了，View 的坐标系统是相对于父控件而言的：

```java
getTop()	//获取子 View 左上角到父 View 顶部的距离
getLeft()	//获取子 View 左上角到父 View 左边的距离
getBottom() //获取子 View 右下角到父 View 顶部的距离
getRight()	//获取子 View 右上角到父 View 左边的距离
    
getBottom() - getTop() = View 的高
getRight() - getLeft() = View 的宽
```

MotionEvent 中的 getXxx 和 getRawXxx 的区别：

```java
event.getX()	//触摸点相对于其所在 View 坐标系的坐标
event.getY()
    
event.getRawX()	//触摸点相对于屏幕坐标系的坐标
event.getRawY()	    
```

##### 颜色

Android 支持的颜色模式有：

| 颜色模式 | 备注                 |
| -------- | -------------------- |
| ARGB8888 | 四通道高精度（32位） |
| ARGB4444 | 四通道低精度（16位） |
| RGB565   | 屏幕默认模式（16位） |

RGB 代表红绿蓝三原色，A 代表透明度，后面的数值表示该类型用多少位二进制来描述。

```java
#f00	//低精度 - 不带透明通道红色
#af00	//低精度 - 带透明通道红色
#ff0000		//高精度 - 不带透明通道红色
#aaff0000	//高精度 - 带透明通道红色	
```

有了基础知识储备，接下来就开始进入自定义 View 了～～～

#### 自定义 View 分类和流程

自定义 View 可以分为两类：一类是自定义 ViewGroup，另一种是自定义 View。自定义 ViewGroup 一般是利用已有的 View 按照特定的布局方式来实现新的组件，比如带自动换行的水平的线性布局等。自定义 View 一般是由于没有现成的 View 可以使用，需要自己实现 onDraw 来绘制。

自定义 View 的流程也是一个通用的套路：

![](https://i.loli.net/2019/02/18/5c6a2e4942328.jpg)

##### 构造函数

```java
public class MyCustomView extends View {

    //在 Activity 中以 new MyCustomView(this) 创建 View
    public MyCustomView(Context context) {}

    //在 xml 中创建 View
    public MyCustomView(Context context, @Nullable AttributeSet attrs) {}

    //为 View 指定样式
    public MyCustomView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {}

    //API > 21
    public MyCustomView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {}
```

我们只需要实现前两个构造函数即可，AttributeSet 用于获取自定义属性等。

##### onMeasure()

用于测量 View 的大小。

你可能会问，既然我们在 xml 里面可以指定 View 的宽高尺寸，为什么还需要自己测量呢？

这是因为，View 的大小不仅由自身所决定，同时也会受父控件的影响，比如我们设置 warp_content 或 match_parent。

```java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //获取宽度尺寸和宽度测量模式
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        
        setMeasuredDimension(widthSize, heightSize);
    }
```

onMeasure 中的参数可以翻译成测量规格，它有两部分组成：宽高实际尺寸和宽高测量模式。

测量模式有三种：

| 模式        | 二进制值 | 描述                                                         |
| ----------- | -------- | ------------------------------------------------------------ |
| UNSPECIFIED | 00       | 默认值，父控件没有给子 View 任何限制，子 View 可以设置为任意大小，一般用在系统中，我们可以不管 |
| EXACTLY     | 01       | 表示父控件已经确切指定了子 View 的大小，对应于 match_parent 和 确切数值 100dp |
| AT_MOST     | 10       | 表示子 View 的大小存在上限，一般是父 View 大小，对应于 warp_content |

所以在测量规格中，只需要两个 bit 就能表示完测量模式，而事实上正是这样做的，测量规格是一个 int 数值，32位，前两位表示测量模式，后三十位表示测量数值。

##### onSizeChanged()

在视图大小发生改变时调用。

既然在测量完 View 并使用 setMeasuredDimension 函数之后 View 的大小基本上已经确定了，那为什么还要再次确认 View 的大小呢？

这是因为 View 的大小不仅由 View 本身控制，而且受父控件的影响，所以我们在确定 View 大小的时候最好使用系统提供的 onSizedChanged 回调函数。

```java
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
    }
```

w、h 即是 View 最终的大小。

##### onLayout()

确定布局的函数是 onLayout，它用于确定子 View 的位置，在定义 ViewGroup 中会用到，它调用的是子 View 的 layout 函数。

在自定义 ViewGroup 中，onLayout 一般是循环取出子 View，然后经过计算得出各个子 View 位置的坐标值，然后用以下函数设置子 View 位置。

```java
child.layout(l,t,r,b)
```

##### onDraw()

onDraw 是实际绘制的部分，使用 Canvas 绘制。

```java
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
    }
```

#### 自定义 View 之 Canvas 常用 API

| 操作类型     | 相关 API                                                     | 备注                                       |
| ------------ | ------------------------------------------------------------ | ------------------------------------------ |
| 绘制颜色     | drawColor、drawRGB、drawARGB                                 | 使用单一颜色填充整个画布                   |
| 绘制基本图形 | drawPoint、drawPoints、drawLine、drawLines、drawRect、drawRoundRect、drawOval、drawCircle、drawArc | 绘制点、线、矩形、圆角矩形、椭圆、圆、圆弧 |
| 绘制图片     | drawBitmap、drawPicture                                      | 绘制位图和图片                             |
| 绘制路径     | drawPath                                                     | 绘制路径，绘制贝塞尔曲线                   |
| 画布裁剪     | clipPath、clipRect                                           | 设置画布的显示区域                         |
| 画布变换     | translate、scale、rotate、skew                               | 位移、缩放、旋转、错切                     |

#### MotionEvent

| 事件          | 简介                               |
| ------------- | ---------------------------------- |
| ACTION_DOWN   | 手指初次接触屏幕时触发             |
| ACTION_MOVE   | 手指在屏幕上滑动时触发，会多次触发 |
| ACTION_UP     | 手指离开屏幕时触发                 |
| ACTION_CANCEL | 事件被上层拦截时触发               |

这里主要说下 ACTION_CANCEL，它的触发条件是事件被上层拦截。但是我们知道，在事件分发中，如果父 View 拦截了事件，那么子 View 是收不到任何事件的。所以这个 ACTION_CANCEL 的正确触发条件是：

**只有父 View 回收事件处理权的时候，子 View 才会收到一个 ACTION_CANCEL 事件。**

举个例子：

上层 View 是一个 RecyclerView，它收到了一个 ACTION_DOWN 事件，由于这可能是个点击事件，所以它先传递给了对应的 ItemView，询问 ItemView 是否需要这个事件，然后接下来又传递过来一个 ACTION_MOVE 事件，且移动的方向和 RecyclerView 的可滑动方向一致，这时候 RecyclerView 判断这个事件是滚动事件，于是要回收事件处理权，这时候对应的 ItemView 就会收到一个 ACTION_CANCEL，并且不会再收到后续事件。

#### 手势检测（GestureDetector）

GestureDetector 可以使用 MotionEvents 检测各种手势和事件，使用起来也很简单～

```java
        final GestureDetector detector = new GestureDetector(this, new GestureDetector.SimpleOnGestureListener() {
            @Override
            public boolean onDoubleTap(MotionEvent e) {
                Toast.makeText(WidgetActivity.this, "双击事件", Toast.LENGTH_SHORT).show();
                return super.onDoubleTap(e);
            }
        });

        mButton.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                return detector.onTouchEvent(event);
            }
        });
```

#### 常见问题汇总

##### 处理 warp_content

1. 自定义 View 的处理

   如果我们不处理自定义 View 中的 warp_content，那么它和 match_parent 的效果一样。这里我们需要在 onMeasure() 里面做特殊处理：

   ```java
       protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
           //获取宽度尺寸和宽度测量模式
           int widthSize = MeasureSpec.getSize(widthMeasureSpec);
           int widthMode = MeasureSpec.getMode(widthMeasureSpec);
   
           int heightSize = MeasureSpec.getSize(heightMeasureSpec);
           int heightMode = MeasureSpec.getMode(heightMeasureSpec);
   
           int defaultWidth = 200; //默认值
           int defaultHeight = 200;
   
           setMeasuredDimension(widthSize, heightSize);
           if (widthMode == MeasureSpec.AT_MOST && heightMode == MeasureSpec.AT_MOST) {
               setMeasuredDimension(defaultWidth, defaultHeight);
           } else if (widthMode == MeasureSpec.AT_MOST) {
               setMeasuredDimension(defaultWidth, heightSize);
           } else if (heightMode == MeasureSpec.AT_MOST) {
               setMeasuredDimension(widthSize, defaultHeight);
           }
       }
   ```

   可以看到，其实我们只是当是 warp_content 的时候设置一个默认值，但是这样不灵活，我们可以在自定义属性中设置。其次，我们可以参考系统对 TextView 的设置，它会根据文字的大小来设置默认宽高。

2. 自定义 ViewGroup 的处理

   ```java
       protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
           super.onMeasure(widthMeasureSpec, heightMeasureSpec);
           //将所有的子 View 进行测量，这会触发每个子 View 的 onMeasure
           //measureChild 是对单个 View 进行测量
           measureChildren(widthMeasureSpec, heightMeasureSpec);
   
           int widthMode = MeasureSpec.getMode(widthMeasureSpec);
           int widthSize = MeasureSpec.getSize(widthMeasureSpec);
           int heightMode = MeasureSpec.getMode(heightMeasureSpec);
           int heightSize = MeasureSpec.getSize(heightMeasureSpec);
   
           int childCount = getChildCount();
           if (childCount == 0) {
               setMeasuredDimension(0, 0);
           } else {
               if (widthMode == MeasureSpec.AT_MOST && heightMode == MeasureSpec.AT_MOST) {
                   int height = getTotalHeight();      //获取子 View 高度加和
                   int width = getMaxChildWidth();     //获取子 View 的最大宽度
                   setMeasuredDimension(width, height);
               } else if (heightMode == MeasureSpec.AT_MOST) {
                   setMeasuredDimension(widthSize, getTotalHeight());
               } else if (widthMode == MeasureSpec.AT_MOST) {
                   setMeasuredDimension(getMaxChildWidth(), heightSize);
               }
           }
       }
   ```

   这里我们以自定义一个垂直的线性布局为例，当 ViewGroup 是 warp_content 的时候，高度为子 View 的高度和，宽度为子 View 中的最大宽度。

##### 处理 padding

1. 自定义 View 的处理

   ```java
       protected void onDraw(Canvas canvas) {
           mPaint.setColor(Color.RED);
           Rect rect = new Rect(0, 0, 100, 100);
           Rect rect1 = new Rect(0 + getPaddingLeft(), 0 + getPaddingTop(), 100 - getPaddingRight(), 100 - getPaddingBottom());
           canvas.drawRect(rect, mPaint);
       }
   ```

2. 自定义 ViewGroup 的处理

   ```java
       protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
           super.onMeasure(widthMeasureSpec, heightMeasureSpec);
           //将所有的子 View 进行测量，这会触发每个子 View 的 onMeasure
           //measureChild 是对单个 View 进行测量
           measureChildren(widthMeasureSpec, heightMeasureSpec);
   
           int widthMode = MeasureSpec.getMode(widthMeasureSpec);
           int widthSize = MeasureSpec.getSize(widthMeasureSpec);
           int heightMode = MeasureSpec.getMode(heightMeasureSpec);
           int heightSize = MeasureSpec.getSize(heightMeasureSpec);
   
           //获取子 View 高度加和 padding 值
           int height = getTotalHeight() + getPaddingTop() + getPaddingBottom();      
           //获取子 View 的最大宽度加 padding 值
           int width = getMaxChildWidth() + getPaddingLeft() + getPaddingRight(); 
   
           int childCount = getChildCount();
           if (childCount == 0) {
               setMeasuredDimension(0, 0);
           } else {
               if (widthMode == MeasureSpec.AT_MOST && heightMode == MeasureSpec.AT_MOST) {
                   setMeasuredDimension(Math.min(width, widthSize), Math.min(height, heightSize));
               } else if (heightMode == MeasureSpec.AT_MOST) {
                   setMeasuredDimension(widthSize, Math.min(height, heightSize));
               } else if (widthMode == MeasureSpec.AT_MOST) {
                   setMeasuredDimension(Math.min(width, widthSize), heightSize);
               }
           }
       }
   ```

##### 处理 margin

自定义 View 里 margin 是生效的，无需处理，只有 ViewGroup 才需要处理 margin。

```java
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int count = getChildCount();
        int currentHeight = t;
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
            int height = child.getMeasuredHeight();
            int width = child.getMeasuredWidth();
            child.layout(l + lp.leftMargin, currentHeight + lp.topMargin, l + width + lp.leftMargin + lp.rightMargin, currentHeight + height + lp.topMargin + lp.bottomMargin);
            currentHeight += height;
        }
    }

    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new MarginLayoutParams(getContext(), attrs);
    
```

#### 实战
##### 圆形进度条
```java
package cn.org.octopus.circle;
 
import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.graphics.Rect;
import android.graphics.RectF;
import android.graphics.Typeface;
import android.util.AttributeSet;
import android.widget.ImageView;
 
public class CircleProcess extends ImageView {
 
	/** 画笔 */
	private Paint mPaint;
	/** 上下文对象 */
	private Context mContext;
	/** 进度条的值 */
	private int mProcessValue;
	
	public CircleProcess(Context context, AttributeSet attrs) {
		super(context, attrs);
		// 初始化成员变量 Context
		mContext = context;
		// 创建画笔, 并设置画笔属性
		mPaint = new Paint();
		// 消除绘制时产生的锯齿
		mPaint.setAntiAlias(true);
		// 绘制空心圆形需要设置该样式
		mPaint.setStyle(Style.STROKE);
	}
	
	/**
	 * 自定义布局实现的 只有 Context 参数的构造方法
	 * @param context
	 */
	public CircleProcess(Context context) {
		super(context);
	}
	
	/**
	 * 自定义布局实现的 三个参数的构造方法
	 * @param context
	 * @param attrs
	 * @param defStyle
	 */
	public CircleProcess(Context context, AttributeSet attrs, int defStyle) {
		super(context, attrs, defStyle);
	}
	
	@Override
	protected void onDraw(Canvas canvas) {
		super.onDraw(canvas);
		
		//获取圆心的 x 轴位置
		int center = getWidth() / 2;
		/*
		 * 中间位置 x 减去左侧位置 的绝对值就是圆半径, 
		 * 注意 : 由于 padding 属性存在, |left - right| 可能与 width 不同
		 */
		int outerRadius = Math.abs(getLeft() - center);
		//计算内圆半径大小, 内圆半径 是 外圆半径的一般
		int innerRadius = outerRadius / 2;
		
		//设置画笔颜色
		mPaint.setColor(Color.BLUE);
		//设置画笔宽度
		mPaint.setStrokeWidth(2);
		//绘制内圆方法 前两个参数是 x, y 轴坐标, 第三个是内圆半径, 第四个参数是 画笔
		canvas.drawCircle(center, center, innerRadius, mPaint);
		
		/*
		 * 绘制进度条的圆弧
		 * 
		 * 绘制图形需要 left top right bottom 坐标, 下面需要计算这个坐标
		 */
		
		//计算圆弧宽度
		int width = outerRadius - innerRadius;
		//将圆弧的宽度设置给 画笔
		mPaint.setStrokeWidth(width);
		/*
		 * 计算画布绘制圆弧填入的 top left bottom right 值, 
		 * 这里注意给的值要在圆弧的一半位置, 绘制的时候参数是从中间开始绘制
		 */
		int top = center - (innerRadius + width/2);
		int left = top;
		int bottom = center + (innerRadius + width/2);
		int right = bottom;
		
		//创建圆弧对象
		RectF rectf = new RectF(left, top, right, bottom);
		//绘制圆弧 参数介绍 : 圆弧, 开始度数, 累加度数, 是否闭合圆弧, 画笔
		canvas.drawArc(rectf, 270, mProcessValue, false, mPaint);
		
		//绘制外圆
		mPaint.setStrokeWidth(2);
		canvas.drawCircle(center, center, innerRadius + width, mPaint);
		
		/*
		 * 在内部正中央绘制一个数字
		 */
		//生成百分比数字
		String str = (int)(mProcessValue * 1.0 / 360 * 100) + "%"; 
		/*
		 * 测量这个数字的宽 和 高
		 */
		//创建数字的边界对象
		Rect textRect = new Rect();
		//设置数字的大小, 注意要根据 内圆半径设置
		mPaint.setTextSize(innerRadius / 2);
		mPaint.setStrokeWidth(0);
		//获取数字边界
		mPaint.getTextBounds(str, 0, str.length(), textRect);
		int textWidth = textRect.width();
		int textHeight = textRect.height();
		
		//根据数字大小获取绘制位置, 以便数字能够在正中央绘制出来
		int textX = center - textWidth / 2;
		int textY = center + textHeight / 2;
		
		//正式开始绘制数字
		canvas.drawText(str, textX, textY, mPaint);
	}
 
	
	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
		/*
		 * setMeasuredDimension 方法 : 该方法决定当前的 View 的大小
		 * 根据 View 在布局中的显示, 动态获取 View 的宽高
		 * 
		 * 当布局组件 warp_content 时 : 
		 * 从 MeasureSpec 获取的宽度 : 492 高度 836 , 
		 * 默认 宽高 都是 120dip转化完毕后 180px 
		 * 
		 * 当将布局组件 的宽高设置为 240 dp : 
		 * 宽度 和 高度 MeasureSpec 获取的都是 360, 此时 MeasureSpec 属于精准模式
		 * 
		 */
		setMeasuredDimension(measure(widthMeasureSpec), measure(heightMeasureSpec));
	}
	
	/**
	 * 获取组件宽度
	 * 
	 * MeasureSpec : 该 int 类型有 32 位, 前两位是状态位, 后面 30 位是大小值;
	 * 		常用方法 : 
	 * 		-- MeasureSpec.getMode(int) : 获取模式
	 *      -- MeasureSpec.getSize(int) : 获取大小
	 *      -- MeasureSpec.makeMeasureSpec(int size, int mode) : 创建一个 MeasureSpec;
	 *      -- MeasureSpec.toString(int) : 模式 + 大小 字符串
	 *      
	 *      模式介绍 : 注意下面的数字是二进制的
	 *      -- 00 : MeasureSpec.UNSPECIFIED, 未指定模式;
	 *      -- 01 : MeasureSpec.EXACTLY, 精准模式;
	 *      -- 11 : MeasureSpec.AT_MOST, 最大模式;
	 *      
	 *      注意 : 这个 MeasureSpec 模式是在 onMeasure 方法中自动生成的, 一般不用去创建这个对象
	 *      
	 * @param widthMeasureSpec
	 * 				MeasureSpec 数值
	 * @return
	 * 				组件的宽度
	 */
	private int measure(int measureSpec) {
		//返回的结果, 即组件宽度
		int result = 0;
		//获取组件的宽度模式
		int mode = MeasureSpec.getMode(measureSpec);
		//获取组件的宽度大小 单位px
		int size = MeasureSpec.getSize(measureSpec);
		
		if(mode == MeasureSpec.EXACTLY){//精准模式
			result = size;
		}else{//未定义模式 或者 最大模式
			//注意 200 是默认大小, 在 warp_content 时使用这个值, 如果组件中定义了大小, 就不使用该值
			result = dip2px(mContext, 200);
			if(mode == MeasureSpec.AT_MOST){//最大模式
				//最大模式下获取一个稍小的值
				result = Math.min(result, size);
			}
		}
		
		return result;
	}
	
	
	/**
	 * 将手机的 设备独立像素 转为 像素值
	 * 
	 * 		公式 : px / dip = dpi / 160
	 * 			   px = dip * dpi / 160;
	 * @param context
	 * 				上下文对象
	 * @param dpValue
	 * 				设备独立像素值
	 * @return
	 * 				转化后的 像素值
	 */
	public static int dip2px(Context context, float dpValue) {
		final float scale = context.getResources().getDisplayMetrics().density;
		return (int) (dpValue * scale + 0.5f);
	}
 
	/**
	 * 将手机的 像素值 转为 设备独立像素
	 * 		公式 : px/dip = dpi/160
	 * 			   dip = px * 160 / dpi
	 * 			   dpi (dot per inch) : 每英寸像素数 归一化的值 120 160 240 320 480;
	 * 			   density : 每英寸的像素数, 精准的像素数, 可以用来计算准确的值
	 * 			   从 DisplayMetics 中获取的
	 * @param context
	 * 				上下文对象
	 * @param pxValue
	 * 				像素值
	 * @return
	 * 				转化后的 设备独立像素值
	 */
	public static int px2dip(Context context, float pxValue) {
		final float scale = context.getResources().getDisplayMetrics().density;
		return (int) (pxValue / scale + 0.5f);
	}
 
	/**
	 * 获取当前进度值
	 * @return
	 * 				返回当前进度值
	 */
	public int getmProcessValue() {
		return mProcessValue;
	}
 
	/**
	 * 为该组件设置进度值
	 * @param mProcessValue
	 * 				设置的进度值参数
	 */
	public void setmProcessValue(int mProcessValue) {
		this.mProcessValue = mProcessValue;
	}
	
}
```

##### 数字进度条
![](https://camo.githubusercontent.com/8d4de2d0ee4b42d27424f490dc54e4e1ecb851ff/68747470733a2f2f692e6c6f6c692e6e65742f323031382f30322f30342f356137363962326435323837372e676966)

```java
public class MyProgressView extends View {

    private Paint mPaint;
    private int mWidth;
    private int mHeight;
    private int textPadding = 5;
    private int progress = 0;

    public MyProgressView(Context context) {
        super(context);
        initPaint();
    }

    public MyProgressView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        initPaint();
    }

    private void initPaint() {
        mPaint = new Paint();
        mPaint.setAntiAlias(true);
        mPaint.setStrokeWidth(3);
        mPaint.setStyle(Paint.Style.FILL);
        mPaint.setTextSize(14);
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        mWidth = w;
        mHeight = h;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        String text = progress + "%";
        float textWidth = mPaint.measureText(text) + textPadding;
        Rect rect = new Rect();
        mPaint.getTextBounds(text, 0, text.length(), rect);
        mPaint.setColor(Color.BLUE);

        canvas.drawLine(0, mHeight / 2, progress * ((mWidth - textWidth) / 100), mHeight / 2, mPaint);
        canvas.drawText(text, progress * ((mWidth - textWidth) / 100) + textPadding, (mHeight - rect.height()) / 2 + 2 * textPadding, mPaint);
        mPaint.setColor(Color.GRAY);
        canvas.drawLine(progress * ((mWidth - textWidth) / 100) + textWidth + textPadding, mHeight / 2, mWidth, mHeight / 2, mPaint);
    }

    public void setProgress(int progress) {
        if (progress > 100) {
            progress = 100;
        } else if (progress < 0) {
            progress = 0;
        }
        this.progress = progress;
        postInvalidate();
    }
}
```
