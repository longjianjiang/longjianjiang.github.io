---
layout: post
title:  "【Cocos2D + Lua】第一个Cocos2D + Lua项目"
date:   2017-05-12
excerpt: "Cocos2D，Lua来进行开发的，因为自己也是第一次接触，所以借此机会，记录下自己开发的经历。（PS：我做的是iOS平台)"
tag:
- Cocos2D
comments: true
---

>Cocos2D，Lua来进行开发的，因为自己也是第一次接触，所以借此机会，记录下自己开发的经历。（PS：我做的是iOS平台）

### 环境安装
- 首先去官网下载[Cocos2D-X](http://www.cocos.com/download)；
- 解压；进入cocos2d-x-3.15文件夹下；
- 执行Python脚本：./setup.py
- 执行完之后，最后一句应该是如下所示：


![Snip20170512_1.png]({{site.url}}/assets/images/blog/cocos2D_first_project_1.png)

此时我们只需要按照它的要求，继续执行给出的指令即可

- 现在我们就可以使用命令来创建一个Cocos2D-X的项目了，只需要执行以下指令 :
>`cocos new firstCocos -p com.longjianjiang.firstCocos -l lua -d "/Users/longjianjiang/Desktop"`
`firstCocos` -> 项目的名字
`-p com.longjianjiang.firstCocos` -> 指定包名
`-l lua` -> 指定语言（PS：有JavaScript，Cpp， Lua三种）
`-d "/Users/longjianjiang/Desktop"`  -> 指定项目存放的位置

### 项目结构
项目创建之后，可以从下图路径下用Xcode打开：

![屏幕快照 2017-05-12 22.56.46.png]({{site.url}}/assets/images/blog/cocos2D_first_project_2.png)

之后我们任务就是在下图的Resources路径下进行Lua开发

![屏幕快照 2017-05-12 22.58.34.png]({{site.url}}/assets/images/blog/cocos2D_first_project_3.png)


### 尾巴
开发游戏谁用SpriteKit只写iOS平台的？😂

