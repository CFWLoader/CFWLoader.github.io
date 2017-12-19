---
title: Effective C++(六)
date: 2017-12-18 17:12:47
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

从这里开始就进入了`设计与声明`主题，这个主题是对入门者，尤其是在校的学生来说是很有用的编程建议。一来是学生项目一般小型，不需要大规模的组织;第二是没有生命周期，完成之后交付即可，所以实战项目让学生非常痛苦的事就是阅读代码，尤其是理顺逻辑。

在经历过实战项目历练之后才会知道组织代码的重要性，包括变量函数命名、写文档(简易文档也有参考性)等等，这样在日后项目有改动的时候不至于不知从何入手。笔者接触的实战项目尚少，也只能写点这样的感慨了。

## 条款18:让接口容易被正确使用，不易被误用

笔者阅读过该条款，认为该问题的主要来源是`C++`的`隐式类型转换`、`缺省参数`、`自动推导`等特性综合起来导致的。扩展开来讲的话，应该是`C++`对`类型`的管理宽松所导致的。虽然`C++`从语法上看是`强类型语言`，但是通过设置编译参数就可以跳过`类型转换`等操作，从`内存`层面来看的话更像是`弱类型语言`，如下例子:
``` c++
class A
{
public:
        int mem1;
};

class B
{
public:
        int mem1;
        float mem2;
};

int main(int argc, char* argv[])
{
        A* objA = new A;

        objA->mem1 = 3;

        B* objB = (B*)objA;

        cout << objB->mem1 << endl;

        objB->mem1 = 4;

        cout << objA->mem1 << endl;

        delete objA;

        return 0;
}
```
编译过程(即便加了`-Wall`参数)没有警告，运行得出:
``` text
3
4
```
甚至可以在违背事实的情况下，作出`objB->mem2=0.12`等操作造成难以发现的隐患。

如果同样在`Java`里面实现一样的代码的话，编译肯定是不通过的，但是这样的操作却在`C++`是允许的。这一方面为`C++`增加了不少的自由度，可以写出很灵活的代码;另一方面却给`C++`入门者一个极大的挑战，本身`指针`这个概念也难倒了不少入门者，当成功跨越这个门槛之后，却发现`指针`背后深藏更多的挑战，造成了开发调试上的困难。

好了，现在引用书上的例子，假设有个日期类:
``` c++
class Date
{
public:
        Date(int year, int month, int day);
        ...
};
```
书上是按美国的日期顺序`月-日-年`，笔者改成了更符合国情的格式。直到程序出现不符合预期之前，一般使用者也不会特意去查看文档(如果有的话)，那么就会有如下的使用:
``` c++
Date d1(2017, 12, 19);
Date d2(12, 19, 2017);
```
因为都是整型，编译器也不会给出什么错误，所以估计出错前也不会有人在意，这样的误用，该怎么样在早期就能提醒？书上给出了`外覆类型(Wrapper types)`，就是让编译器做类型检查:
``` c++
struct Day{
        explicit Day(int d)
                :       val(d)
        {}
        int val;
};

struct Month{
        explicit Month(int m)
                :       val(m)
        {}
        int val;
};

struct Year{
        explicit Year(int y)
                :       val(y)
        {}
        int val;
};

class Date
{
public:
        Date(const Year& y, const Month& m, const Day& d);
        ...
}
```
这样就会强迫使用者:
``` c++
Date d(Year(2017), Month(12), Day(19));
```
当然也可以使用`枚举类型`做一些替换，笔者觉得本质上也是`外覆类型`的一种变化，总之就是启用编译器的`类型检查`来纠错该接口的`误用`。

## 条款19:设计class犹如设计type

这个条款提出了一系列引导思考的问题。

#### 新type的对象应该如何被创建和销毁？
- 构造函数的设计，复制构造函数等。
- 动态分配的成员，在构造中new还是new[]，对应好析构器中的delete形式。
- 结合前面，动态分配的成员在该类复制行为中该如何处理。

#### 对象的初始化和对象的赋值行为该有什么样的差别？
- 初始化构造器。
- `assignment`赋值操作符。

#### 新type的对象如果被`pass by value`(值传递)，意味着什么？

#### 什么是新type的“合法值”？
- 例如上个条款中的`Date`，其月份和日有着相应的值域范围。

#### 你的新type需要配合某个继承图系(inheritance graph)吗？
- virtual关键字声明的时机。

#### 你的新type需要什么样的转换？
- 显式的转换调用增加程序的可靠性，隐式转换增加易用性。

#### 什么样的操作符和函数对此新type而言是合理的？
- 函数接口还能根据场景限制，操作符重载就真的得小心设计了，错误的设计会坑害使用者。

#### 什么样的标准函数应该驳回？
- 条款6.

#### 谁该取用新type的成员？
- 遵照面向对象设计原则，所有成员应该为`private`，然而这个不是万能的。所谓的原则就是在一头雾水的情况下先顶着用的后备方案。

#### 什么是新type的“未声明接口”(undeclared interface)？
- 对效率、异常安全性以及资源运用提供何种保证？依此加上约束条件。

#### 你的新type有多么一般化？
- 通用性很强的话，就是一组`types`了，定一个新的class template会大大减少代码的重复。

#### 你真的需要一个新type吗？
- 不要重复造轮子。

## 条款20:宁以pass-by-reference-to-const替换pass-by-value

书上说得比较多，但是最后的原因是`以值传递`会导致一个新的对象被复制出来，而且严格按照函数接口的类型规定进行。用一个例子说明:
``` c++
class Base
{
public:
	Base(string name_para = "base") : name(name_para)
	{
		cout << "Base::Base(this=" << this << ")" << endl;
	}
        // 在稍后的processVal函数中，涉及到复制构造，如果不重载复制构造函数，就看不到试验结果。
	Base(const Base& rhs) : name(rhs.name)
	{
		cout << "Base::Base(this=" << this << ", &rhs=" << &rhs << ")" << endl;
	}
	virtual void display() const
	{
		cout << "(Base Object)" << name << endl;
	}
	virtual ~Base()
	{
		cout << "Base::~Base(" << this << ")" << endl;
	}
protected:
	string name;	
};

class Derived : public Base
{
public:
	Derived(string name_para = "base") : Base(name_para)
	{
		cout << "Derived::Derived(this=" << this << ")" << endl;
	}
        // Derived也需要重载作为对比。
	Derived(const Derived& rhs) : Base(rhs)
	{
		cout << "Derived::Derived(this=" << this << ", &rhs=" << &rhs << ")" << endl;
	}
	virtual void display() const
	{
		cout << "(Derived Object)" << name << endl;
	}
	~Derived()
	{
		cout << "Derived::~Derived(" << this << ")" << endl;
	}

private:
};

void processVal(Base base)
{
	base.display();
}

// 不能重载processVal函数，否则因为重载决议推断一直使用Derived版，所以这里独立写出一个函数。
void processDerVal(Derived derived)
{
        derived.display();
}

void processRef(const Base& base)
{
	base.display();
}

void processPtr(const Base* const base)
{
	base->display();
}

int main(int argc, char* argv[])
{
	Derived d;

	cout << "----Process by Base value----" << endl;

        processVal(d);

        cout << "----Process by Derived value----" << endl;

        processDerVal(d);

	cout << "----Process by refernce----" << endl;

	processRef(d);

	cout << "----Process by pointer----" << endl;

	processPtr(&d);

	cout << "----All done----" << endl;

	return 0;
}
```
编译运行得到输出:
``` text
Base::Base(this=0x7ffce31ef230)
Derived::Derived(this=0x7ffce31ef230)
----Process by Base value----
Base::Base(this=0x7ffce31ef290, &rhs=0x7ffce31ef230)
(Base Object)base
Base::~Base(0x7ffce31ef290)
----Process by Derived value----
Base::Base(this=0x7ffce31ef2c0, &rhs=0x7ffce31ef230)
Derived::Derived(this=0x7ffce31ef2c0, &rhs=0x7ffce31ef230)
(Derived Object)base
Derived::~Derived(0x7ffce31ef2c0)
Base::~Base(0x7ffce31ef2c0)
----Process by refernce----
(Derived Object)base
----Process by pointer----
(Derived Object)base
----All done----
Derived::~Derived(0x7ffce31ef230)
Base::~Base(0x7ffce31ef230)
```
输出首尾是在`main`函数作用域，不需要关注，看到四个函数中，凡是以值传递的函数都调用了其传入参数对应的构造函数，新建出新的对象，并且造成了`切割(slicing)`现象。

笔者认为切割现象只是以值传递造成的新对象被构造出来的附带现象，其主要原因还是该方式是复制新的对象进入函数作用域中。且不说切割的问题，若该函数被频繁调用，那么频繁的构造和析构也能带来客观的性能开销了，更不要说更大的对象的构造问题了。

## 条款21:必须返回对象时，别妄想返回其reference

(待续)