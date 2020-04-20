---
layout: post
title:  "汇编学习笔记"
date:   2019-05-22
excerpt:  "本文是笔者学习汇编的笔记"
tag:
- Compiler
comments: true
---

# 预备

汇编根据平台和的不同格式有些差别，这里笔者给出差别。

```
+----------------------+-----------------------+-----------------------+
|Intel				   |AT&T                   |ARM                    |
|					   |                       |                       |
|add eax, 1			   |add $1, %eax           |add eax, 1             |
+----------------------+-----------------------+-----------------------+
|                      |                       |                       |
|movl -4(%ebp), %eax   |mov eax, [ebp-4]       |ldr eax, [ebp #4]      |
|                      |                       |                       |
+----------------------+-----------------------+-----------------------+
```

ARM 和 Intel 操作数是，右边赋值到左边；AT&T 则是左边赋值到右边，同时寄存器会加`%`。

# ARM 常用汇编指令

```
mov r0, r1				=> r0 = r1
mov r0, #10				=> r0 = 10
ldr r0, [sp]			=> r0 = *sp
str r0, [sp] 			=> *sp = r0
add r0, r1, r2 			=> r0 = r1 + r2
add r0, r1				=> r0 = r0 + r1
push {r0, r1, r2}		=> push r0, r1, r2 onto the stack;
pop {r0, r1, r2}		=> pop three values off the stack, putting them into r0, r1, r2
stp r0, r1, [sp, #16]	=> push r0, r1 onto the stack, then sp = sp + 16 	 
ldp r0, r1, [sp, #16]	=> pop r0, r1 off the stack at sp+16 position, putting them into r0, r1
```

# Register

## CPSR (Current Program Status Register)

CPSR 是32位寄存器，有如下标记位：

Z, bit[30] : zero condition flag. 最后一条flag-setting的指令结果为0，设置Z bit为1.

[ref](https://developer.arm.com/docs/ddi0595/b/aarch32-system-registers/cpsr)

# Instruction

## tst

tst x10, x10; 用于把一个寄存器的内容和另一个寄存器的内容或立即数进行按位的与运算，并根据运算结果更新CPSR中条件标志位的值。当前运算结果为1，则Z bit=0；当前运算结果为0，则Z bit=1

[ref](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289913099.htm)

# @PAGE, @PAGEOFF

[ref](https://reverseengineering.stackexchange.com/questions/14385/what-are-page-and-pageoff-symbols-in-ida)

