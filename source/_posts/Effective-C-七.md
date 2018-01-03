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

正如这个系列刚开始，笔者探讨`C with class`的问题时提出的场景一样，如果程序动不动就要处理错误，那么为了处理这些错误，可以写出比原来为了满足需求而实现的代码长好几倍的代码。笔者尚未经历过`C++`老项目的维护，但是从其他项目的经验来看，异常处理这一部分并不会特别关注。一个是由于各种框架的出现，使得底层设施这种复用性高，又是关键部分的代码已经得到了很好的异常处理，而使用者仅仅处理业务层的代码，框架一般也提供了事务机制，这就使得使用者只需要在发生错误的时候回滚就行了;二是异常本来就不应该是频发的程序场景，如果一个程序中运行频繁地出现异常，要么就是运行环境太差，要么就是编程人员基础太差。

不过为了满足程序的健壮性，一些异常处理仍然是必须的，但是要处理到何种程度，笔者也不能给出评价标准，毕竟笔者也见过没有任何异常处理的程序稳定运行的案例。如果是在实际项目中，功能的实现是第一目标，异常处理是辅助手段。

书上定义了带有`异常安全性`的函数该有的行为，即异常被抛出的时候:
1. 不泄漏任何资源。
1. 不允许数据败坏。

为了满足这两个行为，笔者给出socket编程中的经典例子:
``` c++
// Initialize server parameters.
listenFileDescriptor = socket(AF_INET, SOCK_STREAM, 0);

bzero(&serverAddr, sizeof(serverAddr));

serverAddr.sin_family = AF_INET;
serverAddr.sin_addr.s_addr = htonl(INADDR_ANY);
serverAddr.sin_port = htons(SERVER_PORT);
// Parameters set. Injecting the server parameters to system.

bind(listenFileDescriptor, (sockaddr*) &serverAddr, sizeof(serverAddr));

listen(listenFileDescriptor, LISTEN_QUEUE);
// Server initialization finished.
```
里面的`socket`、`bind`、`listen`函数都有可能因为系统资源的不足而初始化失败，虽然这里没有用到`C++`的异常语法，但是也能代表为了满足异常处理要怎么写:
``` c++
// Initialize server parameters.
listenFileDescriptor = socket(AF_INET, SOCK_STREAM, 0);

if(listenFileDescriptor < 0)
{
	//	可以输出一些错误提示。
	return ERROR_CODE_SOCKET;	//	或者调用exit等。
}

bzero(&serverAddr, sizeof(serverAddr));

serverAddr.sin_family = AF_INET;
serverAddr.sin_addr.s_addr = htonl(INADDR_ANY);
serverAddr.sin_port = htons(SERVER_PORT);
// Parameters set. Injecting the server parameters to system.

int ret_val = bind(listenFileDescriptor, (sockaddr*) &serverAddr, sizeof(serverAddr));

if(ret_val != 0)
{
	// 	错误提示。
	close(listenFileDescriptor);	// 	因为初始化失败，要释放资源。
	return ERROR_CODE_BIND;			// 	同理可以采用exit。
}

ret_val = listen(listenFileDescriptor, LISTEN_QUEUE);

if(ret_val != 0)
{
	// 	错误提示。
	close(listenFileDescriptor);	// 	因为初始化失败，要释放资源。
	return ERROR_CODE_LISTEN;			// 	同理可以采用exit。
}
// Server initialization finished.
```
为了作出上述的两个保证，代码不得不变长。

异常安全函数(Exception-safe functions)提供以下三个保证之一:
1. `基本承诺`:如果异常被抛出，程序内的任何事物仍然保持在有效状态下。没有任何对象或数据结构会因此而败坏，所有对象都处于一中内部前后一致的状态。然而程序的现实状态(exact state)恐怕不可预料。
1. `强烈保证`:如果异常被抛出，程序状态不改变。如果函数成功则完全成功，如果失败则回滚，具体例子可以看看上面`Server Socket`初始化。
1. `不抛掷(nothrow)保证`:承诺绝不抛出异常，因为它们总是能够完成它们原先承诺的功能。作用于内置类型身上的所有操作都提供nothrow保证。

看完了介绍的概念，笔者认为书上除了例子之外值得做笔记的地方就是`copy and swap`，具体方法是在作用于一个对象前先保存其副本，中途任何失败都放弃修改，返回修改前的对象副本。

## 条款30:透彻了解inlining的里里外外

关于该条款，笔者认为较为重要的是其:
``` text
inline只是对编译器的一个申请，不是强制命令。
```
也就是说给一个函数加上`inline`关键字修饰时，编译器视该修饰为一个建议，不一定会对其进行代码展开。

同时滥用`inline`会导致代码膨胀，甚至造成比不加`inline`时的效率更低的可能。

书上建议将大多数inlining限制在小型、被频繁调用的函数上。因为被`inline`牵涉的函数的调用者在进行二进制升级的时候也会被牵连着更行。

## 条款31:将文件间的编译依存关系降至最低

(待续)