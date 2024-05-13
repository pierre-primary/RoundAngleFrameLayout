##android 通用圆角控件

圆角控件就是对 View的Canvas进行改变轮廓的处理
<font color="#ff00000">
改变轮廓两种方式：
1.剪切（clip()）
剪切clip是对画布进行剪切，只对剪切后的绘制起效果。
ps：Canvas的图形变换平移、放缩、旋转、错切、裁剪都是只对后面的绘制起效果，
对应Matrix中preXXX，Matrix变换分为preXXX，postXXX，setXXX；preXXX将新的变换操作插到队列前，postXXX将新的变换操作插到队列后，setXXX是先reset()清除前面的变换操作并设置新的变换操作，且都是只对后面的绘制起效果。
Canvas的save，restore对变换操作进行保存，和还原，带参的restoreToCount(save())，可以指定还原到第几次保存的状态。
</font>
<font color="#ff00000">
2.遮罩（PorterDuffXfermode）
安卓提供多种遮罩模式选择
![这里写图片描述](http://img.blog.csdn.net/20160420054812946)
遮罩是设置在Paint上的只对 当前绘制的操作有效
</font>


####下面利用这两种方式实现圆角控件
<font color="#ff00000">
onDraw
onDrawForeground
dispatchDraw
这三个回调函数都是可以操作View的Canvas；onDraw，onDrawForeground这两个是在View绘制背景，自身内容和前景时回调的 只有设置了背景、自身内容、前景时才会配回调，并且对这两个函数的参数Canvas上的操作，只对背景、自身内容、前景有效。
dispatchDraw是绘制子控件时的回调，参数Canvas可以对子控件的画布进行处理。
</font>
通用圆角控件必须对子控件的对应位置也是原价所以我门选择在dispatchDraw中进行圆角处理。

剪切（clip()）
```java
    @Override
    protected void dispatchDraw(Canvas canvas) {
        int width = getWidth();
        int height = getHeight();
        Path path = new Path();
        path.moveTo(0, topLeftRadius);
        path.arcTo(new RectF(0, 0, topLeftRadius * 2, topLeftRadius * 2), -180, 90);
        path.lineTo(width - topRightRadius, 0);
        path.arcTo(new RectF(width - 2 * topRightRadius, 0, width, topRightRadius * 2), -90, 90);
        path.lineTo(width, height - bottomRightRadius);
        path.arcTo(new RectF(width - 2 * bottomRightRadius, height - 2 * bottomRightRadius, width, height), 0, 90);
        path.lineTo(bottomLeftRadius, height);
        path.arcTo(new RectF(0, height - 2 * bottomLeftRadius, bottomLeftRadius * 2, height), 90, 90);
        path.close();
        canvas.clipPath(path);
        super.dispatchDraw(canvas);
    }
```
效果图：
![这里写图片描述](http://img.blog.csdn.net/20160420144007744)

上图能看到明显的锯齿 因为 安卓虽提供了抗锯齿功能但是是在Paint上操作的 clip过程没有用到Paint 无法达到抗锯齿目的；

遮罩（PorterDuffXfermode）

####写法一

```java
 @Override
    protected void dispatchDraw(Canvas canvas) {
        super.dispatchDraw(canvas);
        drawTopLeft(canvas);//用PorterDuffXfermode
        drawTopRight(canvas);//用PorterDuffXfermode
        drawBottomLeft(canvas);//用PorterDuffXfermode
        drawBottomRight(canvas);//用PorterDuffXfermode
    }
```

效果图：
![这里写图片描述](http://img.blog.csdn.net/20160420144146871)

上图有黑色底色，view的Canvas底层画布的BItmap是 RGB_565的所以怎么画都有一个黑色底色。

我们可以new Canvas一个底层画布的BItmap是 ARGB_8888的绘制完后 再把这个底层画布的BItmap 绘制到View 的Canvas上

####写法二
```java 
    @Override
    protected void dispatchDraw(Canvas canvas) {
        Bitmap bitmap = Bitmap.createBitmap(canvas.getWidth(), canvas.getHeight(), Bitmap.Config.ARGB_8888);
        Canvas newCanvas = new Canvas(bitmap);
        super.dispatchDraw(newCanvas);
        drawTopLeft(newCanvas);
        drawTopRight(newCanvas);
        drawBottomLeft(newCanvas);
        drawBottomRight(newCanvas);
        canvas.drawBitmap(bitmap,0,0,imagePaint);
//        invalidate();
    }
```
效果图：
![这里写图片描述](http://img.blog.csdn.net/20160420144246653)

实现了，但是这种映射方式实现的 如果子控件中存在滑动控件，滑动时无法实时刷新，用Glide加载image到ImageView中时，WebView load时 都无法实时刷新，出现无法加载的效果，<font color="#ff0000">虽然可以加上invalidate通知刷新 但是掉帧明显</font>

我们只能用回写法一 但是要解决黑色背景的问题
<font color="#ff0000">只要加上一句代码 就能解决 默认黑色背景的问题</font>
```java
canvas.saveLayer(new RectF(0, 0, canvas.getWidth(), canvas.getHeight()), imagePaint,Canvas.ALL_SAVE_FLAG);
```

```java
 @Override
    protected void dispatchDraw(Canvas canvas) {
canvas.saveLayer(new RectF(0, 0, canvas.getWidth(), canvas.getHeight()), imagePaint,Canvas.ALL_SAVE_FLAG);
        super.dispatchDraw(canvas);
        drawTopLeft(canvas);//用PorterDuffXfermode
        drawTopRight(canvas);//用PorterDuffXfermode
        drawBottomLeft(canvas);//用PorterDuffXfermode
        drawBottomRight(canvas);//用PorterDuffXfermode
        canvas.restore();
    }
```
效果图：
![这里写图片描述](http://img.blog.csdn.net/20160420144327780)

因为view的Canvas底层画布的BItmap是 RGB_565
我们只要在保存为图层就行了。用过Photoshop的都知道默认底层画布是不透明的，要先解锁，而这里解锁就是保存为图层。

<font color="#ff0000">相关链接：</font>[http://blog.csdn.net/oyuanwa/article/details/51197546](http://blog.csdn.net/oyuanwa/article/details/51197546)
        



