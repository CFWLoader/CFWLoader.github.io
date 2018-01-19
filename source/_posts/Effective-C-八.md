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

该条款涉及到的知识是`作用域(Scopes)`，在介绍前面条款的时候，笔者也引用到了作用域的部分知识，贴上书上的代码作为例子:
``` c++
int x;					// 全局变量
void fun()
{
	double x;			// 局部变量，此时在fun整个作用域内的x都是该变量
	std::cin >> x;		// 对x进行一些操作
}
```
在经过编程训练之后，笔者会把`fun`函数内变量`x`和全局变量`x`视作不同的变量，即便它们拥有相同的名字。类比成现实的话，就是一个名字可以指的是人，也可以是狗，即便是指人，也有可能是存在多个同名的人，这时候就需要提供更多的信息来对应了。用编译原理的知识来表达的话，就是根据`上下文(Context)`来推导。

至于推导的规则，一般是从当前的作用域内查找，失败就扩大作用域，直到整个上下文环境都查找失败为止。

同样，用书上的例子解释该条款:
``` c++
class Base{
private:
	int x;
public:
	virtual void mf1() = 0;
	virtual void mf1(int);
	virtual void mf2();
	void mf3();
	void mf3(double);
	...
};

class Derived : public Base{
public:
	virtual void mf1();
	void mf3();
	void mf4();
	...
};
```
使用子类:
``` c++
Derived d;
int x;
...
d.mf1();		// OK,使用Derived::mf1
d.mf1(1);		// 错误，Derived::mf1遮掩了所有的Base::mf1
d.mf2();		// OK，Base::mf2
d.mf3();		// OK，Derived::mf3
d.mf3();		// 错误，Derived::mf3遮掩了所有的Base::mf3
```
注意两个mf1都是`virtual`的，笔者在自己做实验做验证的时候的设想是无参版本的mf1被复写了，所以能够正常调用;而带int参数的mf1则因为声明了virtual并且在Base中具有了实现，理应调用Base中的实现，然而实验结果证明了无论函数是否声明为`virtual`，只要在继承类中声明了与基类同名的函数，则基类中所有的同名函数都会被覆盖。

但是笔者同时也在实验中注意到，设置如下代码通过编译:
``` c++
Derived d;
int x = 10;
d.mf1();
d.mf2();
d.mf3();   
```
使用`-S`参数以及`c++filt`工具查看生成的汇编，则会发现编译器为以下:
``` asm
Base::mf1(int)
Base::mf2()
Derived::mf1()
Derived::mf3()
```
这些成员函数生成了汇编码，其它成员函数因为编译器优化探测到没有使用而没有被生成。

有趣的是，生成的汇编码中对代码的解释有`vtable`的描述(经过人工处理):
``` asm
vtable for Derived:
        .quad   Derived::mf1()
        .quad   Base::mf1(int)
        .quad   Base::mf2()
vtable for Base:
        .quad   __cxa_pure_virtual	；这应该是Base::mf1()
        .quad   Base::mf1(int)
        .quad   Base::mf2()
```
其中`Derived`类中的`vtable`竟然存在`Base::mf1(int)`这个描述，但是在代码主体中却没有，笔者也不了解为什么会这样，但是这始终只是一个描述性的部分，决定行为的还是代码主体，鉴于笔者当前的水平也只能先挖出这个奇异点了。

接着使用GDB中的`info fuctions`命令进行运行时查看各个类下的成员函数:
``` text
void Derived::Derived();
void Derived::mf1();
void Derived::mf3();

void Base::Base();
void Base::mf1(int);
void Base::mf2();
```
结果也对上了汇编码。

说了这么多，解决这个问题的方法有两个，一个是:
``` c++
class Derived : public Base{
public:
	using Base::mf1;
	using Base::mf3;
	virtual void mf1();
	void mf3();
	void mf4();
	...
};
```
也就是显式声明让编译器把`Base`中被遮掩的函数在该子类中暴露，或者使用`转交函数(forwarding function)`:
``` c++
class Base{
public:
	virtual void mf1() = 0;
	virtual void mf1(int);
	...
};

class Derived : private Base{	// private继承使得只有这个直系子类能够调用Base中public的部分，Derived的子类则不可以访问Base中的任何部分
public:
	virtual void mf1()
	{							// 转交函数
		Base::mf1();			// 隐式成为了inline
	}
	...
};
```
但是在`Derived`的实例中调用`Base::mf1(int)`仍然是错误的，因为`遮掩`仍在存在。

## 条款34:区分接口继承和实现继承

如果读者使用过`Java`的`interface`关键字做过一些实验的话，并对`设计与实现分离`这个原则有深刻的理解的话，这个条款应该可以跳过了。不过`接口`这个概念在`C++`中也可以很简单地就模拟出来:
- 所有成员都是public
- 没有成员变量
- 所有成员函数都是纯虚函数

根源上来讲`interface`只是类的一种特殊情况，无非是`Java`在语法方面做了限制。正是这样的约束，提高了`Java程序`的质量下限，不需要了解为什么有`interface`关键字，只知道`Java`可以单继承，多实现的语法就行了。顺带一提，`Java 8`开始也支持接口中的方法写`缺省实现`了。

关于该条款，笔者认为书上的飞机例子就够用了:
``` c++
class Airport
{...};

class Airplane{
public:
	virtual void fly(const Airport& destination) = 0;	// 不提供fly的缺省实现，防止编译器让子类自动继承实现
protected:
	void defaultFly(const Airport& destination);		// 假如子类的fly用的缺省方式，则子类的fly实现显式调用该函数
};

void Airplane::defaultFly(const Airport& destination)
{
	// 飞机飞向目的地的缺省行为
}

class ModelA : public Airplane
{
public:
	virtual void fly(const Airport& destination)
	{
		defaultFly(destination);	// 缺省飞行方式
	}
	...
};

class ModelB : public Airplane
{
public:
	virtual void fly(const Airport& destination)
	{
		defaultFly(destination);	// 缺省飞行方式
	}
	...
};

class ModelC : public Airplane
{
public:
	virtual void fly(const Airport& destination)
	{
		// 该型号有别的飞行方式
	}
	...
};
```
如果还是想懒又优雅，可以继续给纯虚函数提供缺省实现:
``` c++
class Airport
{...};

class Airplane{
public:
	virtual void fly(const Airport& destination) = 0;	// 提供fly的缺省实现，子类则需要在不使用缺省方式时显式提供自身的实现
};

void Airplane::fly(const Airport& destination)
{
	// 飞机飞向目的地的缺省行为
}

class ModelA : public Airplane
{
public:
	virtual void fly(const Airport& destination)
	{
		Airplane::fly(destination);	// 缺省飞行方式
	}
	...
};

class ModelB : public Airplane
{
public:
	virtual void fly(const Airport& destination)
	{
		Airplane::fly(destination);	// 缺省飞行方式
	}
	...
};

class ModelC : public Airplane
{
public:
	virtual void fly(const Airport& destination)
	{
		// 该型号有别的飞行方式
	}
	...
};
```
这些例子呈现`设计与实现分离`这个原则不够明显，更体现不出其威力。事实上这个原则有点`解耦`的意味，如果读者有接触过`Java`项目开发的话，应该很熟悉动不动就一个`interface`和一个对应的缺省实现。例如项目中的某个实现发现了可以性能改进的地方，但是不需要改进接口，如果接口和实现放在一起的话，那么意味着这个类需要整个编译一遍;如果采用了接口实现分离，只需要重新编译发生改动的类就可以了。极端一些，假设大片实现都需要更新，而接口不需要更改，这时候编译量的差就很客观了。

笔者入门时也不是太理解这个条款，这里只是提供一个原则，需要经过实践才能体验到这样做带来的好处。

这个条款主要是让读者能够区分`接口继承`以及`实现继承`，读者可以结合`Java`的`interface`理解。

## 条款35:考虑virtual函数以外的其他选择

该条款涉及到:
- [Template Method](https://en.wikipedia.org/wiki/Template_method_pattern)
- [Strategy](https://en.wikipedia.org/wiki/Strategy_pattern)

这两个[设计模式](https://en.wikipedia.org/wiki/Software_design_pattern)，这跟软件开发的范畴十分相关，而且看起来十分相像，不过区分一下`模板方法`是由外部来决定一个具体的实现;而`策略`更多的是由`对象自身的状态`来决定使用什么实现，[wiki](https://www.wikipedia.org/)上给了中国和美国交税的方法不同，笔者根据自身理解写一个例子:
``` c++
// Template Method设计模式样例
class Tax
{
public:
	virtual double payTax() = 0;	// 声明template method
	...
};

class ChnTax : class Tax
{
public:
	virtual double payTax()
	{
		// 一些具体的代码
	}
	...
};

class UsaTax : class Tax
{
public:
	virtual double payTax()
	{
		// 一些具体的代码
	}
	...
};

...

// 使用样例，由外部来决定使用什么实现
Tax tax = new ChnTax();

tax.payTax();				// 交中国税

...

// tax = new UsaTax();

tax.payTax();				// 交美国税

...
```
对比`策略模式`模式的构想:
``` c++
class Tax
{
public:
	enum Country{CHN, USA, ...};	// 对象所处的状态由具体的环境决定，在这个例子里面的状态就是所处国家
	
	Tax(Country ct) : country(ct){}	// 对象在被实例化时就应该赋予状态，或者专门设定一个状态变化的函数

	void payTax()
	{
		if(country == Country::CHN)	// 根据对象的状态采用对应的实现
		{
			chnPayTax();
		}
		else if(country == Country::USA)
		{
			usaPayTax();
		}
		...
	}
private:
	void chnPayTax()				// 这些实现声明为私有
	{
		// 具体的代码
	}

	void usaPayTax()
	{
		// 具体的代码
	}

	Country country;
};

// 使用样例，外部赋予对象初始状态，或者对象自身就有状态

Tax tax = new Tax(Tax::Country::CHN);		// 中国对象实例

tax.payTax();							// 内部已经有了状态，根据自身状态调用对应的策略

... 

tax = new Tax(Tax::Country::USA);		// 或者是一个美国状态实例

...
```
这是两种设计模式的介绍，书中给出的例子虽然跟笔者自己写的例子不是很相像，但是其核心思想是一样的。这也是初学者到进阶的其中一步，也就是遵循某种思想设计出具体的算法，比根据算法的抽象描述或者伪代码实现需要有更加进阶的编程能力要求。

但是笔者经过再读这个模板方法的例子，结合后文的修改:
``` c++
class GameCharacter
{
public:
	int healthValue() const				// 该类子类不重新定义这个函数
	{
		...
		int retVal = doHealthValue();	// 调用实际的实现版本
		...
		return retValue;
	}
	...
protected:					// 最初是private，但是为了让子类能够正确感知到这个实际的实现并可以重新定义，protected比较合理
	virtual int doHealthValue() const
	{
		...			// 缺省的实现
	}

};
```
笔者认为，这样做是把`模板方法`实现，使得调用的接口固定，对使用者友好，这样使用者就可以认为这样的方法只有一个，而不需要考虑多态的问题(虽然内部实现依靠了多态);而对于实现提供者，只需要处理实际实现的函数即可，而不需要关注接口的问题。

而接下来用`策略注入`(该名词为笔者根据)的方法实现`策略模式`。书上第一版的实现用的函数指针，这样的话就无法处理有参数变化的策略函数了，例如书上的:
``` c++
typdef int(*HealthCalcFunc)(const GameCharacter);

GameCharacter::GameCharacter(HealthCalcFunc* hcf = defaultHealthFunc) : healthFunc(hcf);
{}
```
若日后计算的方法发生了改变使得这个计算函数的接口发生了变化，那么接收`策略`的构造器也会跟着出问题。在作者那个时期还没有`C++11`，所以作者用的`boost::tr1`，采用了`tr1::function`代替`函数指针`类型，而且可以使用`bind`解决上述提及的接口变化问题。

书上的例子比较复杂，也比较难直接看出是策略模式的一种实现，笔者还是用上面计算税的例子:
``` c++
class Tax
{
public:
	std::function<double()> CalculateTaxFun;		// 声明没有参数，返回为double的函数类型

	explicit Tax(CalculateTaxFun calcFun) : calcStrategy(calcFun){}	// 构造器接受计算税的策略

	void payTax()			// 交税函数，不过在这个例子的用处是调用注入的计算税的策略
	{
		double taxVal = calcStrategy();		// 调用注入进来的计算税的策略获得应缴税的值
		...					// 处理税务
	}
private:

	CalculateTaxFun calcStrategy;			// CalculateTaxFun的实例
};

enum State{CHN, USA, ...};			// 状态，其实是上个例子的国家

double calculateTax(State state)
{
	double taxVal;

	if(state == State::CHN)
	{
		...						// 在中国，按照基本法算稅
	}
	else if(state == State::USA)
	{
		...						// 美国
	}
	...							// 其他情况

	return taxVal;
}

....
// 使用样例，这时候策略的决定交由外部
Tax::CalculateTaxFun taxCalc(std::bind(calcStrategy, State::CHN, std::placeholders::_1));	// 定义一个中国计算税的策略
/*
 * 上述的过程就是，用std::bind把calcStrategy这个函数的第一个参数绑定为State::CHN，
 * 返回一个std::function<double()>的实例用于初始化一个策略
 * 注意即便声明了using namespace std::placeholders,直接用_1也会直接出错，
 * 所以只能打全称
 */
Tax tax(taxCalc);		// 将计算税的策略注入到税实例中

tax.payTax();			// 交税，计算税的策略已经在里面了

Tax::CalculateTaxFun usaTaxCalc(std::bind(calcStrategy, State::USA, std::placeholders::_1)); // 美国的计算税策略

...
```
上面的例子讲述了用`std::function`实现的策略模式，读者可以细读上个例子，理解策略模式的思想(上面例子没有体现出对象根据自身状态选择策略，而是交给了外部)，以及这个模式的一般实现。