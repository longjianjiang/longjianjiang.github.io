# 构建

构建是一系列的任务的集合，比如编译源文件，链接目标文件，拷贝资源，app签名。每一步其实都是一个命令行工具加上若干参数，所以build的工作是根据build phase 和 build settings 生成一系列的任务。

## 编译

## 链接

每个目标文件，有以下部分：

.text段，也叫代码段，用来存放汇编指令;
.data段，也叫数据段，用来保存程序里设置好的初始化数据信息;
.rela.text段，叫做重定位表(Relocation Table)。在表里，保存了所有未知跳转的指令;
.symtab段，也叫做符号表，用来存放当前文件里定义的函数和对应的地址;
.linkedit段，链接编辑段，链接编辑是链接工具用来告知操作系统和动态链接器在运行时修复问题的元数据;

库定义了你的 target 中不包含的符号：

Dylibs(Dynamic libraries): 动态库（暴露了可执行程序可以使用的代码和数据的 Mach-O 文件）;

[TBDs](http://kejinlu.com/2016/03/tbd-file/)(Text Based Dylib Stubs): 基于文本的 Dylib 桩（只包含符号）,主要是为了减少真机存储动态库，而仅仅包含编译所需的符号，动态库在设备上会存在，这样可以减少xcode的体积；

静态归档文件(Static archives): 

用 "ar" 工具构建的由多个 .o 文件归档而成的文件
只有包含了你已引用的符号的 .o 文件会被包含到你的应用中

链接的过程就是将多个.o文件和库link在一起，具体来说过程如下，linker会扫描所有的目标文件，将所有符号表中的信息收集起来，构成一个全局的符号表，然后再根据重定位表，把所有不确定的跳转指令根据符号表里地址，进行一次修正。最后，把所有的目标文件的section分别进行合并，形成最终的可执行代码。

# 参考

[https://developer.apple.com/videos/play/wwdc2018/415/](https://developer.apple.com/videos/play/wwdc2018/415/)
