---
layout: post
title: Java中的Serializable和Android中的Parcelable && R0.0.6
modified:
categories: 
excerpt: "序列化"
tags: [java, android, serializable, parcelable]
comments: true
image:
  feature:
date: 2013-12-11T20:02:12+08:00
---
#####Base on java 1.6

关于Serializable，引用别人的博客，个人觉得言简意赅[^1][^2]

####Object serialization的定义：
Object serialization 允许你将实现了Serializable接口的对象转换为字节序列，这些字节序列可以被完全存储以备以后重新生成原来的对象。  
serialization不但可以在本机做，而且可以经由网络操作（就是RMI）。这个好处是很大的——因为它自动屏蔽了操作系统 的差异，字节顺序（用Unix下的c开发过网络编程的人应该知道这个概念，我就容易在这上面犯错）等。比如，在Window平台生成一个对象并序列化之， 然后通过网络传到一台Unix机器上，然后可以在这台Unix机器上正确地重构这个对象。 

####Object serialization主要用来支持2种主要的特性： 
Java的RMI(remote method invocation).RMI允许象在本机上一样操作远程机器上的对象。当发送消息给远程对象时，就需要用到serializaiton机制来发送参数和接收返回值。 
Java的JavaBeans。Bean的状态信息通常是在设计时配置的。Bean的状态信息必须被存起来，以便当程序运行时能恢复这些状态信息。这也需要serializaiton机制。 
关于Parcel，这里可以拓展说一下。parcel里面很多方法都是native方法。


Pool的使用在开发中还是比较频繁的，他主要是限制池中的数量和提高复用的性能。下面以ThreadPoolExecutor为例子，看看到底内部的实现，先看代码
{% highlight java %}
 /**
     * Read an integer value from the parcel at the current dataPosition().
     */
    public final native int readInt();

    /**
     * Read a long integer value from the parcel at the current dataPosition().
     */
    public final native long readLong();

    /**
     * Read a floating point value from the parcel at the current
     * dataPosition().
     */
    public final native float readFloat();

    /**
     * Read a double precision floating point value from the parcel at the
     * current dataPosition().
     */
    public final native double readDouble();

    /**
     * Read a string value from the parcel at the current dataPosition().
     */
    public final native String readString();
{% endhighlight %}

并且里面有2个Pool，用来减缓频繁的序列化操作，如图所示。
<figure>
	<a href="/images/2013/12/01.png"><img src="/images/2013/12/01.png"></a>
</figure>
在使用parcelable接口的时候，会有个 Parcelable.Creator变量需要实现，里面有相应的override方法，其实这个是一种类似于callback的方法，只是这次使用的是反射来实现的。
{% highlight java %}
...
...
 try {
       Class c = loader == null ?
       Class.forName(name) : Class.forName(name, true, loader);
       Field f = c.getField("CREATOR");
       creator = (Parcelable.Creator)f.get(null);
       }
...
...
{% endhighlight %}

通过反射，获取静态变量的句柄，然后做相应操作。

Java层的代码不多，现在总结下他们的区别：

* Serializable的作用是为了保存对象的属性到本地文件、数据库、网络流、rmi以方便数据传输，当然这种传输可以是程序内的也可以是两个程序间的。
* Android的Parcelable的设计初衷是因为Serializable效率过慢，为了在程序内不同组件间以及不同Android程序间(AIDL)高效的传输数据而设计，这些数据仅在内存中存在，Parcelable是通过IBinder通信的消息的载体。

##疑问
C和C++层的还有待跟进

#####To be Continue…

###参考
[^1]: <http://www.verydemo.com/demo_c89_i37341.html>
[^2]: <http://www.chinaunix.net/old_jh/26/395684.htm>