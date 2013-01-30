---
layout: post
title: "对Android Drawable Resources的研究"
date: 2012-07-14 16:11
comments: true
categories: android
---
最近一次在android开发时想在GridView被按下的时候改变item的图片。可以通过xml配置selector背景选择器来达到，但是有100个item我是不是就要写100个xml呢？肯定不能够这样！就想到通过java代码来实现selector的效果。于是，找到了StateListDrawable这个类，进而进出对Drawable Resources的深入研究。

> 官方链接：[http://developer.android.com/guide/topics/resources/drawable-resource.html](http://developer.android.com/guide/topics/resources/drawable-resource.html)

首先从字面上理解，绘图资源泛指可以画在屏幕上的东西。

绘图资源可以分为以下几类：

1、位图文件(BitmapDrawable)：位图图像文件，支持jpg、png、gif

2、九片式图片(点9)文件(NinePatchDrawable)：可以拉伸的.9.png图片文件

3、图层列表(LayerDrawable)：一组图片,，index越大的画在最顶层

4、状态列表(StateListDrawable)：不同的状态(正常、按下、获得焦点等)对应不同的资源

5、等级列表(LevelListDrawable)：一组绘图，每张绘图制定了不同的等级，只有符合设定的等级的绘图才显示

6、过渡绘图(TransitionDrawable)：内部可以实现两张图片的渐隐渐现切换

7、插入绘图(InsetDrawable)：使用另一个绘图资源，同时指定了其边距

8、裁剪绘图(ClipDrawable)：可以从原资源上切割一部分显示

9、缩放绘图(ScaleDrawable)：对另一个资源进行缩放

10、形状(ShapeDrawable)：包含颜色和渐变的几何图形

11、动画(AnimationDrawable)

12、颜色(Color)：颜色也可以当做绘图资源使用

位图：可以直接使用包含扩展名的文件名或者R引用作为资源ID。使用xml位图来定义一个指向原位图的引用，可以增加一些属性(比如抗锯齿antialias，抖动处理dither)，使用方法和位图图片文件一样。比如：

`<?xml version=”1.0″ encoding=”utf-8″?>
<bitmap xmlns:android=”http://schemas.android.com/apk/res/android”
android:src=”@drawable/icon”
android:tileMode=”repeat” />`

点9：可以对图片进行伸缩来适用View尺寸的改变。与位图类似，点九也可以创建对应的xml引用文件。比如

`<?xml version=”1.0″ encoding=”utf-8″?>
<nine-patch xmlns:android=”http://schemas.android.com/apk/res/android”
android:src=”@drawable/myninepatch”
android:dither=”false” />`

图层列表：按照index的顺序来绘制。item必须是<selector>的子类，默认情况下回进行缩放来适应。xml中的定义方式如下：

`<?xml version=”1.0″ encoding=”utf-8″?>
<layer-list xmlns:android=”http://schemas.android.com/apk/res/android”>
<item android:drawable=”@drawable/android_red” />
<item android:top=”10dp” android:left=”10dp”>
<bitmap android:src=”@drawable/android_green”
android:gravity=”center” />
</item>
</layer-list>`

状态列表：为每个资源item制定不同的状态。在匹配的时候，第一个与制定状态匹配的item被使用到，不一定是最优的，而是第一个。因此，一需要把默认选项放在最后一个位置。定义示例：

`<?xml version=”1.0″ encoding=”utf-8″?>
<selector xmlns:android=”http://schemas.android.com/apk/res/android”>
<item android:state_pressed=”true”
android:drawable=”@drawable/button_pressed” /> <!– pressed –>
<item android:state_focused=”true”
android:drawable=”@drawable/button_focused” /> <!– focused –>
<item android:state_hovered=”true”
android:drawable=”@drawable/button_focused” /> <!– hovered –>
<item android:drawable=”@drawable/button_normal” /> <!– default –>
</selector>`

级别列表：示例：

`<?xml version=”1.0″ encoding=”utf-8″?>
<level-list xmlns:android=”http://schemas.android.com/apk/res/android” >
<item
android:drawable=”@drawable/status_off”
android:maxLevel=”0″ />
<item
android:drawable=”@drawable/status_on”
android:minLevel=”1″
android:maxLevel=”2″ />
</level-list>`

过渡绘图：只支持最多2个drawable。例如

`<?xml version=”1.0″ encoding=”utf-8″?>
<transition xmlns:android=”http://schemas.android.com/apk/res/android”>
<item android:drawable=”@drawable/on” />
<item android:drawable=”@drawable/off” />
</transition>`

插入绘图：示例：

`<?xml version=”1.0″ encoding=”utf-8″?>
<inset xmlns:android=”http://schemas.android.com/apk/res/android”
android:drawable=”@drawable/background”
android:insetTop=”10dp”
android:insetLeft=”10dp” />`

裁剪绘图：通过设置level来决定裁剪的大小。默认的是0(可见区域为0)，全部裁剪的level是10000。定义的示例：

`<?xml version=”1.0″ encoding=”utf-8″?>
<clip xmlns:android=”http://schemas.android.com/apk/res/android”
android:drawable=”@drawable/android”
android:clipOrientation=”horizontal”
android:gravity=”left” />`

缩放绘图：定义如下：

`<?xml version=”1.0″ encoding=”utf-8″?>
<scale xmlns:android=”http://schemas.android.com/apk/res/android”
android:drawable=”@drawable/logo”
android:scaleGravity=”center_vertical|center_horizontal”
android:scaleHeight=”80%”
android:scaleWidth=”80%” />`

几何形状绘图：根节点为<shape>，包含的属性为shape(rectangle矩形|oval椭圆|line线条|ring环形)。当shape属性为ring时还有innerRadius、innerRadiusRatio、thickness、thicknessRatio、useLevel。它支持以下图形的组合：

<corners>圆角，在矩形的时候生效。注意bottomLeftRadius是右下角，bottomRightRadius是左下角。

<gradient>渐变。包含属性：angle,渐变的方向角度，必须是45的倍数，默认为0，即从左至右，逆时针计算角度；centerX,x的渐变中心位置，0~1.0；centerY，y方向的渐变中心位置；startColor、centerColor、endColor分别是渐变的开始、中间、结束颜色；type，渐变的类型(linear线性|radial中心放射性|sweep扇形)；gradientRadius，渐变半径，中心放射类型的渐变时生效；useLevel，是否作为等级绘图使用。

<padding>内边距。left、top、right、bottom四个方向。

<size>大小尺寸。height、width。

<solid>填充颜色。color。

<stroke>描边。包含属性：width，描边的粗细；color，描边的颜色；dashGap，dashWidth，两个一起使用，使用虚线描边，分别指虚线每条线
的间距和长度。

定义如下：

`<?xml version=”1.0″ encoding=”utf-8″?>
<shape xmlns:android=”http://schemas.android.com/apk/res/android”
android:shape=”rectangle”>
<gradient
android:startColor=”#FFFF0000″
android:endColor=”#80FF00FF”
android:angle=”45″/>
<padding android:left=”7dp”
android:top=”7dp”
android:right=”7dp”
android:bottom=”7dp” />
<corners android:radius=”8dp” />
</shape>`