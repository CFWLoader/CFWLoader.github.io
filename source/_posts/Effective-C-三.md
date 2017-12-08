---
title: Effective C++(三)
date: 2017-12-08 10:56:32
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

前面{% post_link Effective-C-二 Effective C++(二) %}开始讲述了`构造函数`、`赋值函数`以及`析构函数`，在讲述`条款5`时稍微展开讨论了一下细节，没想到篇幅会变长，看来之后有关`Effective C++`每篇博客的讨论的条款数量不定，只能以篇幅来限制，所以这个系列会有多少片，笔者也不清楚，碰到一些比较熟悉的条款应用就展开多讨论一些。

## 条款7:为多态基类声明virtual析构函数

这里涉及到面向对象中的`多态`这个性质，假设有如下继承关系，`Base`基类的析构函数为非虚函数:
``` c++
class Base
{
public:
	~Base()
	{
		cout << "Base::~Base(this=" << this << ")" << endl;
	}
};

class Derived : public Base
{
public:
	~Derived()
	{
		cout << "Derived:~Derived(this=" << this << ")" << endl;
	}
};
```
在析构函数中输出一些信息，有如下的`main`:
``` c++
int main(int argc, char* argv[])
{
	Base* b = new Base();

    Derived* d = new Derived();

    delete b;

    delete d;

    cout << "--------Stg--------" << endl;

    b = new Derived();

    d = new Derived();

    delete b;

    delete d;
}
```
会得到如下的输出:
``` text
Base::~Base(this=0x1d04c20)
Derived:~Derived(this=0x1d04c40)
Base::~Base(this=0x1d04c40)
--------Stg--------
Base::~Base(this=0x1d04c40)
Derived:~Derived(this=0x1d04c20)
Base::~Base(this=0x1d04c20)
```
首先看到第一个由`Derived`指针指向的对象只执行了`Dervied`自身的析构函数，随后正确执行了`Base`部分的析构。

然后因为上一个对象被回收了，导致下一个创建出来的对象用了上一个对象的地址，笔者在此加了一行输出隔开。用一个`Base`指针指向的`Derived`对象，其地址与上一个`Derived`对象一致，然而输出分割线下这个`Base`指向，实际为`Dervied`对象只执行`Base`部分的析构，这样就造成了`Derived`部分的内存泄漏了。

随后修改`Base`:
``` c++
class Base
{
public:
	virtual ~Base()
	{
		cout << "Base::~Base(this=" << this << ")" << endl;
	}
};
```
不改动`main`代码，编译执行，得到如下的输出:
``` text
Base::~Base(this=0x1156c20)
Derived:~Derived(this=0x1156c40)
Base::~Base(this=0x1156c40)
--------Stg--------
Derived:~Derived(this=0x1156c40)
Base::~Base(this=0x1156c40)
Derived:~Derived(this=0x1156c20)
Base::~Base(this=0x1156c20)
```
第三个new出来，由`Base`指针指向的`Derived`对象正确地按照`Derived::~Derived()`、`Base::~Base()`顺序析构了这个对象。

虽然书上给出的心得:"只有当class内含至少一个virtual函数，才为它声明virtual析构函数"，但是笔者认为这只是一个明显的信号，在实践中，一般能够通过分析得出该类是否会被继承，这时候就可以为其声明virtual析构函数了。

注意<font color="red">std::string的析构函数是non-virtual的</font>，所以把std::string作为基类编写自定义类的时候是一个不明智的行为。

书中提到了如果一个带有纯虚析构函数的基类，其声明纯虚函数的作用是为标记此类为抽象类，但是其析构函数仍然具有行为，则可以在声明其纯虚析构函数后继续给出定义:
``` c++
class AWOV 		// Abstract w/o Virtuals
{
public:
	virtual ~AWOV() = 0;
}

AWOV::~AWOV()
{}
```
这样即使抽象类具有数据成员时，因其纯虚的析构函数具有定义，也可执行析构函数成功进行析构动作。

当一个类中带一个纯虚函数的时候，那么该类就是一个抽象类，是不可实例化的。讲到这里，笔者觉得需要与`Java`的类继承机制作为对比，`Java`只允许单继承，而允许多实现，例如:
``` java
class Base
{...}

interface FunPack1
{...}

interface FunPack2
{...}

class Dervied extends Base implements FunPack1, FunPack2
{...}
```
单继承纵然会限制了语言的灵活性，多继承存在如下的问题:
![](Effective-C-三/inher0.jpg)

如果`~A()`是一个虚函数，那么一个`D`的实例被`delete`的时候其`A`部分会被析构几次？这个问题可以在对象模型中找到答案，这里就不展开细说。单继承是为了解决多继承的一些缺点，但是从`Java`多接口实现来看，事实上是多继承的功能的子集，`Java`的接口只允许声明方法的签名以及一些静态量，当用这个接口类型‘指向’其实现的对象时，则可以调用该接口声明的函数，实施一些行为。

笔者在此不是对比单继承多继承的优缺点，而是说明通过限制`C++`多继承可以模拟`Java`的这种方式，提供了面向对象设计的一种思路。毕竟正是因为`C++`的多样的语法导致其编程人员的上限可以很高，也可以很低，当项目集成的时候容易出现致命问题，所以在真正的`C++`项目中会事先做好一些约束。

也就是说，当多继承容易出问题的时候，不妨考虑约束拥有纯虚函数的基类不得拥有成员变量，使之成为一个仅仅声明纯虚函数接口的接口类，而且在后来的条款也会建议，可以用`组合`设计模式解决问题的时候就不考虑`继承`。以此为原则，基本上可以解决很多问题，起码在笔者的经历中还没碰到过非得用上述`菱形`继承的问题。

(待续)