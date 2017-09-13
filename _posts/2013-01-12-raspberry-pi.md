---
id: 437
title: Raspberry Pi 折腾笔记
date: 2013-01-12T19:18:43+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=437
permalink: /archives/437
views:
  - "3921"
categories:
  - Linux
  - 折腾
tags:
  - raspberry pi
---
在去年4月份就入手了一个 Raspberry Pi, 拿到后首先买了个 sd card, USB ttl 串口线 和 hdmi 转 DVI 的线，dd 装完 debian 后，玩了两天就把他装到了原先的盒子里再也没折腾。

这几个月突然心血来潮决定折腾一下，特别是看了 <a href="http://linuxtoy.org/archives/cool-ideas-for-raspberry-pi.html" target="_blank">34 个使用 Raspberry Pi 的酷创意</a>之后，决定先弄个 NAS 玩玩，实现白天远程连接下载，这样就不会浪费了我的 10 M 带宽的网络～

系统有了，可是怎么连上去呢？ 我住这条件简陋，无显示器，只能通过 USB ttl 串口连接了，成功连上后简单配置了一下拨号连接，此时它可以连接到 internet 了，但是问题来了，我的其他设备无法上网了，于是乎把我的那个只有一个 Wan 口功能弱到爆的 TP-LINK 无线路由器拿了出来(怎么突然想起年会大典了 ？)，我的笔记本连 wifi, raspberry pi 连到笔记本的网口上，配置了一下 SNAT 后，raspberry pi 成功联网，这样就更方便我配置 pi 了。

但是这样用起来太不方便，于是思前想后决定买个BUFFALO的路由器，花了我 120 大洋，路由器到手后第一件事就是刷 dd-wrt, <a href="http://www.dd-wrt.com/dd-wrtv2/down.php?path=downloads%2Fothers%2Feko%2FBrainSlayer-V24-preSP2%2F2013%2F01-01-2013-r20453%2Fbuffalo_whr_g300nv2/" target="_blank">firmware</a> , 以前在家的时候刷过，家里用的是 link sys 的路由器，很方便，可以远程 WOL。由于我得到的是公网 ip ,所以我可以远程连到路由器中，在路由器中配置好 ddns 和 DMZ or 端口转发后，我就可以远程连接到我的 Pi 中了。

哈哈，终于搞定了，可以远程下那些剧情场景简单，女主角漂亮的节目了！(新闻联播之类的，不要想歪哦 ～～) 哎，当初买 Raspberry Pi 的目的是研究内核的，结果却 &#8230;&#8230; 罪过啊 

BTW <a href="http://www.gaoqing.tv/other/other_6768.html" target="_blank">Concert YY</a> 终于下完了，<a href="http://v.youku.com/v_show/id_XNDU4MDM5ODcy.html" target="_blank">倾城</a> 真好听！！！

问题又来了，搞这种下载一定要有大容量快速存储设备的支持，本来想买个西数 1T 的硬盘，或者直接干脆就买个 NAS，但是一向勤俭节约的我怎可能买这么贵的东西呢，最终决定过年回家把以前买的那个移动硬盘拿来。哎，做人男，做男人更难啊，愁苦 &#8230;&#8230;

接下来再配置个 Samba NAS 也就简单的实现了，但是一向追求完美的我怎么可能满足呢？ 接下来准备再折腾下 <a href="http://wiki.xbmc.org/index.php?title=Raspberry_Pi" target="_blank">xbmc</a>.