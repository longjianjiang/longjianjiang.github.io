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

# 介绍

我们都知道一个hello world需要经过编译链接成可执行文件，才能运行打印出hello world，而做编译链接事情的就是编译器。

上面说的hello world是用C写的，C是一种编译型语言，如果使用脚本语言写，过程则是另一种情况了。脚本语言没有一个翻译的过程，而是逐行解释，这种类型的程序叫做解释器。

通常编译型的语言执行要比解释型的语言要快，解释型的语言则有更好的错误检查能力。

上面所说的编译器和解释器我们称之为 `Language Processors`。

```
Java的 `Language Processors` 既包含编译器又包含解释器，将 .java -> .bytecode 这一过程是编译器完成的，而JVM执行 .bytecode 则是解释器完成的。正是因为此，Java具有了跨平台的能力。
```

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