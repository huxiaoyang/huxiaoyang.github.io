---
title: CocoaPods 私有仓库使用
tags:
  - CocoaPods
  - iOS组件化
categories: iOS组件化
---

CocoaPods是iOS中使用很广泛的第三方包管理工具，我们可以用来制作自己的开源代码库，也可以制作私有代码库。
基于这些功能，我们可以用它来作为iOS的组件化管理工具。

# 总体步骤

1. 创建私有Spec仓库来管理私有的podspec文件
1. 创建私有的Pod工程文件，并提交到远程git托管平台
1. 创建私有的Pod对应的podspec文件
1. 验证podspec文件的有效性
1. 提交podspec至私有的Spec仓库
1. 创建主工程使用私有的Pod
1. 更新私有的Pod文件

# 创建私有Spec仓库

我们知道安装Cocoapods后，会在本地生成一个官方的Spec仓库master，文件地址在 `~/.cocoapods/repos` 目录，远程地址 `https://github.com/CocoaPods/Specs.git`
私有的 Spec 仓库和 CocoaPods 官方结构一致，用于存放spec文件。私有 Spec 仓库需要 git 托管平台，这里我是用的是 `Bitbucket`
首先，在 `Bitbucket` 上新建项目 `MyPrivateSepc` , `https://huxiaoyang@bitbucket.org/huxiaoyang/MyPrivateSepc.git`
然后，在终端执行：`pod repo add MyPrivateSepc https://huxiaoyang@bitbucket.org/huxiaoyang/MyPrivateSepc.git`
执行完后，打开 `~/.cocoapods/repos` 目录，你会发现 `Cocoapods` 把 `MyPrivateSepc` clone到了本地，这样私有仓库就创建成功了

# 创建私有的pod工程文件

在`Bitbucket`上新建项目用于存放私有的pod工程源码，两种方式创建：

1. 在本地创建工程，然后提交到 `Bitbucket`
    步骤一：切换到您的仓库的目录

        cd /path/to/your/repo
    步骤二：连接您的现有仓库与 Bitbucket

        git remote add origin ssh://git@bitbucket.org/huxiaoyang/test.git
        git push -u origin master

1. 在 `Bitbucket` 创建工程，然后 clone 到本地

        git clone
        git@bitbucket.org:huxiaoyang/XXXX.git`
        cd XXXX
        echo "# My project's README" >> README.md
        git add README.md
        git commit -m "Initial commit"
        git push -u origin master

# 创建私有Pod的 Podspec

1. 在工程的根目录下执行：`pod spec create 工程名`，创建成功后会在根目录下生成 `工程名.podspec` 文件。
1. 根据你的需求修改 `.podspec` 文件，这里不做具体的讲解，可以查看官网教程 [Podspec Syntax Reference](https://guides.cocoapods.org/syntax/podspec.html)
1. push代码到远端

        git add .
        git commit -m " "
        git push origin master
1. 给工程打 tag 并 push 到远端：

        git tag -a "0.0.1" -m " "
        git push --tags
1. 验证`Podspec`文件的有效性，

        pod lib lint 工程名.podspec
        pod spec lint 工程名.podspec

如果有警告，不想修改又想通过验证，在命令后添加：`--allow-warnings`

都验证通过后，`Podspec`文件配置成功

# 提交 Podspec 至私有 Spec 仓库

在 `工程名.podspec` 文件目录执行： `pod repo push MyPrivateSepc 工程名.podspec`

> 这里额外说明一下，如果制作开源库，需要提交到官方仓库，`pod trunk push 工程名.podspec` 

# 新建项目使用

新建 `Xcode` 工程，在工程根目录下执行 ：`pod init` 生成 `Podfile` ，或者直接创建 `.Podfile` 文件。
在 `Podfile` 文件开头添加:

    source 'https://github.com/CocoaPods/Specs.git'
    source 'https://huxiaoyang@bitbucket.org/huxiaoyang/MyPrivateSepc.git'

执行 `pod install` 完成安装

# 更新Pod

1. 修改 podspec 文件中的 s.version.
1. 修改工程代码，打tag，tag名称和 s.version 一致，并 push 到远程仓库.
1. 验证 podspec 文件的有效性.
1. 推送 podspec 文件到远程仓库.
1. 执行 `pod search 工程名` 验证结果.

# TIPS

首次安装 Cocoapods 会下载官方仓库到本地，受GFW的影响有时候会很慢。
可以在终端执行 `cd ~/.cocoapods ` ，然后执行 `du -sh *`， 查看下载进度