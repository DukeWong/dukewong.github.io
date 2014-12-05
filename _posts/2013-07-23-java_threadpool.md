---
layout: post
title: 线程池ThreadPoolExecutor分析&&R0.0.9
modified:
categories: 
excerpt: "线程池"
tags: [java, threadpool]
comments: true
image:
  feature:
date: 2013-04-22T20:02:12+08:00
---
#####Base on java 1.6

Pool的使用在开发中还是比较频繁的，他主要是限制池中的数量和提高复用的性能。下面以ThreadPoolExecutor为例子，看看到底内部的实现，先看代码
{% highlight java %}
Executors.newFixedThreadPool(xxxx,xxxx,....).execute(Runnable r);
{% endhighlight %}

其中Executors调用一个statics的方法newFixedThreadPool（），返回ExecutorService类型，然后执行execute方法，并传入一个runnable类型的变量。
<figure>
	<a href="/images/2013/07/02_0.jpg"><img src="/images/2013/07/02.jpg"></a>
</figure>

类图中，ThreadPoolExecutor中比较特殊的是BlockingQueue<Runnable>和HashSet<Worker>，queue里面存储了Runnable方法，而HashSet里面存储了Threads的集合。

下面介绍下ThreadPoolExecutor的几种状态与状态机的转换，图很容易看懂。
<figure>
	<a href="/images/2013/07/03_0.jpg"><img src="/images/2013/07/03.jpg"></a>
</figure>

流程图如下

<figure>
	<a href="/images/2013/07/04_0.jpg"><img src="/images/2013/07/04.jpg"></a>
</figure>

总结一下：ThreadPoolExecutor主要是传入一个runable的方法，然后内部构建一个Queue把runable方法存入，等到可执行的时候就通过ThreadFactory方法new一个Thread存入hashset，并且执行run方法。

关于LinkedBlockingQueue<E>，本人稍微看了下，虽然是Queue，可是内部却是一个Node<E>的链表，并且内部有两个ReentrantLock，这个拓展开来就太多了，下次再说吧。

另外提下volatile关键字，他使得多线程同步灵活，同时也不会阻塞方法，执行更加高效，可是灵活性太高，用起来难度比较大。

###拓展

ReentrantLock中Condition的用法示例


作为一个示例，假定有一个绑定的缓冲区，它支持 put 和 take 方法。如果试图在空的缓冲区上执行 take 操作，则在某一个项变得可用之前，线程将一直阻塞；如果试图在满的缓冲区上执行 put 操作，则在有空间变得可用之前，线程将一直阻塞。我们喜欢在单独的等待 set 中保存 put 线程和 take 线程，这样就可以在缓冲区中的项或空间变得可用时利用最佳规划，一次只通知一个线程。可以使用两个 Condition 实例来做到这一点。[^2] 
{% highlight java %}
   class BoundedBuffer {
   final Lock lock = new ReentrantLock();
   final Condition notFull  = lock.newCondition(); 
   final Condition notEmpty = lock.newCondition(); 

   final Object[] items = new Object[100];
   int putptr, takeptr, count;

   public void put(Object x) throws InterruptedException {
     lock.lock();
     try {
       while (count == items.length) 
         notFull.await();
       items[putptr] = x; 
       if (++putptr == items.length) putptr = 0;
       ++count;
       notEmpty.signal();
     } finally {
       lock.unlock();
     }
   }
{% endhighlight %}

###疑问
AbstractQueuedSynchronizer、volatile[^1]的用法

#####To be Continue…

###参考
[^1]: <http://www.ibm.com/developerworks/cn/java/j-jtp06197.html>
[^2]: <http://longdick.iteye.com/blog/453615>