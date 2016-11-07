---
layout: post
category: tech
title: 一个Java内存泄漏检测的例子
---
这几天遇到一个问题：一个Java的线上Service经常抛出超时警告。
经过简单的测试，发现是内存快满了，这是JVM经常会出现的一种情况。
JDK带了很多小工具，以帮助软件开发人员了解、分析和解决JVM中可能存在的问题。
其中的大部分，Oracle都声称是实验性质的，但绝对好用。
本文主要是通过这个例子，用于介绍如何通过jstat，jcmd，jmap和jhat这三个JDK自带的工具来发现和监测JVM中的内存问题。

# 发现问题：使用jstat和jcmd了解内存和GC的统计数据

现在夜深人静，寒风正肆无忌惮的刮着。
扫了一眼窗前那皎白的月光，你紧了紧窗帘，打算在被窝里好好睡一觉。
可这个时候，线上的Service抛出超时了，今天又刚刚好轮到你值班。
呃扯远了，经过多次的测试，这个Service的超时情况总是时好时坏的，又是运行在JVM上，让人很快就想到了一种可能：GC Pause。
[jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)是JDK自带的一个工具，使用它可以将JVM的一些统计信息打印出来。
为了查看内存和GC的统计信息，可以使用如下命令：

{% highlight bash %}
jstat  -gcutil 14887 1s
{% endhighlight %}

这个命令中，14887是JVM对应的process id， 1s表示的是一秒钟刷新一次。
然后你就可以看到类似以下的输出：

|---
|S0|S1|E|O|P|YGC|YGCT|FGC|FGCT|GCT|
|---
|0.00|99.71|44.09|99.62|99.14|1587|35.618|65|44.387|80.004|
|0.00|99.71|44.09|99.62|99.14|1587|35.618|65|44.387|80.004|
|0.00|99.71|44.10|99.62|99.14|1587|35.618|65|44.387|80.004|
|0.00|99.71|44.10|99.62|99.14|1587|35.618|65|44.387|80.004|
|0.00|99.71|44.10|99.62|99.14|1587|35.618|65|44.387|80.004|
|0.00|99.71|44.10|99.62|99.14|1587|35.618|65|44.387|80.004|
|0.00|99.71|44.11|99.62|99.14|1587|35.618|65|44.387|80.004|

当然，输出是在console中，不是这么优雅的表格。
表格中O表示的是老年代的内存使用率，FGC表示的是Full GC的次数（假设你比较了解JVM的GC）。
从以上的表格可以看出，老年代的内存使用率的确非常高，而我们的Service在正常情况下根本吃不了这么多内存。
为了进一步说明是Service产生了内存泄漏，我们可以通过jcmd命令来trigger一次GC。

{% highlight bash %}
jcmd 14887 GC.run
{% endhighlight %}
[jcmd](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr006.html)也是JDK中的一个工具，用来发送诊断命令给JVM。
Trigger了一次GC以后，jstat的输出已经多了一次Full GC， 但内存使用率仍然很高，这就基本确定了是由于内存泄漏引起的。

# 备份问题：使用jmap保存JVM内存快照

既然是由于内存泄漏引起的，可以重新部署一下，让Service恢复。
但是为了方便以后继续分析问题，在重新部署Service之前，最好还是先备份好足够多的数据。
[jmap](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jmap.html)是JDK中一个用于分析JVM内存的工具。
我比较喜欢用jmap将每个类所对应实例的内存试用情况打印出来，使用如下命令：

{% highlight bash %}
jmap -histo 14887 | tee temp.txt
{% endhighlight %}

jcmd也有一样的功能，得到的结果如下表所示：

|---
|num|#instances|#bytes|classname|
|---
|1:|47087257|1171123216|[B|
|2:|23434640|563136016|[[B|
|3:|23396433|561514392|com.mysql.jdbc.ByteArrayRow|
|4:|162774|115668640|[Ljava.lang.Object;|
|5:|882259|96180560|[C|
|6:|181578|31957728|com.mysql.jdbc.JDBC4ResultSet|
|7:|341810|28754880|[Ljava.util.HashMap$Entry;|
|8:|188127|27090288|com.mysql.jdbc.Field|
|9:|160233|25637280|com.mysql.jdbc.StatementImpl|

同时，我们还需要将整个JVM的内存dump下来,然后就可以安心重启Service睡觉去了。使用jmap的如下命令:
{% highlight bash %}
jmap -dump:format=b,file=dump.hprof 14887
{% endhighlight %}

# 追踪问题：使用jhat追踪存储引用

从以上的表格中，我们可以发现有几个mysql jdbc中的类有大量的实例。
因此，基本上可以确定这个内存泄漏是由于我们的Service在create一个Statement后忘记close导致的。

当然，为了进一步诊断，我们还是用jhat将dump查看一下dump出来的文件。

{% highlight bash %}
jhat dump.hprof
{% endhighlight %}

[jhat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jhat.html)可以用来parse dump出来的文件并启动一个web服务器让用户可以在浏览器中浏览内存的使用情况。
这个工具的槽点实在是太多了，首先，它非常吃内存，因此以上命令可能需要在一台内存比较大的机器上执行。
其次，让大家看看这个工具的输出：

{% highlight bash %}
Reading from dump.hprof...
Dump file created Mon Oct 31
Snapshot read, resolving...
Resolving 105471306 objects...
Chasing references, expect 21094 dots....
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
{% endhighlight %}

请注意第五行，它后面实际上是很多个点的，它告诉你等到21094个点后，就处理完了，但是这个要怎么数啊，实在弄不明白为啥它不直接输出一下进度。
最后，好不容易出来好了，打开浏览器后任何一个网页都有可能因为内容太多而卡死。
当然，你也可以使用OQL语句告诉Server要查什么，但是很可能出现的情况就是发了一个查询之后等很久都出不来结果。
即便在如此艰难的情况下，我还是追踪到了这么多mysql jdbc相关的对象的源头都是JDBC4Connection的一个对象。
这个对象有一个field叫openStatements，然后statement下面又有result set，然后又是byte array。。。

# 解决问题：三个关闭

首先，对于本文所描述的场景，很多有经验的程序员可能都不会犯这个错误；
其次，虽然我不够了解，但是应该有很多其他的开源和商业的工具用以帮助分析JVM内存使用情况。
本文只是为了介绍一下内存泄漏监测的一个可能的流程，使用的是JDK自带的工具，用起来也还都算方便。

至于解决这种问题：

1. 关闭Statement，是的，你的代码应该保证这一点。
2. 或者，关闭Collection，自己定期回收JDBC的Collection或者使用Collection Pool，推荐使用。
3. 或者，关闭Service，定期重启Service，比较彻底和完美。
