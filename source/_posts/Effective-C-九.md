---
title: Effective C++(九)
date: 2018-01-15 16:53:31
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

`继承与面向对象设计`这个主题内容比较多，在进入下一个主题前需要一篇新的文章放下剩余的内容。

## 条款36:绝不重新定义继承而来的non-virtual函数

这个条款用个例子就能说明:
``` c++
class Base
{
public:
	void mf()
	{
		cout << "Base::mf()" << endl;
	}
};

class Derived : public Base
{
public:
	void mf()
	{
		cout << "Derived::mf()" << endl;
	}
};

void call(Base& base)
{
	cout << "in call():  ";
	base.mf();
}

int main(int argc, char* argv[])
{
	Derived d;

	Base* pb = &d;

	Derived* pd = &d;

	pb->mf();

	pd->mf();

	call(d);

	return 0;
}
```
编译运行，得到:
``` text
Base::mf()
Derived::mf()
in call():  Base::mf()
```
也就是说调用哪个版本的函数是由当前持有对象的`holder`(中文笔者不知道如何表达，代之指针或者普通变量)的类型决定。这个现象是可以通过`C++对象模型`来解释的，不过重点是这样的设计不符合`多态`的原则，容易造成误用。

其根源就是里面所有的表达式都是在编译期完成决议的，所有都是`静态绑定`，以下是静态绑定的定义:
``` text
静态绑定是指在程序编译过程中，把函数（方法或者过程）调用与响应调用所需的代码结合的过程称之为静态绑定。
```

总之如果一个函数是non-virtual的，该类的子类就不要重新定义该函数，避免后续的错误。

## 条款37:绝不重新定义继承而来的缺省参数值

在开始该条款之前，读者应该了解:
- 静态绑定，又名[前期绑定](https://en.wikipedia.org/wiki/Name_binding)
- 动态绑定，又名[后期绑定](https://en.wikipedia.org/wiki/Late_binding)

这个条款要介绍的矛盾是由于`缺省参数值`是`静态绑定`而`virtual函数`是`动态绑定`引起的。用书上的例子:
``` c++
enum ShapeColor{RED, GREEN, BLUE};	// 图形颜色

// 几何图形的基本类
class Shape
{
public:

	virtual void draw(ShapeColor color = ShapeColor::RED) const	// 所有的形状都有一个描绘的函数
	{
		cout << "Calling Shape::draw(ShapeColor=";

		switch(color)		// 加个判断，用于判定输入的参数的实际值
		{
			case ShapeColor::RED : cout << "RED)"  << endl;break;
			case ShapeColor::GREEN : cout << "GREEN)"  << endl;break;
			case ShapeColor::BLUE : cout << "BLUE)"  << endl;break;
		}
	}
};

class Rectangle : public Shape
{
public:
	virtual void draw(ShapeColor color = GREEN) const // 覆写了父类的函数
	{
		cout << "Calling Rectangle::draw(ShapeColor=";

		switch(color)		// 加个判断，用于判定输入的参数的实际值
		{
			case ShapeColor::RED : cout << "RED)"  << endl;break;
			case ShapeColor::GREEN : cout << "GREEN)"  << endl;break;
			case ShapeColor::BLUE : cout << "BLUE)"  << endl;break;
		}
	}
};

class Circle : public Shape
{
public:
	virtual void draw(ShapeColor color) const		// 这个子类强制用户输入颜色
	{
		cout << "Calling Circle::draw(ShapeColor=";

		switch(color)		// 加个判断，用于判定输入的参数的实际值
		{
			case ShapeColor::RED : cout << "RED)"  << endl;break;
			case ShapeColor::GREEN : cout << "GREEN)"  << endl;break;
			case ShapeColor::BLUE : cout << "BLUE)"  << endl;break;
		}
	}
};

int main(int argc, char* argv[])
{
	// 三个变量都是Shape*静态类型
	Shape* ps;

	Shape* pc = new Circle;			// 指向实际类型为Circle

	Shape* pr = new Rectangle;		// 指向实际类型为Rectangle

	pc->draw();

	pr->draw();

	ps = pc;						// ps的动态类型是Circle

	ps->draw();

	ps = pr;						// ps的动态类型是Rectangle

	ps->draw();

	Circle* cpc = new Circle;		// 对照组，用于对比静态类型为子类的情况

    cpc->draw(ShapeColor::BLUE);

    Rectangle* rpr = new Rectangle;

    rpr->draw();

	return 0;
}
```
运行得到:
``` text
Calling Circle::draw(ShapeColor=RED)
Calling Rectangle::draw(ShapeColor=RED)
Calling Circle::draw(ShapeColor=RED)
Calling Rectangle::draw(ShapeColor=RED)
Calling Circle::draw(ShapeColor=BLUE)
Calling Rectangle::draw(ShapeColor=GREEN)
```
这个现象中，所有的`多态`表现正确。`ps`、`pc`、`pr`三个指针的静态类型都是`Shape*`，但是分别指向了`Circle`和`Rectangle`实例，上面的输出显示都根据其动态类型成功地调用了实际类型中的`draw`函数。但是一方面又很奇怪，缺省参数值是根据`静态类型`来决议的，在静态类型是`Shape*`的情况下，所有的缺省参数值都采用了`Shape::draw`的版本，不受实际类型的影响。甚至连`Circle::draw`本身不具有`缺省参数`也采用了父类的版本。

笔者原本以为会引起编译错误，但是仔细想了想也是合理的，首先静态编译期就根据变量类型决议了`缺省参数`的版本，并不影响其运行时。

书上也给出了解决这个问题的技巧，这样的做法并不提倡，因而笔者就不给出方法了。遇到这样的问题更应该从设计上来解决，避免总是用技巧，总体原则就是不要在`virtual函数`里面施加太多的约束，留有灵活性使得子类继承时方便修改。

## 条款38:通过复合塑模出has-a或者"根据某物实现出"

该条款并不是编程方面的问题，而是有关`概念`以及`设计`的问题，例如书上说的"人有一个住址":
``` c++
class Person
{
public:
	...
private:
	Address address;
};
```
而不是"人是一个住址":
``` c++
class Person : public Address
{...};
```
这个是编程人员对所需要实现的软件的认知引起的问题，当然这个例子是很简单的，书上既然给出了这个条款，那么也是提示读者要注重概念上的问题，否则因为这种与编程无关的事情增加了复杂度那就太浪费人力了。正如上面这个"人是一个住址"错误的示例，读者可以想想这样的设计，在把一个`Person`实例`持久化`到数据库的时候，`Address表`和`Person表`的关系，以及`sql语句`的设计。

然后就是`is-a`和`is-implemented-in-terms-of`的区分问题。在编程里面，`集合`和`列表`是两种不同概念的数据结构，书上给的例子是复用`列表`实现`集合`:
``` c++
template<typename T>
class Set : public std::list<T>
{...};
```
这样基本上可不用写多少代码就实现了功能，但是`集合`是不允许集合的重复的而`列表`允许，也就是说产生了概念冲突。

但是如果`集合`仅仅是借`列表`来实现的话:
``` c++
template<typename T>
class Set
{
public:
	...
	void insert(const T& item);
	...
private:
	std::list<T> container;
};

...
template<typename T>
void Set<T>::insert(const T& item)
{
	...		// 加入集合前判断元素是否唯一等概念上的操作
	containter.insert(item);	// 实际插入操作
	...		// 其他处理等
}
...
```
这样就解决了`Set`不是`list`这个概念上的问题，因为这个`Set`是"根据list实现出"来的。

这个条款笔者认为没有太多技术上的东西，更多的是编程人员的基础概念认知的知识问题。

## 条款39:明智而审慎地使用private继承

该条款提到了:
- [空白基类最优化,empty base optimization,EBO](http://blog.csdn.net/buxizhizhou530/article/details/45888203)

笔者没有该条款的实践，毕竟`private继承`在实际应用中太少了，少到几乎没有见到过。

`private继承`意味`implemented-in-terms-of`关系。

例如书上的例子:
``` c++
class Timer
{
public:
	explicit Timer(int tickFrequency);		// 设定计时器频率
	virtual void onTick() const;			// 周期性事件
	...
};
```
然后一个`Widget`需要定时器功能，但是显然`Widget`不是一个`Timer`，需要表现出`is-implemented-in-terms-of`关系才能符合常用逻辑，也就是`Widget`的`定时事件`功能是`根据Time实现`的:
``` c++
class Widget : private Timer
{
private:
	virtual void onTick() const;			// 周期性事件
	...
};
```
这样看起来乖乖的，明明是一种`继承`语法，却是表现不出`is-a`关系设计的实现。这就是为何private继承罕见的原因了。也可以改进为一种更加符合阅读理解的表现形式:
``` c++
class Widget
{
public:
	...
private:
	class WidgetTimer : public Timer 		// 为Widget专门定制的Timer
	{										// 内部类，无法从外部创建实例
	public:
		virtual void onTick const;
		...
	};

	WidgetTimer timer;						// Widget的定时器功能就根据这个定制的Timer实现
	...
};
```
第二种做法还有用到的地方，起码这种实现方式容易让人理解。介绍这个条款并不是推广这种private继承的技巧，而是别无他法的时候才采纳这个实现方案。

## 条款40:明智而审慎地使用多重继承

多重继承(Multiple Inheritance，MI)是`C++`其中一把很著名的双刃剑，一方面给`C++`带来了丰富灵活的继承语法，另一方面又给`C++`的继承带来混乱。其中为了解决这个混乱，`Java`只允许`单一继承`，解决了不少问题，而且引入了`接口(interface)`关键字弥补灵活性的缺失。

首先:
``` c++
class Base1
{
public:
	void f();
	...
};

class Base2
{
public:
	void f();
	...
};

class Derived : public Base1, Base2
{
...
};
```
这样的话，当发生:
``` c++
Derived d;

d.f();
```
就引起了歧义，引起编译错误。为了消除歧义，必须这样指定:
``` c++
d.Base1::f();
```
明确调用哪个版本。

接下来就是`菱形继承问题`:
![](Effective-C-九/inher0.jpg)
写段代码测试:
``` c++
class A
{
public:
	A()
	{
		cout << "A::A(" << this << ")" << endl;
	}
};

class B : public A
{
public:
	B()
	{
		cout << "B::B(" << this << ")" << endl;
	}
};

class C : public A
{
public:
	C()
	{
		cout << "C::C(" << this << ")" << endl;
	}
};

class D : public B, public C
{
public:
	D()
	{
		cout << "D::D(" << this << ")" << endl;
	}
};
```
实例化一个`D`对象时，程序输出:
``` text
A::A(0x7ffecb763bce)
B::B(0x7ffecb763bce)
A::A(0x7ffecb763bcf)
C::C(0x7ffecb763bcf)
D::D(0x7ffecb763bce)
```
也就是说一个`D`实例中的有两份`A`实例，一份是属于`B`的`A`以及一份属于`C`的`A`。

为了解决这样不统一的问题，可以用`virtual继承`:
``` c++
class A
{...};

class B : virtual public A
{...};

class C : virtual public A
{...};

class D : public B, public C
{...};
```
这样实例化一个`D`对象时，程序输出:
``` text
A::A(0x7fffdd669580)
B::B(0x7fffdd669580)
C::C(0x7fffdd669588)
D::D(0x7fffdd669580)
```
这样就能够处理好`公共爷类`的问题了。考虑到继承树下面可能会有`D`的子类也出现这样的情况，所有的继承应用`virtual继承`防止，然而这样会使得实例占用的内存膨胀，也就是说`virtual继承`最好不要滥用。

作者在书上给出的建议，笔者认为就是`Java`的`单一继承`和`多实现`。也就是说约束继承语法使得程序只有`单一继承`原则，而灵活性由`接口`的多继承来补充，关于`接口`笔者在条款34中介绍过。

具体的做法，读者们可以去写写`Java`的`继承`和`接口`，就能总结出在`C++`中怎么写了。