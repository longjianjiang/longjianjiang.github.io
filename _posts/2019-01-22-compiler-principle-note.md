---
layout: post
title:  "编译原理笔记"
date:   2019-01-22
excerpt:  "本文是笔者学习编译原理(Compilers Principles, Techniques, & Tools (purple dragon book) second edition)的笔记。"
tag:
- Compiler
comments: true
---

> 笔者大学期间没有学过编译原理这门课，笔者认为作为一个合格的程序员，有必要了解底层的知识，所以本文就是笔者学习编译原理的笔记。

## 介绍

我们都知道一个hello world需要经过编译链接成可执行文件，才能运行打印出hello world，而做编译链接事情的就是编译器。

上面说的hello world是用C写的，C是一种编译型语言，如果使用脚本语言写，过程则是另一种情况了。脚本语言没有一个翻译的过程，而是逐行解释，这种类型的程序叫做解释器。

通常编译型的语言执行要比解释型的语言要快，解释型的语言则有更好的错误检查能力。

上面所说的编译器和解释器我们称之为 `Language Processors`。

Java的 `Language Processors` 既包含编译器又包含解释器，将 .java -> .bytecode 这一过程是编译器完成的，而JVM执行 .bytecode 则是解释器完成的。正是因为此，Java具有了跨平台的能力。

下面给出一个完整的编译过程：

```
source program
      |
  Preprocessor
      |
modified source program
      |
   Compiler
      |
target assembly program
      |
   Assembler
      |
relocated machine code
      |                      
   linker -----------> | relocatable object files |
   loader -----------> | library files            |
      |                      
 target machine code
```

之前已经说了编译器的工作就是把源程序转变为目标机器代码，这个过程主要分为两步：analysis(分析)、synthesis(合成)。分析部分称为编译器的前端，合成部分称为编译器的后端。分析部分主要拆分源程序，构建语法结构，同时检查错误，给出提示，同时生成符号表。合成部分则根据intermediate representation(中间表示)和符号表构造目标机器代码。下面给出具体的一个流程:

![compiler_principle_note_1]({{site.url}}/assets/images/blog/compiler_principle_note_1.png)

> 实际运行中有些步骤可能会组合起来，组合起来的步骤之间生成的中间表示不需要被明确的构造出来。

# 编译步骤

1> 词法分析（Lexical Analysis）

LA也叫词法分析，LA的输入是程序字符串，输出是一系列的词法单元（token）。

代码作为字符串，逐行读取每个字符，然后去和各个语言里面的token表去对比，生成token。

每个token可以使用<token-name，attribute-value>来表示，token的name代表这个token的类型，value用来记录该token在符号表中的index。

2> 语法分析

Parsing也叫语法分析(syntax analysis)，Parsing的目的是将LA产生的一系列token形成语法树。

比如if/else，while，for，这类语句，生成对应的语法树。

3> 语义分析

所谓语义指的是变量名模糊的情况，一种处理方式严格按照作用域，另一种是判断类型来判断是否为同一个。

4> 优化

优化指的是，编译器会对源代码做一些优化，比如 `int a = 5 * 0;` 可能被优化为 `int a = 0;`。

可以用来减少代码体积，节省了代码容量，代码能运行的更快，消耗内存较少，编译器的优化是有针对性的。

5> 生成代码

生成汇编代码；

转换成平台相关语言；
### 补充

最近笔者再看autoconf和automake的时候了解到其实在预编译之前还有其他的步骤，将其补充到这里：

因为所写的代码是需要跨平台的关系，所以编译器在最开始需要进行所谓的config，这个步骤需要做的工作就是生成一份宏定义(config.h)，方便针对不同平台写条件编译的代码。

同时config步骤还会生成一个Makefile文件，这个文件的说明了源文件之间的依赖关系，假设A文件依赖B文件，那么编译的时候得先编译B然后才编译A，同时当B改变的时候，A需要重新编译。此时也就知道了该程序使用到了那些系统库。此外Makefile中会保存着编译器和连接器的参数选项。

生成config.h 以及 Makefile这一工作的是configure脚本，因为手工写这个脚步比较复杂，所以有了autoconf这个工具。configure脚步会根据`Makefile.in` 模版文件来生成Makefile，为了不让手工写这个模块文件，automake用来自动生成这个模版文件。

所以有了autoconf和automake，我们只需要编写`configure.ac`，一组宏定义，autoconf根据这个文件去生成configure脚本；`Makefile.am` 这个文件里描述文件的依赖关系，automake根据这个文件去生成Makefile.in文件，从而继续生成Makefile。automake不单独使用，需要和autoconf一起使用。

下面是两个工具的具体工作流程图：

![compiler_principle_note_2]({{site.url}}/assets/images/blog/compiler_principle_note_2.png)

我们也可以简单的使用`autoreconf --install`来代替图中的一系列的命令。

## References

[https://book.douban.com/subject/1872069/](https://book.douban.com/subject/1872069/)

[http://darktea.github.io/notes/2012/06/24/autotools.html](http://darktea.github.io/notes/2012/06/24/autotools.html)

[https://geesun.github.io/posts/2015/02/autotool.html](https://geesun.github.io/posts/2015/02/autotool.html)
