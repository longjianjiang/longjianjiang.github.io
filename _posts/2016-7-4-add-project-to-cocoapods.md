---
layout: post
title:  "发布自己的开源框架到Cocoapods"
date:   2016-07-4
excerpt: "自己写了个app内置浏览器，之前一直在Cocoapods上用别人的，今天试着把自己的上传，让其他人也可以用。"
tag:
- Cocoapods
- opensource
comments: true
---


今天自己写了个[App内置浏览器](https://github.com/longjianjiang/LJIn-AppBrowser)，想上传到Cocoapods，之前一直都是用别人的，今天终于可以让人家通过`pod  install`来安装自己的框架到项目中了。
下面就示范下如何一步一步的将自己的框架上传到Cocoapods。
如果你还没用过Cocoapods，可以看看唐巧这一篇文章[用CocoaPods做程序的依赖](http://blog.devtang.com/2014/05/25/use-cocoapod-to-manage-ios-lib-dependency/)

## 去Github上为你自己的框架新建一个仓库

- CocoaPods[项目的源码](https://github.com/CocoaPods/CocoaPods)在Github上管理,所以第一步我们需要创建一个属于自己的仓库。
- 克隆仓库到本地 ，使用命令行：`git clone 仓库地址`
- 在仓库所在目录新建项目

## 配置.podspec文件
该文件为Cocoapods依赖库的描述文件，每个Cocoapods依赖库必须有且仅有那么一个描述文件，简单地讲就是让CocoaPods搜索引擎知道你的代码的作者、版本号、源代码地址、依赖库等信息的文件。文件名称要和我们想创建的依赖库名称保持一致，我的LJIn-AppBrowser依赖库对应的文件名为LJIn-AppBrowser.podspec。

## 如何编写？
- 官方提供了一个模板并附有非常详细的注释说明。` pod spec create LJIn-AppBrowser` 该命令将在本目录产生一个名为`LJIn-AppBrowser.podspec`文件。但是打开创建完的文件你就会发现里面的东西太多了，很多都是我们不需要的。
- 所以通常我们可以拿别人的.podspec文件进行修改，文件内容如下图：
![Snip20160704_3.png](http://ww4.sinaimg.cn/mw690/6b7cdce2gw1f6utm1i1u3j20m107wwi3.jpg)

## 通过trunk推送pod spec文件
- 1、 注册trunk
 * 在注册trunk之前，我们需要确认当前的CocoaPods版本是否足够新。trunk需要pod在0.33及以上版本，如果你不满足要求，打开Terminal使用ruby的gem命令更新pod。
 * 更新结束后，我们开始注册trunk：`pod trunk register 616393956@qq.com 'longjianjiang'  --verbose` --verbose参数是为了便于输出注册过程中的调试信息。执行上面的语句后，点击邮件的链接就完成了trunk注册流程。使用下面的命令可以向trunk服务器查询自己的注册信息：`pod trunk me` 输出如下信息就表示你注册成功.
![Snip20160704_4.png](http://ww4.sinaimg.cn/mw690/6b7cdce2gw1f6utm3atwtj20fl03yaaz.jpg)

- 2、Push项目到Github
 * 终端中用git指令提交代码
```
 git add -A
 git commit -m "first commit for version 1.0.0"
 git push origin master
```
 * 终端中给版本打tag
```
git tag -a 0.0.1 -m 'Version 0.0.1’
git push origin 0.0.1
```
######只有确保了以上两点，CocoaPods才能更准确地找到你的仓库，这个时候我们的目录结构通常如下######

![Snip20160704_6.png](http://ww1.sinaimg.cn/mw690/6b7cdce2gw1f6utm3zukdj20bg07sq4j.jpg)

- 3、上传
 * 执行指令如下 PS：库中用到了第三方框架
`pod trunk push LJIn-AppBrowser.podspec`
然后你会得到下面的错误！

![Snip20160704_7.png](http://ww4.sinaimg.cn/mw690/6b7cdce2gw1f6utm2ce08j20fm02z753.jpg)

 所以当你的库中用到了第三方的框架，上传的时候得加上`--use-libraries`,这个时候上传的podspec文件转成json格式文件，如下图：

![Snip20160704_8.png](http://ww2.sinaimg.cn/mw690/6b7cdce2gw1f6utm4cotij20fl03pwfq.jpg)
此时可以试着搜索你的库，如果有结果则库已审核通过。

![Snip20160704_9.png](http://ww4.sinaimg.cn/mw690/6b7cdce2gw1f6utm2c6w4j20fx03kdgs.jpg)

## 尾巴
第一次写Markdown，感觉棒极了！
