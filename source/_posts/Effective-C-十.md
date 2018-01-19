---
title: Effective C++(十)
date: 2018-01-17 10:50:23
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

从此开始进入`模板与泛型编程`，这也是一般`C++`程序员较少接触的部分。如果读者有使用`C++`里的`标准模板库(Standard Template Librarry,STL)`的话，能够多多少少地体会到`模板`的威力。

正如书上所说，这里的内容不能让读者成为专家级的`template`程序员，但可以让读者成为一个比较好的`template`程序员。笔者认为这里的意思是介绍一些模板编程的基本知识，例如更好地使用STL，或者进行一些简单的模板编程。因为一些更加进阶的玩法，例如在编译器利用模板编程完成某些运算等这些内容，要对`模板元编程(Template Metaprogramming)`十分熟悉，这要求编程人员对`C++`的语法特性十分了解，甚至要对准备编译源文件的编译器十分了解。

笔者自上个主题后期开始的条款就呈现出经验不足，对条款的解释能力开始下降，总结一下无非自己的编程经历也只能理解到书的前半部分，后面部分笔者仍然任重而道远。笔者在剩下的部分里尽自己最大的能力，不足的地方请多指教。

## 条款41:了解隐式接口和编译期多态

有关`C++模板`编程的原理，读者们可以读《C++ Template》这本书了解。在开始介绍条款之前，读者们首先要了解模板是怎么编译的，例如:
``` c++
// 假定有个列表类
template<typename T>
class List
{
public:
	T operator[](int idx) const;	// 假设该List类模板重载下标访问元素函数
...
};
```
在编程中，这个`类模板`被使用了:
``` c++
List<int> intList;
```
在编译期，template是不会被编译的，而是先产生一个`模板类`:
``` c++
// 一般会有mangling发生，例如这个模板类会名为IList之类的，但是这里不考虑mangling的问题
class List
{
public:
	int operator[](int idx) const;	// int版的List，该函数的返回类型T被替换为了int
	...
};
```
然后编译器才会去编译这个`类模板`的实例。介绍这个原理的意义在于要说明，在`类模板`有实例之前，类模板里面有什么内容，编译器是不会理会的，例如:
``` c++
template<typename T>
class Container
{
public:
	void fun(const T& obj)
	{
		obj.call();			// T类具有call函数
	}
};
```
如果来了一个`Container<float>`，可想而知，当然是报错的。编译器也不会对`类模板`以及`模板类`有太多的检查，只要产生出来的模板类有符合模板类里面有的行为即可。笔者仿照书上的例子来说明结合`隐式类型转换`，模板编程会变得多困难:
``` c++
class A
{
public:
	operator int()()		// 假定类A有个向int转换的重载
	{
		return 33;
	}
};

class B
{
public:
	A& size()			// 一般的size认为是返回对象的大小，但是B不按套路来，返回a
	{
		return a;
	}
private:
	A a;
};

template<typename T>
void fun(T& obj)
{
	if(obj.size() > 10)			// 这个函数模板要求T类型有size()函数
	{
		cout << "Greater than 10." << endl;
	}
	else
	{
		cout << "Less than 10" << endl;
	}
}

int main(int argc, char* argv)
{
	B b;

	fun(b);

	return 0;
}
```
看到输出:
``` text
Greater than 10.
```
也就是说，`fun`里面，`B的实例`是符合有`size`函数这个约束的，而`B::size()`返回的`A类型`不能与`int`直接比较，而是调用了`A::operator int()`作了隐式转换之后比较。

这个例子要说明的是，`模板推断`的双面性，一方面`类型推断`的能力为编程人员带来了遍历，可以少写一些类型转换等语句，也就是语法糖吧;坏处就是推断太强大了，以至于做了`隐式转换`可能产生非预期的结果。

而这个`推断`是基于`有效表达式`的。这也是书上要介绍的`编译期多态`，也就是在编译器决议出用什么版本的函数。

## 条款42:了解typename的双重意义

在声明模板的时候:
``` c++
template<typename T>
class Container
{...};
```
与:
``` c++
template<class T>
class Container
{...};
```
里的`typename`和`class`是等价的关键字，但是在用到某个`类模板`里的内置类型时，例如:
``` c++
std::vector<Object>::iterator iter;		// 用到了vector里面的迭代器
```
然而有可能在编译过程中，编译器可能认为`std::vector<Object>::iterator`是`vector`里面的一个成员变量，而真实情况是`vector`里面的一个`嵌套从属类型`，编译器有可能会报错。这时改成:
``` c++
typename std::vector<Object>::iterator iter;
```
问题解决，这时候的`typename`是不能用`class`关键字代替的，这里的`typename`声明`std::vector<Object>::iterator`是一个类型，而不是某种实例变量。

## 条款43:学习处理模板化基类内的名称

该条款涉及到:
- [模板的特化](https://en.wikipedia.org/wiki/Generic_programming#Template_specialization)

(待续)