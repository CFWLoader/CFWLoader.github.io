---
title: Effective C++(七)
date: 2017-12-21 10:28:18
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

从这里开始进入`实现`主题，在此之前先区分好`声明`和`定义`，对于这两者的概念，笔者从网上摘抄下来:
``` text
声明(declaration)指定了一个变量的标识符，用来描述变量的类型，是类型还是对象，或者函数等。声明，用于编译器(compiler)识别变量名所引用的实体。以下这些就是声明:

extern int bar;

extern int g(int, int);

double f(int, double); // 对于函数声明，extern关键字是可以省略的。

class foo; // 类的声明，前面是不能加class的。

定义(definition)是对声明的实现或者实例化。连接器(linker)需要它(定义)来引用内存实体。与上面的声明相应的定义如下:

int bar;

int g(int lhs, int rhs) {return lhs*rhs;} 

double f(int i, double d) {return i+d;} 

class foo {};// foo
```
带着这个基础知识，开始介绍本主题的内容。

## 条款26:尽可能延后变量定义式的出现时间

在大规模的软件项目出现之前，良好的编程习惯是，在该作用域的开始，就把所需要的变量声明以及定义好，这样后面的代码就集中于处理逻辑，代码组织也在某种程度上符合审美标准。但是现在推荐的做法是将在需要该变量前的一刻才定义这个变量，特别是一些涉及到要根据if-else逻辑产生的变量，书上就给出了一个很好的例子:
``` c++
std::string encryptPassword(const std::string& password)
{
	using namespace std;
	if(password.length() < MINIMUM_PASSWORD_LENGTH)
	{
		throw logic_error("Password is too short");
	}
	... // 还有一些针对密码的规定的if-else处理进来的明文。

	string encrypted(password);				// 通过检查，可以加密原文了。
	encrypt(encrypted);						// 加密操作。
	return encrypted;
}
```
当涉及到循环的时候，来看看怎么写:
``` c++
// 方法A:定义于循环外
Widget w;
for(int i = 0; i < n; ++i)
{
	w = 取决于i的某个值;
	...
}
```
这样的开销是: 一次构造 + 一次析构 + n次赋值。

再来看看:
``` c++
// 方法B:定义于循环内
for(int i = 0; i < n; ++i)
{
	Widget w(取决于i的某个值);
	...
}
```
开销: n次构造 + n次析构。

然后对比的就是构造+析构和赋值的开销了，根据场景取舍。

## 条款27:尽量少做转型动作

在开始讨论这个条款之前，先介绍转型语法，首先旧式的转型:
- `(T)expression`，C风格转型动作。
- `T(expression)`，函数风格的转型动作。

`C++`提供4中新的转型动作:
- `const_cast<T>(expression)`，移除对象的常量性(cast away the constness)。
- `dynamic_cast<T>(expression)`，将一个父类对象`安全向下转型(safe downcasting)`为继承体系下的某个子类型。
- `reinterpret_cast<T>(expression)`，执行低级转型，实际动作和结果取决于编译器，使得程序变得不可移植。例如将`pointer to int`转为一个`int`。
- `static_cast<T>(expression)`，强迫隐式转换，笔者认为效果等同与上述两种旧式转型，但是无法去除对象的常量性。

介绍完转型语法之后，书上也讲了不少转型的例子和分析，但是总的来讲就是程序设计不佳，使得在使用这份代码的时候不得不进行一些类型转换。在实际应用中，`static_cast`大多数用于基础类型的转换，例如`int`转换成`float`等;`const_cast`这种移除常量性的需求笔者尚未遇到过;`dynamic_cast`的使用表征着这个继承体系的基类成员接口没有设计好，笔者也只是在少数情况下才采用这个转型;`reinterpret_cast`估计只有写嵌入式程序方面的人才有可能用到。

`dynamic_cast`的问题一般很好解决，例如连串:
``` c++
class Window{...};
...	// Window的子类定义。
typedef std::vector<std::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for (VPW::iterator iter = winPtrs.begin();
		iter != winPtrs.end(); ++iter)
{
	if(SpecialWindow1 * psw1 = dynamic_cast<SpecialWindow1*>(iter->get())) {...}
	else if(SpecialWindow2 * psw2 = dynamic_cast<SpecialWindow2*>(iter->get())) {...}
	else if(SpecialWindow3 * psw3 = dynamic_cast<SpecialWindow3*>(iter->get())) {...}
	...
}
```
那证明这个体系中，其子类共有的操作可以成为基类中的虚函数，是类型设计上的问题。

尽量避免使用转型，如果非转型不可，也尽可能使用新式转型语法。

不过考虑到现今的`C++`项目大多数开始都是老项目，而轻易修改老项目原代码是大忌，所以这些转型还是看着点用。

## 条款28:避免返回handles指向对象内部成分

这一条笔者认为与`条款22`有强烈联系，属于`面向对象`的设计原则之一，例如:
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
既然已经把数据与类成员函数绑定在了一起，原本已经声明了成员变量都是`private`，那么如果返回指向对象内部的handles(引用，指针，迭代器等)，那就相当于破坏了封装性，`getters`和`setters`形同虚设，使用者可以绕开这些设定好的接口和约束直接修改内部成员，极有可能破坏既定的行为。

因此，在没有特别的理由支持下，保持封装性比开放访问性是个更好的选择。

## 条款29:为“异常安全”而努力是值得的

(待续)