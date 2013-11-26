---
layout: post
title: C++与Java的多态性实现
---
#C++与Java的多态性实现
多态是面向对象语言一个非常重要的特征，在实际软件开发中也占有重要地位，然而它的实现方式很容易被忽略，甚至有时候一个很普通的重载，会被我们当成多态。
##C++的多态性
在《The C++ Programming Language》这本书提到，C++的多态性只能通过关键字实现
{% highlight java %}
public class HelloWorld {
    public static void main(String args[]) {
		System.out.println("Hello World!");
    }
}
{% endhighlight %}
{% highlight c++ %}
class A{
public:
int a;
}
{% endhighlight %}
