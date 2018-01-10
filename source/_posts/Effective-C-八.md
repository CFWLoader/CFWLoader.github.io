---
title: Effective C++(八)
date: 2018-01-04 16:19:54
tags:
	- C++
	- C++11
	- Effective C++
categories: C++技术
---

这里开始进入到`继承与面向对象设计`主题。虽然这是`Effective C++`书上的内容，但是笔者认为这部分的知识不仅仅局限于语言，毕竟从上个世纪`面向对象`这个概念被提出之后就被普遍地实践，`C++`的前身也是因为要适应这股潮流才被创造出来的。

笔者认为`面向对象`这个思想更关注的是代码的逻辑结构符不符合人的直观感受，或者是经验感受，接下来开始介绍这个主题下的条款吧。

## 条款32:确定你的public继承塑模出is-a关系

省略书上开头的例子，第一个涉及到的有关OOP的概念是`里斯科夫替换原则(Liskov Substitution Principle,LSP)`,简称[里式替换原则](https://en.wikipedia.org/wiki/Liskov_substitution_principle)，这里再扩展一下`SOLID`原则，有兴趣的读者可以深入了解`https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)`。LSP主要观点是“派生类（子类）对象能够替换其基类（超类）对象被使用”。假设有个继承关系`A`是`B`的父类，`A`可以使用的场景，那么`B`一定可以应用在该场景中;反之不一定成立。

书上随后给出了`学生`和`人`这个基本的`is-a`关系，论证了`public继承`可以符合这个关系的实现。而后给出了“企鹅是一种鸟”，“鸟会飞”这两个基本事实，事实上应该说”鸟一般会飞“这个描述更为准确一些。用public继承描述了这样的关系:

``` c++
class Bird{
public:
	virtual void fly();			// 鸟”一般“可以飞
	...
};

class Penguin : public Bird{	// 企鹅是一种鸟
	...
};
```
然后就违背了“企鹅不会飞”这个事实。为了解决这个问题，鸟首先被分为了`可以飞`和`不会飞`的两个类，然后企鹅继承`不会飞`的鸟类，problem solved:
``` c++
class Bird{
	...
};

class FlyingBird : public Bird{
public:
	virtual void fly();
	...
};

class Penguin : public Bird{
	...
};
```
这样便开始增加了继承体系的复杂性。这个复杂性笔者认为是来自于业务，因为构造这样的继承体系需要生物分类的基本事实，除非是为了开发一个覆盖现已发现的所有生物的相关的业务系统，否则继承体系不应该100%还原生物分类体系，并且还有一些分类不清的生物也增加了设计这个继承体系的难度，解决这个复杂度笔者认为不是数据结构设计上的问题，更多的是相关业务开发人员对该业务领域的认知能力。

书上给出了一个针对解决企鹅不会飞这个特例的解决方案:
``` c++
classs Penguin : public Bird{
public:
	virtual void fly(){
		throw std::exception("Attempt to make a penguin fly!");		// 原书用一个自己实现的error函数
	}
};
```
无论是原书用自己实现的一个error函数，还是笔者自行修改的抛出异常，本质上都是运行时报错。这个对代码的使用人员太不友好，也只能算是下下策。

还有办法就是取消鸟类里面的飞的函数，直到某个层次开始分类出能够飞或者不会飞的类属，这样在编译器就可以阻止使用者的错误的使用方式。但是这样问题就回到前面提到的继承体系复杂度的问题。

书上继续给出了`矩形`与`正方形`的例子，贴上书上的代码:
``` c++
class Rectangle{
public:
	virtual void setHeight(int newHeight);
	virtual void setWidth(int newWidth);
	virtual int height() const;
	virtual int width() const;
	...
};

void makeBigger(Rectangle& r)				// 	增加r的面积
{
	int oldHeight = r.height();
	r.setWidth(r.width() + 10);				//	增加r的宽10个单位
	assert(r.height() == oldHeight);		//	笔者不理解为什么要断言这个变化，这个约束也太严格了
}
```
上述代码保证了只改变矩形的宽，利用断言确定高度不会被改变，`正方形`的出场带来了一个难题:
``` c++
class Square : public Rectangle{...};
Square s;
...
assert(s.width() == s.height())				//	对正方形的正常约束检查
makeBigger(s);								//	1.问题来了
assert(s.width() == s.height())				//	对于正方形，这个约束应该成立
```
一个稍微认真考虑过`正方形`的设置长宽函数实现的编程人员应该会下意识的让`setHeight`和`setWidth`的行为一致，这个就会马上与`makeBigger`中的断言发生冲突。或者说忘记了正方形的性质，而导致正方形的长宽一致的断言失败。

书上没有讨论是谁的错，但是笔者认为是`makeBigger`函数的不合理导致的:
1. 既然是改变面积的功能，从OOP角度考虑可以设计为图形的成员函数，并声明为virtual，赋予子类应对场景的能力;
1. 也是从OOP角度来考虑，“makeBigger”这个函数对功能的描述模棱两可，根据行为来看，设置为“makeWidder”更加容易让人读懂。

所有的例子强调了`public继承`就要实现好`is-a`这个模型，使得其符合里氏替换原则，不然要么就是代码有问题，要么就是本身需要实现的模型就有问题。

此外还有常见的`has-a`(有一个)以及`is-implemented-in-terms-of`(根据某物实现出)这个关系，分别在条款38和39中介绍。

## 条款33:避免遮掩继承而来的名称

(待续)