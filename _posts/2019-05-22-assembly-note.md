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

## CPU

CPU由运算器，控制器，寄存器，中断系统组成。

## 段

可以将一段内存定义为一个段，一个段地址指示段，偏移地址访问段内单元。

代码段，数据段，栈段。都是这样的段。

CPU如何知道访问的是哪个段，取决于当前段地址寄存器是哪一个。对于数据段来说，段地址放到DS，代码段来说，段地址放在CS。

默认段地址在ds中，可以显式的给出段地址，例如ds:[bx]。

---

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
Each register can be used as a 64-bit X register (X0..X30), or as a 32-bit W register (W0..W30)

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

```
cbz    x19, 0x18bfd2d6c => (compare branch zero) x19地址中的值和0比较，如果等于0，跳转到PC+offset;
```

```
orr    w4, wzr, #0x4 => w4 = wzr | 0x4; wzr(wzr 是 word zero register，类似于 /dev/zero，写入无效，读出为 0)
```

```
csel  Wd, Wn, Wm, cond 
# 等价于
Wd = cond ? Wn : Wm
```

# Register

## lr (link register)

用于保存函数调用的返回地址，即存放执行BL或者BLX指令后的PC的值。

x29是fp寄存器、x30是lr寄存器

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

[ref](https://juejin.im/post/5c9df4c4e51d4502c94c16dd)

# Other

- .p2align x

进行2的x次方进行对齐；

- .quad 0

表示0占4个字，8个字节；

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

---

arm64寄存器分为两种，x0-x30是64位寄存器。w0-w30是32位寄存器。64位寄存器可以当作两个32位寄存器来使用。

整型返回值被保存在 x0 和 x1 中，而浮点值则保存在向量寄存器 v0 - v3 中。同时使用多个寄存器可以返回一个小型结构体类型返回值。

如果返回值为比较大的结构体，那么寄存器可能就变的不够用了。此时就需要调用者做出一些配合。调用者会在一开始为该结构体分配一块内存，然后将其地址提前写入到 x8 寄存器中。在设置返回值的时候，直接往该地址中写数据即可。

# 实战

Trampoline通常都和跳转相关。本文提到的Trampoline是一个特定的地址，该地址指向特定的功能，待功能执行完毕之后，立马跳出Trampoline回到正常执行路径。

RN中的一个Trampoline实现，汇编地址[这里](https://github.com/facebook/react-native/blob/main/React/Profiler/RCTProfileTrampoline-arm64.S)，用到的函数[这里](https://github.com/facebook/react-native/blob/8bd3edec88148d0ab1f225d2119435681fbbba33/React/Profiler/RCTProfile.m)。

[ref](https://www.rookie2geek.cn/react-native/2018/12/02/react-native-Trampoline%E5%AE%9E%E7%8E%B0.html#%E5%AF%84%E5%AD%98%E5%99%A8)

`imp_implementationWithBlock`的实现也用到了Trampoline，有个简单的实现[ref](https://github.com/landonf/plblockimp)。

block实现imp，系统的实现里面用到了虚拟内存的两个函数。`vm_allocate`分配一块虚拟内容，`vm_remap`将申请好的虚拟内存进行重新映射到一块虚拟内存，此时相当于有两个虚拟内存地址指向的是同一块内存区域。

[vm_remap](https://juejin.cn/post/6844903773031284750)
