
# target & scheme

一个target代表一个具体的“工程”，可能是app，可能是framework，可能是单元测试；

而scheme则是对这个target的一系列操作的配置，比如有Build操作，Run操作，Archive操作。Run操作内存管理诊断里可以指定zombie检测；

[ref](https://stackoverflow.com/questions/20637435/xcode-what-is-a-target-and-scheme-in-plain-language)

# project.pbxproj

这个文件是Xcode用来描述工程结构的，结构是类似json的kv形式。

描述了工程的目录结构，build settings，build phase，所有文件。

# shortcut

```
shift + control + F/B: select text;
```

[ref](http://www.monobjc.net/xcode-project-file-format.html)

[ref](http://yulingtianxia.com/blog/2016/09/28/Let-s-Talk-About-project-pbxproj/)
