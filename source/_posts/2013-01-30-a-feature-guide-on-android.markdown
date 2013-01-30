---
layout: post
title: "一个Android程序中新手引导功能实现方式的变迁"
date: 2012-07-14 16:12
comments: true
categories: android
---
最近做的一个项目中，引入了首次进入程序时的显示新手引导的功能，全屏显示，可以手指拖动左右翻页。效果类似于[淘宝Android客户端](http://android.taobao.com/)3.0版本中新手引导的效果。

下面说说这个需要全屏的新手引导都经历了一些什么变化吧：

#### 1、第一阶段：加在首页的PopupWindow上面

`private void addFeatureGuide() {
LayoutInflater layoutInflater = LayoutInflater.from(this);
LinearLayout popContentView = (LinearLayout) layoutInflater.inflate(R.layout.popup, null);
PopupWindow popupWindow = new PopupWindow(popContentView, LayoutParams.FILL_PARENT, LayoutParams.FILL_PARENT);
popupWindow.showAtLocation(this.findViewById(R.id.main_root), Gravity.CENTER, 0, 0);
}`

在onResume的时候调用addFeatureGuide()方法。

这样应该OK了吧？不！Run的时候报错了：

`FATAL EXCEPTION: main
java.lang.RuntimeException: Unable to resume activity {com.example/com.example.MyActivity}: android.view.WindowManager$BadTokenException: Unable to add window — token null is not valid; is your activity running?
at android.app.ActivityThread.performResumeActivity(ActivityThread.java:3128)
at android.app.ActivityThread.handleResumeActivity(ActivityThread.java:3143)
at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2684)
at android.app.ActivityThread.access$2300(ActivityThread.java:125)`

从错误日志中得出，在activity显示出来之前，拿不到activity将要显示在它上面的window的token。没有了window token，PopupWindow这个寄生的对象也就加不上去了。

那好吧，既然这样，我就在activity显示出来之后再加popupWindow上去吧。先这样试试：

`mHandler.sendEmptyMessageDelayed(MSG_WHAT_SHOWPOP, 1000);`

这样Activity将在1秒之后（先假设activity在resume一秒钟之后显示出来了）接收到消息，将popupWindow加上去。
结果，暂时是能够把PopupWindow显示出来。

问题：PopupWindow加上去的时延无法判断。显示太晚，会把首页内容看到之后一段时间才看得到新手引导；太早，同样会出现拿不到window token的问题，导致程序崩溃。所以这种方法不可取。

#### 2、第二阶段：作为一个单独的Activity，在Welcome之后首页之前显示。

新手引导作为一个独立的Activity的contentView。Welcome页面完了之后，跳到新手引导的Activity；在用户左右滑动翻到新手引导的最后一页的时候，跳到首页MainActivity。

问题：进入首页需要的时间太久。首页MainActivity是个庞然大物，进入的时候要显示N多界面元素，还要启动几个线程。用户在新手引导滑动完了之后，需要停滞几秒钟才能看到首页。不能忍受！

#### 3、第三阶段：加在导航栏里面

导航栏是一个利用WindowManager实现的东西(感觉很心虚)。导航栏是一个独立于Activity之外的东西，在欢迎页面的时候就会去实例化。因此，如果我把新手引导加在导航栏上面，那么，就能在首页显示出来之前把新手导航显示出来，并且在新手导航消失之后马上看到首页(仅仅把新手引导的view remove掉就是)。

问题：导航栏的功能不包括新手引导，不应该把多余的东西加入到导航栏上面。

#### 4、第四阶段：加在WindowManager上面

想了一下，为什么前面要把新手引导加到导航栏上面呢？其实是项利用导航栏的某些特性，比如独立于activity，可以在覆盖住整个屏幕。那么，我想要的其实不是导航栏，而是实现导航栏所使用的WindowManager。

既然如此，我就把新手引导放WindowManager里面，实现的地方跟导航栏独立开来，和导航栏是同一个级别的东西。

在首页MainActivity create的时候，我就把新手引导的视图生成，然后加到WindowManger上面。

问题：

A、在新手引导要消失的时候，需要把新手引导的view remove掉，然后让导航栏显示出来。导航栏出来得比较慢，首页下部会显示白色很久然后导航栏才显示。

B、由于新手引导要求全屏，首页又非全屏，在锁屏之后：按下手机解锁键，有状态栏显示；滑动解锁之后，需要重新设置window flag，即使马上调用新手导航view的requestLayout方法，也会出现界面显示出来后突然向上跳一下的效果，体验不好。

C、而且，使用WindowManager这种更偏底层的元素，可能会引发一些未知的风险。

#### 5、第五阶段：作为首页Activity的一个view

最后只能这样了，前面的几种尝试，都存在很多缺陷。只能用最原始的方式了。

首页在create的时候，判断是否要显示新手引导。如果要显示，就把新手引导的view生成，然后加到activity里面。

这样做也有一些不好的地方。比如：

首页上界面元素多，新手引导在滑动翻页的时候布局改变的时候，首页控件树中的其他view也会计算，流畅度不够好；

物流、旺旺等后台提示都仍旧会其作用；

在处理onKeyDown，onTrackballEvent等事件的时候，需要判断是否新手引导存在来做处理，增加了复杂度。

虽然最终的方案从功能上来说是最满足需求一个，但是还不完美。