
# 关闭 dex2oat

export WITH_DEXPREOPT=false

# Android 13 添加模拟器支持
1. 打开 build/make/target/product/AndroidProducts.mk
2. 在 COMMON_LUNCH_CHOICES 选项中添加 sdk_phone_x86_64-eng \ 
       后面的 -eng 表示工程师版，最后的 反斜线 不要忘记添加。建议用 userdebug 版本，eng 版本某些模块有问题。
3. 可以看到在 build/make/target/product 文件夹中有 sdk_phone_x86_64.mk 等一系列文件只是没添加到 lunch 中。

# Android 13 预编译中加入自己的 app 编译失败的问题
	
	在 build/make/target/product/handheld_system 中加上自己的 app
	soucre build/envsetup.sh 后加上
	export DISABLE_ARTIFACT_PATH_REQUIREMENTS="true"


# 源码导入 Android Studio

1. mmm development/tools/idegen 
2. 运行 development/tools/idegen/idegen.sh // 生成  android.iml 和 android.ipr 两个工程配置文件
3. Android Studio 打开 android.ipr 等待索引完成。

# 编译步骤：

1. source build/envsetup.sh
## 选择需要的环境，可以不带参数，直接使用 lunch 来看看都提供了哪些环境
2. lunch aosp_arm64-eng
3. make -j8   或者用  make framework-minus-apex 编译特定的模块
4. emulator -writable-system
5. mmm package_path  或者进入到要编译的目录 mm ，mmma 和 mma 都会编译依赖项
6. make snod  重新生成 system.img 消耗时间很少，类似 WinCE 的 makeing 过程，如果只修改了一些数据文件（如音乐、视频）时比较有用。
7. make cts 编译 CTS 套机，编译结果放在 out 目录对应的 data/app 目录下，CTS 测试时有用。
8. make installclean 清除out目录下对应板文件夹中的内容，也就是相当于make clean，通常如果改变了一些数据文件（如去掉）、最好执行以下make installclean，否则残留在out目录下的还会被打包进去。
9. mm/mm -B 在修改了的目录下执行这条命令，就能智能地进行编译，输出的文件在通过adb推送到目标机，可以很方便地调试。
10. make bootimage 用这条命令可以生成boot.img,这个镜像文件中包含Linux Kernel，Ram disk，生成的boot.img只能通过fastboot进行烧写，这在只修改了Linux内核的时候有用。
11. make systemimage 编译完后，out\target\product\XXX\system.img会更新。

## 编译 framework.jar 模块

	模拟器要通过 emulator -writable-system 启动，所以模拟器至少应该是 userdebug 模式的

1. make framework-minus-apex
2. adb root
	成功后显示： restarting adbd as root
3. adb disable-verity
	成功后显示： verity is already disabled
4. adb remount
	成功后显示： remount succeeded
5. adb push xxxx/framework.jar /system/framework

adb root && adb disable-verity && adb reboot && adb root && adb remount

### 重启 zygote 方式

6. adb shell
7. rm -r /system/framework/oat
8. rm -r /system/framework/arm
9. rm -r /system/framework/arm64
10. exit

	上面这 5 步可以尝试用 adb shell sync 代替，看行不行？

// 重启 zygote
11. adb shell stop
12. adb shell start

### 直接重启模拟器

6. adb shell sync
7. adb reboot

# 切换 aosp 环境
aosp 切换到了 repo init -u https://android.googlesource.com/platform/manifest -b remotes/origin/android-10.0.0_r17 分支
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-10.0.0_r17

# 错误解决
编译环境：Ubuntu 20.04 LTS
1.当编译完成后 执行 emulator 时，出现以下错误：
	emulator: WARNING: encryption is off
	emulator: ERROR: AdbHostServer.cpp:102: Unable to connect to adb daemon on port: 5037
无法在端口 5037 连接到 adb daemon 。那么 “端口 5037“ 可能被占用了，如果被占用杀死占用的进程；也有可能 adb daemon 服务未启动，这里推荐先通过 adb start-server 尝试启动下，如果是该原因造成的则会打印下面的语句：
	* daemon not running; starting now at tcp:5037
	* daemon started successfully
	然后，再执行 emulator 命令就行了。
	
# 桌面应用开发

如果想开发桌面应用，只需APK的AndroidMainfest.xml中修改如下即可：

	<activity
	 android:name="com.yly.launcher.activity.HomeActivity"
	 android:label="@string/app_name"
	 android:launchMode="singleTask"
	 android:clearTaskOnLaunch="true"
	 android:stateNotNeeded="true"
	 android:taskAffinity=""
	 android:windowSoftInputMode="adjustNothing">
	 <intent-filter>
	 <action android:name="android.intent.action.VIEW" />
	 <action android:name="android.intent.action.MAIN" />
	 <category android:name="android.intent.category.HOME" />
	 <category android:name="android.intent.category.DEFAULT" />
	 </intent-filter>
	</activity>

## 植入并替换系统应用

在开发完成将桌面应用编译为apk后。将apk文件拷贝到对应的CustomLauncher目录下并创建Android.mk文件。

	LOCAL_PATH := $(call my-dir)
	include $(CLEAR_VARS)
	LOCAL_MODULE := CustomLauncher
	LOCAL_MODULE_TAGS := optional
	LOCAL_SRC_FILES := $(LOCAL_MODULE).apk

该句的意思是说，系统原生的Launcher3等桌面应用将不会被编译进系统，而被这个应用给替换了。

	LOCAL_OVERRIDES_PACKAGES := Home Launcher2 Launcher3 
	LOCAL_MODULE_CLASS := APPS
	LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
	LOCAL_CERTIFICATE := PRESIGNED
	LOCAL_PRIVILEGED_MODULE := true
	LOCAL_DEX_PREOPT := false
	include $(BUILD_PREBUILT)

然后在 /device/平台/.../项目目录下找到相应的版本，打开其中的 “项目名.mk” 文件， 添加：

	// 编译项目时，会在对应的版本中添加上这个apk。
	PRODUCT_PACKAGES +=CustomLauncher

## 清除之前的缓存

调用如下命令可以清除所有旧模块/映像的输出目录（不会删除所有输出目录）：

	make installclean

## 编译并打包

在修改完成后，接下来重新编译system.img镜像并烧写：

	//第二次编译只会编译修改过的模块
	m
	adb reboot bootloader
	fastboot falsh system system.img
	fastboot reboot
	
## AOSP 常见模块及路径

![AOSP 常见模块及路径](./AOSP%20常见模块及路径.png)
