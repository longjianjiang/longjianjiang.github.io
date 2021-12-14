# monkeydev

安装monkeyDev过程中的错误，issue中有方案，记录一下。

[xcode 12 Types.xcspec not found](https://github.com/AloneMonkey/MonkeyDev/issues/266)

[curl: (7) Failed to connect to raw.githubusercontent.com port 443: Connection refused](https://github.com/AloneMonkey/MonkeyDev/issues/242)

# 手机越狱

用iOS App signer 重签 uncOver；

## cydia 打开报错

报错： The package <tweak.name> needs to be reinstalled, but I can't find an archive for it. cydia 无法正常显示内存。

解决方案root下执行命令，就可以了。

```
dpkg --remove --force-remove-reinstreq <tweak.name>
```

## cydia 无法连接网络

报错：posix: network is down

解决方案root下执行命令，就可以了。

```
rm /Library/Preferences/com.apple.networkextension.plist
killall -9 CommCenter
```

# 砸壳

- 电脑安装frida

```
brew install python; // python3
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py
```

```
sudo easy_install pip; // python2.7 系统自带
```

用python下的pip包管理工具：`pip3 install frida`.

- 手机安装frida

添加源`https://build.frida.re`，然后安装frida。

- usb 连接

1. 安装usbmuxd ：`brew install usbmuxd`;

2. 映射端口： `iproxy 2222 22`(将手机22端口映射到电脑的2222端口);

3. ssh连接： `ssh -p 2222 root@localhost`(root表示手机用户名)

> 2, 3 两步都是在电脑终端执行即可。前提是手机通过USB连接到电脑。

4. 下载`frida-ios-dump`，在该目录下执行`python3 dump.py app名称`。

这一步需要注意：

4.1> 首先需要修改dump.py 里的用户名和密码，密码就是越狱手机终端的密码；

4.2> 期间可能会报错，xx module not found，`pip install module`即可；

4.3> dump期间，需要打开app，这样才能进行内存dump；

4.4> dump的时候，需要在dump.py中指定用户名和密码，同时用户需要是root，否则可能会出现SCPException，导致砸壳不成功；

打开终端，如果当前用户是mobile，运行 `su root` 输入密码后切到到root用户。

[可以导出包，但是没有砸壳成功](https://github.com/AloneMonkey/frida-ios-dump/issues/118)

dump.py 报 segmentation fault 错误，手机的frida版本需要升级到最新就正常了。
dump.py 报 Unable to connect (connection refused) 错误，电脑frida版本比手机高会报错，需要保持手机电脑frida版本一致。

# class-dump

安装步骤：

```
1. 下载地址：http://stevenygard.com/projects/class-dump/
2. open /usr/local/bin
3. 把dmg文件中的class-dump文件复制到/usr/local/bin
4. 更改权限：终端输入, sudo chmod 777 /usr/local/bin/class-dump
5. class-dump --help
```

拿到砸壳的ipa后，首先dump出二进制文件的头文件信息。

class-dump这个工具使用Runtime特性，将mach-O文件中的@interface和@protocol的信息提取出来，生成对应的.h文件。

查看头文件可以帮助我们后面的定位。

`class-dump -H /path/xx.app -o /output_path`;

`otool -L /path/xx.app/xx` 查看二进制文件使用的所有动态库；

结合[Lookin](https://lookin.work/)查看界面结构，View的名称，定位到具体的控制器。

# debugserver 

```
ps -e | grep appname
debugserver localhost:1234 -a pid
lldb: process connect connect:1234 // ip is 1234
```
# lldb

反反调试工具，[https://github.com/4ch12dy/xia0LLDB](https://github.com/4ch12dy/xia0LLDB)。

# CaptainHook

这一步使用`CaptainHook`提供的API，以及上一步头文件和UI的搜索，尝试Hook具体某个类的某个方法。

[Logos](https://iphonedevwiki.net/index.php/Logos#.25orig)

# dpkg

手机越狱后cydia网络问题无法安装插件，看到网上越狱手机使用[reveal](https://www.jianshu.com/p/24897f77b39e)调试页面。所以搜了下可以使用dpkg在命令行中安装。

安装需要两个deb，一个是 Reveal2Loader_1.0-3_iphoneos-arm.deb ，一个是 extensionlist_1.0-1.deb。

安装过程是，使用iFunBox将这个两个deb文件放入到/var/mobile/Documents 目录下面，然后依次执行：

```
dpkg -i extensionlist_1.0-1.deb
dpkg -i Reveal2Loader_1.0-3_iphoneos-arm.deb
```

可以从[这个地址](http://www.cydiacrawler.com/index.php?cat=search&keyword=com.zidaneno5.extensionlist#.YRTi5VMzb_Q)去搜索deb文件。

安装好reveal后，拖入到Patcher软件中，然后打开Reveal.app输入任意序列号即可激活。

# 原理

## 砸壳

因为Apple将我们上传到App Store的二进制进行了加密，而我们进行逆向分析的二进制需要是未加密的。所以需要一个解密的过程。

其实这个过程的原理很简单，虽然Apple将二进制进行了加密，但是当二进制加载进内存的时候，肯定需要解密从而运行App。所以`dumpdecrypted`的原理也是用了这一点，将内存中解密的镜像dump出来，写到外部文件，这样也就达到了解密的效果。

`dumpdecrypted`具体实现也不复杂，通过`_dyld_register_func_for_add_image`添加回调，每次尝试将加密的load command分三步写进文件。

> 根据代码可以知道，只有一个load command是加密的；也就是`LC_CODE_SIGNATURE`，codesign的时候会对二进制文件进行签名。

1. 尝试将当前加密load command之前的内容写进文件；
2. 写当前加密的内容；
3. 尝试将加密load command之后的内容写进文件；

# References

[https://github.com/AloneMonkey/frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump)

[https://www.tylinux.com/2018/03/12/how-dumpdecrypted-dylib-works/](https://www.tylinux.com/2018/03/12/how-dumpdecrypted-dylib-works/)

[https://github.com/stefanesser/dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)

[http://stevenygard.com/projects/class-dump/](http://stevenygard.com/projects/class-dump/)

[https://github.com/rpetrich/CaptainHook](https://github.com/rpetrich/CaptainHook)

[https://pandara.xyz/2016/08/13/fake_wechat_location/](https://pandara.xyz/2016/08/13/fake_wechat_location/)
