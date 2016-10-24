---
title: "Android userdebug 版本无法remount问题"
layout: post
date: 2016-10-24 16:57
image: /assets/images/markdown.jpg
headerImage: false
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
 
---

### 问题解决

根据上述官方文档我们可以知道userdebug版本拥有的权限比eng版本小，因此出现无法remount的问题。问题的解决也比较方便只要执行两句adb命令就能够解决上述问题。

* 首先执行 adb disable-verity 命令
* 其次执行 adb reboot 重启

好了两行命令解决adb remount 问题，另外特别提醒一下如果遇到无法 adb disable-verity 那么可能是你的 adb 版本太 out 了，你可以从网上下最新的，这里就不给出链接了。如果你编译过整套安卓源代码那么可以直接从 编译 out 目录下获取最新的 adb 拿博主的 MTK 平台代码举例：

* /code-Litem/out/host/linux-x86/bin$  adb 就在这个目录下




