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

> 函数调用是基于栈的，栈地址从高往低，新入栈的数据在低地址，`sp`寄存器指向栈顶地址。

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



