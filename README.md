# 前言

前面俩篇进行了艰苦的爬坑之旅，主要是对官方代码进行了修改和编译。
这一篇我们就来干点轻松的事吧——
- 给ZXing美美容
- 调调API，^_^。

# 为ZXing美容
一般的需求都会涉及到四个方面：
- 矩形框大小
- 矩形框四角的图形&颜色
- 扫描激光动画
- 噪点

## 矩形框大小调整 
主要集中在这个类$CameraManager.java

### 相关的代码： 
**getFramingRect()**
```java
public synchronized Rect getFramingRect() {
              int width = findDesiredDimensionInRange(screenResolution.x, MIN_FRAME_WIDTH, MAX_FRAME_WIDTH);
	      int height = findDesiredDimensionInRange(screenResolution.y, MIN_FRAME_HEIGHT, MAX_FRAME_HEIGHT);
	
	      int leftOffset = (screenResolution.x - width) / 2;
	      int topOffset = (screenResolution.y - height) / 2;
	... ...
}
```

**findDesiredDimensionInRange**
```java
private static int findDesiredDimensionInRange(int resolution, int hardMin, int hardMax) {
    int dim = 5 * resolution / 8; // Target 5/8 of each dimension
    if (dim < hardMin) {
      return hardMin;
    }
    if (dim > hardMax) {
      return hardMax;
    }
    return dim;
  }
```

**还有几个常量**
```java
  private static final int MIN_FRAME_WIDTH = 240;
  private static final int MIN_FRAME_HEIGHT = 240;
  private static final int MAX_FRAME_WIDTH = 1200; // = 5/8 * 1920
  private static final int MAX_FRAME_HEIGHT = 675; // = 5/8 * 1080
```
### 那么我们具体要怎么改呢？： 
在不考虑适配的情况下，为了简单起见，譬如这么改：
```java
public synchronized Rect getFramingRect() {
//      int width = findDesiredDimensionInRange(screenResolution.x, MIN_FRAME_WIDTH, MAX_FRAME_WIDTH);
//      int height = findDesiredDimensionInRange(screenResolution.y, MIN_FRAME_HEIGHT, MAX_FRAME_HEIGHT);

      int width = 350;
      int height = 500;
	
	      int leftOffset = (screenResolution.x - width) / 2;
	      int topOffset = (screenResolution.y - height) / 2;
	... ...
}
```
然后修改出来的效果是：
![](https://wx3.sinaimg.cn/mw690/0061ejqJgy1fmpo15et24j307l0dfaf4.jpg)

## 矩形框四角的图形&颜色
$ViewfinderView.java

### 声明字段：
```java
//  private int laserColor = Scanner.color.VIEWFINDER_LASER;//扫描线颜色
//  private int laserFrameBoundColor = laserColor;//扫描框4角颜色
  private int laserLineTop;// 扫描线最顶端位置
  private int laserLineHeight;//扫描线默认高度
  private int laserMoveSpeed;// 扫描线默认移动距离px
  private int laserFrameCornerWidth = 20;//扫描框4角宽
  private int laserFrameCornerLength = 20;//扫描框4角高
  private int laserLineResId;//扫描线图片资源
  private String drawText = "将二维码放入框内，即可自动扫描";//提示文字
  private int drawTextSize;//提示文字大小
  private int drawTextColor = Color.WHITE;//提示文字颜色
  private boolean drawTextGravityBottom = true;//提示文字位置
  private int drawTextMargin;//提示文字与扫描框距离
```

### 声明方法：
```java
  /**
   * 绘制扫描框4角
   *
   * @param canvas
   * @param frame
   */
  private void drawFrameCorner(Canvas canvas, Rect frame) {
    paint.setColor(getResources().getColor(android.R.color.holo_green_dark));
    paint.setStyle(Paint.Style.FILL);
    // 左上角
    canvas.drawRect(frame.left - laserFrameCornerWidth, frame.top, frame.left, frame.top
            + laserFrameCornerLength, paint);
    canvas.drawRect(frame.left - laserFrameCornerWidth, frame.top - laserFrameCornerWidth, frame.left
            + laserFrameCornerLength, frame.top, paint);
    // 右上角
    canvas.drawRect(frame.right, frame.top, frame.right + laserFrameCornerWidth,
            frame.top + laserFrameCornerLength, paint);
    canvas.drawRect(frame.right - laserFrameCornerLength, frame.top - laserFrameCornerWidth,
            frame.right + laserFrameCornerWidth, frame.top, paint);
    // 左下角
    canvas.drawRect(frame.left - laserFrameCornerWidth, frame.bottom - laserFrameCornerLength,
            frame.left, frame.bottom, paint);
    canvas.drawRect(frame.left - laserFrameCornerWidth, frame.bottom, frame.left
            + laserFrameCornerLength, frame.bottom + laserFrameCornerWidth, paint);
    // 右下角
    canvas.drawRect(frame.right, frame.bottom - laserFrameCornerLength, frame.right
            + laserFrameCornerWidth, frame.bottom, paint);
    canvas.drawRect(frame.right - laserFrameCornerLength, frame.bottom, frame.right
            + laserFrameCornerWidth, frame.bottom + laserFrameCornerWidth, paint);
  }
```
### 方法调用的位置：
```java
  @SuppressLint("DrawAllocation")
  @Override
  public void onDraw(Canvas canvas) {

	    ......

    if (resultBitmap != null) {
      // Draw the opaque result bitmap over the scanning rectangle
      paint.setAlpha(CURRENT_POINT_OPACITY);
      canvas.drawBitmap(resultBitmap, null, frame, paint);
    } else {

	    ......


		//add code
		drawFrameCorner(canvas,frame);
		//end add code

	    ......

	}

  }


```

### 看效果：
![](https://wx4.sinaimg.cn/mw690/0061ejqJgy1fmpp44w3nwj307k0ddaf4.jpg)


## 扫描激光动画调整
还是在这个类：$ViewfinderView.java

**onDraw()内**

代码中有注释，扫描动画是如下部分代码：
```java
  // Draw a red "laser scanner" line through the middle to show decoding is active
      paint.setColor(laserColor);
      paint.setAlpha(SCANNER_ALPHA[scannerAlpha]);
      scannerAlpha = (scannerAlpha + 1) % SCANNER_ALPHA.length;
      int middle = frame.height() / 2 + frame.top;
      canvas.drawRect(frame.left + 2, middle - 1, frame.right - 1, middle + 2, paint);
```

**分析：**

- 颜色绘制：
> paint.setColor(laserColor);

- 动画绘制：
>  paint.setAlpha(SCANNER_ALPHA[scannerAlpha]);
>  canvas.drawRect(frame.left + 2, middle - 1, frame.right - 1, middle + 2, paint);

**那么我们修改的部分就是这俩部分。**

### 修改颜色
颜色不用多说，直接将其颜色值更改就O了，这里为了简单期间，我直接是这么设置的：
> laserColor = resources.getColor(android.R.color.holo_green_dark);

### 修改动画
其实也算不上动画，我们来看,真正绘制到屏幕上是这一句：
> canvas.drawRect(frame.left + 2, middle - 1, frame.right - 1, middle + 2, paint);

我们玩过自定义控件都知道，这句话的意思就是画一个矩形：
```java
  public void drawRect(float left, float top, float right, float bottom, Paint paint) {
        throw new RuntimeException("Stub!");
    }
```
看见没，ondraw的时候我们不停的改变它的上坐标，下坐标，那么绘制的位置是不是就会跟着改变？
**答案是否定的，但是思路是这个思路!**
我们跟着这个思路对drawRect进行修改。

- **step 1：定义一个新的扫描线方法：根据上面的分析思路**
在ViewfinderView.java初始化的时候进行"线"素材的初始化（我这里就直接拿默认图了，真实项目上肯定是要用一张比较漂亮的图来替换的）。
```java
    scanLight = BitmapFactory.decodeResource(this.getResources(), R.drawable.launcher_icon);
```

- **step 2：定义一个新的扫描线方法：根据上面的分析思路**
```java
声明全局变量：
  int scanLineTop;
  int SCAN_VELOCITY = 15;

/**
   * 绘制扫描线方法
   * @param canvas
   * @param frame
   */
  private void drawScanLight(Canvas canvas, Rect frame) {

    if (scanLineTop == 0) {
      scanLineTop = frame.top;
    }

    if (scanLineTop >= frame.bottom - 30) {
      scanLineTop = frame.top;
    } else {
      scanLineTop += SCAN_VELOCITY;// SCAN_VELOCITY可以在属性中设置，默认为5
    }
    Rect scanRect = new Rect(frame.left, scanLineTop, frame.right, scanLineTop + 30);
    canvas.drawBitmap(scanLight, null, scanRect, paint);
  }
```
- **step 3：注释掉原来的绘制语句，调用新定义的方法：**
```java
//      canvas.drawRect(frame.left + 2, mTop, frame.right - 1, middle + 2, paint);
      drawScanLight(canvas,frame);
```


- **step 4：优化显示内容...：**
这里我只是提供一个基本实现的思路，就暂时不费那个劲了，至于怎么样让这个界面看着更漂亮，那就是你的事儿啦...
我的建议是：
~闪烁这个效果可以去掉了(setAlpha)。
~速率可以再调整快些(SCAN_VELOCITY越大越快)。


### 效果观察：
![](https://wx3.sinaimg.cn/mw690/0061ejqJgy1fmsuxm0pkkg307y0f54qp.gif)

本篇就到这里，下篇我们来玩玩API。

# Demo
https://github.com/zj614android/niceZxing
