---
title: Effective C++(五)
date: 2017-12-11 16:58:47
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

前面探讨了`C++`中构造、析构以及赋值方面的问题，随后书上第三个主题就开始是`资源管理`。在当前流行的`Java`、`Python`等语言中，甚至早期的`Scheme`也有`垃圾回收(Garbage Collection, GC)`机制。使用者只需要在需要新的对象的时候用`new`语义创建对象实例，之后就不需要手动用`delete`语义释放这个对象或考虑对象何时会被释放，完全由运行时环境(虚拟机是运行时环境的子集)决定。

说回`C++`，因其历史原因等，是几乎不可能让语言集成垃圾回收机制，这样极有可能导致以后版本的编译器无法兼容老旧的代码，甚至造成语言自身分裂成不同的阵容@Python。而且垃圾回收本身也占用系统资源，这对于某些场景(如高频交易)来说，1毫秒的延迟都能造成不小的金融损失。所以`C++`只能在选择集成垃圾回收机制的以外做抉择。虽然`C++11`才正式地内建`shared_ptr`、`weak_ptr`、`scoped_ptr`等对象管理指针的方法，其实早在`0x`时代`Boost`库就已经做好了这些功能了，所以笔者觉得`C++11`很大部分就是把`Boost`中用得多的特性吸收了进来(Effective C++最后一条也是建议熟悉Boost)。

垃圾回收机制的威力不仅仅体现在释放使用者的脑力，使其能够更加专注与业务代码的构建。在现在硬件性能上升以及计算处理量越来越大的年代，`多线程(Multithreaind)`编程也变得十分常见。在单线程的场景下，使用者可以通过调试和使用一些内存泄漏检测工具(如`valgrind`)最终得到一个内存管理十分精确的版本。但是在多线程的环境下，前人积累下来的经验毁于一旦，在这个线程某个对象尚未初始化完成，可能就要接受另一个线程的访问;更恐怖的是在这个线程这个对象已经被析构了，但是被另外一个线程访问了。笔者曾经试过写一个多线程的程序，调试了三天，最终发现是某个变量在主线程被析构了，别的线程仍在访问。但是如果不释放一些对象的话，施以`互斥锁`应该可以解决对象的正确访问问题，但是这样可能程序就不能长久运行了，毕竟系统内存有限。

因此，结合`C++11`，使用者也可以摆脱手动管理指针的烦恼，再掌握一些良好的多线程编程习惯，多多少少可以减轻多线程编程恐惧。

## 条款13:以对象管理资源

这个条款，书上首先介绍了`资源取得时机便是初始化时机(Resource Acquisition Is Initialization,RAII)`，狭义理解为在构造函数中尽可能地使用初始化列表初始化成员。广义上就是资源一分配下来就放入到对象中管理。

在介绍了一个场景之后，就演示了一个用`auto_ptr`的例子，但是`auto_ptr`在`C++11`中被弃用了，用`unique_ptr`:
``` c++
void fun()
{
	std::unique_ptr<Object> objPtr(new Object(...));

	objPtr->{调用一些成员函数};
}
```
这是一个`RAII`的样例，fun一开始`unique_ptr`就申请了一个`Object`对象并且马上置入到`unique_ptr`对象中;fun结束之后，`unique_ptr`对象结束了在其作用域内的寿命，调用了析构函数，而这个析构函数就是`delete`一个`Object*`。

当一个对象在多处被引用，其析构的时机当然是在最后一个引用结束之后，但是这样如何得知其最后一个引用是在哪里？这里提出了一个`引用计数器`的方法，书上称之为`引用计数型智慧指针(reference-counting smart pointer, RCSP)`。简要的说，一个对象被引用一次，其引用计数就+1，引用其的某个对象结束了生命周期，该对象的引用便-1。当引用计数为0的时候，就释放该对象。但是这里的引用计数其有个`循环引用`的问题:
``` c++
void cyclesOfReference()
{
	shared_ptr<Object> obj1(new Object(...)), obj2(new Object(...));

	obj1->attach(shared_ptr<Object>(obj2));			// 假设attach函数是为对象建立某种关系，引起了引用+1;

	obj2->attach(shared_ptr<Object>(obj1));
}
```
这样在`cyclesOfReference()`的作用域内，`obj1`和`obj2`对象的引用数至少为2;当函数执行结束的时候，各自减了1，引用数至少为1。因为它们的引用数不为0,所以没被析构释放，但是它们再也不起作用了。其它集成了垃圾回收机制的语言中用了`可达性分析`的方法解决这个问题，在这里暂不展开讨论。

注意<font color="red">unique_ptr</font>和<font color="red">shared_ptr</font>等智能指针在析构的时候对对象用的的`delete`而不是`delete[]`，所以很遗憾它们只能管理单个对象。

另外`shared_ptr`也只能保证对象一定会在引用数为0时被析构，而不能保证对象只会被析构1次。

虽然`C++`也开始提供对象管理内存的机制，但是方便性不及集成垃圾回收机制的语言，使用者当作是个辅助性的手段就好，决不能依赖。

## 条款14:在资源管理类中小心copying行为

对于该条款，笔者的理解是有些资源是不能共享的，大约是现在笔者已经过了入门时期，自然而然地觉得对象设计根据场景就能得出对象该如何设计，例如书上举例的互斥锁:
``` c++
class Lock
{
public:
	explicit Lock(Mutex* pm) : mutexPtr(pm)
	{
		lock(mutexPtr);
	}

	~Lock()
	{
		unlock(mutexPtr);
	}
}
```
原理也是一个`Lock`在其所处的作用域中初始化时调用构造函数，在作用域结束时编译器调用其析构函数，这是一个C++编程上的小技巧。

但是:
``` c++
void fun()
{
	Mutex m;
	Lock m11(&m);
	Lock m12(m11);
}
```
编译器只能保证在`fun`的末尾肯定会插入`m11`和`m12`的构造函数，但是不能保证其顺序:
``` c++
void fun()
{
	Mutex::Mutex(&m);

	Lock::Lock(&m11, &m);

	Lock::Lock(*m12, &m11);

	Lock::~Lock(&m12);

	Lock::~Lock(&m11);

	Mutex::~Mutex(&m);
}
```
以上是编译为`fun`编译产生的一种中间代码，`m12`析构比`m11`要早，但是因为`m12`没有获取到锁，因而不能释放锁，程序在这里卡死。

书上给出了如下的解决方法:
1. 禁止复制，这个很自然就想到了。
1. 对底层资源祭出`引用计数法`，在这个例子中就更像把`互斥锁`升级为了`信号量`。
1. 复制底部资源，深度复制法，但是对这个场景不适用。
1. 转移地步资源拥有权。

第`3`、`4`点，笔者认为是程序设计阶段的问题，对于这样的复制行为，应该作好协议，规定这类行为的代码规范。

因而这个条款笔者不觉得有太多可讨论的地方。

## 条款15:在资源管理类中提供对原始资源的访问

其实这个条款也是直觉上就能理解的，毕竟面向对象的编程原则不是万能的，总有不适用的场景存在，例如这个主题中经常出现的智能指针`shared_ptr`等。`C++11`智能指针提供了`operator->`的重载，也就是提供了隐式转换:
``` c++
Object* pobj = new Object();

std::shared_ptr<Object> sobj(pobj);

pobj->fun();

sobj->fun();
```
其中`shared_ptr::operator->()`的源码:
``` c++
_Tp* operator->() const noexcept
{
	_GLIBCXX_DEBUG_PEDASSERT(_M_ptr != 0);
	return _M_ptr;
}
```
笔者也疑惑，这样处理之后的代码应该是:
``` c++
Object* pobj = new Object();

std::shared_ptr<Object> sobj(pobj);

pobj->fun();

(sobj::operator->())fun();
```
中间似乎缺少了一个`->`，不过这也只能解释为语言特性了。

总之为了补全封装性带来的程序设计缺点，总有场景需要提取封装中的内容物，这时候只能提供原始资源访问的接口，接下来就是显式接口，如`Java`的`String`要显式地使用`charAt(int index)`来获取指定索引的字符，这样虽然不方便，但是会防止误用;而隐式接口，`C++`的`string`提供的`operator[](int index)`很方便地就符合使用者的直觉。

## 条款16:成对使用new和delete时要采取相同形式

这个条款提醒了初学者，如果用了new，那么释放的时候就用delete;如果用了new[]，就用delete[]释放。这个规则是递归使用的，如果用了多维的定义，那么就得先delete[]低一维之后才释放该维:
``` c++
int** matrix = new int[10];

for(int row = 0; row < 10; ++row)
{
	matrix[row] = new int[10];
	for(int col = 0; col < 10; ++col)
	{
		matrix[row][col] = 0;		// 初始化。
	}
}

... 	// 一顿操作。

for(int row = 0; row < 10; ++row)
{
	delete[] matrix[row];			// 释放低一维的变量。
	matrix[row] = nullptr;			// 释放完之后设置空指针是个良好的编程习惯。
}

delete[] matrix;					// 再释放高一维变量。
```
这样的申请释放才不会造成内存泄漏。

注意书上提及的:
``` c++
std::string* stringArray = new std::string[100];
...
delete stringArray;
```
这里是用了`new[]`申请，但是用的`delete`释放，这样只会释放`stringArray[0]`。

注意前面也提过无论new的时候是什么形式，智能指针只会用`delete`释放资源，所以用智能指针管理资源的时候就要注意这个细节了。

## 条款17:以独立语句将newed对象置入智能指针

书上给了一个例子，也提供了具体的场景，函数原型如下:
``` c++
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);
```
有如下的调用:
``` c++
processWidget(std::shared_ptr<Widget>(new Widget), priority());
```
即使有`智能指针`的辅助，仍然不能根绝`内存泄漏`的问题，试想上述调用前可能会产生如下的构造:
``` c++
Widget* pw = new Widget();
int prio = priority();
std::shared_ptr spw(pw);
```
但是注意上述三行代码在原代码中是处于同一行的，也就是说，如果调用`proirity()`中出现了异常，导致了`std::shared_ptr`未构造，这时候显然`new`出来的对象就逃逸了。

书上给出的建议，笔者认为一个是编程上的技巧，也是编程上的良好的习惯，代码如下:
``` c++
std::shared_ptr<Widget> pw(new Widget);

processWidget(pw, priority());

...
```
这样就不会因为`priority()`的失败导致一个原始的`Widget`对象脱离了管理，事实上在调用一个函数的时候，也尽量简化这个调用的表达式，又有如下一个假想例子:
``` c++
Type ret_val = func(
	std::shared_ptr(new Object),
	para1 == ANY_VALUE ? para1 : OTHER_VALUE,
	func1(),
	...
);
```
这样一长串的调用，使得一个调用中的内容过于复杂，不方便后期维护审阅之外，其中任何一个参数的构造失败使得其它已经构造好的参数变得没有意义，也使对象不便于管理。

这个条款一个是提供编程上的小技巧，笔者认为更加像是一个良好的编程习惯。