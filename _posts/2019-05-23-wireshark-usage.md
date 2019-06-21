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

# rvictl

apple引进的一种流量分析的特性(remote virtual interface)。原理是Mac上建立一个虚拟网络接口作为iOS设备的网络栈，这样iOS设备的流量都会经过这个虚拟接口，所以在Mac上就可以使用wireshark工具进行分析。

rvi的唯一要求就是使用USB连接iOS设备到Mac上，连接完成以后有如下命令：

- rvictl -l ：显示当前活跃的设备
- rvictl -s UDID : 开始监听
- rvictl -x UDID : 结束监听

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
 
 - 查找特定的Flags

 `tcp.flags.reset == 1`, `tcp.flags.syn == 1`
