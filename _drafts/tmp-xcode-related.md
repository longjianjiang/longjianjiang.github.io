
# target & scheme

一个target代表一个具体的“工程”，可能是app，可能是framework，可能是单元测试；

而scheme则是对这个target的一系列操作的配置，比如有Build操作，Run操作，Archive操作。Run操作内存管理诊断里可以指定zombie检测；

[ref](https://stackoverflow.com/questions/20637435/xcode-what-is-a-target-and-scheme-in-plain-language)

# search paths

$(PROJECT_DIR)代表的是整个项目，xcodeproj文件的上一层目录；

$(SRCROOT)代表的是项目根目录下，xcodeproj文件所在目录；

${PODS_ROOT}代表的是pod目录；

`header search paths` 会尝试两种方式<>, "", `user header search paths` 只会使用 ""的方式。<>表示从系统目录空间搜索文件，""表示从用户目录空间搜索文件。

[ref](https://www.jianshu.com/p/d41e05e6d9fa)

# project.pbxproj

这个文件是Xcode用来描述工程结构的，结构是类似json的kv形式。

描述了工程的目录结构，build settings，build phase，所有文件。

# link framework & embed framework

link库，静态库会将代码链接进可执行文件。静态库则不用，运行时加载进内存，可以做到多个进程共用。

embend，把这个库嵌入到最终输出的程序的bundle里面。

如果link了，但是没有embed，会报Termination Description: DYLD, dependent dylib '@rpath/MyFramework.framework/MyFramework'，操作。

有的时候我们是只需要 link 而不需要 embed 的，比如一个 Framework 依赖另外一个 Framework 的时候（Apple 并不提倡这种操作），比如 Framework A 依赖 Framework B，主工程依赖 Framework A，这时在 Framework A 中，只要 Link Framework B，不需要也不能( Xcode 对于 Framework 工程没有对应的操作界面) Embed Framework B，主工程中在同时 Embed Framework A 和 Framework B。

具体可以看 [Embedding Frameworks In An App](https://developer.apple.com/library/archive/technotes/tn2435/_index.html) 的 Apps with Dependencies Between Frameworks 片段

# shortcut

```
shift + control + F/B: select text;
control + 6: search method in files;
command + option + 0: show/hide utilities panel;
command + 0: show/hide navigator panel;
command + shift + f: search;
```

[ref](https://supereasyapps.com/blog/2014/9/15/14-xcode-time-saving-shortcuts-memorize-and-improve-your-productivity)

# References

[ref](http://www.monobjc.net/xcode-project-file-format.html)

[ref](http://yulingtianxia.com/blog/2016/09/28/Let-s-Talk-About-project-pbxproj/)
