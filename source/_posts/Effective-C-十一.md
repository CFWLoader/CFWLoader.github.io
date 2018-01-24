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

(待续)