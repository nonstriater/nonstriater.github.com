---
layout: post
title: "iOS包依赖管理工具cocoapods的安装和使用"
description: ""
category: "工具"
tags: []
---


###使用cocoaPods有哪些好处？

cocoaPods是第三方库管理工具，他可以帮我们导入第三方源码库的文件到项目中，还会自动的帮我们导入依赖的framework，设置可能需要的编译参数（如：-fno-objc-arc 或 -all_load,-objc等）


<br/>
###cocoaPods的安装和升级

CocoaPods是使用 ruby gem 分发的，，在mac下只需要运行简单的命令就可安装cocoaPods：
```sh
 sudo gem install cocoaPods
```
这个命令要写系统文件，普通用户没有权限，需要用sudo切换到超级用户，才可以执行  

升级：
```sh
 sudo gem update cocoapods
```

安装后执行下
```sh
    pod setup
```
这条命令会将Spec项目复制到当前用户的.cocoapods\master目录下,以后的查找、安装使用都是基于该本地目录的.

<br/>
###cocoaPods使用

现在在你的项目中使用cocoaPods，如果是一个已有的项目：

<br/>
#### 在项目根目录创建Podfile，在里面加入第三方库

```sh
platform :ios, '6.0'
pod 'JSONKit', '~> 1.4' 
pod 'Reachability', '~> 3.0.0' 
pod 'ASIHTTPRequest' 
pod 'RegexKitLite'
```

<br/>
#### 执行 pod install

接下来，cocoaPods将为你下载第三方源码库文件和自动设置编译依赖环境。

注意，每次更改了Podfile文件，你需要重新执行一次pod install命令 或 pod update。

<br/>
#### 常用的cocoaPods命令

```sh
pod search       在pod中心搜索第三方库
pod list             列出pod 中心所有的第三方库，截止目前，已经有2千多个第三方库提交到pod center
pod podfile-info  可以列出项目中pod管理的第三方库
pod outdated       检查是否有更新，有的话，我们可以使用pod update或pod install 来更显我们的库
```

pod repo  spec项目管理
Commands:

    * add      Add a spec repo.
    * lint     Validates all specs in a repo.
    * remove   Remove a spec repo
    * update   Update a spec repo.



<br/>
#### 使用cocoaPods后项目的变化   
现在来看看cocoaPods带来的变化（cocoapods的原理）：

- 打开工程现在使用.xcworkplace,而不是以前的.xcodeproj 文件
- 多出了一个Pods的项目，该项目会为每一个依赖创建一个target，也就是第三方源码库的管理都移到了pods项目中
- 多了一个podfile.lock文件
- 在你自己的project多了几个东西：frameworks里面多了一个libPods.a，我们的project依赖这个静态库;还有一个Pods.xcconfig配置文件,这个文件用来设置编译时的依赖和参数
- 在pods项目中，CocoaPods提供了一个名为Pods-resources.sh脚本，该脚本在每次项目编译的时候都会执行，将第三方库的各种资源文件复制到目标目录中。


<br/>
###使用过程的几个问题

**问题1 ：我们在项目中使用头文件的时候（import），没有提示且找不到头文件？**

设置头文件的目录，在项目的Target的里设置一下”User header Search Paths“：${SRCROOT},后面选上recursive。

<br/>
**问题2：The platform of the target `Pods` (iOS 4.3) is not compatible with 。。。。**

这是因为一些第三方库最低版本号大于Pods的版本号不兼容导致的，打开项目中 Podfile 文件   修改其 iOS平台为对应平台，如iOS5.0，podfile没写时默认是4.3.

<br/>
**问题3：Undefined symbols for architecture armv7:
"_OBJC_METACLASS_$_AFHTTPClient", referenced from。。**

这是因为AFNetworking在2.0x以后去掉了AFHTTPClient类，加入了对NSURLSession的支持。在podfile中修改afnetworking的版本为1.3.3之后，任然出现了问题：

暂时将afnetworking移除到pods管理。【待解决】


<br/>
**问题4：Unknown class 【GMGridView】 in Interface Builder** file。在sb布局里面用到了GMGridView类。

发生这种错误一般都是在xib文件里调用Framework或者是静态库存的Class, 这样直接调用是会出错的，需要在目标Target的Build Settings里设置"Other Linker Flags"成"-all_load" "-ObjC"


命令加上”--verbose“，可以看到命令的执行过程（log信息）。在pod install --verbose，可以看到创建pod project的整个过程：

Analyzing dependencies
Downloading dependencies
Generating Pods project
Integrating client project


<br/>
**问题5：对于没有开源的第三方SDK的处理**

例如微信sdk这种，这个pod就没法管理，只能我们自己管理配置了。

<br/>
**问题6：我们需要的三方库不再cocoaPods中**

创建自己项目的spec描述文件。这里有篇文字参考：
http://ishalou.com/blog/2012/10/16/how-to-create-a-cocoapods-spec-file/
http://iiiyu.com/2013/12/19/learning-ios-notes-thirty-one/
   
<br/>
###提交你的开源库到cocoaPods

参考上面的spec描述文件制作。

<br/>
###对git代码管理的影响

分下面2种情况

1 将pods纳入版本管理中：

这个没有什么太大得问题，注意选择正确的schemes。

<br/>
2 不将pods纳入版本管理中：

但需要保证下面文件纳入版本管理，保证使用的pod的版本。
Podfile
.xcworkspace
Podfile.lock

记得bulid之前，都要执行pod install命令，因为podfile里面可能是别的同事加入了新的开源库。通常是把pod install命令加入到build步骤中来。
