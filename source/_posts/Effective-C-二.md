---
title: Effective C++(二)
date: 2017-12-07 15:29:23
tags:
- C++
- C++11
- Effective C++
categories: C++技术
---

上一篇{% post_link Effective-C-一 Effective C++(一) %}介绍了认知C++方面的内容，这一部分将讨论的是`构造`、`析构`、`赋值`方面的问题。

## 条款5:了解C++默默编写并调用那些函数

这一条款主要是讲述C++编译器为类合成`构造函数`、`析构函数`方面的知识，其实应该结合《深度探索C++对象模型》来讲述会更加清楚，但是这一部分展开介绍实在会增加篇幅，笔者将在日后的对象模型博客中详细地介绍。

首先书上给的是C++编译器会如何处理空类，其中手动编写空类的定义:
``` c++
class Empty
{};
```
编译器会自动生成一些成员函数，这时你的类定义可能会看起来是这样:
``` c++
class Empty
{
public:
	Empty(){}
	Empty(const Empty& rhs){}
	~Empty(){}
	Empty& operator=(const Empty& rhs){}
};
```
只有代码中用到这一部分的函数的时候，编译器才会去处理这些成员函数。为了查看编译器是不是真的会实施这个行为，用`-S`选项并分析其生成的汇编码({% post_link 使用Linux工具分析g-生成的代码 使用Linux工具分析g++生成的代码 %})。

首先编写一个没有用到`Empty`的`main`函数:
``` c++
int main(int argc, char* argv[])
{
	return 0;
}
```
查看只为`main`生成的汇编码:
``` asm
main:
        pushq   %rbp
        movq    %rsp, %rbp
        movl    %edi, -4(%rbp)
        movq    %rsi, -16(%rbp)
        movl    $0, %eax
        popq    %rbp
        ret
```
更改`main`函数为:
``` c++
int main(int argc, char* argv[])
{
	Empty e1;
	return 0;
}
```
几乎也不会有什么变化:
``` asm
main:
        pushq   %rbp
        movq    %rsp, %rbp
        movl    %edi, -20(%rbp)
        movq    %rsi, -32(%rbp)
        movl    $0, %eax
        popq    %rbp
        ret
```
只是传入到main中参数发生了一些细微的变化，并没有看到`构造器`以及`call`行为发生。

那将`main`改成了书上的样子:
``` c++
int main(int argc, char* argv[])
{
	Empty e1;

	Empty e2(e1);

	e2 = e1;

	return 0;
}
```
可惜的是，这样的代码也没有促使编译器合成构造器等函数，不过考虑到这本书已经有出版时间稍微有点间隔，以及笔者使用的是查看汇编码而不是其他编译器产生的中间代码，不过汇编码可以算是比其他中间代码更有证明力的部分。既然没合成，要么就是编译器已经可以针对这样的空类优化了，要么就是作者方便读者理解而设立出来的架空代码。

不过在此笔者继续探索，假设定义了一个类，其成员非空，先不为其编写构造函数等:
``` c++
class Empty
{
private:
	int mem1;
};
```
首先也是`main`发生上述三个行为时的时候，生成的汇编码中也没有产生成员函数，倒是`e2 = e1`时的汇编码:
``` asm
main:
        pushq   %rbp
        movq    %rsp, %rbp
        movl    %edi, -20(%rbp)
        movq    %rsi, -32(%rbp)
        movl    -4(%rbp), %eax			; Empty e2(e1)
        movl    %eax, -8(%rbp)
        movl    -4(%rbp), %eax			; e2 = e1
        movl    %eax, -8(%rbp)
        movl    $0, %eax
        popq    %rbp
        ret
```
编译器多多少少处理了赋值行为，但是也没有明显地为其生成了一个专门的函数。

如果将`Empty`类的构造函数、赋值函数以及析构函数给予定义:
``` c++
class Empty
{
public:
        Empty() : mem1(0){}
        Empty(const Empty& rhs) : mem1(rhs.mem1) {}
        Empty& operator=(const Empty& rhs)
        {
                mem1 = rhs.mem1;
        }
private:
        int mem1;
};
```
在`main`函数没有任何行为的时候，编译器也不会为这个`Empty`生成任何函数，而当开始需要构造函数的时候:
``` c++
int main(int argc, char* argv[])
{
        Empty e1;
        return 0;
}
```
编译器则为其生成了:
``` asm
Empty::Empty():
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        movq    -8(%rbp), %rax
        movl    $0, (%rax)
        nop
        popq    %rbp
        ret

main:
        pushq   %rbp
        movq    %rsp, %rbp       
        subq    $32, %rsp
        movl    %edi, -20(%rbp)
        movq    %rsi, -32(%rbp)
        leaq    -4(%rbp), %rax
        movq    %rax, %rdi
        call    Empty::Empty()
        movl    $0, %eax
        leave
        ret       
```
明显地生成了该类的构造函数，也有`call`调用函数的行为，再把`main`修改:
``` c++
int main(int argc, char* argv[])
{
        Empty e1;

        Empty e2(e1);

        e2 = e1;

        return 0;
}
```
则生成了:
``` asm
Empty::Empty():
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        movq    -8(%rbp), %rax
        movl    $0, (%rax)
        nop
        popq    %rbp
        ret

Empty::Empty(Empty const&):
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        movq    %rsi, -16(%rbp)
        movq    -16(%rbp), %rax
        movl    (%rax), %edx
        movq    -8(%rbp), %rax
        movl    %edx, (%rax)
        nop
        popq    %rbp
        ret

Empty::operator=(Empty const&):
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        movq    %rsi, -16(%rbp)
        movq    -16(%rbp), %rax
        movl    (%rax), %edx
        movq    -8(%rbp), %rax
        movl    %edx, (%rax)
        nop
        popq    %rbp
        ret

main:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $32, %rsp
        movl    %edi, -20(%rbp)
        movq    %rsi, -32(%rbp)
        leaq    -4(%rbp), %rax
        movq    %rax, %rdi
        call    Empty::Empty()
        leaq    -4(%rbp), %rdx
        leaq    -8(%rbp), %rax
        movq    %rdx, %rsi
        leaq    -4(%rbp), %rdx
        leaq    -8(%rbp), %rax
        movq    %rdx, %rsi
        movq    %rax, %rdi
        call    Empty::Empty(Empty const&)
        leaq    -4(%rbp), %rdx
        leaq    -8(%rbp), %rax
        movq    %rdx, %rsi
        movq    %rax, %rdi
        call    Empty::operator=(Empty const&)
        movl    $0, %eax
        leave
        ret
```
在这个例子看来，现在的编译器不会默默地为类合成构造器等函数，并且是在已经使用到的情况下也只是产生了对应的行为，但是不会合成相应的函数，从某种角度来看，原书的说法还是正确的，不过要说明是编译器会合成这些行为，但是不会产生一个专门的函数，起码是在汇编码层面看不到。

## 条款6:若不想使用编译器自动生成的函数，就应该明确拒绝

书上这一条款看起来主要是针对复制构造函数以及赋值操作符，应用场景一般为禁止复制的对象。具体一些的思路就是让复制构造函数以及赋值操作附不可用，但是如果不去声明这些函数的话，编译器会自动合成这些函数。那么手动声明定义这些函数并设置其不可用状态，一般通过声明其为`private`并且不去实现:
``` c++
class Object
{
public:
        ...
private:
        ...
        Object(const Object&);
        Object& operator=(const Object&);
}
```
在`C++11`下，可以这么写:
``` c++
lass Object
{
public:
        Object(const Object&) = delete;
        Object& operator=(const Object&) = delete;
        ...
private:
        ...
}
```
这样就声明了这两个函数是删除并且不可用的。

如果实际应用中很多对象都是禁止复制的话，一般是声明一个不可复制的基类:
``` c++
class Uncopyable
{
public:
        Uncopyable() {}
        Uncopyable(const Uncopyable&) = delete;
        Uncopyable& operator=(const Uncopyable&) = delete;
        ~Uncopyable() {}
private:
};
```
这样在写新的不可复制的类是就可以写:
``` c++
class Object : private Uncopyable
{
        ...
}
```
那么如果使用者对该类实例进行了复制操作，编译器会尝试生成该类的复制构造函数或者赋值操作符时会以失败告终并告知编译失败。