---
title: GraphQL和Protobuf-iOS端实践调研
date: 2018-06-14 14:57:02
tags: 
  - GraphQL 
  - Swift
categories: GraphQL
---

## 什么是GraphQL？

简单的说，GraphQL 是（一种描述请求数据方法的语法 / 一种API查询语言），通常用于客户端从服务端加载数据。
GraphQL 有以下三个主要特征：

* 它允许客户端指定具体所需的数据。
* 它让从多个数据源汇总取数据变得更简单。
* 它使用了类型系统来描述数据。

> 大白话版：
GraphQL 为数据通信而生。你有一个客户端和一个服务器，它们需要相互通信。客户端需要告知服务器需要哪些数据，服务器需要用实际的数据来满足客户端的数据需求。GraphQL 是此种通信方式的中介

### GraphQL是怎么诞生的呢？

GraphQL 是由 Facebook 开发的，用于解决他们巨大、老旧的架构的数据请求问题。并于2015年将 GraphQL 开源。

目前已经形成完善的社区

* [GraphQL](https://github.com/graphql)
* [awesome-graphql](https://github.com/chentsulin/awesome-graphql)
* [GraphQL 中国](http://graphql.cn/)

### GraphQL介绍和优缺点

这里我不再详细的做个人分析，献上几篇译文：

* [我经常听到的 GraphQL 到底是什么？](https://juejin.im/post/58fd6d121b69e600589ec740)
* [安息吧 REST API，GraphQL 长存](https://www.zcfy.cc/article/rest-apis-are-rest-in-peace-apis-long-live-graphql)
* [阻碍你使用 GraphQL 的十个问题](http://jerryzou.com/posts/10-questions-about-graphql/)
* [GraphQL vs. REST](https://juejin.im/post/59793f625188253ded721c70)

* [淘宝前端团队：深入理解 GraphQL](http://taobaofed.org/blog/2016/03/10/graphql-in-depth/)

### GraphQL 在 iOS 端的使用

GraphQL 在客户端的使用需要借助 Relay 或者 Apollo-Client。但是 Relay 不支持 OC 和 Swift。
所以这里主要调研了 Apollo-iOS

* [GraphQL 和 Apollo-iOS](https://blog.csdn.net/kmyhy/article/details/77161966)
* [apollo-client-ios](https://github.com/apollographql/apollo-ios)

> Tips: 在服务端更新node后，客户端必须要更新schema.json文件，再次运行下面的命令。

``` ruby
apollo-codegen download-schema __SIMPLE_API_ENDPOINT__ --output $(文件路径)/schema.json
# __SIMPLE_API_ENDPOINT__ 替换成服务端地址
```

## 什么是Protobuf？

Protocol Buffers (简称 Protobuf) 是 Google 公司内部的混合语言数据标准，目前已经正在使用的有超过 48,162 种报文格式定义和超过 12,183 个 .proto 文件。他们用于 RPC 系统和持续数据存储系统。

Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。

### Protobuf的使用

它并不难使用，相反，简单易上手还是它的优点，参考：

* [Introduction to Protocol Buffers on iOS](https://www.jianshu.com/p/25baebc411fe)
* [swift-protobuf](https://github.com/apple/swift-protobuf)

### GraphQL 和 Protobuf

这里说一下，难点在于服务端使用 Protobuf 源生成统一的GraphQL schema。

Google开源了方案，使用 [rejoiner](https://github.com/google/rejoiner) 框架。

客户端反而简单，只需要多写一个.proto文件，然后在使用 GraphQL 创建 Query 后，结合Apollo-client 获取返回值后用Prototbuf解析就可以了

``` swift

extension ApolloClient {
  @discardableResult
  func fetch<Query: GraphQLQuery>(query: Query, resultHandler: ResultHandler<Query>? = nil) -> Cancellable {
    return fetch(query: query, cachePolicy: CachePolicy.fetchIgnoringCacheData, queue: DispatchQueue.main) { (result, error) in
      if let error = error {
        resultHandler?(.failure(SunTravelClientError.underlyingError(error)))
      } else if let errors = result?.errors {
        resultHandler?(.failure(SunTravelClientError.underlyingErrors(errors)))
      } else if let data = result?.data {
        /// 关键的异步：解析.proto文件
        let contact = try? Contact(protobuf: data)
        resultHandler?(.success(contact))
      } else {
        resultHandler?(.failure(SunTravelClientError.invalidResponse))
      }
    }
  }
}

```