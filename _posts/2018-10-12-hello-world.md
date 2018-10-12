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
	.section	__TEXT,__text,regular,pure_instructions
	.build_version macos, 10, 14
	.globl	_main                   ## -- Begin function main
	.p2align	4, 0x90
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