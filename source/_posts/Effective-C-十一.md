---
title: Effective C++(十一)
date: 2018-01-24 11:17:42
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

从此开始进入`定制new和delete`主题，因为`C++`手动管理内存的双面性，因此了解`new`和`delete`是十分必要的。时至今日，`垃圾回收(Garbage Collection，GC)`十分流行，主流的的语言里面也只有`C`和`C++`没有`GC`的支持，下图是2018的TIOBE语言使用统计:
![](Effective-C-十一/tiobe_index2018jan.jpg)

可以看到`Top 10`的语言里面除`C`和`C++`以外都是具有`GC`的语言。GC固然会带来程序性能的下降，但是免去手动管理内存带来的开发效率的提高更加明显，`Top 1`的`Java`就很好地说明了这点。而且在`多线程`的挑战下，手动内存管理变得尤其困难，笔者曾尝试过写一个多线程Socket程序，却总是崩溃，找了三天才找出是自己没有正确管理好内存。当一个执行线程开始使用一个对象的时候，该对象已经在其他线程中被析构了。因此从这点看来，`C++`需要耗费不少精力才能学好。

`Effective C++`里面只有`operator new`的出现，如果读者已经阅读过《深度探索C++对象模型》的话，应该也接触过`new operator`，要注意这两者的区别。

## 条款49:了解new-handler的行为

当`operator new`因无法满足所要求的内存需求而抛出异常前，其中一步会调用客户制定的错误处理函数，即所谓的`new-handler`:
``` c++
namespace std
{
	typedef void (*new_handler)();
	new_handler set_new_handler(new_handler p) throw();
}
```
也就是说一旦调用了`set_new_handler`之后，在申请内存失败的时候则会调用输入`p`函数。而一个设计良好的`new-handler`则应该有以下的行为:
- 让更多的内存可被使用。类似GC过程，或者在程序初始化时开辟更大块的内存。
- 安装另外一个new-handler。让有能力处理的new-handler接手。
- 卸除new-handler。直接让operator new失败时抛出异常。
- 抛出bad_alloc异常。
- 不返回，调用abort()或者exit()。

一般没有太大的必要去定制`new-handler`，如果非要定制的话，引用书上的实现:
``` c++
class Widget
{
public:
	static std::new_handler set_new_handler(std::new_handler p) throw();
	static void* operator new(std::size_t size) throw(std::bad_alloc);
private:
	static std::new_handler currentHandler;
};
```
除非是`const`的静态成员，否则需要把定义式写在类的定义外:
``` c++
std::new_handler Widget::currentHandler = 0;	// 初始化为null
```
标准版的`set_new_handler`:
``` c++
std::new_handler Widget::set_new_handler(std::new_handler p) throw()
{
	std::new_handler oldHandler = currentHandler;

	currentHandler = p;

	return oldHandler;
}
```
`Widget::operator new`行为如下:
1. 调用标准set_new_handler，告知Widget的错误处理函数。会将Widget::currentHandler = ${global new-handler}。
1. 调用global operator new，执行实际的内存分配。如果分配失败则调用Widget::currentHandler。
1. global operator new成功分配，返回内存指针。Widget::~Widget()会管理global new-handler，会恢复`Widget::operator new`调用前global new-handler。

后面原作也用了详细的代码介绍了上面的三个过程，这里省略掉一是因为这部分可以查原书代码了解，二是笔者对这部分了解不深，基本上仍然处于初次接触的水平，三是该条款重点是了解`set_new_handler`以及相关的过程即可。

## 条款50:了解new和delete的合理替换时机

对于绝大多数的`C++`程序员，语言提供的原始的`operator new`和`operator delete`基本上可以满足程序的需求，很少人想到去重载(原书上用的“替换”)这两个`operator`。书上列出了三个最常见的理由:
- 用来检测运用上的错误。
- 为了强化效能。例如插入一些`内存碎片`整理的算法。
- 为了收集使用上的统计数据。

总的来说，就是改变原有的行为，修改分配规则，或者插入一些信息记录的步骤。书上也给出定制`operator new`的一般方法:
``` c++
// 让分配的对象具有签名
static const int signatrue = 0xDEADBEEF;	// 特征值
typedef unsigned char Byte;

// 暂时忽略一些小错误
void* operator new(std::size_t size) throw(std::bad_alloc)
{
	using namespace std;

	size_t realSize = size + 2 * sizeof(int);	// 空出2个int存放signature

	void* pMem = malloc(realSize);				// 调用malloc分配内存

	if(!pMen) throw bad_alloc();

	*(static_cast<int*>(pMen)) = signatrue;		// 在对象头存放signature

	// 找到分配块的最后两个字节，并写入signatrue
	*(reinterpret_cast<int*>(static_cast<Byte*>(pMem) + realSize - sizeof(int))) = signatrue;

	// 返回原本申请大小对象实际指针位置，在开头signature后的第一个字节
	return static_cast<Byte*>(pMem) + sizeof(int);
}
```
例子中的主要缺陷是没有遵循`operator new`的原则，如应该有一个循环调用`new-handling`的行为，直到所有尝试都失败，具体可以看条款51。

`对齐(alignment)`的问题，因为`C++`底层的代码必然接触到操作系统、甚至硬件相关的这样的细节，重载`operator new`可以做到对齐的修正，提高运行速度。

一般需要修改默认的`operator new`和`operator delete`的理由如下:
- 为了检测运用错误。
- 为了收集动态分配内存的使用统计信息。
- 为了增加分配和归还的速度。
- 为了降低缺省内存管理器带来的额外空间开销。
- 为了弥补缺省分配器的非最佳对齐(suboptimal alignment)。例如指针值在32位系统是4倍数等。
- 为了将相关对象成簇集中。减少`缺页异常(Page Faults)`。
- 为了获得非传统的行为。

## 条款51:编写new和delete时固守常规

先给出结论:
- `operator new`应该内含一个无穷循环，并在其中尝试分配内存，如果它无法满足内存需求，就该调用`new-handler`。它也应该有能力处理`0 bytes`申请。Class专属版本则应该处理“比正确大小更大的(错误)申请”。
- `operator delete`应该在收到null指针时不做任何事。Class专属版本则还应该处理“比正确大小更大的(错误)申请”。

C++规定，即便要求客户要求`0 bytes`，`operator new`也会返回一个合法指针。一份常规的`non-member operator new`伪码:
``` c++
void* operator new(std::size_t size) throw(std::bad_alloc)	// 定制的函数可以接受额外的参数
{
	using namespace std;

	if(size == 0)			// 0-byte提为1-byte请求
	{
		size = 1;
	}

	while(true)				// 直到申请成功为止，或者程序崩溃
	{
		尝试分配 size bytes;

		if(成功分配)
		{
			return 成功分配的指针值;
		}

		// 分配失败
		new_handler globalHandler = set_new_handler(0);

		set_new_handler(globalHandler);

		if(globalHandler)
		{
			(*globalHandler)();
		}
		else
		{
			throw std::bad_alloc();
		}
	}
}
```
如果是Class专属版的`operator new`:
``` c++
class Base
{
public:
	static void* operator new(std::size_t size) throw(std::bad_alloc);
	...
};

class Derived : public Base 	// Derived并没有声明自己的operator new
{...};

Derived* pd = new Derived;
```
那么问题就很明显了，`Derived`的大小大于等于`Base`，这样重载的`operator new`执行出来的`Derived`的尺寸是正确的吗？

虽然很怪异，但是如果`Base::operator new`这样实现:
``` c++
void* Base::operator new(std::size_t size) throw(std::bad_alloc)
{
	if(size != sizeof(Base))		// 尺寸匹配不上
	{
		return ::operator new(size);	// 交给标准的operator new处理
	}
	...					// 正常流程
}
```
把责任甩开可以解决问题。

如果重载`operator new[]`的话，考虑到`Base`的子子孙孙，`bytes % sizeof(Base)`的值就不一定为零了，所以重载的时候要考虑的问题还真不少。

接下来就是`operator delete`的问题，首先记住“删除null指针永远安全”，如下一份`non-member operator delete`的伪码:
``` c++
void operator delete(void* rawMemory) throw()
{
	if(rawMemory == 0) return;		// 不处理空指针

	...				// 剩余的delete动作
}
```
同样，考虑到`Base`的子子孙孙，`Base`专属版的operator delete`应该这么实现:
``` c++
class Base
{
public:
	static void* operator new(std::size_t size) throw(std::bad_alloc);
	static void operator delete(void* rawMemory, std::size_t size) throw();
	...
};

void Base::operator delete(void* rawMemory, std::size_t size) throw()
{
	if(rawMemory == 0) return;

	if(size != sizeof(Base))		// 甩锅给标准版operator delete
	{
		::operator delete(rawMemory);
		return;
	}

	...		// 剩余的归还动作。
}
```
注意，如果没有`virtual`关键字修饰的析构函数，`operator delete`也不会接收到正确的`size`。所以在一个继承里面，`virtual`关键字是必须要考虑的。

## 条款52:写了placement new也要写placement delete

`placement new`和`placement delete`应该是更少见的语法了。什么是`placement new`:
``` text
如果operator new接受的参数除了一定会有的那个size_t之外还有其他，这边是所谓的placement new。
```
有很多版本的`placement new`，其中最为有用的一个是“接受一个指针指向对象该被构造之处”:
``` c++
void* operator new(std::size_t, void* pMemory) throw();
```
同样，如果接受额外参数的`operator delete`也成为`placement delete`。原书上给出了加了日志功能的`placement new/delete`的例子:
``` c++
class Widget
{
public:
	...
	static void* operator new(std::size_t, std::ostream& logStream) throw(std::bad_alloc);

	static void operator delete(void* pMemory) throw();

	static void operator delete(void* pMemory, std::ostream& logStream) throw();

	...
};
```
`placement new`一定要有对应版本的`placement delete`，若`placement new`过程抛出了异常，需要恢复而调用`operator delete`时，是寻找参数匹配的`placement delete`，假如没有提供的话，那么就不会执行恢复步骤产生内存泄漏。例如上述代码的成对的`placement new/delete`。当:
``` c++
Widget* pw = new(std::cerr) Widget;
```
发生异常，除了原有的log输出到`std::cerr`中，因为具有参数匹配的`placement delete`被调用，因而不会有泄漏的问题。

还有，写了`placement new`就会产生名字掩盖问题:
``` c++
class Base
{
public:
	...
	static void* operator new(std::size_t size, std::ostream& logStream) throw(std::bad_alloc);
	// 这里只有placement new而没有operator new，global operator new被掩盖了
};

Base* pb = new Base;		// 调用失败，正常的operator new被掩盖了

Base* pb = new (std::cerr) Base;		// OK
```
同理:
``` c++
class Derived : public Base
{
public:
	...
	static void* operator new(std::size_t size) throw(std::bad_alloc);

	...
};

Derived* pd = new (std::clog) Derived;		// 错误，Base中的placement new被掩盖了

Derived* pd = new Derived;					// OK
```
所以在写自己的`placement new/delete`时，要考虑该类需不需要一般形式的`operator new`。

即是，声明`placement new/delete`时，不要忘记了这样会掩盖正常版本的`operator new/delete`。