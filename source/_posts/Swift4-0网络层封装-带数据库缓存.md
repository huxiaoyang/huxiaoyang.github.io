---
title: Swift4.0网络层封装(带数据库缓存)
date: 2017-11-06 16:29:06
tags:
  - Swift
  - RxSwift
  - Realm
categories: Swift
---

> 关于Swift3.0的网络层封装，请参考之前的博客 [Moya + RxSwift + HandyJSON 实现函数式网络请求](http://www.walled.men/2017/07/27/Moya-RxSwift-HandyJSON-%E5%AE%9E%E7%8E%B0%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BD%91%E7%BB%9C%E8%AF%B7%E6%B1%82/)

# 本篇项目环境和依赖框架版本

- Swift 4.0 +
- RxSwift 4.0 +
- Moya 10.0 +
- ObjectMapper 3.0 +
- Realm 3.0 +

# 使用

具体的使用可查看之前的博客，我信奉的是 `No BB Show Code`

`HandyJSON` 更换成 `ObjectMapper`，是因为 `HandyJSON` 对 `Realm` 的适配不是很友好，而`ObjectMapper`正好相反

使用 `Realm` 而不是 `SQLite` 是因为 `Realm` 的API对 `Swift` 很友好，功能强大而且使用简单

使用 `RxSwift` 是因为在 `Swift` 语言中，函数是一等公民，而且响应式的编程确实很爽，关于函数式编程，我也是小白一个，正在深入的学习中...

这篇博客不想写太多，确实没有需要注意的点需要深入讲解，也没有太多的新知识，更多的是对 `Swift4.0` 的适配

本篇博客的目的只是为了在 `Swift5.0` 推出后，证明我适配过 `Swift4.0`  ^-^

这里放一个 [Demo](https://github.com/huxiaoyang/Moya-Realm-ObjectMapper)，供大家参考！