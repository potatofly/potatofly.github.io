---
title: "Android  低电提示需求"
layout: post
date: 2016-11-07 18:57
image: /assets/images/markdown.jpg
headerImage: false
tag: android
blog: true
author: potatofly
description: android framework 层添加低电量弹框提示
---

## 描述

最近项目上提出了一个低电弹框需求，而项目上又恰好缺少framework层的开发，很快项目经理就物色上了我ps:我来做驱动前一直做的是应用的开发，对framework层代码也了解一些。项目经理都说话了而且最近我头上问题也不是特别多就爽快的答应了。在一些前辈的帮助下我也圆满的完成了项目需求，现在在此简单梳理一下以做备忘。

### 项目需求分析

 {% highlight raw %}
设备重新开机后流程处理提示：
| 触发时机        | 电源无连接           | 电源有连接	|用户交互	|用户插电	|电源断开后	|
| --------------------|:------------------------:|:-------------------:|:------------------:|:------------------:|:-------------------:|
|>10%	|无弹窗	|无弹窗	|无弹窗	|无弹窗	|无弹窗	|
|==10%	|有弹窗	|无弹窗	|用户点击弹窗 周边后低电提示框消失	|用户插电后低 电提示框消失	|有弹窗	|
|>1% & <10%	|有弹窗	|无弹窗	|用户点击弹窗 周边后低电提示框消失，且不再提示	|用户插电后低 电提示框消失	|有弹窗	|
|<=1%	|有弹窗（电量 危急图示）	|无弹窗	|且收到系统电量广播后会再次提示。（此时已经濒临系统自动关机	|用户插电后危 电提示框消失	|有弹窗（电量 危急图示）	|
 {% endhighlight %}

### 项目思路

* 实现低电量弹框可以用自定义Dialog实现。
* 判定条件：通过BatteryProperties这个类我们可以获取电量信息（电量百分比）和电池充电状态通过查找/code-Litem/frameworks/base/core/java/android/BatteryManager.java我们可以得出充电状态对应的值是public static final int BATTERY_STATUS_CHARGING = 2
* 了解Dialog弹出时机通过网上查找资源和自己查看源码可以在/code-Litem/frameworks/base/services/core/java/com/android/server/BatteryService.java 里面的主要方法private void processValuesLocked(boolean force)中实现这一功能即在此方法中判断逻辑并调用方法。

### 编译重点
* mmm frameworks/base/core/res/ -B 先编译资源
* make update-api   再更新api
* mmm frameworks/base/services/  最后使用资源

###  具体实现

* 低电逻辑判断
 {% highlight raw %}
private void showWaringDialog() {

        if (mBatteryProps.batteryStatus == 2){  //如果在充电状态就不显示警告弹框
            if (mCriticalDialog != null && mCriticalDialog.isShowing()) mCriticalDialog.dismiss();
            if (mUrgencyDialog != null && mUrgencyDialog.isShowing()) mUrgencyDialog.dismiss();
			if (mRangeDialog!= null && mRangeDialog.isShowing()) mRangeDialog.dismiss();
            isNeedShowCriticalWarningDialog = true;
            isNeedShowUrgencyWarningDialog = true;
			isNeedShowRangeDialog = true;
            return;
        }

        if (mBatteryProps.batteryLevel == BATTERY_CRITICAL_LEVEL) {  //电量等于1%时弹出警告框
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    //if showing , no longer show again
                    if (mRangeDialog != null && mRangeDialog.isShowing())
                        mRangeDialog.dismiss();

                    if ((mCriticalDialog != null && mCriticalDialog.isShowing()) || !isNeedShowCriticalWarningDialog)
                        return;

                    mCriticalDialog = new WarningDialog(mContext, BATTERY_CRITICAL_LEVEL);
                    mCriticalDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
                    mCriticalDialog.show();
                    isNeedShowCriticalWarningDialog = false;
                }
            });
        }


		if (mBatteryProps.batteryLevel < BATTERY_URGENCY_LEVEL){  //电量大于1%小于10%时显示的低于10%电量警告框
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    if (mBatteryProps.batteryLevel <= BATTERY_CRITICAL_LEVEL){
                         return;
                     }
                    //if showing , no longer show again
                    if (mUrgencyDialog != null && mUrgencyDialog.isShowing()){
                        mUrgencyDialog.dismiss();
						isNeedShowUrgencyWarningDialog = true;
						}

                    if ((mRangeDialog != null && mRangeDialog.isShowing()) || !isNeedShowUrgencyWarningDialog || !isNeedShowRangeDialog)
                        return;            //这部分逻辑做过多次优化，当电量从10%降到低于10%的过程中
			//在用户没有交互取消弹框情况下弹框会从10%提示替换为低于10%的提示
                    mRangeDialog= new WarningDialog(mContext, BATTERY_BELOW_LEVEL);
                    mRangeDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
                    mRangeDialog.show();
                    isNeedShowRangeDialog= false;
                 }
             });
         } else if (mBatteryProps.batteryLevel == BATTERY_URGENCY_LEVEL){   //10%电量提示框
				mHandler.post(new Runnable() {
                @Override
                public void run() {

                    //if showing , no longer show again
                    if ((mUrgencyDialog != null && mUrgencyDialog.isShowing()) || !isNeedShowUrgencyWarningDialog)
                        return;
                    mUrgencyDialog = new WarningDialog(mContext, BATTERY_URGENCY_LEVEL);
                    mUrgencyDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
                    mUrgencyDialog.show();
                    isNeedShowUrgencyWarningDialog = false;
                 }
             });

         }

    }
 {% endhighlight %}

* 自定义Dialog
 {% highlight raw %}
class WarningDialog extends Dialog implements DialogInterface.OnClickListener {

        private int batteryLevel;

        WarningDialog(Context context, int batteryLevel) {
            super(context, com.android.internal.R.style.DialogWarning);
            this.batteryLevel = batteryLevel;
        }

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(com.android.internal.R.layout.battery_warning_dialog);
            ImageView icon = (ImageView) findViewById(com.android.internal.R.id.battery_warning_dilog_icon);
			TextView textView = (TextView) findViewById(com.android.internal.R.id.battery_warning_dilog_text);
			switch(batteryLevel){

				case BATTERY_CRITICAL_LEVEL:
					icon.setImageResource(com.android.internal.R.drawable.battery_percent_1);
					textView.setText(com.android.internal.R.string.percent_1_battery_warning);
					break;

				case BATTERY_BELOW_LEVEL:
					icon.setImageResource(com.android.internal.R.drawable.battery_percent_15);
					textView.setText(com.android.internal.R.string.percent_10_battery_warning);
					break;
				case BATTERY_URGENCY_LEVEL:
					icon.setImageResource(com.android.internal.R.drawable.battery_percent_15);
					textView.setText(com.android.internal.R.string.percent_15_battery_warning);
					break;

			}

        }

        @Override
        public void onClick(DialogInterface dialogInterface, int i) {
            dismiss();
        }

    }
 {% endhighlight %}

* 完整修改
 {% highlight raw %}
From 9447b4915136c026e9fddda0d0d7a7d185e357d9 Mon Sep 17 00:00:00 2001
From: 
Date: Mon, 07 Nov 2016 09:46:28 +0800
Subject: [PATCH] add function 10% battery warning dialog

Change-Id: I59e3f37aadb0fde810acb8f5356841c7b8fa319d
---

diff --git a/core/res/res/drawable-hdpi/battery_percent_1.png b/core/res/res/drawable-hdpi/battery_percent_1.png
new file mode 100644
index 0000000..b0be57a
--- /dev/null
+++ b/core/res/res/drawable-hdpi/battery_percent_1.png
Binary files differ
diff --git a/core/res/res/drawable-hdpi/battery_percent_15.png b/core/res/res/drawable-hdpi/battery_percent_15.png
new file mode 100644
index 0000000..aea8b8b
--- /dev/null
+++ b/core/res/res/drawable-hdpi/battery_percent_15.png
Binary files differ
diff --git a/core/res/res/drawable-hdpi/battery_warning_pop_up.png b/core/res/res/drawable-hdpi/battery_warning_pop_up.png
new file mode 100644
index 0000000..21e2750
--- /dev/null
+++ b/core/res/res/drawable-hdpi/battery_warning_pop_up.png
Binary files differ
diff --git a/core/res/res/drawable/warning_dailog_style.xml b/core/res/res/drawable/warning_dailog_style.xml
new file mode 100644
index 0000000..9d981d7
--- /dev/null
+++ b/core/res/res/drawable/warning_dailog_style.xml
@@ -0,0 +1,6 @@
+<?xml version="1.0" encoding="utf-8"?>
+<shape xmlns:android="http://schemas.android.com/apk/res/android"
+    android:shape="rectangle">
+    <solid android:color="#80000000" />
+    <corners android:radius="14dip"/>
+</shape>
diff --git a/core/res/res/layout/battery_warning_dialog.xml b/core/res/res/layout/battery_warning_dialog.xml
new file mode 100644
index 0000000..3e36220
--- /dev/null
+++ b/core/res/res/layout/battery_warning_dialog.xml
@@ -0,0 +1,26 @@
+<?xml version="1.0" encoding="utf-8"?>
+<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
+    android:layout_width="384dp"
+    android:layout_height="236dp"
+    android:orientation="vertical"
+    android:background="@drawable/battery_warning_pop_up"
+    android:gravity="center">
+        <ImageView
+            android:scaleType="fitXY"
+            android:id="@+id/battery_warning_dilog_icon"
+            android:layout_marginTop="52dp"
+            android:layout_marginBottom="37dp"
+            android:src="@drawable/battery_percent_1"
+            android:layout_width="wrap_content"
+            android:layout_height="wrap_content" />
+        <TextView
+            android:id="@+id/battery_warning_dilog_text"
+            android:textSize="22sp"
+            android:textColor="#ffffff"
+            android:layout_marginLeft="75dp"
+            android:layout_marginRight="75dp"
+            android:layout_marginBottom="51dp"
+            android:text="@string/percent_1_battery_warning"
+            android:layout_width="wrap_content"
+            android:layout_height="wrap_content" />
+</LinearLayout>
diff --git a/core/res/res/values/strings.xml b/core/res/res/values/strings.xml
index de442dd..6f3aab4 100644
--- a/core/res/res/values/strings.xml
+++ b/core/res/res/values/strings.xml
@@ -4188,4 +4188,7 @@
 
     <!-- MOD PR2514030 by liu.zhixiang 2016.7.29-->
     <string name="mismatchPin">No Disturb</string>
+    <string name="percent_1_battery_warning">1% battery remaining please connect charger</string>
+    <string name="percent_15_battery_warning">10% battery remaining please connect charger</string>
+    <string name="percent_10_battery_warning">Battery less than 10% please connect charger</string>
 </resources>
diff --git a/core/res/res/values/styles.xml b/core/res/res/values/styles.xml
index 4bad16d..4e6d12e 100644
--- a/core/res/res/values/styles.xml
+++ b/core/res/res/values/styles.xml
@@ -1386,4 +1386,10 @@
         <item name="padding">16dp</item>
     </style>
 
+    <style name="DialogWarning" parent="@android:style/Theme.Dialog">
+        <item name="android:windowFrame">@null</item>
+        <item name="android:windowNoTitle">true</item>
+        <item name="android:windowBackground">@android:color/transparent</item>
+    </style>
+
 </resources>
diff --git a/core/res/res/values/symbols.xml b/core/res/res/values/symbols.xml
index aa00909..5d994aa 100644
--- a/core/res/res/values/symbols.xml
+++ b/core/res/res/values/symbols.xml
@@ -2333,4 +2333,14 @@
   <java-symbol type="drawable" name="power_reboot_button" />
   <java-symbol type="layout" name="power_action_layout" />
 
+
+  <java-symbol type="layout" name="battery_warning_dialog" />
+  <java-symbol type="string" name="percent_1_battery_warning" />
+  <java-symbol type="string" name="percent_15_battery_warning" />
+  <java-symbol type="string" name="percent_10_battery_warning" />
+  <java-symbol type="id" name="battery_warning_dilog_icon" />
+  <java-symbol type="id" name="battery_warning_dilog_text" />
+  <java-symbol type="drawable" name="battery_percent_15" />
+  <java-symbol type="drawable" name="battery_percent_1" />
+  <java-symbol type="style" name="DialogWarning" />
 </resources>
diff --git a/services/core/java/com/android/server/BatteryService.java b/services/core/java/com/android/server/BatteryService.java
index 2dca4ba..fbd8388 100644
--- a/services/core/java/com/android/server/BatteryService.java
+++ b/services/core/java/com/android/server/BatteryService.java
@@ -22,14 +22,15 @@
 package com.android.server;
 
 import android.database.ContentObserver;
-import android.os.BatteryStats;
 
-import com.android.internal.app.IBatteryStats;
-import com.android.server.am.BatteryStatsService;
-import com.android.server.lights.Light;
-import com.android.server.lights.LightsManager;
 
-import android.app.ActivityManagerNative;
+import android.view.WindowManager;
+import android.widget.ImageView;
+import android.widget.TextView;
+import android.widget.Toast;
+import android.app.Dialog;
+import android.content.*;
+import android.os.*;
 import android.content.ContentResolver;
 import android.content.Context;
 import android.content.Intent;
@@ -52,6 +53,18 @@
 import android.os.UEventObserver;
 import android.os.SystemProperties;
 import android.os.UserHandle;
+import android.os.BatteryStats;
+
+
+
+import com.android.internal.app.IBatteryStats;
+import com.android.server.am.BatteryStatsService;
+import com.android.server.lights.Light;
+import com.android.server.lights.LightsManager;
+
+import android.app.ActivityManagerNative;
+
+
 import android.provider.Settings;
 import android.util.EventLog;
 import android.util.Log;
@@ -111,11 +124,17 @@
 
     // This should probably be exposed in the API, though it's not critical
     private static final int BATTERY_PLUGGED_NONE = 0;
+	private static final int BATTERY_CRITICAL_LEVEL = 1;
+ 	private static final int BATTERY_URGENCY_LEVEL = 10;
+	private static final int BATTERY_BELOW_LEVEL = 9;
 
     private final Context mContext;
     private final IBatteryStats mBatteryStats;
     private final Handler mHandler;
 
+	private WarningDialog mCriticalDialog;
+    private WarningDialog mUrgencyDialog;
+	private WarningDialog mRangeDialog;
     private final Object mLock = new Object();
 
     private BatteryProperties mBatteryProps;
@@ -151,6 +170,9 @@
 
     private Led mLed;
 
+	private boolean isNeedShowCriticalWarningDialog = true;
+    private boolean isNeedShowUrgencyWarningDialog = true;
+	private boolean isNeedShowRangeDialog = true;
     private boolean mSentLowBatteryBroadcast = false;
 
     private boolean mIPOShutdown = false;
@@ -300,7 +322,7 @@
     private void shutdownIfNoPowerLocked() {
         // shut down gracefully if our battery is critically low and we are not powered.
         // wait until the system has booted before attempting to display the shutdown dialog.
-        if ((mBatteryProps.batteryLevel <= 1) && !isPoweredLocked(BatteryManager.BATTERY_PLUGGED_ANY)) {
+        if ((mBatteryProps.batteryLevel <= 0) && !isPoweredLocked(BatteryManager.BATTERY_PLUGGED_ANY)) {
             mHandler.post(new Runnable() {
                 @Override
                 public void run() {
@@ -352,6 +374,83 @@
                 mLastBatteryProps.set(props);
             }
         }
+    }
+
+	private void showWaringDialog() {
+
+        //if charging dismiss dialog and not show warning
+
+		assert mBatteryProps !=null;
+
+        if (mBatteryProps.batteryStatus == 2){
+            if (mCriticalDialog != null && mCriticalDialog.isShowing()) mCriticalDialog.dismiss();
+            if (mUrgencyDialog != null && mUrgencyDialog.isShowing()) mUrgencyDialog.dismiss();
+			if (mRangeDialog!= null && mRangeDialog.isShowing()) mRangeDialog.dismiss();
+            isNeedShowCriticalWarningDialog = true;
+            isNeedShowUrgencyWarningDialog = true;
+			isNeedShowRangeDialog = true;
+            return;
+        }
+
+        if (mBatteryProps.batteryLevel == BATTERY_CRITICAL_LEVEL) {
+            mHandler.post(new Runnable() {
+                @Override
+                public void run() {
+                    //if showing , no longer show again
+                    if (mRangeDialog != null && mRangeDialog.isShowing())
+                        mRangeDialog.dismiss();
+
+                    if ((mCriticalDialog != null && mCriticalDialog.isShowing()) || !isNeedShowCriticalWarningDialog)
+                        return;
+
+                    mCriticalDialog = new WarningDialog(mContext, BATTERY_CRITICAL_LEVEL);
+                    mCriticalDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
+                    mCriticalDialog.show();
+                    isNeedShowCriticalWarningDialog = false;
+                }
+            });
+        }
+
+
+		if (mBatteryProps.batteryLevel < BATTERY_URGENCY_LEVEL){
+            mHandler.post(new Runnable() {
+                @Override
+                public void run() {
+                    if (mBatteryProps.batteryLevel <= BATTERY_CRITICAL_LEVEL){
+                         return;
+                     }
+                    //if showing , no longer show again
+                    if (mUrgencyDialog != null && mUrgencyDialog.isShowing()){
+                        mUrgencyDialog.dismiss();
+						isNeedShowUrgencyWarningDialog = true;
+						}
+
+                    if ((mRangeDialog != null && mRangeDialog.isShowing()) || !isNeedShowUrgencyWarningDialog || !isNeedShowRangeDialog)
+                        return;
+
+                    mRangeDialog= new WarningDialog(mContext, BATTERY_BELOW_LEVEL);
+                    mRangeDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
+                    mRangeDialog.show();
+                    isNeedShowRangeDialog= false;
+                 }
+             });
+         } else if (mBatteryProps.batteryLevel == BATTERY_URGENCY_LEVEL){
+				mHandler.post(new Runnable() {
+                @Override
+                public void run() {
+
+                    //if showing , no longer show again
+                    if ((mUrgencyDialog != null && mUrgencyDialog.isShowing()) || !isNeedShowUrgencyWarningDialog)
+                        return;
+                    mUrgencyDialog = new WarningDialog(mContext, BATTERY_URGENCY_LEVEL);
+                    mUrgencyDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
+                    mUrgencyDialog.show();
+                    isNeedShowUrgencyWarningDialog = false;
+                 }
+             });
+
+         }
+
     }
 
     private void processValuesLocked(boolean force) {
@@ -407,6 +506,7 @@
             // Should never happen.
         }
 
+		showWaringDialog();
         shutdownIfNoPowerLocked();
         shutdownIfOverTempLocked();
 
@@ -936,6 +1036,48 @@
         }
     }
 
+	class WarningDialog extends Dialog implements DialogInterface.OnClickListener {
+
+        private int batteryLevel;
+
+        WarningDialog(Context context, int batteryLevel) {
+            super(context, com.android.internal.R.style.DialogWarning);
+            this.batteryLevel = batteryLevel;
+        }
+
+        @Override
+        protected void onCreate(Bundle savedInstanceState) {
+            super.onCreate(savedInstanceState);
+            setContentView(com.android.internal.R.layout.battery_warning_dialog);
+            ImageView icon = (ImageView) findViewById(com.android.internal.R.id.battery_warning_dilog_icon);
+			TextView textView = (TextView) findViewById(com.android.internal.R.id.battery_warning_dilog_text);
+			switch(batteryLevel){
+
+				case BATTERY_CRITICAL_LEVEL:
+					icon.setImageResource(com.android.internal.R.drawable.battery_percent_1);
+					textView.setText(com.android.internal.R.string.percent_1_battery_warning);
+					break;
+
+				case BATTERY_BELOW_LEVEL:
+					icon.setImageResource(com.android.internal.R.drawable.battery_percent_15);
+					textView.setText(com.android.internal.R.string.percent_10_battery_warning);
+					break;
+				case BATTERY_URGENCY_LEVEL:
+					icon.setImageResource(com.android.internal.R.drawable.battery_percent_15);
+					textView.setText(com.android.internal.R.string.percent_15_battery_warning);
+					break;
+
+			}
+
+        }
+
+        @Override
+        public void onClick(DialogInterface dialogInterface, int i) {
+            dismiss();
+        }
+
+    }
+
     private final class LocalService extends BatteryManagerInternal {
         @Override
         public boolean isPowered(int plugTypeSet) {
 {% endhighlight %}











 

