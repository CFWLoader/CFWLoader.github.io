---
title: 使用Linux工具分析g++生成的代码
date: 2017-11-21 22:30:05
tags:
	- Linux
	- C++
	- gdb
categories: C++技术
---

近来想用实际代码实验来验证《Effective C++》、《深度探索C++对象模型》中的知识，通过反汇编等手段查看编译器生成的代码，原本想着看能不能设置好编译参数，使得编译器可以输出书本中的中间代码，可惜的是暂时还没找到，还一度以为只能通过强行分析汇编码。经过一周的摸索，总算弄出了一个可以接受的方案，这篇博客主要说明几个命令之间生成代码的信息。

## 环境支持

- Linux
- g++
- gdb
- nm
- objdump
- c++filt
- readelf

其中`g++`本身就可以通过添加`-S`参数就可以生成汇编码了，`nm`命令用来查看生成二进制文件的函数表，`objdump`是查看更多关于二进制文件的信息，`c++filt`命令使用来`demangling`，`readelf`用于读取`ELF`文件信息。

## 目标

《深度探索C++对象模型》中的构造语义章节中提到过，如果有一个`Object`类，而在声明一个实例的时候:
``` c++
Object obj;
```
编译器可能会生成如下的中间代码:
``` c++
Object obj;
Object::Object(&obj);
```
而生成的函数原型是:
``` c++
void Object::Object(Object* const this);
```
此实验的目的就是验证编译器确实是生成了这样的中间代码。结论是目前只有`gdb`能做到，在呈现成果的结果前，也试试用别的命令看看能够尝试到什么样的结果。

## x86_64汇编相关的准备知识

一旦开始做这类C++相关代码分析的工作，汇编是逃不开的，不过现在也基本不用写，会读就可以了。`g++`生成的汇编码是`AT&T`格式的，64位机的条件下，每个涉及到操作数的命令中后面会带有`b`、`w`、`l`、`q`等字母，分别代表操作一个1Byte(8 bit)、2 Byte(16bit)、4 Byte(32bit)以及8 Byte(64bit)。例如:
- `movb %al, %bl`代表把`ax`寄存器的低8bit赋值给`bx`寄存器的低8bit。
- `movw %ax, %bx`代表把`ax`中16bit的值赋值给`bx`寄存器。
- `movl %eax, %ebx`代表把`eax`中32bit的值赋值予`ebx`寄存器。
- `movq %rax, %rbx`代表把`rax`中64bit的值赋予`rbx`寄存器。

x86_64有16个64bit寄存器，分别为%rax，%rbx，%rcx，%rdx，%esi，%edi，%rbp，%rsp，%r8，%r9，%r10，%r11，%r12，%r13，%r14，%r15。笔者所用的`Linux`下的`g++`编译器一般会这样划分寄存器的用途:
- %rax 作为函数返回值使用。
- %rsp 栈指针寄存器，指向栈顶。
- %rdi，%rsi，%rdx，%rcx，%r8，%r9 用作函数参数，依次对应第1参数，第2参数...
- %rbx，%rbp，%r12，%r13，%14，%15 用作数据存储，遵循被调用者使用规则，简单说就是随便用，调用子函数之前要备份它，以防他被修改。
- %r10，%r11 用作数据存储，遵循调用者使用规则，简单说就是使用之前要先保存原值。

## 实验用的代码
``` c++
class Object
{
public:
	Object()
	{}
private:
};

int main(int argc, char* argv[])
{
	Object obj;
	return 0;
}
```
这里需要定义`Object::Object()`，因为编译器在这里不会自动合成`Object`类的构造函数。如果连`Object`类是个空类，没有函数成员，也没有数据成员，编译器会为这个类生成怎样的代码?这个问题也在《深度探索C++对象模型》中提到过，有兴趣者可以通过搜索`空基类优化`关键字获得相关的知识，这里就不展开细说了。

## nm命令

`nm`命令是用于列出目标文件中的符号，在这个实验中用作输出函数签名，查找看能不能输出期望的`void Object::Object(Object * const)`的函数签名。

首先使用
``` bash
$ g++ main.cpp
```
生成目标文件`a.out`。随后:
``` bash
$ nm a.out | c++filt
```
可以看到终端输出(经过处理):
``` code
0000000000400566 T main
0000000000400588 W Object::Object()
0000000000400588 W Object::Object()
```
信息还不少，可惜没有找到期待的`void Object::Object(Object * const)`。

## objdump

不改动上述的`a.out`文件，执行:
``` bash
$ objdump -t a.out | c++filt
```
也是输出了不少信息(经过处理):
``` code
0000000000400566 g     F .text	0000000000000022              main
0000000000400588  w    F .text	000000000000000b              Object::Object()
0000000000400588  w    F .text	000000000000000b              Object::Object()
```
当前只关注能不能找到`void Object::Object(Object * const)`，所以这里也不展开介绍这个命令输出的内容。

## readelf

不改动上述的`a.out`文件，执行:
``` bash
$ readelf -s a.out | c++filt
```
也是输出了不少信息(经过处理):
``` code
55: 0000000000400588    11 FUNC    WEAK   DEFAULT   11 Object::Object()
60: 0000000000400588    11 FUNC    WEAK   DEFAULT   11 Object::Object()
63: 0000000000400566    34 FUNC    GLOBAL DEFAULT   11 main
```
也能获得不少关于符号表中的信息，可惜没有期待的`void Object::Object(Object * const)`。

## 直接查看g++生成的汇编码

还是上述的`C++`代码，不过不是分析编译器生成目标文件，而是通过:
``` bash
$ g++ -S main.cpp
```
生成汇编文件`main.s`，不过需要给这个文件进行一下`demangling`，不然没法看:
``` bash
$ cat main.s | c++filt 
```
会输出(经过处理):
``` code
Object::Object():
	pushq	%rbp
	movq	%rsp, %rbp
	movq	%rdi, -8(%rbp)
	nop
	popq	%rbp
	ret

main:
	pushq	%rbp
	movq	%rsp, %rbp
	subq	$32, %rsp
	movl	%edi, -20(%rbp)
	movq	%rsi, -32(%rbp)
	leaq	-1(%rbp), %rax
	movq	%rax, %rdi
	call	Object::Object()
	movl	$0, %eax
	leave
	ret
```
还是看不到《对象模型》中所说的`void Object::Object(Object * const)`函数签名。不过注意到`Object::Object():`中有一句:
``` asm
movq	%rdi, -8(%rbp)
```
这句汇编的意思是把%rdi寄存器中的值赋予%rbp前移8个字节的内存地址中，即复制了64bit数据。
前面提到:
``` code
%rdi，%rsi，%rdx，%rcx，%r8，%r9 用作函数参数，依次对应第1参数，第2参数...
```
那是不是有什么参数传入到了这个函数过程中了呢?不过我们现在暂时没有办法得知，需要后面提到的`gdb`帮助下才能验证。

## 强力工具gdb

本实验只用到`gdb`的一小部分功能，之前使用搜索引擎的时候，发现都可以用`gdb`调试多线程程序了，命令也挺简单，可以进入某一线程中进行调试。

回到本实验上，使用`gdb`前编译目标文件:
``` bash
$ g++ -g main.cpp
```
添加`-g`是为了让生成的目标文件带有调试信息。

执行:
``` bash
$ gdb a.out
```
进入调试。首先使用:
``` bash
$ (gdb) info functions
```
会看到输出(经过处理):
``` code
File main.cpp:
void Object::Object();
int main(int, char**);
```
还是看不到我们想要的`void Object::Object(Object * const)`，但是使用了万能的:
``` bash
$ (gdb) print Object::Object
```
看到了输出:
``` code
$1 = {void (Object * const)} 0x400588 <Object::Object()>
```
这里说明了经过输出美化的`Object::Object()`真正的函数签名是`void Object::Object(Object * const)`，这个实验的基本目的就达到了。

上面提到生成的汇编码中有暗示传入参数到`Object::Object()`的疑点，现在就用步进模式运行程序:
``` bash
$ (gdb) start
```
然后键入`s`或者`step`一步一步执行，进入了`Object::Object()`函数执行内部:
``` code
Object::Object (this=0x7fffffffddcf) at main.cpp:5
5		{}
```
注意到这个`this=0x7fffffffddcf`，这个是什么变量的地址值?在这里执行:
``` bash
$ (gdb) print &obj
```
可惜当前`Object::Object()`上下文不存在这个变量，只能键入`s`进行到下一步回到`main()`中执行，得到:
``` code
$2 = (Object *) 0x7fffffffddcf
```
证明了`void Object::Object(Object * const)`是需要传入一个`Object * const`参数作为执行的上下文的。到了这里，忘了在`Object::Object()`上下文中检查上面提到的寄存器们了，只能用`s`或者`n`执行完这次，然后再次`start`并进入到`Object::Object()`的上下文中了，此时执行:
``` bash
$ (gdb) info registers
```
得到:
``` code
rax            0x7fffffffddcf	140737488346575

rdi            0x7fffffffddcf	140737488346575

rbp            0x7fffffffdda0	0x7fffffffdda0
rsp            0x7fffffffdda0	0x7fffffffdda0
```
好了，根据上面所说的`rdi`作为存储函数参数地址的寄存器，那么其中存储的`0x7fffffffddcf`跟`&obj`的输出值对上了，也证明了`Object::Object()`真正的函数签名是`void Object::Object(Object * const)`。

证明完了之后，上面还特意贴出了`rax`寄存器中的值，也是`0x7fffffffddcf`，上面也提到:
``` code
%rax 作为函数返回值使用。
```
那么，最终Object::Object真正函数签名是`Object* Object::Object(Object * const)`呢?是的，然而这个函数是返回`void`，只要我们使用着这个编译器，我们便无法从其提供的语法层面获得这个返回值。

这个简单的实验，也从侧面说明了`C++`这个语言是有多复杂，编译器做了多少的小动作，不花点心思和用点工具，使用者只能看着编译出来的黑箱运行，用点`printf`或者`cout`看看是否正确运行。

至此，这个寻找类构造函数真正的签名的实验到此为止。