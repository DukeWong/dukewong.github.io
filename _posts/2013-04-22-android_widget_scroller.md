---
layout: post
title: Android Widget之Scroller浅析 && R0.1.0
modified:
categories: 
excerpt: "关于Scroller那些事儿"
tags: [android, widget, scroller]
comments: true
image:
  feature:
date: 2013-04-22T16:48:42+08:00
---
#####Base On Android 4.2.1

最近做一个Android上的滑动效果，看到facebook上的一个浮动的功能，觉得很有意思，于是决定看看到底如何实现，知识点一拓展开，才发觉原来 自己懂的太少了。从开始研究到最后的ending，花了将近一周的时间，下面来总结下这一周的研究成果。[^1][^2][^3] 滑动效果涉及的widget有ViewFlipper，ViewPager，还有个Scroller，笔者猜测，ViewFilpper和 ViewPager最终都调用的是Scroller中的方法，这个还有待证实。 首先分析三个widget的继承结构
<figure>
	<a href="/images/2013/04/01.png"><img src="/images/2013/04/01.png"></a>
	<figcaption><a href="/images/2013-0.png" title="ViewFlipper">ViewFlipper</a>.</figcaption>
</figure>

<figure>
	<a href="/images/2013/04/02.png"><img src="/images/2013/04/02.png"></a>
	<figcaption><a href="/images/2013-0.png" title="ViewPager">ViewPager</a>.</figcaption>
</figure>

<figure>
	<a href="/images/2013/04/03.png"><img src="/images/2013/04/03.png"></a>
	<figcaption><a href="/images/2013-0.png" title="Scroller">Scroller</a>.</figcaption>
</figure>

其中，ViewFlipper和ViewPager都为ViewGroup的子类，可以判断所谓的滑动效果应该是一组View之间的Animation运 作，只有Scroller比较特殊开始一个单独拓展出来的组件，笔者查看代码后画出简单的时序图，从图中看的话应该会比较清楚。
<figure>
	<a href="/images/2013/04/04.png"><img src="/images/2013/04/04.png"></a>
</figure>
ViewFlipper继承至FrameLayout，是一组View的集合，当调用showNext（）后，最终会调用到View的startAnimation（）。

<figure>
	<a href="/images/2013/04/05.png"><img src="/images/2013/04/05.png"></a>
</figure>
ViewPager需要配合setAdapter（…）使用，同样也要传入一组View，最后通过View的scrollTo达到滑动的效果。

<figure>
	<a href="/images/2013/04/06.png"><img src="/images/2013/04/06.png"></a>
</figure>

其中scroller比较麻烦，他不属于主动调用的类型，查看代码后发现属于一种被动调用，通过回调实现的，在每次redraw一组view的时候，回掉用computeScroll（）函数，所以重载它是达到实现scroll的方法。其中还有一点就是每个view的坐标位置在View中有个mScrollerX和mScrollY来定义，而打log发现Scroller的startScroller函数与mScrollerX和mScrollY有关联，具体调用在此不做详细讨论。 综上所述，滑动为一组View的相互交替作用，而不同的view使用不同的layout布局，可以实现叠加，间隔，连续等效果。

###疑问
startAnimation与scrollTo是否有间接联系？

#####To be Continue…

原创文章，转载请注明： 转载自 <a href="http://archcodev.com">:-X archcodev</a>

###参考
[^1]: <http://blog.csdn.net/zhouyuanjing/article/details/8290454>
[^2]: <http://blog.csdn.net/qinjuning/article/details/7247126>
[^3]: <http://blog.csdn.net/gemmem/article/details/7321910>