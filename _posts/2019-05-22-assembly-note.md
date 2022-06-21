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
sub R0，R1，R2 // R0 = R1 - R2
sub R0，R1，#256 //R0 = R1 - 256
sub R0，R2，R3，LSL#1 // R0 = R2 - (R3 << 1)
push {r0, r1, r2}		=> push r0, r1, r2 onto the stack;
pop {r0, r1, r2}		=> pop three values off the stack, putting them into r0, r1, r2
stp r0, r1, [sp, #16]	=> *(sp + 16) = r0, *(sp + 16 + 8) = r1; // push r0, r1 onto the stack, then sp = sp + 16 	 
ldp r0, r1, [sp, #16]	=> r0 = *(sp + 16), r1 = *(sp + 16 + 8); // pop r0, r1 off the stack at sp+16 position, putting them into r0, r1
```

r(register), p(pair)


```
ldr x0, [x1, #8]		=> Load from address X1 + 8
ldr x0, [x1, #8]!		=> Pre-index: Update X1 first (to X1 + #8), then load from the new address 
ldr x0, [x1], #8		=> Post-index: Load from the unmodified address in X1 first, then update X1 (to X1 + #8)
```

```
temporaryLabel:
    b temporaryLabel # 类似 jmp指令，此时不会将下一个指令地址保存到lr；
	bl temporaryLabel # 同样是跳转到某个位置，但是执行完后回来继续执行下一个指令；
```

# Register

## lr (link register)

用于保存函数调用的返回地址，即存放执行BL或者BLX指令后的PC的值。

## CPSR (Current Program Status Register)

CPSR 是32位寄存器，有如下标记位：

Z, bit[30] : zero condition flag. 最后一条flag-setting的指令结果为0，设置Z bit为1.

[ref](https://developer.arm.com/docs/ddi0595/b/aarch32-system-registers/cpsr)

## rip

rip 指令寄存器，下一个指令会访问到的内存地址。ARM 下是pc（program counter）寄存器；

# Instruction

## tst

tst x10, x10; 用于把一个寄存器的内容和另一个寄存器的内容或立即数进行按位的与运算，并根据运算结果更新CPSR中条件标志位的值。当前运算结果为1，则Z bit=0；当前运算结果为0，则Z bit=1

[ref](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289913099.htm)

# @PAGE, @PAGEOFF

在汇编中，可以发现数据和代码是存放在一起的，数据也是一种代码，所以代码中需要取数据区的值的时候，就需要根据pc当前指令距离数据区偏移来获取地址。

指令长度是32位，所以不能存放更长的地址，所以需要使用@PAGE先取基于PC的偏移的高21位，才使用@PAGEOFF取偏移的12位，这样就可以取到完整的偏移，加上PC就可以找到实际的地址；

下面是一个例子：

```
_addCount:
    ; omit function start
    adrp    x8, _counter@PAGE
    add     x8, x8, _counter@PAGEOFF
    ; omit code for save new value to w1
    str     w1, [x8]
    ; omit function end
```

分成两部，adrp+add，取到_counter距离pc的完整偏移，x8就是_counter的地址，将w1的数值进行更新_counter。

[ref](https://reverseengineering.stackexchange.com/questions/14385/what-are-page-and-pageoff-symbols-in-ida)
[ref](https://juejin.im/post/5c9df4c4e51d4502c94c16dd)

# Debug Notes

## calling convention

x86，函数参数前6个放在寄存器，后面的压栈。

• FirstArgument:RDI
• SecondArgument:RSI
• ThirdArgument:RDX
• FourthArgument:RCX
• FifthArgument:R8
• SixthArgument:R9

rax寄存器存放函数返回值。

• Nybble: 4 bits, a single value in hexadecimal
• Half word: 16 bits, or 2 bytes
• Word: 32 bits, or 4 bytes
• Double word or Giant word: 64 bits or 8 bytes.

# 实战

Trampoline通常都和跳转相关。本文提到的Trampoline是一个特定的地址，该地址指向特定的功能，待功能执行完毕之后，立马跳出Trampoline回到正常执行路径。

RN中的一个Trampoline实现，汇编地址[这里](https://github.com/facebook/react-native/blob/main/React/Profiler/RCTProfileTrampoline-arm64.S)，用到的函数[这里](https://github.com/facebook/react-native/blob/8bd3edec88148d0ab1f225d2119435681fbbba33/React/Profiler/RCTProfile.m)。

[ref](https://www.rookie2geek.cn/react-native/2018/12/02/react-native-Trampoline%E5%AE%9E%E7%8E%B0.html#%E5%AF%84%E5%AD%98%E5%99%A8)

`imp_implementationWithBlock`的实现也用到了Trampoline，有个简单的实现[ref](https://github.com/landonf/plblockimp)。
