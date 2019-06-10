---
layout: post
title:  "Wireshark 小记"
date:   2019-05-23
excerpt:  "本文是笔者使用Wireshark时记录下面的一些操作，方便日后查看" 
tag:
- Tutorial
comments: true
---

> 本文是笔者使用Wireshark时记录下面的一些操作，方便日后查看。

# 过滤

## ip

- source

`ip.src == ''`

- destination

`ip.dst == ''`

- either

`ip.addr == ''`

## tcp

- 查找`Keep-Alive`包

`tcp.analysis.keep_alive`
 
