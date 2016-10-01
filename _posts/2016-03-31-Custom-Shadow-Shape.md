---
title: 自定义阴影形状
---

[Material Design](https://material.google.com/ "Material Design") 中介绍了 [Elevation and shadows](https://material.google.com/material-design/elevation-shadows.html "Elevation and shadows") 的概念，通常来说我们们只需要使用 `View.setElevation()` 这个方法来设置投影就可以了。

但是 Android 默认只支持给**圆形**、**矩形**、**圆角矩形**这 3 种形状的 View 添加投影，如果遇到其他特别的形状，比如三角形，系统就没法很好地支持。这种情况下你可以选择自己切图来模拟，但是效果通常都不是很好，如果要一句话形容的话，大概就是没有系统绘制的投影带来的那种**质感**。所以我们就要自己来实现相关方法来让系统支持。

为了让系统支持自定义形状的投影，我们需要了解 [Outline](https://developer.android.com/reference/android/graphics/Outline.html?hl=zh-cn "Outline") 的概念。 Outline 也就是轮廓，向系统描述了如何绘制投影。一般我们有两种方式来设置 Outline ，一种是通过重写背景 Drawable 的 getOutline() 方法，另一种则是直接使用 `View.setOutlineProvider()` 来设置。两种方法都差不多，选取第一种方法来说明。

首先我们需要手动绘制出 Outline 所需要的路径，这个路径是有要求的，必须是**凸多边形**，维基百科对凸多边形的[描述](https://zh.wikipedia.org/wiki/%E5%87%B8%E5%A4%9A%E8%BE%B9%E5%BD%A2%E5%92%8C%E5%87%B9%E5%A4%9A%E8%BE%B9%E5%BD%A2 "凸多边形和凹多边形")是：

 - 每个内角小于或等于 180 度。
 - 任何两个顶点间的线段位于多边形的内部或边界上。

简而言之就是说，三角形是凸的，矩形也是凸的，但是五角星形就不是凸的。虽然在自然世界里五角星形也是有投影的，但是在 Android 中不支持。如果你给 Outline 设置了凹多边形，那么系统会直接抛出异常，可以在设置前使用 `Path.isConvex()` 方法事先判断一下是不是凸多边形。

好啦噜，现在让我们在界面上画一个三角形的例子吧：

{% highlight java %}
@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    FrameLayout frame = (FrameLayout) findViewById(R.id.frame);
    frame.setBackground(buildBackground());
    frame.setElevation(8.0f);
    ...
}

private Drawable buildBackground() {
    return new ShapeDrawable(new RectShape() {
        @Override
        public void draw(Canvas canvas, Paint paint) {
            paint.setColor(Color.WHITE);
            paint.setStyle(Paint.Style.FILL);
            canvas.drawPath(buildConvexPath(), paint);
        }

        @Override
        public void getOutline(Outline outline) {
            outline.setConvexPath(buildConvexPath());
        }

        private Path buildConvexPath() {
            Path path = new Path();
            path.lineTo(rect().left, rect().top);
            path.lineTo(rect().right, rect().top);
            path.lineTo(rect().left, rect().bottom);
            path.close();
            return path;
        }
    });
} {% endhighlight %}

绘制出来的效果图是这样的：

![Triangle](/assets/img/2016-03-31-Triangle.png "Triangle")

怎么样，看起来很不错吧。

那如果我非要给一个凹多边形添加投影怎么办？其实也很简单，具有**相同投影高度**的图形拼接在一起时，拼接处的阴影会被自动合并而消失掉，所以整体看起来你就拥有了一块完整的图形。三角形恰好又是计算机图形学中面的最基本单位，所以你可以把你的凹多边形分解为一个个小的三角形分别绘制，再拼接起来就 OK 。

好啦，现在你可以尽情给你想要的形状添加投影，超赞！