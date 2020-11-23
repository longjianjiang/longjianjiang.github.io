# 前端基础笔记

本文记录笔者整理的一些前端的知识。

## dom

所谓dom(document object model) 其实是用来操作html的api。整个html可以被看作是一个dom树，<div></div> 这些标签都可以看作是树的一个节点。

document 是dom树的根节点，包含了标题(document.title)，url(document.URL)等属性，子节点(document.getElementById("xx"))，可以直接在JS中访问。

## bom

所谓bom(browser object model) 是用来控制浏览器的api。

location 是bom中的一个对象，比如常用的`location.href = "xx"`。

window 也是bom中的一个对象。
