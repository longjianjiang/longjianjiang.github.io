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

## 数据长度如何判断

汇编中的数据如何判断是以一个字节还是两个字节去读取或者写入呢。

1. 根据寄存器来决定，不同的寄存器存储的位数是固定的；

2. 当没有寄存器当时候，需要声明操作内存单元的长度；

3. 特定指令默认来内存长度；

## 内存寻址的多种方式

[idata]使用常量来表示地址，可以直接定位到某个内存单元；

[bx]使用一个变量表示地址，可以间接定位到某个内存单元；

[bx+idata]使用一个变量和一个常量来表示地址，可以使用这种方式来遍历数组，数组首地址就是idata，bx每次更新进行遍历；

[bx+si]使用两个变量来表示地址；

[bx+si+idata]使用两个变量和一个常量表示地址；

## 转移指令

可以修改IP寄存器，或者修改CS，IP寄存器的指令都是转移指令。

转移指令有两种形式，一种是给定距离IP的偏移实现转移，一种是给出具体的CS，IP地址实现转移。

常见的jmp，ret，loop都是转移指令；

```
assume cs:code
code segment
	start: 	mov ax, 1
			mov cx, 3	
			call s
			mov bx, ax
			mov ax, 4c00h
			int 21h
	s:		add ax, ax
			loop s
			ret
code ends
end start
```

call 和 ret 配合使用可以实现函数的嵌套。

cpu将`call s`指令机器码读入后，IP会指向`call s`的下一条指令`mov bx, ax`。

将IP地址进行push进栈，修改IP，加上偏移定位到标号s指令地址。

执行完标号s后，执行ret，ret将之前push的IP的地址进行pop到IP，此时IP执行`call s`的下一条指令`move bx, ax`处，CPU继续执行。

函数调用过程中，需要进行传递参数和返回结果，简单的方式是参数和结果都使用寄存器进行存储，不过如果参数过多的时候寄存器可能会不够用，这个时候可以存到内存中，返回首地址即可。

函数调用过程中，还存在的一个问题在于寄存器的覆盖问题，多个函数使用同一个寄存器，解决方式是子函数在开始之前将所有用到的寄存器入栈，结束后将所有用到的寄存器进行出栈恢复，达到保存状态的作用。

## 中断

中断的来源分为CPU内部和CPU外部，内中断，外中断。

所谓中断就是CPU在执行完一条指令后，转去执行其他处理。

比如程序中打了一个断点，当CPU执行到这个断点处，就会暂停，显示出当前状态各个变量的值。

这个过程大致可以这样理解，当CPU执行到断点处指令时，会触发一个特定到中断，此时CPU转向去执行Debug提供的中断处理程序。一般情况下这个程序会展示变量状态，并且提供一个输入页面等待输入指令进行下一步操作。

因为中断随时可以发生，所以中断处理程序需要常驻在内容中，并且有一张中断表，中断编号以及对应中断处理程序的地址。通过中断地址表项按一定规则计算CS，IP转到中断处理程序进行执行。转之前还是需要做必要的寄存器（CS，IP）压栈操作，以便返回继续往下执行。

`int 中断编号` 跳转到指定中断处理程序。
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
