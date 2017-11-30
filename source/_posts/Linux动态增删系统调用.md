---
title: Linux动态增删系统调用
date: 2017-11-24 16:42:58
tags:
	- Linux
	- 系统调用
	- 动态加载模块
	- C
categories: 系统运维
---

## 环境支持
- Fedora 25
- gcc

## 系统调用

`Linux`的系统调用是操作系统提供给用户程序调用的一组接口，用户程序可以在`用户态`下调用这些接口获得内核提供的服务，而无须进入内核态，这样就会少了很多权限上的问题，而且用户态与内核态的切换是存在开销的。

### 增删系统调用的方法

#### 1.重新编译内核

修改内核库函数的相关文件，将准备添加的系统调用声明以及实现代码等加入进去，然后使用`make`命令重新编译一个新的内核，添加到`grub`启动列表等等。

过程挺麻烦的，而且涉及到编译内核，要准备不少环境不说了，一旦与搜索引擎搜索出来的教程环境不一样的话，还有不小概率会编译失败。

#### 2.动态加载模块

这个方法不需要修改内核库相关的文件，自己的代码源码和makefile写好，基本上就可以了。主要的原理是利用动态加载模块的时候的权限访问以及修改系统相关的一些信息。而系统调用表是操作系统级的数据，所以在动态加载的过程中获得系统调用表并修改之。

这个过程也不简单，而且编译成功也需要花费点心思，毕竟大多数搜出来的教程什么的，不是祖传的就是拷贝的，当自己的实验环境跟教程不一样的话，只能靠自己摸索了。

## 环境参数收集

#### 1.获取当前系统已使用的系统调用号:
``` bash
$ cat /usr/include/asm/unistd.h
```
然后根据自身系统的位数，例如笔者的`Fedora 25`是64位机:
``` bash
$ cat /usr/include/asm/unistd_64.h
```
获得最后一个系统调用号:
``` code
#define __NR_statx 332
```
这样在后续设置准备添加的系统调用号就为`333`以后的值，然而这个是祖传教程里面可以采用的做法，笔者尝试了在本机64位`Fedora 25`下添加调用号为`333`的系统调用，失败了，也没能找到类似`NR_syscalls`这类的宏定义来修改。只能通过替换`332`以内的值来替换系统当前使用中的系统调用。

替换已有的系统调用号，在卸除自己添加的系统调用的时候还原回去。但是系统调用会被众多系统中的应用调用的，一个不小心可能会导致系统中一些应用的崩溃，所以这个是能是个实验，看来在64位机上要做应用级的系统调用还是得重新编译内核。

#### 2.获取系统系统调用表的内存地址:

解决这个问题的方法有很多，笔者采用的是:
``` bash
$ sudo cat /proc/kallsyms | grep sys_call_table
```
就会得到:
``` code
ffffffff90a00200 R sys_call_table
ffffffff90a00e80 R ia32_sys_call_table
```
只需要用到`sys_call_table`的值即可。

注意到它们的只读`R`属性，这意味着即使拿到了超级权限，也无法修改，怎么解决这个问题后面会提及。

## 演示目标说明

既然是一个实验，那肯定需要用一个演示程序及其演示结果来让实验者得知目标已达成，在这个实验里面，用户态的程序是发起系统调用，获得系统正在运行进程信息，并显示出来，在这里虽然设计了深度来表示进程是有几个前驱进程创建出来的，但是没有表现父子间的关系，只有简单的层数。

那么被调用的系统调用自然就是查询系统中的进程的数据结构，采集需要的数据并存到起来，返回给用户态的程序。

## 定义结构体

这个结构体是用来采集系统中进程的信息，这个定义在用户态程序的代码和内核态程序的代码要分开定义，但是要保持一致。

``` c
struct process_info_t
{
	unsigned int pid;

	unsigned int depth;

	char process_name[TASK_COMM_LEN];
};
```

## 定义准备替换的函数的功能

定义准备替换系统调用的函数功能，注意函数参数中的`__user`定义了该参数是从用户态传入。调用`process_tree`访问系统中的进程信息并采集，随后调用`copy_to_user`函数将数据复制到用户态内存区域上。

``` c
asmlinkage long __my_syscall__(char __user* buf)
{
	int depth = 0;

	process_counter = 0;

	reset_proc_info_t_arr_val(pro_info_array_kernel, 512);

	struct task_struct* p;

	printk("Evan's system call is executing.\n");
	
	p = &init_task;					// 获得系统进程信息的表头。

	process_tree(p, depth);

	if(copy_to_user((struct process_info_t*)buf, pro_info_array_kernel, 512 * sizeof(struct process_info_t)))
	{
		return -EFAULT;
	}
	else
	{
		return sizeof(pro_info_array_kernel);
	}
}
```

具体执行采集进程信息的`process_tree`函数:
``` c
void process_tree(struct task_struct* _p, int _depth)
{
	struct list_head* l;

	pro_info_array_kernel[process_counter].pid = _p->pid;

	pro_info_array_kernel[process_counter].depth = _depth;

	strncpy(pro_info_array_kernel[process_counter].process_name, _p->comm, TASK_COMM_LEN);

	++process_counter;					// 全局变量，用于记录采集信息数组的下标值。

	// 递归复制子进程的信息。
	for(l = _p->children.next; l != &(_p->children); l = l->next)
	{
		struct task_struct* t = list_entry(l, struct task_struct, sibling);

		process_tree(t, _depth + 1);
	}
}
```

## 替换系统调用的函数代码

前面提到过系统调用表的具有不可修改的标志，甚至用超级权限也是，解决办法是用汇编访问到状态寄存器`CR0`，而`CR0`中的从低到高数20号索引(注意索引从0开始)是内存写保护位，将其设置为0之后即可为所欲为了。

所以思路是修改系统调用表前设置`CR0`状态，设置写保护关闭，改完表之后设回原来的值。代码如下:
``` c
// 关闭写保护的函数
uint64_t clr_and_ret_cr0(void)
{
	uint64_t cr0 = 0;

	uint64_t ret;

	asm("movq %%cr0, %%rax":"=a"(cr0));					// 获取CR0的值。

	ret = cr0;

	// cr0 &= 0xfffffffffffeffff;						// 两句是等价的，设置20号位为0。
    cr0 &= ~0x10000LL;

	asm("movq %%rax, %%cr0"::"a"(cr0));					// 设置CR0的值。

	return ret;											// 将修改前的值返回到调用者。
}

// 复原状态的函数
void setback_cr0(uint64_t val)
{
	asm volatile("movq %%rax, %%cr0"::"a"(val));
}
```
这里用到了汇编的知识，之前的教程都是32位的汇编码，如果不加C编译参数设置编译环境是32位的话，当然在64位机编译失败。刚好笔者前一段时间科普了一下汇编，才稍微懂得把32位汇编代码改到64位机可用。

## 定义加载模块时的行为

修改系统调用和编写内核模块是分开两个技术范畴，只是在这个实验中借着加载以及卸载内核模块的过程修改系统调用表。前面定义了准备替换的函数行为以及修改系统写权限的代码，现在定义程序是如何实施修改系统调用表行为。

在加载内核模块的时候，告知系统加载模块时的行为用`module_init`，而告知系统卸载时的行为用`module_exit`。

加载内核模块的时候修改系统调用表:
``` c
static int __init __init_extra_syscall__(void)
{
	printk("Evan's System call successfully added.\n");

	__sys_call_table_ptr__ = (unsigned long*)__SYS_CALL_TABLE_ADDR__;

	printk("System call's address: %lx. TASK_COMM_LEN's value: %d.\n", __sys_call_table_ptr__, TASK_COMM_LEN);

	funptr = (int(*)(void)) (__sys_call_table_ptr__[__MY_SYS_CALL_NUM__]);				// 保存原来表中该函数的指针。

	orig_cr0 = clr_and_ret_cr0();														// 关闭写保护。

	__sys_call_table_ptr__[__MY_SYS_CALL_NUM__] = (unsigned long)&__my_syscall__;		// 替换为自定义的函数。

	setback_cr0(orig_cr0);																// 复原。

	return 0;
```
卸载内核模块的时候还原系统调用表:
``` c
static void __exit __exit_extra_syscall__(void)
{
	orig_cr0 = clr_and_ret_cr0();

	__sys_call_table_ptr__[__MY_SYS_CALL_NUM__] = (unsigned long)funptr;

	setback_cr0(orig_cr0);

	printk("Evan's system call exited.\n");
}
```
然后告知系统加载卸载内核模块行为:
``` c
module_init(__init_extra_syscall__);

module_exit(__exit_extra_syscall__);

MODULE_LICENSE("GPL");
```
最后一句许可是不可省略的，不然会编译失败。

## 用户态程序行为

``` c
int main(int argc, char* argv[])
{
	int i, j, err;

	// 系统调用332号，并传入参数。
	err = syscall(332, &processes);

	printf("Syscall result: %d.\n", err);

	if(err != 0)
	{
		printf("%s\n", strerror(errno));
	}

	printf("Syscall result: %d.\n", syscall(332, &processes));

	// 打印进程信息。
	for(i = 0; i < 512; ++i)
	{
		for(j = 0; j < processes[i].depth; ++j)
		{
			printf("|-");
		}

		printf("%d-%s\n", processes[i].pid, processes[i].pname);

		if(processes[i + 1].pid == 0)
		{
			break;
		}
	}

	return 0;
}
```
用户态的程序很简单，就是调用，然后显示返回的数据。

## Makefile的编写

`make`命令已经集成好编译内核模块，指定好参数，然后`make`即可。注意`obj-m`要对应好，如笔者保存的修改系统调用表的源码文件命名为`ProcessTree.c`，所以需要`obj-m:=ProcessTree.o`。

``` Makefile
KERNEL_VERSION = `uname -r`

obj-m:=ProcessTree.o
# EXTRA_CFLAGS=-m32	# 在64位系统编译32位代码的标志。

build: kernel_modules test_run

kernel_modules:
	make -C /lib/modules/${KERNEL_VERSION}/build/ M=${CURDIR} modules

test_run:
	gcc -o test_run test_run.c

clean:
	make -C /lib/modules/${KERNEL_VERSION}/build M=${CURDIR} clean; \
	rm -rf ${CURDIR}/test_run
```

[完整源码](https://github.com/CFWLoader/Toys/tree/master/SystemCall/64bit)再此。

## 编译加载运行

在目录下执行:
``` bash
$ make
```
即会产生目标文件，随后执行加载模块命令:
``` bash
$ sudo insmod ProcessTree.ko
```
查看加载模块信息:
``` bash
$ sudo dmesg
```
例如，在这个例子中会看到如下输出:
``` code
[18128.569537] Evan's System call successfully added.
[18128.569540] System call's address: ffffffffa5a00200. TASK_COMM_LEN's value: 16.
```
执行用户态程序，顺利的话，会得到运行结果，输出太长了就不贴了。

再用`dmesg`可以看到:
``` code
[18158.067630] Evan's system call is executing.
```
表明这个系统调用成功执行了。

卸载模块:
``` bash
$ sudo rmmod ProcessTree
```
再用`dmesg`查看:
``` code
[18190.508284] Evan's system call exited.
```
模块卸载了，系统调用表也还原了。

动态修改系统调用表的尝试至此为止了。之前在用32位的代码做实验的时候挺顺利的，后来想想在64位机上理论上也可以做到同样的效果，将代码改为支持64的汇编还好。中间是卡在了之前添加系统调用的设想上，明明获取了正确的系统调用表内存地址，`dmesg`也表明修改成功了，但是在用户态调用添加的333号调用上老是报错函数未实现，但是直接替换系统当前已有的332号就可以。所以猜想系统中应该存在类似`NR_syscalls`这类变量控制好系统调用表的边界，然而动用了`Google`搜索引擎也没找到期望的结果，只能放弃了。

从这个实验看，`Linux`系统的安全性也越来越好了。