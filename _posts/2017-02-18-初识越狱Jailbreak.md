---
layout: post
title: '初识越狱 -- Jailbreak '
subtitle: '越狱 Jailbreak'
date: 2017-02-18
categories: 技术
cover: 'https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E9%80%86%E5%90%91/day01/day01-header.jpg'
tags: 逆向
---


### 什么是iOS Jailbreak？
- 利用iOS系统的漏洞，获取iOS系统的最高权限（Root），解开之前的各种限制（合法行为）

### iOS Jailbreak的优点
- 打造个性化、与众不同的iPhone
	- 自由安装各种实用的插件、主题、APP
	- 修改系统APP的一些默认行为
- 自由安装非AppSore来源的APP
	- “付费APP”秒变“免费APP”
	- 未越狱iPhone安装APP的途径
	- AppStore、真机调试、通过证书打包签名ipa安装
- 灵活管理文件系统，让iPhone可以像U盘那样灵活
- 给开发者提供了逆向工程的环境

### iOS Jailbreak的缺点
- 不予保修
- 费电，越狱后的iOS系统会常驻一些进程，耗电速度约提升10%~20%
- 在新的iOS固件版本出来的时候，不能及时地进行更新
	- 每个新版本的固件，都会修复上一个版本的越狱漏洞，使越狱失效
	- 如果需要保持越狱状态，要等待新的越狱程序发布时，才能升级相应的固件版本
- 不再受iOS系统默认的安全保护，容易被恶意软件攻击，个人隐私有被窃取的风险
- 如果安装了不稳定的插件，容易让系统变得不稳定、变慢，甚至出现“白苹果”等问题

### 完美越狱和不完美越狱
- 完美越狱
	- 越狱后的iPhone可以正常关机和重启

- 不完美越狱
	- iPhone一旦关机后再开机时，屏幕就会一直停留在启动画面，也就是“白苹果”状态
	- 能正常开机，但已经安装的破解软件都无法正常使用，需要将设备与PC连接后，使用软件进行引导才能使用

- 一般说来，在苹果发布新的iOS固件后，针对该固件的不完美越狱会先发布，随后完美越狱才可能发布
	- 一般较新的系统版本，均为不完美越狱

- 越狱方法推荐
    - [PP助手](http://jailbreak.25pp.com/)

### 如何判断是否越狱成功？
- 桌面是否有Cydia
- 工具判断（比如PP助手）

### Cydia
- 越狱后的“App Store”
	- 可以在Cydia中安装各种第三方的软件（插件、补丁、APP）
- 作者
	- Jay Freeman (saurik)

### Cydia安装软件的步骤
- 添加软件源（不同软件的软件源可能不同）
![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E9%80%86%E5%90%91/day01/day01-6.png)
![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E9%80%86%E5%90%91/day01/day01-7.png)
![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E9%80%86%E5%90%91/day01/day01-8.png)

- 进入软件源找到对应的软件，开始安装
![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E9%80%86%E5%90%91/day01/day01-9.png)
![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E9%80%86%E5%90%91/day01/day01-10.png)
![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E9%80%86%E5%90%91/day01/day01-11.png)
- 如果软件源中的软件太多，可以搜索查找

### SpringBoard
- 有时候通过Cydia安装完插件后，可能会出现以下界面
![](https://fuqionglin-blog.oss-cn-qingdao.aliyuncs.com/%E9%80%86%E5%90%91/day01/day01-13.png)
- SpringBoard就是iOS的桌面

### Apple File Conduit "2"
- Apple File Conduit "2"补丁的作用
	- 可以访问整个iOS设备的文件系统
	- 类似的补丁还有：afc2、afc2add
- 软件源
	- [http://apt.saurik.com](http://apt.saurik.com)
	- [http://apt.25pp.com](http://apt.25pp.com)

### AppSync Unified
- AppSync Unified补丁的作用
	- 可以绕过系统验证，随意安装、运行破解的ipa安装包
- 软件源
	- [http://apt.25pp.com](http://apt.25pp.com)

### iFile
- iFile的作用
	- 可以在iPhone上自由访问iOS文件系统
	- 类似的还有Filza File Manager、File Browser
- 软件源
	- [http://apt.thebigboss.org/repofiles/cydia](http://apt.thebigboss.org/repofiles/cydia)
	
### PP助手
- 可以利用PP助手自由安装海量APP
- 软件源
	- [http://apt.25pp.com/](http://apt.25pp.com/)
	
### Mac必备
- iFunBox
	- 管理文件系统
- PP助手
	- 自由安装海量APP
	- 卸载APP
	- 备份APP为ipa安装包（iOS9开始，不再支持备份APP）

### 安装包
- 通常情况下
	- 通过Cydia安装的安装包是deb格式的（结合软件包管理工具apt）
	- 通过PP助手安装的安装包是ipa格式的
- 如果通过Cydia源安装deb失败
	- 可以先从网上下载deb格式的安装包
	- 然后将deb安装包放到/var/root/Media/Cydia/AutoInstall
	- 重启手机，Cydia就会自动安装deb
	
### 如何在iOS代码中判断设备是否越狱？
- 针对不同iOS版本的判断方法可能不一样
- 最简单的一种方法：判断手机上是否安装了Cydia

<pre><code class="language-objectivec">-（BOOL）re_isJailbreak
{
    return [[NSFileManager defaultManager] fileExistsAtPath:@"/Applications/Cydia.app"];
}
</code></pre>

