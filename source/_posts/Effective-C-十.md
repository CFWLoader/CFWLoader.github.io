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

笔者反复看这个条款，最后也完全理解这个条款想讲的是什么。但是推断应该是与`类设计`相关的问题。引用书上的例子:
``` c++
class CompanyA
{
public:
	// 每个公司都有向该公司发送文本的函数
	void sendCleartext(const std::string& msg);
	void sendEncryted(const std::string& msg);
	...
};

class CompanyB
{
public:
	// 公司B也有
	void sendCleartext(const std::string& msg);
	void sendEncryted(const std::string& msg);
	...
};

...

class MsgInfo {...};		// 承载信息实体的类

template<typename Company>
class MsgSender				// 发送消息给公司的类模板
{
public:
	...
	void sendClear(const MsgInfo& info)
	{
		std::string msg;
		// 根据info产生信息
		Company c;
		c.sendCleartext(msg);	// 模板中调用sendCleartext()函数发送消息
	}

	void sendSecret(const MsgInfo& info)
	{
		std::string msg;
		// 根据info产生信息
		Company c;
		c.sendEncryted(msg);	// 模板中调用sendEncryted()函数发送加密消息
	}
};
```
接着为了让发送消息有日志，编写一个可以记日志的`LoggingMsgSender`:
``` c++
template<typename Company>
class LoggingMsgSender : public MsgSender<Company>
{
public:
	...
	void sendClearMsg(const MsgInfo& info)
	{
		...	//	发送前记日志
		sendClear(info);		// 调用MsgSender::sendClear()，但是无法通过编译
		... //	发送后记日志
	}
	...
};
```
问题的根源在于，当编译器处理`class template LoggingMsgSender`定义式时，首先要知道`MsgSender`的定义式，但是`MsgSender`是一个模板，也就是说需要一个`MsgSender<Company>`的实例的定义式。而`MsgSender`有可能有`特化`的版本，`特化`的版本与一般的模板又不一样，如:
``` c++
class CompanyZ			// Z公司类
{
public:
	...
	// 这个类不提供sendCleartext函数
	void sendEncryted(const std::string& msg);	// 仅提供发送加密消息的函数
	...
};

template<>				// 为了CompanyZ与众不同的行为提供一个全特化版本的类模板
class MsgSender<CompanyZ>
{
public:
	...
	void sendSecret(const MsgInfo& info)	// 因此这个特化的类模板也只提供发送密文的函数
	{...}
};
```
这个时候，`LoggingMsgSender`:
``` c++
template<typename Company>
class LoggingMsgSender : public MsgSender<Company>
{
public:
	...
	void sendClearMsg(const MsgInfo& info)
	{
		...	//	发送前记日志
		sendClear(info);		// 调用Company == CompanyZ时候，基类就不提供sendClear()函数了
		... //	发送后记日志
	}
	...
};
```
因为`MsgSender`有可能存在全特化的版本，因而编译器不会去处理基类相关的内容。

为了能够正常编译代码，有三个办法(假设所有的Company行为一致，不会像CompanyZ有与众不同的行为):
- 用`this`关键字
``` c++
template<typename Company>
class LoggingMsgSender : public MsgSender<Company>
{
public:
	...
	void sendClearMsg(const MsgInfo& info)
	{
		...	//	发送前记日志
		this->sendClear(info);		// 告诉编译器这个函数是类成员，强迫编译器从整个继承体系搜索
		... //	发送后记日志
	}
	...
};
```
- 用`using`声明式:
``` c++
template<typename Company>
class LoggingMsgSender : public MsgSender<Company>
{
public:
	using MsgSender<Company>::sendClear;		// 告诉编译器假定sendClear由基类提供
	...
	void sendClearMsg(const MsgInfo& info)
	{
		...	//	发送前记日志
		sendClear(info);		// 假设sendClear存在于基类，则编译通过
		... //	发送后记日志
	}
	...
};
```
- 显式指出函数的版本:
``` c++
template<typename Company>
class LoggingMsgSender : public MsgSender<Company>
{
public:
	...
	void sendClearMsg(const MsgInfo& info)
	{
		...	//	发送前记日志
		MsgSender<Company>::sendClear(info);		// 告诉编译器用MsgSender<Company>::sendClear
		... //	发送后记日志
	}
	...
};
```
第二三个方法显式地声明了函数，会使得`virtual`函数的多态行为失效。

还有，如果采用了解决方案之后，还出现`LoggingMsgSender<CompanyZ>`的实例化，编译当然就失败了。

## 条款44:将与参数无关的代码抽离templates

该条款第一个提到的问题是`非参数类型`带来的`代码膨胀`，`template`支持如下的语法:
``` c++
template<typename T, std::size_t size>	// 假设该类为size*size大小的方阵
class SquareMatrix
{...};
```
在写下:
``` c++
SquareMatrix<int, 3> m1;

SquareMatrix<int, 4> m2;

...
```
编译器会为`SquareMatrix<int, 3>`以及`SquareMatrix<int, 4>`模板类各自生成实例，可能产生如下的代码:
``` c++
class I3SquareMatrix
{
public:
	...
};

class I4SquareMatrix
{
public:
	...
};
```
当需要产生很多大小不一的方阵实例的时候，`类模板`的实例代码随之膨胀。然而仅仅只是因为尺寸的问题就带来代码膨胀，这是不必要的。通过`成员变量`一般可以解决问题:
``` c++
template<typename T>
class SquareMatrix
{
public:
	...
private:
	...
	std::size_t matrix_size;		// 通过一个成员变量记录方阵的大小
	T* data_ptr;					// 存储实际数据的指针，也可以是T**类型
	...
};
```
这样就解决了来自于`非参数类型`导致的代码膨胀。

而对于来自与`类型参数`的代码膨胀，一般不需要处理，要么就要有高超的编程技巧。例如`SquareMatrix<int>`、`SquareMatrix<const int>`和`SquareMatrix<long>`明明在`二进制表示`上一致(一般平台int和long一样)，但是编译器会分别为它们生成不同的类模板实例。往往这样做的话需要用到`void*`类型来操作一些手法，这个就是很底层的代码的问题了，而且`STL`大部分容器是做了这个优化的，因而笔者认为该部分大多数人了解即可。

## 条款45:运用成员函数模板接受所有兼容类型

书上用了`智能指针`来作说明，如下一个继承体系:
``` c++
class Top {...};

class Middle : public Top {...};

class Bottom : public Middle {...};
```
第一版的`智能指针`类定义:
``` c++
template<typename T>
class SmartPtr
{
public:
	explicit SmartPtr(T* ptr);	// 提供原生的指针进行初始化
	...
};
```
原生的继承体系，用父类的指针指向子类实例:
``` c++
Top* top1 = new Middle;

Top* top2 = new Bottom;
```
都没有任何问题，但是:
``` c++
void fun(SmartPtr<Top> top)
{
	// 处理一些只与Top类型相关的操作
}

...

SmartPtr<Middle> middlePtr(new Middle);	// 假定开发者在这个作用域知道实际的类型

fun(middlePtr);				// 根据继承体系，传入一个Middle实例应该是没问题的
```
但是问题就出现在这里了，原生的继承体系`Top<-Middle<-Bottom`是没有问题的，一般的编译器会为上面的使用生成如下的类模板实例:
``` c++
// 模板的实例化
class TopSmartPtr			// 为SmartPtr<Top>生成的模板类
{
public:
	explicit TopSmartPtr(Top* ptr);
	...
};

class MiddleSmartPtr			// 为SmartPtr<Middle>生成的模板类
{
public:
	explicit MiddleSmartPtr(Middle* ptr);
	...
};
```
注意并不是:
``` c++
class MiddleSmartPtr : public TopSmartPtr
{...};
```
也就是说`SmartPtr<Middle>`与`SmartPtr<Top>`没有任何关系，目前也没有代码提供它们之间的`显式`或者`隐式`类型转换，自然编译不能通过了。

为了解决这个问题，让`SmartPtr<Top> top(SmartPtr<Middle>(new Middle))`这类的语法能够正常工作，`SmartPtr`的类定义则应该这么写:
``` c++
template<typname T>
class SmartPtr
{
public:
	template<typename U>
	explicit SmartPtr(U* instance);		// T兼容U，例如例子中的继承关系

	template<typename U>
	SmartPtr& operator=(const SmartPtr<U>& sptr);	// 兼容类型的复制函数
	...
};
```
更具体的设计可以查看`std::shared_ptr`的源码，在这里要提示一点，在写下上面的成员函数的定义时，应该是:
``` c++
template<typename T>
	template<typename U>		// 笔者习惯加个缩进表示U参数是在T参数的作用域内
SmartPtr<T>& operator=(const SmartPtr<U>& sptr)
{
	...					// 实现代码
}
```
而不能写成不能通过的:
``` c++
template<typename T, typename U>
SmartPtr<T>& operator=(const SmartPtr<U>& sptr)
{
	...					// 实现代码
}
```
这样就解决了泛化的问题。

## 条款46:需要类型转换时请为模板定义非成员函数

问题类似与条款24,不同的是这次加入了模板，假设有个有理数的类模板:
``` c++
template<typename T>
class Rational
{
public:
	Rational(const T& numerator = 0, const T& denominator);		// 条款20
	const T numerator() const;					// 条款28
	const T denominator() const;				// 条款3
	...
};

template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs)
{...}
```
根据常理，有理数直接的操作:
``` c++
Rational<int> oneHalf(1, 2);

Rational result = oneHalf * 2;
```
应该是没问题的，但是在`强类型`语言中，编译器试图寻找匹配的函数，但是目前也只能匹配到:
``` c++
template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs)
```
显然根据地一个参数，`Rational<int>`被推导出来了，然而`template实参推导过程`不会考虑`通过构造函数而发生`的隐式类型转换。在编程人员尚未提供规则使得`int`可以向`const Rational<int>`转换，在产生模板函数过程自然推导失败了。

书上提供的解决方法:
``` c++
template<typename T>
class Rational
{
public:
	...
	// 将operator*重载变为类的友元函数
	friend const Rational operator*(const Rational& lhs, const Rational& rhs);
};

template<typename T>
const Rational<T> operator*(const Rational& lhs, const Rational& rhs)
{...}
```
这样做的原理是让编译器在产生`Rational<int>`类模板实例时，带上友元函数一起产生，使其省略上一个版本让编译器先为`operator*`推导的步骤，然后迫使`oneHalf * 2`套用函数时，对`int`强制做隐式类型转换。

编译通过，但是会链接失败，神奇的是，把`operator*`的定义放回类内部又可以了:
``` c++
template<typename T>
class Rational
{
public:
	...
	friend const Rational operator*(const Rational& lhs, const Rational& rhs)
	{
		...	// 函数定义
	}
};
```
`friend`关键字原本是用于让类外部的函数，或者类，可以访问私有成员，但是这里并没有这样的使用。

当类模板相关的函数需要隐式类型转换的时候，将函数定义为`类模板内部的friend函数`。

## 条款47:请使用traits classes表现类型信息

这个条款对于写轮子的人来说比较有用，`STL`用的人应该不少，但是阅读过STL源码的人可能不多，连笔者至今也没有系统读过C++ STL的源码，而STL中才会有大量涉及`traits classes`代码。

<font color="blue">Traits并不是C++关键字或者一个预先定义好的构建:它们是一种技术，也是一个C++程序员共同遵守的协议。</font>

因为模板编程的初衷是将与类型无关的操作提取出来，减少代码的重复性。不从实现角度来讨论，在该条款里面，模板编程表现得像`接口`一样，引用书上`advance`函数的例子:
``` c++
// advance作为一个STL算法实现，作用是让迭代器向前移动d步
// DistT是距离变量的类型
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
	if(iter是一个随机访问的迭代器)
	{
		iter += d;	// 直接跳d步
	}
	else
	{
		if(d >= 0){while(d--) ++iter;}		// 一步一步跳d步
		else {while(d++) --iter;}			// 当d小于0时是回退d步
	}
}
```
在`advance`例子里面，`if`分支里面的描述用到的就是输入的迭代器的`Traits`信息，假设迭代器的遵循`Traits`协议:
``` c++
template<typename IterT>
struct IteratorTraits			// Traits通常用struct做实现
{
	const static bool is_random_access = false;		// 在特化前，所有的迭代器默认没有随机访问的能力
	...							// 其他有关迭代器的Traits描述
};

...

template<>				// 全特化，假定存在RandomAccessIterator这个类
struct IteratorTraits<RandomAccessIterator>	// 针对该迭代器特化一个模板
{
	const static bool is_random_access = true;		// 该类迭代器具备随机访问能力
	...					// 其他Traits描述
};
```
有了类似这样的实现，那么上面的`advance`例子的判断语句可以具体为:
``` c++
if(IterT::is_random_access)
```
这时的`advance`就像是一个`接口`，你只需要输入兼容的迭代器即可，无需关心该迭代器的特性会影响如何完成这个操作。

笔者前面的模拟实现是需要运行时的判断，而模板元编程强大之处，也是设计思想，尽可能地在编译期完成计算，提高运行时的效率。那么`Traits`应该改成较为合理的形式(参考STL):
``` c++
template<...>		// 省略参数
class deque			// STL中的双向队列
{
public：
	class iterator 	// 双向队列自带的迭代器实现
	{
		typedef random_access_iterator_tag iterator_category;	// 说明自己的分类是随机迭代器
		...
	};
	...
};

// 根据上面STL中提供的迭代器定义，可以简单地提取其中自带的Traits信息，一般的实现
template<typename IterT>
struct iterator_traits 
{
	typedef typename IterT::iterator_category iterator_category;		// 说明自身是何种迭代器的traits
	...
};
```
可以根据STL的设计，提供一个运行时的判断:
``` c++
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
	// 判断遵循标准使用typeid实现
 	if(typeid(typename std::iterator_traits<IterT>::iterator_category) ==
 		typeid(std::random_access_iterator_tag))
 	...
}
```
为了优化运行时，同时不改变接口的情况下，这时候`advance`更像一个接口了:
``` c++
template<typename IterT, typename DistT>
void doAdvance(IterT iter, DistT d,
				std::random_access_iterator_tag)	// 用于随机迭代器的advance实际执行函数
{
	iter += d;
}

...		// 其他种类迭代器的doAdvance实现
```
这时的`advance`仅仅是单纯`转发(forwarding)`:
``` c++
template<typename IterT, typename DistT>
void advance(IterT iter, DistT d,)
{
	// 用Traits提供迭代器种类信息，随后让编译器决议用哪个版本的doAdvance
	doAdvance(iter, d,
		typename std::iterator_traits<IterT>::iterator_category)
	);
}
```
以上介绍了`Traits Classes`的基本原理和实现。

## 条款48:认识template元编程

(待续)