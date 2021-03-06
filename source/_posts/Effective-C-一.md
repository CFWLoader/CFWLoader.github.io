---
title: Effective C++(一)
date: 2017-12-06 15:29:56
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

Scott Meyers的`Effective C++`是C++程序员推荐的读物之一，其中充满了作者的总结出来的编程经验，当年笔者刚入门编程的时候一口气从谭浩强的《C语言编程》开始，《C Primer》、《C++ Primer Plus》等等加起来有10本书学习了如何在C++下编程。当年读这本书的时候也不具备什么具体的编程实践，除了做了书上的一些代码的实验之外，无非也就是在平时的编程中有意地使用书中的编程技巧。时至今日，笔者目前的工作虽然不是以C++来编写实践项目，像一般的`Web`项目优先考虑的是快捷的`Java`开发，但是学过的编程技巧却是跨越语言的。并且从这几年的学习编程的经验来看，真正对自己编程能力有真正提升的并不是学习了XX语言。

现在总结起来，笔者的编程能力有了突变式的提升是在大一下学期有一门《数据结构》，学习每种经典的数据结构时，笔者对ADT(Abstract Data Type,抽象数据类型)毫无招架之力，书中对这些经典的数据结构(栈、队列等)有一个抽象的描述，用脑子想似乎很正常，但是在具体实现时常常不知从何下手，除了一遍一遍地抄书上的代码实践之外抄到熟之外别无他法。这样下来，锻炼到遇到一个问题下来，首先是问题的分解，分解到可以直接想到这部分的代码该是什么样子的，把每个最小的部分想好之后向上整合，形成解决整个问题的具体思路。

到了后来实践具体项目的时候，客户的需求描述比课本的描述更加抽象，更多的是靠工程经验想像出客户的期望，也有实际的工程工具(软件开发模型等)帮助项目的一步一步实现。终归到底，依靠的还是问题分解。

笔者虽然已经读过了`Effective C++`，也在这几年中生搬硬套应用过书中的知识，多少都有了些心得，于是在打算在这段较为空闲的时间中写下，也算是复习这本书。

## 条款1:视C++为一个语言联邦

书中提到了应该视C++拥有4个次语言:
1. C
1. Object-Oriented C++
1. Template C++
1. STL

第1点当然很容易理解，C++兼容C的语法，并且可以直接使用C的库，但是在笔者的观点上看，使用C++更多的是为了第2点，理由稍后描述。而在笔者的编程经历上看，很少(除了用VC写C)用C++写纯C语言项目，尤其是在`Linux`下。举个`Socket`编程的例子，用`C`的话一般这么写:
``` c
int fd = socket(AF_INET, SOCK_STREAM, 0);

if(fd < 0)
{
	// 做一些申请失败的处理代码，如果程序不能离开这个资源的话，甚至需要退出整个程序。
}
```
这里一小段还好，如果是写服务器端的代码的话，用到`bind`、`listen`、`accept`等函数，每次调用完之后都要判断返回值，后续还有设置其他参数等，这样的代码组织方面不会说出大的问题，但是不雅观。那么用`C++`一般是这么写的:
``` c++
Socket socket(8080);
```
当然`Socket`类里面封装的代码大体还是跟原来的`C`部分相同，这里用到的是第2点中的`封装`性质，但是这里又出一个问题了，原来的`C`代码中，每一步一旦申请资源失败都可以在调用者方进行处理;进行了封装之后，中间的过程申请资源失败了，那么该怎么处理？是在本函数体内处理完异常？还是返回错误信息给调用者？如果是内部处理异常的话，那么发生了调用者不期望的行为，而调用者放心地把所有问题交给了封装好的对象，那么调用者不理会的话，程序会因为严重错误而终止运行;如果是返回错误信息给调用者的话，那么这个封装的意义何在？所以有的论调是用`C++`不如用`C`进行项目开发，除了平添开销之外没有好处，从上述的例子上看多少有点道理。但是看看隔壁的`Java`面向对象风生水起，证明了面向对象的开发可以大大提高软件的开发效率(注意是开发效率)。总之就是见仁见智吧。

第3点是`Template C++`，泛型编程，为了减少代码的重复而存在，例如大小的比较，总不可能一个int就写一个专门的int型比较函数，一个float型就写一个专门的float型比较函数吧。当然这只是基础的背景，事实上泛型编程有更多的应用，学习难度更高，例如泛型元编程(template metaprogramming)，然而笔者还没在应用中见识过这样的东西，一般都是做着实验玩的居多。

第4点是标准模板库(STL,Standard Template Library)，其实就是把经典的数据结构及其算法集成在语言基础库中，也是为了减少代码的重复，不然一个项目重新做一个轮子，这样的时间耗费基本没有意义。STL提供的库可以显著地改变代码风格，其中的`迭代器(Iterator)`是一个很重要的概念，用了STL提供的概念之后，自己实现的代码中很可能都不需要写任何算法方面的代码，也没有循环的存在，有可能两三句就把一个功能完成了。

## 条款2:尽量以const,enum,inline替换#define

这一条款书上提供了具体的例子，`#define`定义的常量不计入符号表，到时报错的时候很可能就是一个简单的数值，这让开发者无从着手调试。而用`const`定义的常量，一是具有类型，不怕使用者错误地使用这个常量值;二是这样定义的常量记入符号表，这样编译报错起码之后错了的变量名。

使用`define`无法限制该变量的作用域，即便是在类中声明定义的`define`也能在全局中访问到。

`enum`取代`define`书上没有太多的陈述，总体来说就是为了维护代码的可读性吧，加上`C++11`支持`class enum`，即强类型的`enum`。例如:
``` c++
enum class Day
{MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY};
```
在`C++11`之前不能用`enum class`而用`enum`的时候，编程者可以通过
``` c++
Day afDay = Day::MONDAY;

aDay == 0;
```
直接对比`Day`类型和`0`的值，虽然一般情况下这样第一个枚举量的值是0,但是不排除会有别的赋值，这样会导致非期望的行为，这也是编程者错误使用的一种方式。

而加入`enum class`之后，编译器会检查这个枚举实例的类型，如果类型不一致就会报错，从而避免而编程者的错误使用，提高了代码的可维护性。

当某一段代码重复出现的频率很高时，会考虑将这部分代码抽象为一个函数，例如`max`、`min`等;调用函数会产生`上下文切换`开销，当函数调用频率非常高，频繁的上下文切换开销会变得非常客观。当然现在的编译器有针对这个情况的优化，在这里暂时忽略。所以用`#define`抽象这个过程，一来可以抽象这段代码，而来又不会招致额外的开销。但是这样做也有明显的缺点，例如书中提到的例子:
``` c++
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```
当开始调用这个宏定义的时候:
``` c++
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);
CALL_WITH_MAX(++a, b+10);
```
会被展开为:
``` c++
f((++a) > (b) ? (++a) : (b));
f((++a) > (b) ? (++a) : (b+10));
```
第一行因为是`++a`比较大，所以`++a`执行了两次，大概一般的程序员都不会期待这样的结果。

替代的方法就是:
``` c++
template<typename T>
inline void callWithMax(const T& a, const T& b)		// 条款20
{
	f(a > b ? a : b);
}
```
这样`++a`就不会重复两遍了。

注意，`inline`只是提供优化意见给编译器，编译器并不会强制执行。

## 条款3:尽可能使用const

依笔者的经验，用`const`限制变量的修改性是比较有意义的事，这样可以很大程度地避免使用者错误地使用你提供的代码，例如在类成员函数加上`const`表示该方法不能修改该类实例中的成员变量。但是奇怪的是C++又提供了`mutable`关键字修饰成员变量，使之可以在`const`成员函数中被修改，当然这样也是因为其特殊的应用场景，毕竟应用代码的场景太多了，不能因为自己没有遇到过而否定这样的编程方式。

其他情况诸如`friend`友元关键字，明明声明了一个类成员是`private`，但是声明了友元之后却可以在类外部被直接访问，这样又破坏了面向对象中的封装性。但是不通过这样的编程方式的话，又难以达成
``` c++
cout << {自定义的类实例};
```
这样的语法糖。造成C++今日这种语法复杂，各种原则衍生，互相破坏的原因大概也是因为前人设计该语言的能力有限，但是也证明了通过兼容各种设计原则才能让C++支撑到今日。

书上关于`const`的介绍例子很多，但是笔者认为只要程序员认为该处不应该发生修改行为，就可以施用`const`语法。但是存在一个问题，一个类的成员函数作用是返回其中一个成员的引用，但是该函数用了`const`来修饰，如果调用者修改了这个引用，应该就是发生了非期望的行为。这个编程习惯应该避免。

另外摘抄书上一段关于`const`指针声明，毕竟这个经常让人混乱:“如果关键字const出现在星号左边，表示被指物是常量;如果出现在星号右边，表示指针自身是常量;如果出现在星号两边，表示被指物和指针两者都是常量”。

## 条款4:确定对象被使用前已先被初始化

其实这也是一个编程习惯上的问题，例如书上的例子:
``` c++
int x;

class Point
{
public:
// getters and setters
private:
	int x,y;
};
```
一般情况下声明这些变量实例的时候，这些`int`的实例会被初始化为`0`，但是显然可能发生别的值的问题，还是显式地声明初始值比较好:
``` c++
int x = 0;

class Point
{
public:
	explicit Point(int x_, int y_) : x(x_), y(y_)
	{}
//getters and setters
private:
	int x,y;
};

Point point(0, 0);
```
加上`explicit`关键字，强迫实例必须具有初始值，这样就可以保证实例能被正确地初始化。注意示例代码中用了`初始化列表`，这个是常用的编程技巧，结合《深度探索C++对象模型》来讲述，如果用初始化列表会产生如下的构造器代码(猜测):
``` c++
Point::Point(Point* this, int x_, int y_)
{
	this->x(x_);
	this->y(y_);
}
```
但是如果将构造器原始代码改成:
``` c++
class Point
{
public:
	explicit Point(int x_, int y_)
	{
		x = x_;
		y = y_;
	}
//getters and setters
private:
	int x,y;
};
```
则可能产生如下的代码:
``` c++
Point::Point(Point* this, int x_, int y_)
{
	this->x();
	this->y();
	this->x=x_;
	this->y=y_;
}
```
以上代码仅仅作一个示例说明，在此就不详细展开了，日后笔者在写C++对象模型的博客时会提到。

书上还提示要记住初始化列表的顺序要与类变量在声明时的顺序保持一致。

这一条款中笔者比较少碰到的就是跨文件类实例初始化问题。例如`Point`中设定一个原点:
``` c++
class Point
{
public:
	explicit Point(int x_, int y_) : x(x_), y(y_)
	{}
//getters and setters
private:
	int x,y;
};

extern Point origin(0, 0);
```
而当在外部源码文件使用到`origin`这个文件时，因为
``` text
C++对“定义于不同的编译单元内的non-local static对象”的初始化相对顺序并无明确定义。
```
也就是说假设使用者定义了一个二维坐标系的类实例并用到了`origin`变量时，这个二维坐标系实例已经完成了初始化，而`origin`可能尚未初始化。

解决这个问题用的技巧则是依赖于"C++保证函数内的local static对象会在‘该函数被调用期间’、‘首次遇上该对象之定义式’时被初始化"。转化成代码则是在原来的文件上添加:
``` c++
Point& getOrigin()
{
	static Point origin(0,0);
	return origin; 
}
```
这样就可以确保这个`origin`类实例可以在使用前完成初始化。