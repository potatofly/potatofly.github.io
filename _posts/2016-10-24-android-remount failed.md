---
title: "Android userdebug 版本无法remount问题"
layout: post
date: 2016-10-24 16:57
image: /assets/images/markdown.jpg
headerImage: true
tag: android
blog: true
author: potatofly
description: remount failed describe
---

## 描述

    最近调试安卓平板时遇到一个问题，每次刷服务上的daily版本时，adb root 总是无法adb remount。但是刷本地版本时能够remount。通过网上寻找答案发现问题出现的主要原因是公司服务器上每天release的是userdebug版本而我本地编译的eng版本。下面简单来说明一下各版本区别，以及问题的解决。
    大家看接口
