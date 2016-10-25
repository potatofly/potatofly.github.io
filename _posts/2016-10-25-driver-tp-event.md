---
title: "初步检查TouchPanel 驱动是否正常工作"
layout: post
date: 2016-10-25 17:50
image: /assets/images/markdown.jpg
headerImage: false
tag: driver
blog: true
author: potatofly
description: TP 报点
---

## 描述

使用 adb shell getevent 命令检查 TouchPanel是否正常工作

### 解决

1. 使用如下命令找出 mtk-tpd 对应的event设备

* adb shell getevent -i

2. 使用此设备读取 TouchPanel上报的事件

* adb shell get event /dev/input/event3
* 每一行有三个数值,通过查询
* /code-Litem/kernel-3.18/include/uapi/linux/input.h
* 可以得到其含义,举例如下:

 {% highlight raw %}
1=EV_KEY	0x14a(330) = BTN_TOUCH				/*Down=1, Up=0*/ 
3=EV_ABS	0x30(48) = ABS_MT_TOUCH_MAJOR		/* Major axis of touching ellipse */
3=EV_ABS	0x31(49)=ABS_MT_TOUCH_MINOR		/* Minor axis (omit if circular) */	
3=EV_ABS	0x3a(58)=ABS_MT_PRESSURE			/* Pressure on contact area */	
3=EV_ABS	0x35(53) = ABS_MT_POSITION_X		/* Center X touch position */
3=EV_ABS	0x36(54) = ABS_MT_POSITION_Y		/* Center Y touch position */
3=EV_ABS	0x39(57) = ABS_MT_TRACKING_ID		/* Unique ID of initiated contact */ 
0=EV_SYN	0x2(2) = SYN_MT_REPORT	0
0=EV_SYN	0x0(0) = SYN_REPORT	0				/* 同步上报信息*/
 {% endhighlight %}
