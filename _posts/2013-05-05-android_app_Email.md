---
layout: post
title: Android App之Email浅析 && R0.0.7
modified:
categories: 
excerpt: "工作那会儿的Email App"
tags: [android, app, Email]
comments: true
image:
  feature:
date: 2013-05-05T12:23:02+08:00
---
##### Base On Android 4.2.1

 刚入职那会儿就干着email的移植啊，bug啊之类的，索性写下点什么，以后翻起来也方便点，以下不保证正确性，仅供 娱乐参考。 Email的概念层次图，可以对Email有个宏观的了解（pop端口一般是995，而smtp一般是465）[^1]。
<figure>
	<a href="/images/2013/05/01.png"><img src="/images/2013/05/01.png"></a>
</figure>

代码部分，定位到java层次 packages/apps/Email/src/com/android/email
<figure>
	<a href="/images/2013/05/02.png"><img src="/images/2013/05/02.png"></a>
</figure>

各个部分的代码作用区域大概就是这些了，有些时候其实看UML图比我说千百遍都来的很好理解 下面看下Email使用的SMTP协议（一个建驻在TCP协议上的协议），是通过发送指令下达消息的，具体可以参考文章后面的链接，上文提到的 transport文件夹是用来处理通信的，找到smtpSender.java文件，里面有一段代码用来发送email data以及cc bc等，而其他的java只是做了些用户交互的东西。

{% highlight java %}
...
try {
    executeSimpleCommand("MAIL FROM: " + "<" + from.getAddress() + ">");
    for (Address address : to) {
        executeSimpleCommand("RCPT TO: " + "<" + address.getAddress() + ">");
    }
    for (Address address : cc) {
        executeSimpleCommand("RCPT TO: " + "<" + address.getAddress() + ">");
    }
    for (Address address : bcc) {
        executeSimpleCommand("RCPT TO: " + "<" + address.getAddress() + ">");
    }
    executeSimpleCommand("DATA");
...
{% endhighlight %}

接下来从总体架构看下email的消息走向先从setupxxx.java配置账户，在此注意xml的解偶作用，随后Controller把消息传送到 messagingController，注意这是一个死循环的Thread，内置一个Queue消息队列，接下来就是一些异步Task和监听的 Listener 回掉，总感觉还是看图方便，希望这个图能让大家看懂。

<figure>
	<a href="/images/2013/05/03_0.png"><img src="/images/2013/05/03.png"></a>
</figure>

其中synchronizeMailboxGeneric中操作比较多，下面列一下，英文的，看不懂翻翻字典咯。 8-O

* Get the message list from the local store and create an index of the uids
* Open the remote folder and create the remote folder if necessary
* Open the remote folder. This pre-loads certain metadata like message count
* Trash any remote messages that are marked as trashed locally
* Get the remote message count
* Determine the limit # of messages to download
* Create a list of messages to download
* Download basic info about the new/unloaded messages (if any)
* Refresh the flags for any messages in the local store that we didn’t just download
* Remove any messages that are in the local store but no longer on the remote store
* Clean up and report results

 在多个账户同时接收邮件的过程中常常遇到多个线程的交互，这样就必须用到锁机制，随便搜索了下，大多数是些读写操作，这也很符合PV操作的概念，凑合看下吧，可能以后会做详细解释。

接下来看看数据库的db文件，从里面挑选了几个比较常用的Table，account存储账户信息，mailbox对应邮件文件夹，message对应的是邮件正文[^2]。

{% highlight java %}
BEGIN TRANSACTION;

insert into  ("_id", "displayName", "emailAddress", "syncKey", "syncLookback", "syncInterval", "hostAuthKeyRecv", "hostAuthKeySend", "flags", "isDefault", "compatibilityUuid", "senderName", "ringtoneUri", "protocolVersion", "newMessageCount", "securityFlags", "securitySyncKey", "signature", "policyKey", "notifiedMessageId", "notifiedMessageCount") 
    		values ('3', 'archcodevxxx@163.com', 'archcodevxxx@163.com', NULL, '-1', '15', '5', '6', '2057', '0', 'cd4354ba-03c2-4089-8f42-61abec9c1744', 'archcodev', 'content://settings/system/notification_sound', NULL, '0', NULL, NULL, NULL, '0', '0', '0');

insert into  ("_id", "displayName", "serverId", "parentServerId", "parentKey", "accountKey", "type", "delimiter", "syncKey", "syncLookback", "syncInterval", "syncTime", "unreadCount", "flagVisible", "flags", "visibleLimit", "syncStatus", "messageCount", "lastSeenMessageKey", "lastTouchedTime") 
			values ('17', '已发送', '已发送', NULL, '-1', '3', '5', '47', NULL, '0', '0', '0', '0', '1', '24', '25', NULL, '0', '0', '1368427825017');

insert into  ("_id", "syncServerId", "syncServerTimeStamp", "displayName", "timeStamp", "subject", "flagRead", "flagLoaded", "flagFavorite", "flagAttachment", "flags", "clientId", "messageId", "mailboxKey", "accountKey", "fromList", "toList", "ccList", "bccList", "replyToList", "meetingInfo", "snippet", "protocolSearchInfo", "size") 
			values ('78', '1319384458', '1357011679000', 'Postmaster@163.com', '1357011678000', '系统退信', '1', '2', '0', '0', '0', NULL, '<50E25ADE.7D4E04.17696@163smtp12>', '25', '3', 'Postmaster@163.com', 'archcodevxxx@163.com', '', '', '', NULL, '抱歉，您的邮件被退回来了…… 原邮件信息： 时 间： 2012-12-31 11:27:34 主 题： 回复: 验证金山快盘邮箱，免费获取1G空间 收件人： kuaipan@wps.cn 抄 送： xxx 密 送： yyy 退信原因： 您可能写了外星人的邮件地址；或者是网络不给力，我们无法与收信方联络上。 英文说明:Can not connect to wps.cn:114.112.66.62:2', NULL, '13427');

COMMIT;
{% endhighlight %}

关于数据流图下面简单画了一下

<figure>
	<a href="/images/2013/05/04.png"><img src="/images/2013/05/04.png"></a>
</figure>

Android 3.0为了适配平板加入了Fragement的概念，下面以MessageListFragement为例子，看下消息是如何从view传到 database的？本人比较关注结构方面，有点小建筑控 :lol:，先看下这个类的结构，其中implements4个package外的接口，然后内部定义了2个接口

<figure>
	<a href="/images/2013/05/05.png"><img src="/images/2013/05/05_0.png"></a>
</figure>

在此，关注一个接口回掉的方法，下面是类图，简单解释下，当点击onclick（）的时候调用回掉接口中定义的内容，其中 getTargetFragment（）的用法比较有趣，和setTargetFragment（）搭配使用。关于callback的使用最早接触是在C 中了解到了，他定义了一个接口模板，然后通过内嵌的方式突破权限的限制，使得Class A可以调用Class B的方法，同时能拆开Class AB的耦合，一举两得，不能在google工作也就只能拜读他们的代码咯。

### 疑问
在有网络时，是否也每次都是从数据库中取数据，换句话说就是先得从网络把数据存储到sqlite，然后再从db中取出显示？

##### To be Continue…

原创文章，转载请注明： 转载自 <a href="http://archcodev.com">:-X archcodev</a>

### 参考
[^1]: <http://www.cnblogs.com/CrazyWill/archive/2006/07/03/441795.html>
[^2]: <http://blog.sina.com.cn/s/blog_5d6ee3360100r1my.html>