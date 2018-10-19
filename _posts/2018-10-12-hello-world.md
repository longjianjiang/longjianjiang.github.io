---
layout: post
title:  "Hello World 入门"
date:   2018-10-12
excerpt:  "本文是笔者对以Hello World 学习汇编，Mach-O格式以及fishhook的笔记"
tag:
- Assembly
comments: true
---

笔者第一个程序就是C语言的Hello World。

```C
#include <stdio.h>

int main(int argc, const char * argv[]) {
    printf("Hello World");
    return 0;
}
```

上述的Hello World 程序非常简单，在控制台输出 "Hello World"， 下面笔者一步一步分析这个简单的程序。

## 汇编

我们用`clang -S helloworld.c` 生成helloworld.s, 也就是汇编代码：

```
    ## 下面四行以 . 开始的是汇编指令
	.section	__TEXT,__text,regular,pure_instructions				## 指定所在seciton
	.build_version macos, 10, 14		
	.globl	_main                   								## 指定开始函数 main，对外可见
	.p2align	4, 0x90												## 指定后面代码 2^4 也就是 16字节对齐， 如果需要用0x90补齐

## 冒号结尾的（例如L_.str:) 称为一个label用来标识一段汇编代码的名字，是这段汇编代码的入口地址。以_开头的label表示是一个函数（例如_main: )，可以用`call` 调用
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
subq	$32, %rsp

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
### 寄存器

上述汇编代码中用到了一些寄存器，下面介绍常见的寄存器：

```
sp：栈顶指针
bp：栈底指针
eax: 一般存储函数返回值
```

### 地址偏移

类似 `-20(%rbp)` 表示在rbp地址-20的一个新地址，类似C语言数组的下标操作。

汇编器会将汇编代码转换为机器码，同时创建一个目标文件，也就是 `.o` 文件。
最后由这些目标文件和库会被链接成一个可执行文件。

## Mach-O

我们用`clang helloworld.c` 生成可执行文件a.out, 使用`file a.out` 我们可以看到改可执行文件的类型是 `Mach-O 64-bit executable x86_64`

Mach-O 包含三部分内容

![Mach-O](http://ocigwe4cv.bkt.clouddn.com/hello_world_1.png)

如上图所示可执行文件由header、Load Commands、Data组成。

### header

我们可以使用 `otool -hv a.out` 查看可执行文件的header信息

```
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64  X86_64        ALL LIB64     EXECUTE    15       1216   NOUNDEFS DYLDLINK TWOLEVEL PIE
```

header的结构体定义如下：

```
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
```

其中header中的flags表示dyld的加载标志位，具体取值可以去 `#include <mach-o/loader.h>` 中查看，头文件中有详细的注释。
所以综上，header中包含一些类型的信息，表示运行的环境（cpu架构类型）。同时flags会影响可执行文件执行的流程。

### load commands

加载命令表示了可执行文件的逻辑结构和内存布局。每个加载命令前面都包含了加载命令的类型和加载命令的的size。

```
struct load_command {
	uint32_t cmd;		/* type of load command */
	uint32_t cmdsize;	/* total size of command in bytes */
};
```

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

## data

之前load commadns中 `LC_SEGEMENT_64` 将segment加载进内存 ，所谓segement就是一段数据，这种数据内部又分不同的section.

### segement
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

### section

section就是将segement中的数据按不同的分类分别存放。

我们可执行文件中有下面不同的section：

```
__TEXT,__text			: 主程序代码
__TEXT,__stubs			: 用来重定向指针表（懒加载和非懒加载）
__TEXT,__stub_helper	: 
__TEXT,__cstring		: C语言字符串
__TEXT,__unwind_info    : 可执行代码的展开信息，用于异常处理

__DATA,__nl_symbol_ptr	: 非懒加载的指针表
__DATA,__la_symbol_ptr  : 懒加载的指针表
```

Mach-O 文件格式大致就是以上所述。

## fishhook

了解完 Mach-O 格式后，我们可以看一看fishhook，一个可以hook C语言系统函数的库。

还是Hello World的例子

```C
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
```

上述程序输出的是 `Hello World [modified]`, 因为使用了fishhook，动态改变了printf函数。
下面笔者分析下fishhook 动态改变printf函数的原理。

### dyld
ld(static linking) responsible for transfroming symbol references in code into indirect symbol lookups for dyld use later.
