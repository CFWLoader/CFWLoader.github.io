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

经过上个条款的学习之后，大部分初学者应该会不留余力地去除`pass-by-value`这样的使用，以至于一个函数的返回值都返回一个对象的引用。其实这也是初学者常犯的一个错误，但是一旦经过学习，这个编程习惯还是挺容易就记住的。这里面涉及到的主要的知识点无非是`变量的作用域`，或者说是`变量的生命周期`，看看上个条款中笔者给出的例子，分析得出:
``` text
在main构造一个Derived实例derived;
        ---过渡带，保存一些当前上下文的信息，准备进入processVal函数上下文
        以derived实例为原本复制一个Base实例，暂且变量名base;
                ---进入processVal函数的作用域
                调用base.display();
                ---退出processVal作用域
        析构base实例;
        ---切换上下文，回到main
...     //      涉及到以值传递的函数都会发生上述过程。
在main中析构derived实例;
```
假设有个函数新建一个Base对象，并且返回其引用，原型如下:
``` c++
const Base& generateBase()
{
        Base base;
        return base;
}
```
并在main中使用:
``` c++
int main(int argc, char* argv[])
{
        generateBase().display();
        return 0;
}
```
幸运的话也许还能正常输出，不过基本上都是报错告终的了。

把main像上面那样翻译一下:
``` text
        ---保存main上下文
                ---进入generateBase函数作用域
                构造一个Base实例;
                析构这个Base实例;
                返回这个Base实例的引用;
                ---退出generateBase函数作用域
        把返回的引用加入到main上下文;
        ---切换回main的上下文
调用这个返回的base对象的display函数
```
显然，`Base`对象已经被析构了，调用一个被析构的对象当然产生为定义的行为。

记得被调用的函数的作用域肯定比调用该函数的函数的作用域段，那么被调用的函数里面的对象当然不能返回其引用给上一级了。

这个时候要么乖乖地返回一个值，向性能效率妥协;要么就在堆上新建对象返回其指针，对自己的程序设计有信心的话，手动管理内存，幸好现在智能指针是标准库的一部分，也可以考虑考虑。

## 条款22:将成员变量声明为private

在学习过`Java`之后，笔者都忘了为何要这么设计一个类的。不过从书上看来，这个是前人总结出来的经验，如果不是对自己的项目设计能力特别有信心的话，循着经验来总不会错的。

将成员变量设置为`private`，然后通过`getter`、`setter`的实现来控制好外部对该类内部成员变量的控制。前人的经验，虽然有时候会有些不方便，但是有个总体的原则在，程序就不会太乱了。

## 条款23:宁以non-member、non-friend替换member函数

对于该条款，笔者再次阅读的时候也没完全理解，不过大概的意思就是`过度封装`的问题，`面向对象`的设计思想确实很强大，也很好用，但是滥用也是会出问题的。也有可能有时候从业务逻辑上看，某个函数是某个对象的成员函数不太符合直观感觉;或者某个种行为对该模块里面的类通用，抽取出来变为一个通用的函数。

其实这样的问题会在`Java`这类完全以对象为基础的语言中更加突出，例如库函数`sin`、`cos`等明明可以成为一个独立的函数，在`Java`中却必须定义在某个类中，哪怕是`static`也好。例如进行一次`sin`:
``` java
Math.sin(1);
```
而`C++`的`容器类`(vector, map, set等)和`算法库`(&lt;alogorithm&gt;)就是一个对抗过度封装的很好的例子，举一个`find`的例子:
``` c++
vector<int> vecCon;
map<int,int> mapCon;
vector<int>::iterator vIter = find(vecCon.begin(), vecCon.end(), 3);
map<int,int>::iterator mIter = find(mapCon.begin(), mapCon.end(), make_pair(1, 3));
```
算法库中有不少这样的函数，对于每种容器类，其通用操作查找，添加等，如果为每个容器都写一次，那么代码的重复性就太高了，当然使用者会很感谢的。并且这样的操作本身也不会与容器中的类中的成员有绑定现象。

其实归根结底还是写代码的人对与场景的分析。

## 条款24:若所有参数皆需要类型转换，请为此采用non-member函数

这个条款从书上的例子出发:
``` c++
class Rational          // 有理数类
{
public:
        Rational(int numerator = 0, int denominator = 1);
        int numerator() const;
        int denominator() const;
        const Rational operator* (const Rational& rhs) const;   // 用于支持有理数相乘
private:
        ...
};
```
有如下的调用:
``` c++
Rational oneEight(1, 8);
Rational oneHalf(1, 2);
Rational result = onHalf * oneEight;    // ok
result = oneHalf * 2;                   // ok
result = 2 * oneHalf;                   // 编译不通过
```
对于第二个相乘，编译器产生如下代码:
``` c++
const Rational temp(2);
result = oneHalf * temp;                // 调用Rational::operator*(const Rational& rhs);
```
但是对于数值`2`，起码`C++`没有为数值提供类定义，只是一个普通的数值，如果非得用面向对象来看的话那么第三个相乘操作调用的是:
``` c++
const Rational int::operator*(const Rational& rhs);
```
显然就算有`int`这个类，因为是内建类型，修改其定义是非常疯狂的行为。因为无法定义这个函数，编译器也无法获得`Rational`到`int`的隐式转换，当然编译不通过了。于是书上将相乘操作提出来成为一个独立的函数:
``` c++
const Rational operator*(const Ratinal& lhs, const Rational& rhs)
{
        return Rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
}
```
这样原来的`2 * oneHalf`在`自动推导`中会得出:
``` c++
const Rational temp(2);
result = temp * oneHalf;
```
完成隐式类型转换并推导出使用`const Rational operator*(const Ratinal& lhs, const Rational& rhs)`这个函数，编译通过。

## 条款25:考虑写一个不抛异常的swap函数

笔者对该条款没有什么特别的感受，所以简述书上的总结带过好了。

1. 当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不抛出异常。
1. 如果你提供一个member swap，也该提供一个non-member swap用来调用前者。对月classes(而非templates)，也请特化std::swap。
1. 调用swap时应针对std::swap使用using声明式，然后调用swap并且不带任何“明明空间资格修饰”。
1. 为“用户定义类型”进行std templates全特化是好的，但千万不要尝试在std内加入某些对std而言全新的东西。