
# 手机越狱

用iOS App signer 重签 uncOver；

# 砸壳

- 电脑安装frida

用python下的pip包管理工具：`pip install frida`.

- 手机安装frida

添加源`https://build.frida.re`，然后安装frida。

- usb 连接

1. 安装usbmuxd ：`brew install usbmuxd`;

2. 映射端口： `iproxy 2222 22`(将手机22端口映射到电脑的2222端口);

3. ssh连接： `ssh -p 2222 root@localhost`(root表示手机用户名)

> 2, 3 两步都是在电脑终端执行即可。前提是手机通过USB连接到电脑。

4. 下载`frida-ios-dump`，在该目录下执行`./dump.py app名称`。

这一步需要注意：
4.1> 首先需要修改dump.py 里的用户名和密码，密码就是越狱手机终端的密码；
4.2> 期间可能会报错，xx module not found，`pip install module`即可；
4.3> dump期间，需要打开app，这样才能进行内存dump；

# class-dump

拿到砸壳的ipa后，首先dump出二进制文件的头文件信息。

class-dump这个工具使用Runtime特性，将mach-O文件中的@interface和@protocol的信息提取出来，生成对应的.h文件。

查看头文件可以帮助我们后面的定位。

`class-dump -H /path/xx.app -o /output_path`;

`otool -L /path/xx.app/xx` 查看二进制文件使用的所有动态库；

结合Reveal查看界面结构，View的名称，定位到具体的控制器。

# CaptainHook

这一步使用`CaptainHook`提供的API，以及上一步头文件和UI的搜索，尝试Hook具体某个类的某个方法。

# ToDo

了解具体的步骤及原理。

# References

[https://github.com/AloneMonkey/frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump)
[http://stevenygard.com/projects/class-dump/](http://stevenygard.com/projects/class-dump/)
