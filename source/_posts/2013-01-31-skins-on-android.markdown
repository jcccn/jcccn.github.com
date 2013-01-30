---
layout: post
title: "Android程序如何实现换肤？"
date: 2012-07-16 11:51
comments: true
categories: android
---
安卓上换肤可以通过那些方式来实现呢？

源码下载地址：[http://www.kuaipan.com.cn/file/id_80676665698552.htm](http://www.kuaipan.com.cn/file/id_80676665698552.htm)
 
#### 方式一：切换程序的语言版本。

* 原理：系统根据一定的规则去资源文件夹res下寻找资源。通过改变程序的配置（比如语言或者地区），就能加载不同资源文件夹下面的资源，从而实现简单的换肤。

* 实践：举例说明，我要通过中文(系统默认)和英文两个语言版本来实现换肤。首先增加两个资源文件夹drawable-en-mdpi和layout-en，如下图。当在默认的中文环境下，使用的是drawable-mdpi和layout下面的资源。切换至英文环境之后，程序则会使用drawable-en-mdpi和layout-en下面的资源，在这两个文件夹下找不到的时候才去使用默认的drawable-mdpi和layout下面的资源。
![http://jcccn-wordpress.stor.sinaapp.com/uploads/2012/06/skin-test-1.png](http://jcccn-wordpress.stor.sinaapp.com/uploads/2012/06/skin-test-1.png)       

要实现语言环境的切换，同时不改变系统的语言环境，我们就要改变当前应用的语言，使用Resources实例的updateConfiguration方法即可。示例代码如下：

`changeLanguage(Locale.ENGLISH);
private void changeLanguage(Locale newLocale) {
Resources resources = getResources();
Configuration config = resources.getConfiguration();
DisplayMetrics dm = resources.getDisplayMetrics();
config.locale = newLocale;
resources.updateConfiguration(config, dm); 
this.onCreate(null); // 用于立即刷新界面
}`

但是这种方式有一些局限性：只能更换本apk内置的资源且受限于语言种数；需要重新create activity等方式才能刷新界面；可能影响程序其他依赖于语言的元素。这种方式属于一种投机取巧的方法，不建议使用。

总结：两步实现，第一步，增加不同语言版本的资源文件夹；第二步，程序内部切换语言。

#### 方式二：安装主题apk。

* 原理：通过获取其他程序的context来获取皮肤资源。我们知道android程序中要获取drawable、layout等资源，都要通过context.getResources().getXXX的方式。关键就在这儿了，如果我们可以拿到其他程序的context，那么那个程序就可以作为皮肤程序来提供资源给主程序使用了。android中两个程序相互读取数据的条件是：两个程序的共享用户id相同，通过AndroidManifest.xml中的android:sharedUserId属性配置；两个程序签名相同。想要改变皮肤时，改变提供资源的context为皮肤程序的context，然后刷新即可。

* 实践：有一点要注意，要保证能正确获取到皮肤包中的资源，需要编译出来的皮肤包与主程序中的R.java文件一致，即资源对应要一致（主程序中有layout、color、drawable、value等多少类资源，皮肤包中也需要有相同数量的资源）。

`Context skinContext = createPackageContext(skinPackageName, Context.CONTEXT_IGNORE_SECURITY);    contentView.setBackgroundDrawable(skinContext.getResources().getDrawable(R.drawable.gloal_background));`


#### 方式三：使用皮肤资源zip包。

* 原理：直接从文件（SD卡或者data目录）中读取资源文件并解码，然后设置给相关的控件。

* 实践：实现方式可以是直接一个包含皮肤资源（图片、控制布局的一些数值文本文件等）的压缩包，通常后缀名被命名为自己独有的名字，比如搜狗的sga，百度的bds等，使用时被解压拷贝到手机存储上的皮肤文件夹里面；也可以是包含在一个单独的apk安装包里面，安装应用皮肤后皮肤压缩包（一般放在皮肤apk的asset目录下）被解压拷贝到data目录下以供使用。

这种方式需要注意的一个地方就是内存管理的问题。一般是new一些bitmap或者BitmapDrawable之类的对象出来，在不再使用的时候要注意释放。一个思路是参照系统Resources类的管理方式，详细实现方式后面再研究研究。