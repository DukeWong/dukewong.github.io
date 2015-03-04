---
layout: post
title: Android Widget之SurfaceView浅析 && R0.0.7
modified:
categories: 
excerpt: "开始搞游戏了，看看surfaceView"
tags: [android, widget, surfaceView]
comments: true
image:
  feature:
date: 2013-10-21T19:18:02+08:00
---
#####Base On Android 4.2.1

近日开始研究SurfaceView，对于它为何能够在非UI线程更新动画而不报CalledFromWrongThreadException错（android.view.ViewRoot$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views）产生了兴趣，所以就深入看下为何如此。

国际惯例，看看他的结构图
<figure>
	<a href="/images/2013/10/01.png"><img src="/images/2013/10/01.png"></a>
</figure>

继承至View，而主线程初始至View?

首先看看SurfaceView创建的过程。Android4..x的代码多了个choreographer类，官方说法是当收到时序同步心跳的时候会主动刷新子系统帧到下一帧，而里面是通过一个FrameHandler（handler）来实现的，当收到MSG_DO_FRAME消息时，触发doFrame方法，最后通过层层调用，到了ViewRootImpl的doTraversal方法，此方法最终调用到performTraversals方法，这个方法比较复杂，稍后介绍。

{% highlight java %}
private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    } 
{% endhighlight %}

最后到了SurfaceView中的onPreDraw->updateWindow->surfaceCreated。至此，SurfaceView启动了后面的应该就很明了了，一般用户会在里面create一个loop方法，然后用canvas绘图。

到此为止，启动的过程已经有个了解了，可是似乎有种几乎什么都没有发现的感觉。

* 1、为何surfaceView可以不在主线程UI绘图？还是不明白，而且脑中有几个疑问，一直没有得到解决。
* 2、普通的widget（如TextView等）创建过程它他有何不同？
* 3、window窗口在刷新这两种widget时是否分开刷新？
* 4、他的帧率又是如何控制的？

下面先给出顺序图：

<figure>
	<a href="/images/2013/10/02_0.png"><img src="/images/2013/10/02.png"></a>
</figure>

对于图需要解释下，其中黑色部分是公共的部分，而各种颜色的部分是单独的部分，把他们画在一起是为了做比较。

####1、对于View
view在创建和更新的时候都是最终调用到canvas的drawXXX方法。需要说明的是performTraversals（）会被周期性的调用（具体可以参看scheduleTraversals()），然后view在创建的时候就会调用到view的draw方法。而invalidate（）方法会触发performTraversals（）。

####2、对于ViewGroup
ViewGroup其实最后都会调用到child view的Draw方法。

####3、对于SurfaceView
里面有个ViewTreeObserver接口，最后会回调到updateWindow方法。对于ViewTreeObserver，从字面意思上看是一个观察者，仔细看看是一个类，里面绑定了相关的interface（OnPreDrawListeners），然后实现onPreDraw（）的回调。这里还有个问题，observer直接调用的是一个group（本质上是ArrayList）里面的onPreDraw方法，具体代码如下
{% highlight java %}
....
        if (mOnPreDrawListeners != null && mOnPreDrawListeners.size() > 0) {
            final ArrayList listeners =
                    (ArrayList) mOnPreDrawListeners.clone();
            int numListeners = listeners.size();
            for (int i = 0; i < numListeners; ++i) {
                cancelDraw |= !(listeners.get(i).onPreDraw());
            }
        }
....
{% endhighlight %}

向上查看代码后，才知道这些listener是在 host.dispatchAttachedToWindow(attachInfo, 0)时调用到ViewGroup或者View的onAttachedToWindow的时候添加进去的，这个时候多态发挥了作用，具体代码大家自己去看吧。

下面来解答上面提出的问题。

* 1、必须要在主线程中刷新的原因是每次invalidate会调用checkThread，此方法会检查是否为当前主线程，后面接连调用scheduleTraversals，这个方法非线程安全，所以需要检查线程是否为主线程。（具体原因是什么日后还得继续研究）

* 2、上面的解释已解

* 3、刷新的时候并不分开刷新，而是有个层的概念，在刷新的时候会设置透明的区域，层层叠加到一起，形成了一组图，此操作都在performTraversals中[^1]，如图
<figure>
	<a href="/images/2013/10/03.jpg"><img src="/images/2013/10/03.jpg"></a>
</figure>

* 4、帧率控制有个FrameHandler，在Choreographer中。

###疑问
 为何scheduleTraversals不是线程安全？

#####To be Continue…

原创文章，转载请注明： 转载自 <a href="http://archcodev.com">:-X archcodev</a>

###参考
[^1]: <http://blog.csdn.net/luoshengyang/article/details/8661317>