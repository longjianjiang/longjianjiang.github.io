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

