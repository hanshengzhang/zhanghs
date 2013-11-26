---
layout: post
title: C++与Java的多态性实现
---
#C++与Java的多态性实现
多态是面向对象语言一个非常重要的特征，在实际软件开发中也占有重要地位，然而它的实现方式很容易被忽略，甚至有时候一个很普通的重载，会被我们当成多态。
##C++的多态性
《The C++ Programming Language》这本书提到，C++的多态性只能通过**virtual**关键字实现。也就是说，如果不在父类中声明某个函数是virtual的，那么当父类的指针或引用指向子类的对象，再通过其调用被子类重写了的函数时，会调用父类的函数。下面有一个例子。
{% highlight c++ %}
#include <iostream>
using namespace std;
class A{
public:
    void print(){ //virtual void print()
		cout << "A" << endl;
    }
};
class B:public A{
public:
    void print(){
		cout << "B" << endl;
    }
};
int main(void){
	B  b;
	A & a = b;
	a.print();
	b.print();
	return 0;
}
{% endhighlight %}
上述例子的运行结果为
{% highlight c++ %}
A
B
{% endhighlight%}
而当我们在A中的print函数前添加了virtual关键字后，其运行结果则为下图所示
{% highlight c++ %}
B
B
{% endhighlight%}
&nbsp;
##Java的多态性
与C++不同的是，Java默认的就是建立类似于C++的虚函数表来查找函数，因此，Java的多态性是不需要添加任何关键字说明的。下面有个简单的例子。
{% highlight java %}
//A.java
public class A {
	public static void main(String[] args){
		B b = new B();
		A a = b;
		System.out.println(a.str());
		System.out.println(b.str());
	}
	public String str(){
		return "A";
	}
}
//B.java
public class B extends A{
	public String str(){
		return "B";
	}
}
{% endhighlight %}
以上程序的运行结果为
{% highlight java %}
B
B
{% endhighlight %}
&nbsp;