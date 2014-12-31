---
layout: post
title: Android Widget之AsyncTask浅析 && R0.1.0
modified:
categories: 
excerpt: "AsyncTask简单分析"
tags: [android, widget, asyncTask]
comments: true
image:
  feature:
date: 2013-06-29T02:21:32+08:00
---
#####Base On Android 4.2.1

<figure>
	<a href="/images/2013/06/01_0.png"><img src="/images/2013/06/01.png"></a>
</figure>

####AsyncTask 

AsyncTask[^1][^2][^3]中有2个主要的Pool，其中一个是串行的，另一个是并行的。串行的由ArrayDeque维护，而并行的由BlockingQueue和一个ThreadFactory管理。

ArrayDeque通过一个数组作为载体，其中的数组元素在add等方法执行时不移动，发生变化的只是head和tail指针，而且指针是循环变化，数组容量不限制，当tail追上head（(this.tail = this.tail + 1 & this.elements.length - 1) == this.head）时，数组容量翻一倍。

如果BlockingQueue是空的，从BlockingQueue取东西的操作将会被阻断进入等待状态，直到BlockingQueue进了东西才会 被唤醒，同样，如果BlockingQueue是满的，任何试图往里存东西的操作也会被阻断进入等待状态，直到BlockingQueue里有空间时才会 被唤醒继续操作。

#####To be Continue…

原创文章，转载请注明： 转载自 <a href="http://archcodev.com">:-X archcodev</a>

###参考
[^1]: <http://blog.csdn.net/zlb824/article/details/7091814>
[^2]: <http://blog.csdn.net/microsoftq/article/details/5737514>
[^3]: <http://hubingforever.blog.163.com/blog/static/1710405792010964339151>