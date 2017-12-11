---
title: Effective C++(四)
date: 2017-12-11 15:16:08
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

## 条款10:令Operator=返回一个reference to *this

这个条款很直观，就是为了提供一个语法糖类似的功能使之可以:
``` c++
x = y = z;
```
这样的连续赋值，按照惯例贴下样例实现吧:
``` c++
class Object
{
public:
	...
	Object& operator=(const Object& rhs)
	{
		...
		return *this;
	}
	...
}
```

## 条款11:在operator=中处理“自我赋值”

例如可能发生如下的情况:
``` c++
Object o;

o = o;
```

最简单的方法就是做一次判断，如果是自己就什么都不做:
``` c++
class Object
{
public:
	...
	Object& operator=(const Object& rhs)
	{
		if(this == &rhs)
		{
			return *this;
		}
		...
		return *this;
	}
	...
}
```
当然书上也给出了另外一个应对异常安全的做法，就是定制一个`swap()`，这个方法会在条款29提及，以及笔者感觉特意为了这样的自我赋值做一个判断就够了。

## 条款12:复制对象时切勿忘其每一个成分

书中这个条款只是提醒了类变量成员的复制问题，而且前提是这些成员是对象成员，而不是一个指针，拥有指针时的复制就更加需要注意了。

先来简要介绍一下仅含对象成员:
``` c++
class Base
{
public:
	Base(const Base& rhs) : 初始化列表
	{
		...
	}
	...
	Base& operator=(const Base& rhs)
	{
		...
	}
private:
	...
};

class Mem1
{
public:
	Mem1(const Mem1& rhs) : 初始化列表
	{
		...
	}
	Mem1& operator(const Mem1& rhs)
	{
		...
	}
private:
	...
};

class Mem2
{
public:
	Mem2(const Mem2& rhs) : 初始化列表
	{
		...
	}
	Mem2& operator(const Mem2& rhs)
	{
		...
	}
private:
	...
};

class Com : public Base
{
public:
	Com(const Com& rhs) : Base(rhs), mem1(rhs.mem1), mem2(rhs.mem2), ...
	{
		...
	}
	Com& operator=(const Com& rhs)
	{
		Base::operator=(rhs);
		this.mem1 = rhs.mem1;
		this.mem2 = rhs.mem2;
		...
	}
private:
	Mem1 mem1;
	Mem2 mem2;
	...
}
```
注意好复制构造函数以及赋值操作符重载基本上问题就不大了，就是编写的时候不要漏掉该处理的成员，调用基类的赋值操作符重载补全复制不到的基类成员。

但是当成员中存在指针的时候，就涉及到`深度复制`的问题了，设想上述的改动:
``` c++
class Com : public Base
{
public:
	Com(const Com& rhs) : Base(rhs), mem1(rhs.mem1), mem2(rhs.mem2), ...
	{
		...
	}
	Com& operator=(const Com& rhs)
	{
		Base::operator=(rhs);
		this.mem1 = rhs.mem1;
		this.mem2 = rhs.mem2;
		...
	}
	Mem1& getMem1() const
	{
		return *mem1;
	}
	...
	~Com()
	{
		delete mem1;
		delete mem2;
	}
private:
	Mem1* mem1;
	Mem2* mem2;
	...
}
```
当使用该类的代码这么写:
``` c++
Com* com1 = new Com(...);

Com* com2(com1); 			// com2 = com1;

delete com1;

com->getMem1().调用一些Mem1的成员函数
```
此时成员`mem1`和`mem2`已经被释放了，所以com2中mem1指向的是一块未定义的内存，这样的调用是十分危险的，基本上都会导致程序意外终止，所以涉及到指针的复制要更加慎重。

在`Java`里面，所有类的根类是`Object`，而其拥有的九个方法其中之一就是`clone()`，可惜`C++`编程太自由了，无法强制规定使用者要遵循一些编程规范。以笔者的经验来看，在`C++`里面一般涉及到深度复制的之后，也只能靠自身养成的良好的编程习惯，例如给`Mem1`类添加`copy()`或者`clone()`函数:
``` c++
class Mem1
{
public:
	...
	Mem1* clone() const
	{
		Mem1* newObj = new Mem1(this->{自身一些非指针型成员数据的初始化列表});

		// 如果具有指针型的成员则递归调用其clone或者copy函数，否则在此手动深度复制
	}
private:
	...
};
```
这样定义之后，`Com`可以这样深度复制:
``` c++
class Com : public Base
{
public:
	Com(const Com& rhs) : Base(rhs), mem1(rhs.mem1.clone()), mem2(rhs.mem2.clone()), ...
	{
		...
	}
	Com& operator=(const Com& rhs)
	{
		Base::operator=(rhs);
		this.mem1 = rhs.mem1.clone();
		this.mem2 = rhs.mem2.clone();
		...
	}
	Mem1& getMem1() const
	{
		return *mem1;
	}
	...
	~Com()
	{
		delete mem1;
		delete mem2;
	}
private:
	Mem1* mem1;
	Mem2* mem2;
	...
}
```
经过这样的处理，即使原来的实例已经被释放了，但是因为复制的数据是完整的，两个对象之间并没有指针引用，所以就不会造成上述的错误。

当然实际应用场景中也有需要对象引用相同的成员的的需求，这时候要依据具体的需求设定好接口，不过`clone`和`copy`这类函数的语义最好设定为深度复制。

当然使用指针之后的问题就是需要手动管理内存了，在这里暂时不展开这个话题，毕竟`C++`内存管理一直是个大问题。