---
title: Moya + RxSwift + HandyJSON 实现函数式网络请求
date: 2017-07-27 16:33:06
tags:
---

# Moya

`Moya` 是什么？

一种基于 `Alamofire` 封装的网络抽象层库。使用 `Moya` 可以很方便快捷的实现一个 `Swift` 项目的网络请求 `Router`。

根据 `Moya` [GitHub](https://github.com/Moya/Moya) 提供的文档，我们可以很方便的了解它和 `Alamofire` 的关系：

{% asset_img moya1.png Moya %}

# RxSwift

响应式编程框架，这里不做过多解释，可以去看官方文档 [GitHub](https://github.com/ReactiveX/RxSwift) 或者我的另一篇 [博客](http://www.walled.men/2017/07/14/%E5%AD%A6%E4%B9%A0RxSwift-%E5%9F%BA%E7%A1%80%E8%AF%AD%E6%B3%95/) 。
因为 `Moya` 原生的支持 `RxSwift` ，更方便了我们去使用它

# HandyJSON

阿里巴巴开源的 `Swift` 版本 `JSON To Model` 解析工具。这里使用它，仅仅是作为一个工具来分析 `Moya` 的实用性。
`Demo` 中可以把 `HandyJSON` 替换成 `ObjectMapper` 或者 `SwiftyJSON`，都没有问题

# Moya的实际使用分析

## 基本使用

使用 `Moya` 创建网络请求，我们需要遵守它的 `TargetType` 协议。而`Swift` 强大的 `enum` 功能，可以让我们很方便的创建一个网络请求的 `Router` 。

``` swift
enum MoyaApi {
    case threadList(mod: String, page: String)
}

extension MoyaApi: TargetType {

    var baseURL: URL {
        return URL(string: Moya_baseURL)!
    }

    /// 要附加到`baseURL'后以形成完整的URL的路径
    var path: String {
        switch self {
        case .threadList:
            return "/api.php"
        }
    }

    /// 请求中使用的HTTP方法
    var method: Moya.Method {
        return .get
    }


    /// 请求中要编码的参数
    var parameters: [String : Any]? {
        switch self {
        case .threadList(let mod, let page):
            var p = [String: Any]()
            p["mod"] = mod
            p["page"] = page
            return p
        }
    }

    /// 用于参数编码的方法
    var parameterEncoding: ParameterEncoding {
        return URLEncoding.default
    }

    /// 要执行的HTTP任务的类型
    var task: Task {
        return .request
    }

    /// 提供存根数据用于测试
    /// 只在单元测试文件中有作用
    var sampleData: Data {
        return "".data(using: String.Encoding.utf8)!
    }

}

```

我们实现了这个 `enum` ，我们就相当于完成了一个网络请求所需要的一个 `endpoint` ，`endpoint` 其实就是一个结构，包含了我们一个请求所需要的基本信息，比如 `url` ，`parameter` 等。 `endpoint` 好了这时候我们就需要一个工具去发送这个请求。于是 `Moya` 提供一个发送请求的 `Provider`

``` swift
let provider = RxMoyaProvider<MoyaApi>()
```

然后，我们可以使用这个 `provider` 发起一个网络请求：

``` swift
provider.request(.threadList(mod: "getAppIndexThreadList", page: "1")) { result in
    switch result {
    case let .success(response):
    .......
    case let .failure(error):
    ......
    }
}
```

## Endpoint Closure

`Moya` 的这个功能，可以很方便的让我们处理网络请求的 `Header` 或者公共参数：

``` swift
let headerFields: [String: String] = ["agent" : "XXXX"]

var publicParameters: [String : Any] = ["platform"      : "iOS",
                                        "clientVersion" : "1.0.0",
                                        "network"       : "WIFI"]

private let endPointClosure = { (target: MoyaApi) -> Endpoint<MoyaApi> in
    let defaultEndpoint = MoyaProvider<MoyaApi>.defaultEndpointMapping(for: target)
    return defaultEndpoint.adding(parameters: publicParameters as [String : AnyObject]?, httpHeaderFields: headerFields)
}

let provider = RxMoyaProvider<MoyaApi>(endpointClosure: endPointClosure)])

```

## 插件机制

`Moya` 的插件机制也很好用，提供了两个接口，willSendRequest 和 didReceiveResponse，可以在请求发出前和请求收到后做一些额外的处理，并且不和主功能耦合。

`Moya` 本身提供了打印网路请求日志的插件和 NetworkActivityIndicator 的插件。

例如网络请求过程中展示 `HUD` 的插件：

``` swift
final class MoyaPlugin: PluginType {

    func willSend(_ request: RequestType, target: TargetType) {
       // 获取请求的URL
//        guard let requestURLString = request.request?.url?.absoluteString else { return }

        UIViewController.showHUD()
    }

    public func didReceive(_ result: Result<Moya.Response, MoyaError>, target: TargetType) {
        // 只监听失败
//        guard case Result.failure(_) = result else { return }

        UIViewController.hideHUD()
    }

}
```

使用自定义的插件也很方便：

``` swift
let provider = RxMoyaProvider<MoyaApi>(endpointClosure: endPointClosure, plugins:[NetworkLoggerPlugin(verbose: true), MoyaPlugin()])
```

## 扩展

使用 `Moya + RxSwift`， 我们可以对 `Observable` 进行扩展，这样我们就可以用一种优雅的方式实现链式语法。

`Moya` 自带了一些 `Observable`，这里我们介绍一个我们自己的实现。

比如，我们实现一个扩展，在网络请求失败的时候，弹出 `Toast` :

``` swift
extension Observable {

    func showErrorToast() -> Observable<Element> {
        return self.do(onNext: { response in

            let json = response as! [String : Any]
            if let c = json[RESULT_CODE] as? Int, c != 0 {
                // 错误信息，弹出Toast
                UIViewController.showStatus(title: json[RESULT_MESSAGE] as! String)
            }

        }, onError: { error in
            // 错误信息，弹出Toast
            UIViewController.showStatus(title: error.localizedDescription)
        })
    }

}
```

使用它：

``` swift
provider.request(.threadList(mod: "getAppIndexThreadList", page: "1"))
            .filterSuccessfulStatusCodes()
            .showErrorToast() { result in
            switch result {
            case let .success(response):
                .......
            case let .failure(error):
                ......
            }
        }
```

## JSON To Model

这里我们介绍一种通过扩展 `Observable` 实现 `JSON To Model` 的方式：

``` swift
extension Observable {

    /// HandyJSON To Object
    // 从json data中获取Object数据
    func mapObjectFromData<T: HandyJSON>(type: T.Type) -> Observable<T> {

        return mapObject(type: type, designatedPath: RESULT_DATA)

    }

    // 从指定json位置获取Object数据
    func mapObject<T: HandyJSON>(type: T.Type, designatedPath: String? = nil) -> Observable<T> {

        return self.map { response in

            guard let dict = response as? [String : Any] else {
                throw RxSwiftMoyaError.parseJSONError
            }

            if dict["code"] != nil, let code = dict[RESULT_CODE] as? Int, code != 0 {
                throw RxSwiftMoyaError.noData
            }

            guard let data = try? JSONSerialization.data(withJSONObject: dict) else {
                throw RxSwiftMoyaError.parseJSONError
            }

            let jsonString = String(data: data, encoding: .utf8)

            guard let object = JSONDeserializer<T>.deserializeFrom(json: jsonString, designatedPath: designatedPath) else {
                throw RxSwiftMoyaError.parseJSONError
            }

            return object
        }

    }



    /// HandyJSON To Array
    func mapArrayFromData<T: HandyJSON>(type: T.Type) -> Observable<[T]> {
        return mapArray(type: type, designatedPath: RESULT_DATA)
    }

    func mapArray<T: HandyJSON>(type: T.Type, designatedPath: String? = nil) -> Observable<[T]> {

        return self.map { response in

            guard let dict = response as? [String : Any] else {
                throw RxSwiftMoyaError.parseJSONError
            }

            if dict["code"] != nil, let code = dict[RESULT_CODE] as? Int, code != 0 {
                throw RxSwiftMoyaError.noData
            }

            guard let data = try? JSONSerialization.data(withJSONObject: dict) else {
                throw RxSwiftMoyaError.parseJSONError
            }

            let jsonString = String(data: data, encoding: .utf8)

            guard let array = JSONDeserializer<T>.deserializeModelArrayFrom(json: jsonString, designatedPath: designatedPath) as? [T] else {
                throw RxSwiftMoyaError.parseJSONError
            }

            return array
        }

    }

}
```

使用：

``` swift
provider.request(.threadList(mod: "getAppIndexThreadList", page: "1"))
            .filterSuccessfulStatusCodes()
            .mapJSON()
            .showErrorToast()
            .mapArrayFromData(type: ListModel.self)
            .subscribe ({ event in
                switch event {
                case .next(let list):
                    self.showAlertView(title: "SUCCESS", message: "request is success", action: {
                        print(list)
                    })

                case .error(let error):
                    print(error.localizedDescription)

                case .completed:
                    break

                }
            })
            .addDisposableTo(disposeBag)
```

# DEMO

Demo: [GitHub](https://github.com/huxiaoyang/RxMoyaHandyJSONDemo)

# 参考
<https://liuduo.me/2016/07/24/rxswiftmoyanetwork/>

<https://github.com/Moya/Moya/tree/master/docs/Examples>

<http://www.jianshu.com/p/d642865f0a64>