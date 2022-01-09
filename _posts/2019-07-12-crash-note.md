---
layout: post
title: "Crash相关学习笔记"
date: 2019-07-12
excerpt: "nope"
tag:
- iOS
comments: true
---
# Crash的类型
## Mach 异常 / BSD 信号

当我们的程序中代码写的有问题，比如说访问了一块已经释放的内存地址，会触发`EXC_BAD_ACCESS`异常。

当出现了某个`EXC_XX`异常时，Mach的异常处理程序`exception_triage()`，调用`exception_deliver`将异常通过消息机制进行分发到一个`exception port`。 分发首先会尝试往当前线程上，然后会往进程上，最后会往Host进行分发。如果没有一个port返回值是成功，也就是没有进行捕获，那么程序就会被终止。

线程，进程，Host都有一个异常端口数组，通过调用`xxx_set_exception_ports()`（xxx为thread、task或host）可以设置这些异常端口，默认值是NULL。

BSD层会在第一个进程启动的时候，调用`host_set_exception_ports()`，注册一个`exception port`，这样所有Mach层的异常都会被转发到BSD层，然后这些异常再被转成各种信号。比如`EXC_BAD_ACCESS`可能会被转成`SIGSEGV`。通过方法`threadsignal()`将信号投递到出错线程。可以通过方法signal(x, SignalHandler)来捕获signal。

硬件产生的信号(通过CPU Trap)被Mach层捕获，然后才转换为对应的Unix信号；苹果为了统一机制，于是操作系统和用户产生的信号(通过调用kill和pthread_kill)也首先沉下来被转换为Mach异常，再转换为Unix信号。

所以捕获异常可以在Mach层注册port，也可以在BSD层监听signal，但是从上面的流程可以知道，为了获取所有的异常，需要在Mach层进行捕获。

```
// 1. 创建port
mach_port_allocate
// 2. 设置port发送权限，这样可以往这个port发送消息
mach_port_insert_right
// 3. 将创建的port设置为当前进程的exception port
task_set_exception_ports
// 4. 最后循环等待异常消息
```

## OC 异常

`NSException` 是应用层面的抛出异常，提供了更多的信息，比如reason，name，callStackSymbols信息，可以方便快速的定位问题。

`NSException` 可以通过注册 `NSUncaughtExceptionHandler` 捕获异常信息。

未被try catch的NSException会发出kill或pthread_kill信号-> Mach异常-> Unix信号（SIGABRT）。

无论设置了NSSetUncaughtExceptionHandler与否，最终都会被转成Unix信号，只要该NSException没有被try catch。区别在于设置了，则在其ExceptionHandler中无法获得最终发送的Unix信号类型。

## 特殊

watch dog机制是系统为了避免app长时间无法响应，一般是主线程阻塞，超过一定时间触发watch dog，将app 杀死。[ref](https://developer.apple.com/documentation/xcode/addressing-watchdog-terminations)

watch dog机制的堆栈信息有时候并不是真正耗时的原因，因为比如要求5s完成，但是前4s耗时操作完成了，然后堆栈停留在了正常工作的那一秒，这个时候就需要使用timer porfiler去查看。

oom（out of memory），当app 一直申请内存，一般是递归调用导致栈溢出，触发oom，app会被杀死。

# 文件格式

## DSYM 

`.DSYM`（debugging symbols)就是所谓的调试符号表，文件目录如下所示：

编译的时候加上`-g`选项，编译器会在输出文件中包含debug信息，同时生成`.DSYM`调试符号表。

也可以从Mach-O中提取`.DSYM`文件，命令如下:

```
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/dsymutil /Users/wangzz/Library/Developer/Xcode/DerivedData/YourApp-cqvijavqbptjyhbwewgpdmzbmwzk/Build/Products/Debug-iphonesimulator/YourApp.app/YourApp -o YourApp.dSYM
```

Xcode也是通过这种方式来生成符号表文件的。

## DWARF

`.DSYM`文件目录如下所示：
```
Contents:
	Info.plist
	Resources:
		DWARF:
			xxxx
```

可以看到 `.DSYM` 中保存了一个DWARF（Debugging With Arbitrary Record Formats），`DWARF`是Mach-O文件用来存储和处理调试信息的格式。

![crash_note_1]({{site.url}}/assets/images/blog/crash_note_1.png)

结构如上图所示，类似Mach-O，不同的数据存储在不同的section中，section都以`_debug`开头。

# 符号化流程

所谓符号化，其实就是将地址还原为符号信息带上行号，这样就可以快速定位。

做到解析需要用到`DWARF`中`_debug_info` 和 `_debug_line` 两个section的信息。可以通过如下命令进行提取section中的数据：

```
dwarfdump -debug-info YourPath/YourApp.dSYM/Contents/Resources/DWARF > info-e.txt
```

解析步骤有如下几步：

## 检查一致性

这一步我们有了符号表文件，同时有了crash的日志文件也就是一个地址堆栈信息。此时我们需要确认两者对应的是不是同一个可执行文件，通过UUID来验证一致性。

![crash_note_2]({{site.url}}/assets/images/blog/crash_note_2.png)

如上图所示，crash日志有一个`Binary Images`模块，给出了加载地址段的信息，二进制架构，UUID信息。

`dwarfdump --uuid Your.app.dSYM`可以提取符号表的UUID，因为二进制文件可能支持多个指令集合，UUID会存在多个。

## 计算crash地址对应符号表中的地址

```
Thread 0:
0 CoreFoundation	0x0000000235c9598c 0x0000000235b7d000 + 1149324
1 libobjc.A.dylib	0x0000000234e6e9f8 objc_exception_throw + 56
2 CoreFoundation	0x0000000235b9fbc0 0x0000000235b7d000 + 142272
3 Foundation		0x00000002365f02a4 0x00000002365e8000 + 33444
4 Your 				0x0000000100644068 0x0000000100600000 + 278632
5 Your              0x00000001006443ac 0x0000000100600000 + 279468
6 libdispatch.dylib	0x00000002356d3a38 0x0000000235674000 + 391736
7 libdispatch.dylib	0x00000002356d47d4 0x0000000235674000 + 395220
8 libdispatch.dylib	0x0000000235682008 0x0000000235674000 + 57352
9 CoreFoundation	0x0000000235c2732c 0x0000000235b7d000 + 697132
10 CoreFoundation	0x0000000235c22264 0x0000000235b7d000 + 676452
11 CoreFoundation	0x0000000235c217c0 CFRunLoopRunSpecific + 436
12 GraphicsServices	0x0000000237e2279c GSEventRunModal + 104
13 UIKitCore		0x00000002622d4c38 UIApplicationMain + 212
14 Your 			0x000000010061aa70 0x0000000100600000 + 109168
15 libdyld.dylib	0x00000002356e58e0 0x00000002356e4000 + 6368
```

上面是一个crash堆栈，从第4行看到，crash发生在运行时地址`0x0000000100644068`，后面的地址`0x0000000100600000`则是进行开始地址，加号后面的数字是十进制表示的crash地址距离开始地址的偏移。

```
0x0000000100644068 = 0x0000000100600000 + 0x00044068(278632)
运行时堆栈地址 = 运行时开始地址 + 偏移
```

crash日志的地址都是运行时地址，根据偏移量不变，可以定位到符号表中的堆栈地址。

所以根据符号表TEXT段的开始地址加偏移就可以定位到crash在符号表中的地址。而TEXT段的开始地址可以通过load command查看，使用`$otool -l Your.app.dSYM/Contents/Resources/DWARF/Your`命令。

这个时候得到crash对应符号表中的地址。记做`0x57`。

`_debug_info`中的基本单元是`DIE`(Debug Information Entry)。所以这个时候我们需要根据之前求到的符号表中的地址定位到是哪个DIE的。

具体寻找是根据之前导出的`_debug_info`内容中，通过搜索算法找到一个DIE地址范围包含目标地址，下面给出一个示例：

```
0x000432a2:   DW_TAG_subprogram
                DW_AT_low_pc	(0x000000000000efe8)
                DW_AT_high_pc	(0x000000000000f054)
                DW_AT_frame_base	(DW_OP_reg7 R7)
                DW_AT_object_pointer	(0x000432bf)
                DW_AT_name	("-[AudioNotationItem initWithNotation:time:]")
                DW_AT_decl_file	("/Users/zl/Projects/xsj/EleCalligraphy/ECalligraphy/Models/数据模型/作业/AudioNotationItem.m")
                DW_AT_decl_line	(13)
                DW_AT_prototyped	(0x01)
                DW_AT_type	(0x0004342b "AudioNotationItem*")
                DW_AT_APPLE_optimized	(0x01)
                DW_AT_APPLE_isa	(0x01)
```

可以发现通过这个DIE可以获取到crash所在文件，所在方法，方法在文件的行号。

下面最后一步就是定位到crash发生在这个方法的哪一行，这个时候需要使用到`_debug_line`。

根据DIE的开始地址，定位到方法的开始，然后根据我们最初计算出的符号表中的地址，最终定位到crash发生的具体行数。

到了这里，解析也就结束了。

以上过程可以使用`atos`命令，格式如下：

```
atos -arch <Binary Architecture> -o <Path to dSYM file>/Contents/Resources/DWARF/<binary image name> -l <load address> <address to symbolicate>
```

# 总结

大致了解了这样一个过程，还有一些疑惑，这里记录下来。

调试的符号表和二进制中的符号表有什么关系？

crash 地址如何对应到符号表中的地址？

# References

[ref1](http://foggry.com/blog/2015/07/27/ru-he-shou-dong-jie-xi-crashlog/)

[ref2](http://foggry.com/blog/2015/08/10/ru-he-shou-dong-jie-xi-crashlogzhi-yuan-li-pian/)

[ref3](https://developer.apple.com/library/archive/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-INTRODUCTION)

[ref4](https://faisalmemon.github.io/ios-crash-dump-analysis-book/)

# 附

记录遇到的crash。

- dict存储nil

dict中存储string对象，但实际为nil，导致crash。解决方案： obj?:@"";

- 对象类型错误

消息中detail字段根据不同消息类型可能是不一样的类型，取的时候未判断导致取property的时候出现了unrecognized selector sent to instance。

解决方案：取detail时进行消息类型的判断；

- 数组类型（字符串）操作，时刻注意越界

`range` 操作需要注意，range.location + range.length <= length;

- Attempted to dereference null pointer.

类中存在实例变量的时候，使用`->`去操作的时候，得加空判断，否则会出现EXC_BAD_ACCESS；

- 多线程错误

EXC_BAD_ACCESS 错误一般有两种情况，一种是访问了非法地址，一种是执行了非法指令。对于CPU来说，同一段信息有时候看作是数据有时候看作是指令。

如何区分，看指令寄存器，如果指令寄存器和异常地址一样，那么就是执行了非法指令，否则就是访问了非法地址。

访问了非法地址，一般是地址被改动，等到正常访问的时候出错了，一般涉及到多线程，此时时序无法保证。可以使用xcode提供到[sanitizer](https://developer.apple.com/documentation/xcode/diagnosing-memory-thread-and-crash-issues-early)工具进行排查。

arm下，如果发生了执行非法指令异常，可以从lr寄存器中获取执行非法之前的调用地址，通过atos进行解析到调用记录。

[ref](https://developer.apple.com/documentation/xcode/investigating-memory-access-crashes)
