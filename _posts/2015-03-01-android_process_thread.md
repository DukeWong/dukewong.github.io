---
layout: post
title: "Android应用程序中进程与线程的关系 && M0.0.8"
modified:
categories: 
excerpt: "全面整体看process和thread"
tags: [android, process, zygote, thread]
image:
  feature:
date: 2015-03-01T00:021:20+08:00
---
#####Base On Android 4.0

去年8月份开始研究android 的C++层面，为了看懂avm，还去翻了hotspot相关的知识，随后再是dalvik，再到后面的计算机系统概念，随后解决刚入行时对android的进程线程的疑问。

下面画了张图，手绘的，别介意。

<figure>
	<a href="/images/2015/03/01.png"><img src="/images/2015/03/01.png"></a>
</figure>


* 蓝色的代表进程。

* 当一个app启动的时候，是由ActivityManagerService通知zygoteInit进程，然后fork一个zygote进程，启动activity的。[^1] [^2]

* java层的New Thread操作最终会进过jni转化成linux pthread_create操作，底层的创建线程也是通过pthread的。

* linux没有线程的概念，线程算作精简的进程。

* dvm限制单个app的heap空间。

* activity和thread、process不是一个范畴的东西，他是android大牛发明的概念。

* Activity task是一类操作的集合，和thread也没关系，相类比的概念有activityStack，activityRecord等等。[^3]

* looper是anroid大牛在java层想到的线程间通信的方法，和linux层面的不同，linux层面是通过static或者传参共享的。[^4]


总之，归纳这些概念，整整花了2天，一两句也说不清楚，要理解这些还真得自己的参透。

###疑问
service与activity不同在哪里？又和thread如何区分？

#####To be Continue…

原创文章，转载请注明： 转载自 <a href="http://archcodev.com">:-X archcodev</a>

###参考
[^1]: <http://blog.csdn.net/luoshengyang/article/details/6689748>
[^2]: <http://blog.csdn.net/luoshengyang/article/details/8923484>
[^3]: <http://blog.csdn.net/qinjuning/article/details/7262769>
[^4]: <http://blog.csdn.net/luoshengyang/article/details/6817933>