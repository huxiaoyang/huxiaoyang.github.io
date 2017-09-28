---
title: iOS 持续部署
date: 2017-08-02 10:29:59
tags:
  - iOS
  - 工具
categories: iOS 
---

为什么使用持续部署？

工作多年的老司机会白你一眼，『为什么要手动部署？』。

1. 为了追求手动生成各种证书时候的快感？
1. 为了享受在iTunes Connect上传ipa的等待？
1. 为了展示Archive各种测试包的熟练？

以上，如果有一项能够让你觉得舒服。请自动忽略本篇博客。

# Fastlane

一组工具套件，旨在实现 `iOS` 应用发布流程的自动化，并且提供一个运行良好的持续部署流程，只需要运行一个简单的命令就可以触发这个流程。

# 安装 + 使用

[Fastlane 官方教程](https://docs.fastlane.tools/)

官方的文档很详细，还有很多的示例。可以充分满足你的需求，如果还有不明白或者不理解的：

[GitHub 提问题](https://github.com/fastlane/fastlane/issues)

会有`很多人` `非常快速` `完美的` 解决你的问题。

# Fastlane 工具链

- [fastlane](https://fastlane.tools/)：自动化iOS和Android应用的beta部署和发布
- [deliver](https://github.com/fastlane/fastlane/tree/master/deliver)：将截图，元数据和应用上传到App Store
- [snapshot](https://github.com/fastlane/fastlane/tree/master/snapshot)：在每个设备上自动执行iOS应用的本地化屏幕截图
- [frameit](https://github.com/fastlane/fastlane/tree/master/frameit)：快速截屏并将截屏放入设备中
- [pem](https://github.com/fastlane/fastlane/tree/master/pem)：自动生成和更新推送通知配置文件
- [produce](https://github.com/fastlane/fastlane/tree/master/produce)：使用命令行在iTunes Connect和Dev Portal上创建新的iOS应用程序
- [cert](https://github.com/fastlane/fastlane/tree/master/cert)：自动创建和维护iOS代码签名证书
- [spaceship](https://github.com/fastlane/fastlane/tree/master/spaceship)：Ruby库访问Apple Dev Center和iTunes Connect
- [pilot](https://github.com/fastlane/fastlane/tree/master/pilot)：从终端管理TestFlight测试人员和构建的最佳方式
- [boarding](https://github.com/fastlane/boarding)：邀请TestFlight beta测试人员
- [scan](https://github.com/fastlane/fastlane/tree/master/scan)：运行iOS和Mac应用程序测试
- [match](https://github.com/fastlane/fastlane/tree/master/match)：使用git同步团队的证书和配置文件
- [precheck](https://github.com/fastlane/fastlane/tree/master/precheck)：使用社区驱动的App Store评估规则检查您的应用程序，以避免被拒绝

# TIPS

不管你是看官方文档，还是Google。大神们都已经把 `Fastlane` 讲解的很清楚了，使用也很简单。我这里就不再过多的解释怎么使用了。

有一个注意的点，安装完成后，会在项目根目录生成 `Gemfile` 文件。打开查看，一定要保证 `source` 跟你本地安装 `Gem` 的 `source` 的源一样。

查看本地gem源，终端 `gem source l`

# 参考

[小团队的自动化发布－Fastlane带来的全自动化发布](https://whlsxl.github.io/fastlane1/)

[使用 fir 和 fastlane 实现 iOS 持续集成](http://www.jianshu.com/p/002e1061ee08)

[fastlane docs](https://docs.fastlane.tools/actions/)