---
layout: post
title:  "Hammerspoon 小记"
date:   2019-01-19
excerpt:  "本文是笔者日常使用 Hammerspoon 完成一些摆脱鼠标操作的记录"
tag:
- Tutorial
comments: true
---

Hammerspoon 是一款开源的自动化工具，允许我们通过Lua脚本来操纵操作系统提供的操作，比如窗口管理，鼠标移动等操作，具体可以参看[API](http://www.hammerspoon.org/docs/)。而且 Hammerspoon 提供了自定义快捷键绑定操作，也就是说我们可以使用键盘完成原先需要鼠标点击的一些操作。

启动Hammerspoon 后会自动加载配置文件 `~/.hammerspoon/init.lua`, 我们自定义的一些操作也写在这里。

# 有道字典

笔者经常使用有道字典查单词，但是查完单词后想听读音，因为此时双手都在键盘上，所以每次都得切换到鼠标，很麻烦。看到了Hammerspoon，所以想到使用键盘操作鼠标，移动到读音的按钮，点击，完成听语音的操作。

下面笔者给出自己写的快捷绑定的方法, 基本能完成需求，不过也需要按几次键盘，属于可以接受的范围:

{% highlight lua %}
local hotkey = require "hs.hotkey"
local mouse = require "hs.mouse"
local app = require "hs.application"
local tap = require "hs.eventtap"

-- mouse move
local mouse_move_value = 5

function moveMouseToYouDao()
    youdaoApp = app.frontmostApplication()
    win = youdaoApp:allWindows()[1]
    top_left = win:topLeft()
    top_left.x = top_left.x + 250
    top_left.y = top_left.y + 150
    tap.leftClick(top_left) 
end

hs.hotkey.bind({"ctrl"}, "S", function()
	moveMouseToYouDao()
end)

hs.hotkey.bind({"ctrl"}, "T", function()
	p = hs.mouse.getAbsolutePosition()
	p.y = p.y - mouse_move_value
	hs.mouse.setAbsolutePosition(p)
end)

hs.hotkey.bind({"ctrl"}, "B", function()
	p = hs.mouse.getAbsolutePosition()
	p.y = p.y + mouse_move_value
	hs.mouse.setAbsolutePosition(p)
end)

hs.hotkey.bind({"ctrl"}, "L", function()
	p = hs.mouse.getAbsolutePosition()
	p.x = p.x - mouse_move_value
	hs.mouse.setAbsolutePosition(p)
end)

hs.hotkey.bind({"ctrl"}, "R", function()
	p = hs.mouse.getAbsolutePosition()
	p.x = p.x + mouse_move_value
	hs.mouse.setAbsolutePosition(p)
end)

hs.hotkey.bind({"ctrl"}, "C", function()
    p = hs.mouse.getAbsolutePosition()
    tap.leftClick(p)
end)
{% endhighlight %}

ctrl S 首先将鼠标定位到有道字典默认窗口的左上方，ctrl TBLR, 上下左右移动鼠标到播放按钮，ctrl C点击鼠标播放声音。