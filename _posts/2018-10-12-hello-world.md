---
layout: post
title:  "Hello World 学习笔记"
date:   2018-10-12
excerpt:  "本文是笔者对以Hello World 学习汇编，Mach-O格式以及fishhook的笔记"
tag:
- Compiler
comments: true
---

笔者第一个程序就是C语言的Hello World。

{% highlight cpp %}
#include <stdio.h>

int main(int argc, const char * argv[]) {
    printf("Hello World");
    return 0;
}
{% endhighlight %}

上述的Hello World 程序非常简单，在控制台输出 "Hello World"， 下面笔者一步一步分析这个简单的程序。

## 汇编

我们用`clang -S helloworld.c` 生成helloworld.s, 也就是汇编代码：

```
    ## 下面四行以 . 开始的是汇编指令
	.section	__TEXT,__text,regular,pure_instructions				## 指定所在seciton
	.build_version macos, 10, 14		
	.globl	_main                   								## 指定开始函数 main，对外可见
	.p2align	4, 0x90												## 指定后面代码 2^4 也就是 16字节对齐， 如果需要用0x90补齐

## 冒号结尾的（例如L_.str:) 称为一个label用来标识一段汇编代码的名字，也是这段汇编代码的入口地址。以_开头的label表示是一个函数（例如_main: )，可以用`call` 调用
_main:                                  ## @main
	.cfi_startproc
## %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	subq	$32, %rsp
	leaq	L_.str(%rip), %rax
	movl	$0, -4(%rbp)
	movl	%edi, -8(%rbp)
	movq	%rsi, -16(%rbp)
	movq	%rax, %rdi
	movb	$0, %al
	callq	_printf
	xorl	%ecx, %ecx
	movl	%eax, -20(%rbp)         ## 4-byte Spill
	movl	%ecx, %eax
	addq	$32, %rsp
	popq	%rbp
	retq
	.cfi_endproc
                                        ## -- End function
	.section	__TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
	.asciz	"Hello World!\n"


.subsections_via_symbols
```

### 调用帧(Call Frame Information)

```
## 函数开始结尾成对插入
.cfi_startproc
.cfi_endproc

## 主要是为了调试需要，debugger可以根据这些信息展开调用栈
.cfi_def_cfa_offset 16
.cfi_offset %rbp, -16
.cfi_def_cfa_register %rbp
```

### 栈

> 函数调用是基于栈的，栈地址从高往低，新入栈的数据在低地址

main函数中调用`printf` 过程如下所示：

```
## 首先main函数开头将 `rbp` 寄存器入栈，因为main函数中可能会调用其他函数，这样当调用函数返回时我们可以知道main函数的执行地址。
pushq	%rbp  

## 然后将 `rbp` 指向 栈顶
movq	%rsp, %rbp

## 将栈顶前移32个字节，也就是函数调用的位置
subq	$32, %rs

## 将字符串的地址放入 `rax` 寄存器； `leaq` 是地址操作
leaq	L_.str(%rip), %rax

## 将main函数(int argc, const char * argv[])的参数存储起来
movl	$0, -4(%rbp)  ## 字节对齐
movl	%edi, -8(%rbp)
movq	%rsi, -16(%rbp)

## 准备 `printf` 函数的参数，分别为 `rdi`, `al` 
movq	%rax, %rdi
movb	$0, %al
callq	_printf

## 相当于 `movl $0 %ecx`, 也就是将 `ecx` 寄存器赋值为0
xorl	%ecx, %ecx

## 将 `eax`寄存器中的值入栈， 这里是存储printf 函数的返回值
movl	%eax, -20(%rbp)         ## 4-byte Spill

## 将 `eax` 寄存器赋值为0， 也就是main函数的返回值
movl	%ecx, %eax

## 收回之前栈前移的32字节
addq	$32, %rsp

## 恢复之前存入栈的 `rbp` 寄存器中的值
popq	%rbp

## 函数执行return
retq
```

下面笔者介绍**stack frame**，也就是所谓的栈帧，函数调用就是通过栈帧来实现的，结构如下所示:

```
high address
                                 +---------------+
                                 |               |
                                 |      Old      |
                                 |     Stack     |
                                 |     Frame     |
                                 |               |
                --------------   +---------------+
                      ^     ^    |   Parameters  |
                      |   caller +---------------+
                      |     v    |  Return addr  |
                           ---   +---------------+
                Stack Frame ^    |    Old  ebp   |   <------- ebp  Frame Pointer
                            |    +---------------+
                      |  callee  |   Registers   |
                      |     |    +---------------+
                      v     v    |   Local vars  |
                --------------   +---------------+   <------- esp  Stack Pointer

low address  
```

> 根据上图可以看到一个函数调用对应一个栈帧。

当产生函数调用时，首先caller将其参数列表从右往左进行入栈，紧接着将返回地址入栈，完成后跳转到callee进行执行被调用函数。

此时callee将caller的帧地址入栈，这个帧地址就是之前某个方法调用caller所构成的一个函数调用。使用当前栈顶指针更新帧寄存器，然后执行callee，保存某些寄存器中的值，为callee内部的局部变量分配内存。

当callee执行结束后，一个函数调用也就结束了，此时整个栈帧的数据都会被弹出。

最后继续执行caller的下一条指令。

### 寄存器

上述汇编代码中用到了一些寄存器，下面介绍常见的寄存器：

```
sp：栈寄存器，指向栈顶
bp：帧寄存器，指向当前的帧
eax: 一般存储函数返回值
```

### 地址偏移

类似 `-20(%rbp)` 表示在rbp地址-20的一个新地址，类似C语言数组的下标操作。

汇编器会将汇编代码转换为机器码，同时创建一个目标文件，也就是 `.o` 文件。
最后由这些目标文件和库会被链接成一个可执行文件。

## Mach-O

我们用`clang helloworld.c` 生成可执行文件a.out, 使用`file a.out` 我们可以看到改可执行文件的类型是 `Mach-O 64-bit executable x86_64`

Mach-O 包含三部分内容

![Mach-O]({{site.url}}/assets/images/blog/hello_world_1.png)

如上图所示可执行文件由header、Load Commands、Data组成。

### header

我们可以使用 `otool -hv a.out` 查看可执行文件的header信息

```
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64  X86_64        ALL LIB64     EXECUTE    15       1216   NOUNDEFS DYLDLINK TWOLEVEL PIE
```

header的结构体定义如下：

{% highlight cpp %}
/*
 * The 64-bit mach header appears at the very beginning of object files for
 * 64-bit architectures.
 */
struct mach_header_64 {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
	uint32_t	reserved;	/* reserved */
};
{% endhighlight %}

其中header中的flags表示dyld的加载标志位，flags会影响可执行文件执行的流程。具体取值可以去 `#include <mach-o/loader.h>` 中查看，头文件中有详细的注释。
所以综上，header中包含一些类型的信息，运行的环境（cpu架构类型），加载命令的信息（加载命令的个数和大小）。

### load commands

加载命令表示了可执行文件的逻辑结构和内存布局。每个加载命令前面都包含了加载命令的类型和加载命令的的size。

{% highlight cpp %}
struct load_command {
	uint32_t cmd;		/* type of load command */
	uint32_t cmdsize;	/* total size of command in bytes */
};
{% endhighlight %}

下面给出我们可执行文件中的所有load commands

```
LC_SEGEMENT_64 			: 		将segement中的所有section数据加载进内存
LC_DYLD_INFO_ONLY		:		可执行文件中动态连接的信息
LC_SYMTAB				: 		可执行文件的符号表，动态/静态连接器会用到这些信息
LC_DYSYMTAB				: 		动态符号表，动态连接器会用到该信息
LC_LOAD_DYLINKER		: 		启动动态加载连接器(/usr/lib/dyld)加载可执行文件
LC_UUID					:		可执行文件的唯一标示符
LC_SOURCE_VERSION		: 		可执行文件的版本	
LC_MAIN					: 		可执行文件入口地址和栈大小
LC_LOAD_DYLIB			: 		可执行文件需要加载的动态库的名字，版本等信息
LC_FUNCTION_STARTS		:		可执行文件的函数地址表
LC_DATA_IN_CODE			: 		可执行代码中数据的位置，用来反汇编
```

### data

之前load commands `LC_SEGEMENT_64` 将segment加载进内存 ，所谓segement就是一段数据，这种数据内部又分不同的section.

#### segement
我们可执行文件中有四种不同的segement：

```
// 捕捉空指针
#define	SEG_PAGEZERO	"__PAGEZERO"	/* the pagezero segment which has no */
					/* protections and catches NULL */
					/* references for MH_EXECUTE files */

// 代码/只读数据段
#define	SEG_TEXT	"__TEXT"	/* the tradition UNIX text segment */

// 数据段
#define	SEG_DATA	"__DATA"	/* the tradition UNIX data segment */

// 连接器使用的符号和表文件
#define	SEG_LINKEDIT	"__LINKEDIT"	/* the segment containing all structs */
					/* created and maintained by the link */
					/* editor.  Created with -seglinkedit */
					/* option to ld(1) for MH_EXECUTE and */
					/* FVMLIB file types only */
```

#### section

section就是将segement中的数据按不同的分类分别存放。

我们可执行文件中有下面不同的section：

```
__TEXT,__text			: 主程序代码
__TEXT,__stubs			: 用来重定向指针表（懒加载和非懒加载）
__TEXT,__stub_helper	: 懒加载指针表中指针指向该区域，当用到懒加载中的符号时，通过该区域表中的offset进行重定向
__TEXT,__cstring		: C语言字符串
__TEXT,__unwind_info    : 可执行代码的展开信息，用于异常处理

__DATA,__nl_symbol_ptr	: 非懒加载的指针表
__DATA,__la_symbol_ptr  : 懒加载的指针表
```

Mach-O 文件格式大致就是以上所述。

## fishhook

了解完 Mach-O 格式后，我们可以看一看fishhook，一个可以hook C语言系统函数的库。

还是Hello World的例子

{% highlight cpp %}
static int (*orig_printf)(const char *, ...);

int new_printf(const char * str, ...) {
    return orig_printf("%s [modified]",str);
}

int main(int argc, const char * argv[]) {
    struct rebinding printf_rebinding = { "printf", new_printf, (void *)&orig_printf };
    rebind_symbols((struct rebinding[1]){printf_rebinding}, 1);
    
    printf("Hello World");
    return 0;
}
{% endhighlight %}

上述程序输出的是 `Hello World [modified]`, 因为使用了fishhook，动态改变了printf函数。
下面笔者分析下fishhook 动态改变printf函数的原理。

### dyld

dyld (dynamic linker) ，简单的说负责将可执行文件中需要的镜像加载到内存中，同时也负责绑定懒加载和非懒加载的symbol。

所以fishhook的原理就是，首先注册了一个回调，`_dyld_register_func_for_add_image`, 该方法会执行当有新的镜像（也就是系统的动态库）加载时执行，同时第一次也会为之前已经加载进内存的镜像执行一次回调。这个时候更新可执行文件中symbol，用新的函数指针替换原先的函数指针，从而达到hook的效果。

### 源码

如果去看fishhook的源代码，里面的主要工作就是在之前我们说到的懒加载符号表和非懒加载符号表（这两个表保存这字符串对应的函数指针）中寻找和我们需要hook的函数一致的函数名，然后将双方的函数指针指向进行替换。

> 所谓懒加载表中的符号就是指，当用到时dyld才会通过`dyld_stub_binder`去链接到指定的动态链接库，然后再去执行；而非懒加载则当动态链接库第一次被绑定时就会被加载。

所以上述fishhook的主要工作就是在于寻找到需要hook的函数名，这里用到了额外三张表，动态符号表，符号表和字符串表,这三个表都是在`__LINKEDIT`段的。

```
Symbol Table（symtab）: 符号表记录了符号的映射。在 Mach-O 中，符号表是由结构体 n_list 构成。

Dynamic Symbol Table(Indirect Symbols): 动态符号表主要是为了提供index从而在符号表中找到对应的符号，其本身就是一个uint32_t的index数组。

String Table（strtab）: 字符串表存放了符号表中可读的字符串，字符串末尾自带的 \0 为分隔符（机器码00）
```

所以fishhook寻找的主要步骤如下：

### rebind_symbols_for_image 方法
- 找到`LC_SEGEMENT_64(__LINKEDIT)`, `LC_SYMTAB`, `LC_DYSYMTAB` 三个加载命令

- 根据以上三个加载命令，找到动态符号表、符号表和字符串表的地址

- 找到懒加载表和非懒加载表section，准备进行下一步操作（替换该section中的符号对应的函数指针）

### perform_rebinding_with_section 方法

- 首先找到section对应的动态符号表的数组指针，然后遍历section，从动态符号表中取到符号表的index，然后根据index取到结构体n_list, 接着根据n_list中的n_strx也就是该符号名在字符串表中的偏移量，最后用字符串表的地址就可以得到该符号的名字。这个时候遍历我们开始传递过去的一个`rebinding`结构体，比较name是否相等，则进行函数指针替换。

所以我们调用printf，会调用到new_printf, 而new_printf中调用orig_printf会调用原来的printf，因为我们将先前传递的函数指针的实现替换成了系统的函数的实现。

上述步骤中涉及了很多关于偏移的计算，其实这些规则在apple提供的头文件中都有说明，这里就不具体说明了。

唯一想说明的是动态符号表中其实区分了多个不同的section，所以开始取的时候需要根据section的`reserved1`偏移和动态符号表的地址进行获取。

以上就是fishhook的全部流程，具体各位可以去参看源代码。

## 最后

一个小小的hello world，背后的东西，远远比想象的多的多，本文也只是简单的分析了下其中的一些方面。

## References

[https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html](https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html)
[https://www.mikeash.com/pyblog/friday-qa-2012-11-30-lets-build-a-mach-o-executable.html](https://www.mikeash.com/pyblog/friday-qa-2012-11-30-lets-build-a-mach-o-executable.html)
[https://www.objc.io/issues/6-build-tools/mach-o-executables/](https://www.objc.io/issues/6-build-tools/mach-o-executables/)
[http://wsfdl.com/linux/2015/05/16/%E7%90%86%E8%A7%A3stack-func.html](http://wsfdl.com/linux/2015/05/16/%E7%90%86%E8%A7%A3stack-func.html)