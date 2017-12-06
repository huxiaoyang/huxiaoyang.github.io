---
title: 关于Swift页面跳转路由的设计思路
date: 2017-11-08 15:30:16
tags:
  - Swift
categories: Swift
---

为什么要设计一个页面跳转的路由？

因为，不想在整个项目中到处播撒下面的代码...

``` swift
let VC = UIViewController()
self.navigationController?.pushViewController(VC, animated: true)
```

不易维护，而且耦合严重。

如果你觉得以上这些都不是事儿，我只管写不管维护，干三个月我就走人了。

> 请自动忽略下面的内容，并 `cmd+w` 二连

# 正题

以前写OC代码，使用的 [casatwy](https://casatwy.com/iOS-Modulization.html) 大神的组件化方案，可以实现页面跳转逻辑的解耦。

> 本篇不讨论组件化，只是借鉴

Swift虽然可以继续使用该方案，但是我对在Swift中使用运行时语法很抵触，莫名其妙 ^_^'

于是借鉴了一下 `Moya` ，设计一个简单易用的pageRouter

## 设计思路

要求：

- 解耦
- 枚举映射
- 参数传递方便
- 使用简单

首先，想到页面跳转，就想到了 `push` 、 `present` 等各种方式

所以我们先设计一个枚举，列出所有可用到的跳转方式：

> 这里只是提供思路，具体的各种跳转逻辑可以自己研究

``` swift
enum JumpOperation {
    case push
    case present
    case presentNav(parent: UINavigationController.Type)
}
```

下面，整个路由设计中最重要的，我们需要设计一个协议，来控制整个路由

协议里面包含跳转过程中需要的一些数据

``` swift
/// 待跳转 controller 信息协议
protocol ControllerConvertible {
    /// 目标页面
    func asController() -> UIViewController.Type
    /// 跳转方式
    func method() -> JumpOperation
    /// 目标的title
    func title() -> String?
}
```

有了协议，需要再设计一个 `Provider` 去提供一个跳转方法。

设计 `Provider` 的时候，我发现少了一个东西，少了一个 `源 Controller`，这时候我本来想在协议中再加入一个参数或者函数来表示这个 `源`，但是后来我放弃了，一般情况下，页面的跳转是最前端的表现，此时的 `源` 应该就是当前 `window` 上最顶端的 `viewController` 。

所以我写了一个扩展，用来获取最顶端的 `viewController`

``` swift
extension UIViewController {

    private class func getCurrentWindow() -> UIWindow? {
        var window: UIWindow? = UIApplication.shared.keyWindow
        if let w = window, w.windowLevel != UIWindowLevelNormal {
            for tempWindow in UIApplication.shared.windows {
                if tempWindow.windowLevel == UIWindowLevelNormal  {
                    window = tempWindow
                    break
                }
            }
        }
        return window
    }


    // MARK: 获取当前window上controller
    private class func getTopWindowController(_ viewController: UIViewController?) -> UIViewController? {

        guard let controller = viewController else {
            return nil
        }

        if let VC = controller.presentedViewController {
            return getTopWindowController(VC)
        }

        if controller.isKind(of: UISplitViewController.self) {
            let VC = controller as! UISplitViewController

            if VC.viewControllers.count > 0 {
                return getTopWindowController(VC.viewControllers.last)
            }

            return VC
        }

        if controller.isKind(of: UINavigationController.self) {
            let VC = controller as! UINavigationController

            if VC.viewControllers.count > 0 {
                return getTopWindowController(VC.topViewController)
            }

            return VC
        }

        if controller.isKind(of: UITabBarController.self) {
            let VC = controller as! UITabBarController

            guard let controllers = VC.viewControllers, controllers.count > 0 else {
                return VC
            }

            return getTopWindowController(VC.selectedViewController)
        }

        return controller
    }


    class func topWindowController() -> UIViewController? {

        guard let window = UIViewController.getCurrentWindow() else {
            return nil;
        }

        return getTopWindowController(window.rootViewController!)
    }

}

```

有个 `源 Controller` 我们可以设计 `Provider` 了。

首先，我们决定 `Provider` 中只有一个方法

``` swift
func jump(_ router: target, parameters: [String : Any] = [:])
```

有 `parameters` ，使用字典类型，易扩展，也易使用。

那么接下来我们怎么去存取这些参数呢?

我们写一个协议，决定存取方法

``` swift
protocol ViewControllerIntent {
    /// 存
    func putExtra<T>(_ key: String, _ value: T)
    /// 取
    func getExtra<T>(_ key: String) -> T?
}
```

然后写一个 UIViewController 的扩展

> 注意，整个Router中的 `target` 对象都应该是 `UIViewController`

``` swift
extension UIViewController: ViewControllerIntent {

    private struct intentStorage {
        static var extra: [String : Any] = [:]
    }

    func putExtra<T>(_ key: String, _ value: T) {
        intentStorage.extra[key] = value
    }

    func getExtra<T>(_ key: String) -> T? {
        return intentStorage.extra[key] as? T
    }

}
```

好了，最后我们实现 `Provider`

``` swift
final class PageProvider<target: ControllerConvertible> {

    func jump(_ router: target, parameters: [String : Any] = [:]) {

        let viewController = router.asController().init()

        if let title = router.title() {
            viewController.title = title
        }

        if !parameters.isEmpty {

            for (key, value) in parameters {

                viewController.putExtra(key, value)

            }

        }

        DispatchQueue.main.async {

            switch router.method() {

            case .push:

                guard let controller = UIViewController.topWindowController(), let nav = controller.navigationController else {
                    return
                }

                nav.pushViewController(viewController, animated: true)

            case .present:

                guard let controller = UIViewController.topWindowController() else {
                    return
                }

                controller.present(viewController, animated: true)

            case .presentNav(let parent):

                guard let controller = UIViewController.topWindowController() else {
                    return
                }

                let nav = parent.init(rootViewController: viewController)

                controller.present(nav, animated: true)

            }

        }

    }

}
```

## 实例：

定义自己的 `Router` 实现 `ControllerConvertible` 协议

``` swift
extension PageControlRouter: ControllerConvertible {

    /// 处理跳转页面，这里最好不使用default，每一个枚举对应一个Controller.Type
    func asController() -> UIViewController.Type {

        switch self {

        case .none:
            return BSViewController.self
        case .login:
            return LoginViewController.self
        case .register:
            return RegisterViewController.self
        case .forgotPassword:
            return ForgotPasswordViewController.self
        case .productDetail:
            return ProductDetailViewController.self
        case .cardList:
            return CardListViewController.self
        }

    }

    /// 处理跳转方式，默认push方式
    func method() -> ControllerOperation {

        switch self {

        case .login:
            return .presentNav(parent: BSNavigationController.self)

        default:
            return .push

        }

    }

    /// 处理跳转页面的title
    func title() -> String? {

        switch self {

        case .none:
            return "测试空白页"

        case .register:
            return "注册账号"

        case .forgotPassword:
            return "找回密码"

        default:
            return nil

        }

    }

}
```

初始化一个 `Provider`

``` swift
let Page = PageProvider<PageControlRouter>()
```

使用：

``` swift
/// 1. 不带参数
Page.jump(.login)

/// 2. 带参数
Page.jump(.login, parameters: ["key": "value"])
/// 参数使用 target Controller内
let value = getExtra("key")

```

## DEMO

[GitHub](https://github.com/huxiaoyang/PageRouter/tree/master)