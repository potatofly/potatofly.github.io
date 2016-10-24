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

---

### Android 编译选项 eng、user、userdebug 

* eng：debug 版本（工程版本）
* user: release 版本（最终用户版本）
* userDebug版本：部分debug版本（调试测试版本）

 {% highlight raw %}
Google 官方描述: USER/USERDEBUG/ENG 版本的差异, 参考alps/build/core/build-system.html 的详细说明
eng This is the default flavor. A plain make is the same as make eng.
* Installs modules tagged with: eng, debug, user, and/or development.
* Installs non-APK modules that have no tags specified.
* Installs APKs according to the product definition files, in addition to tagged APKs.
* ro.secure=0
* ro.debuggable=1
* ro.kernel.android.checkjni=1
* adb is enabled by default.
* Setupwizard is optional
user make user
This is the flavor intended to be the final release bits.
* Installs modules tagged with user.
* Installs non-APK modules that have no tags specified.
* Installs APKs according to the product definition files; tags are ignored for APK modules.
* ro.secure=1
* ro.debuggable=0
* adb is disabled by default.
* Enable dex pre-optimization for all TARGET projects in default to speed up device first boot-up
userdebug make userdebug
The same as user, except:
* Also installs modules tagged with debug.
* ro.debuggable=1
* adb is enabled by default. 
 {% endhighlight %}




