---
layout: post
title: "Android中的SharedPreferences陷阱"
date: 2012-07-14 16:16
comments: true
categories: android
---
将保存SharedPreferences的xml文件删除了，能够彻底删除对应的SharedPreferences吗？

一次开发过程中，一个功能是需要将程序缓存清除掉，包括SharedPreferences文件。

第一次做的方式是，把相关的文件删除。但是发现有问题：程序退出后，再次进入程序仍然能够读取到应该被删除掉的SharedPreferences的值，但是DDMS查看要删除的pref文件确实都不在了。

为什么文件都不在了还能够取到值呢？

换了另一种清除SharedPreferences的方式：使用SharedPreferences.Editor的commit方法。结果证明，这样做是能够起作用的，调用后不退出程序都马上生效。

既然这样，第一个反应就是取SharedPreferences是首先内存缓存中取的。那为什么重启程序都还能去得到呢？

就进入源代码中看了一下，有几下几个地方可以了解了解：

取SharedPreferences实际上是在ContextImpl这个类中完成的。

* 1、context.getSharedPreferences(pref_name, mode)的流程：
	* A 在sSharedPrefs这个map(同步的)中以pref_name为键取SharedPreferencesImp对象sp。如果sp不为空并且对应的pref文件未被异常修改，就返回这个对象。否则进入B。
	* B 如果sp为空，重新生成一个SharedPreferenceImp对象并且加入到sSharedPrefs这个map中。
	* C 同步的：从pref文件中解析出map对象并用之替换SharedPreferenceImp对象中原有的存放pref键值对的mMap成员对象。如果pref文件解析异常导致map为null，就保持原有对象而不替换。 如果备份的pref文件(…pref_name.xml.bak)存在，就使用备份文件。
	* D 返回SharedPreferenceImp对象sp。
注意：sSharedPrefs在程序中是静态的：private static final HashMap sSharedPrefs = new HashMap(); 如果退出了程序但Context没有被清掉，那么下次进入程序仍然可能取到本应被删除掉的值。

* 2、从SharedPreference中取值getString(String key, String defValue)：
从SharedPreferencesImp对象的mMap成员对象中根据key取出相应的对象v。如果取得的对象v为空，返回默认对象defValue；否则，返回对象v。

* 3、commit过程：
	* A 在内存中提交，即用要提交的map去刷新已有的mMap对象。如果map对象中某个键的值指向editer对象自身，就代表要移除这个键值对。
	* B 将步骤A返回的MemoryCommitResult对象加入到写入本地的队列中，写入本地文件。这一步目前是在同一个线程中做的，因为性能表现得很良好。在写入文件前，如果同名文件已经存在，则会原文件重命名为备份文件名，如果写入成功，才删除bak备份文件。
	* C 通知SharedPreferences的监听状态改变了。返回提交是否内存成功的状态。

* 4、EditorImpl内部类：
内部有一个Map成员对象mModified，用来保存将要提交的pref键值。
apply方法与commit方法的区别：前者先提交到内存中，再异步写到文件，并且不需要返回写入成功与否的状态；后者同步写入内存和文件。

* 5、MemoryCommitResult内部类：
用来存放Editor提交到内存的返回状态，包括是否有键值改变、将要写入文件中的map对象，写入文件成功与否等。

总结一下：要想及时并安全清除SharedPreferences一定要使用Editor去clear并commit，不要直接暴力地删除其xml文件。

测试用的源代码(GBK编码)：[http://www.kuaipan.cn/file/id_80676665698551.htm](http://www.kuaipan.cn/file/id_80676665698551.htm)