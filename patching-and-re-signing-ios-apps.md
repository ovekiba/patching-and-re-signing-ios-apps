# 如何对iOS App进行打补丁和重新签名

参考: [Patching and Re-Signing iOS Apps](http://www.vantagepoint.sg/blog/85-patching-and-re-signing-ios-apps)

作者: [kiba](https://github.com/ovekiba/)



--------



## 一、硬件环境

> 1. 一台装有macOS的电脑

> 2. 一台iOS设备



## 二、准备

### 1. XCode

从App Store安装即可。


### 2. optools

```shell
$ git clone https://github.com/alexzielenski/optool.git

$ cd optool/

$ git submodule update --init --recursive

$ xcodebuild
```
生成的optool在目录build/Release下。


### 3. ios-deploy

#### a. 先安装homebrew
```shell
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

#### b. 再安装npm
```shell
$ brew install npm
```

#### c. 最后安装ios-deploy
```shell
$ npm install -g ios-deploy
```


### 4. frida

#### a.首先到pyhton的官方网站下载python3并安装

#### b. 安装frida
```shell
$ pip3 install frida
```

> 第一次安装frida时，一般会失败，提示:
> ```shell
> urllib2.URLError: <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:749>
> ```
> 这是因为下载软件安装包时，https的SSL证书验证失败！
> 
> **解决方法**：运行python3安装目录中的Install Certificates.command文件即可，该文件的路径在：
> ```shell
> /Applications/Python 3.6/Install Certificates.command
> ```
> 执行完了后，再安装frida就不会报错了。


### 5. 用于签名的证书码

```shell
$ security find-identity -p codesigning -v
1) 8ABAFA6FFABE09DD00688EF32841F4C975A68D01 "iPhone Developer: xxxxxx@gmail.com (FFFDF2W2QC)"
```
结果中的**8ABAFA6FFABE09DD00688EF32841F4C975A68D01**即是后面要用到的用于签名的证书码


### 6. FridaGadget.dylib

#### a. 从 https://build.frida.re/frida/ios/lib/FridaGadget.dylib 下载此文件


#### b. 为FridaGadget.dylib签名
```shell
$ codesign -f -s 8ABAFA6FFABE09DD00688EF32841F4C975A68D01 FridaGadget.dylib
```

> 如果为FridaGadget.dylib签名命令的最后提示有"main executable failed strict validation"，说明签名失败，一个有可能的原因就是文件未下载完整，需重新下载文件。


### 7. UnCrackable_Level1.ipa
从 https://raw.githubusercontent.com/OWASP/owasp-mstg/master/Crackmes/iOS/Level_01/UnCrackable_Level1.ipa 下载此文件。



## 三、操作步骤

### 1. 获取embedded.mobileprovision文件

#### a. 打开XCode，新建一个iOS设备类型的应用，新建完成后，在项目属性中，将**General -> Signing -> Team**设置为自己开发账号（如果没有则需要添加，输入自己的iCloud账号密码即可）。

#### b. 将iOS设备用USB线连接到macOS电脑上，将XCode项目运行的设备设置为该iOS设备，点击运行，待应用程序成功在iOS设备上运行时，找到XCode项目Products文件夹下文件名以.app结尾的文件，右键点击Show in Finder，然后再在app文件上右键点击Show Package Contents，弹出新窗口中的embedded.mobileprovision文件，即是需要的文件，将此文件复制保存到其他文件夹，保存成功后，就可以停止在iOS设备上运行应用程序并退出XCode了。

> * 如果Show Package Contents后，文件夹中没有embedded.mobileprovision文件，可能是因为程序没有在iOS设备上运行，而是在模拟器中运行的。这时需要检查程序运行的设备是否是连接电脑的iOS设备。
> 如果是第一次在该iOS设备上安装运行自己创建应用程序，需要先在iOS设备上的**设置 -> 通用*中选择信任证书。


### 2. 处理embedded.mobileprovision文件

```shell
$ security cms -D -i embedded.mobileprovision > profile.plist

$ /usr/libexec/PlistBuddy -x -c 'Print :Entitlements' profile.plist > entitlements.plist
```

最后得到的entitlements.plist文件内容与下面的内容相似：
```shell
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>application-identifier</key>
	<string>VUDNES7LPD.com.ovesec.Homepwner</string>
	<key>com.apple.developer.team-identifier</key>
	<string>VUDNES7LPD</string>
	<key>get-task-allow</key>
	<true/>
	<key>keychain-access-groups</key>
	<array>
		<string>VUDNES7LPD.*</string>
	</array>
</dict>
</plist>
```
需要注意的是文件中"com.ovesec.Homepwner"，后面的步骤会用到。


### 3. 对UnCrackable_Level1.ipa应用进行修改和重签名

```shell
$ unzip UnCrackable_Level1.ipa

$ cp FridaGadget.dylib Payload/UnCrackable\ Level\ 1.app/

$ optool install -c load -p "@executable_path/FridaGadget.dylib" -t Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
Found FAT Header
Found thin header...
Found thin header...
Load command already exists
Successfully inserted a LC_LOAD_DYLIB command for arm
Load command already exists
Successfully inserted a LC_LOAD_DYLIB command for arm64
Writing executable to Payload/UnCrackable Level 1.app/UnCrackable Level 1...

$ cp embedded.mobileprovision Payload/UnCrackable\ Level\ 1.app/

$ /usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier com.ovesec.Homepwner" Payload/UnCrackable\ Level\ 1.app/Info.plist

$ rm -fr Payload/UnCrackable\ Level\ 1.app/_CodeSignature/

$ /usr/bin/codesign --force --sign 8ABAFA6FFABE09DD00688EF32841F4C975A68D01 --entitlements entitlements.plist Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
Payload/UnCrackable Level 1.app/UnCrackable Level 1: replacing existing signature
```


### 4.运行重签名后的应用程序

#### a. 将iOS设备连接至电脑


#### b. 运行重签名后的应用程序
```shell
$ ios-deploy --debug --bundle Payload/UnCrackable\ Level\ 1.app/
```

> 如果b步骤命令最后输出了下面的信息：
> ```shell
> Unable to locate DeviceSupport directory. This probably means you don't have Xcode installed, you will need to launch the app manually and logging output will not be shown!
> ```
> 则说明运行失败，原因是iOS设备的系统太新了，XCode没有设备支持的相关文件。
>
> **解决方法**: 查看iOS设备的系统版本，发现是10.3.2 (14F89)，而目录/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/DeviceSupport中只有支持到10.3.1 (14E8301)的版本。
> 10.3.2和10.3.1的版本变化不大，可以使用10.3.1来启动10.3.2的设备。
> ```shell
> $ sudo ln -s /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/DeviceSupport/10.3.1 \(14E8301\)/ /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/DeviceSupport/10.3
> ```


#### c. 使用frida来验证
```shell
$ sudo frida-ps -U
PID  Name
----  ------
1423  Gadget
```
