---
title: iOS横竖屏问题
date: 2017-07-20 13:37:41
tags:
  - Swift
  - iOS
categories: iOS
---

iPhone具有横竖屏开关，所以iOS应用应该具有横竖屏的功能。但是我们又想随意的控制界面是否允许横竖屏或者强制横竖屏。

下面内容，主要介绍代码控制屏幕旋转，包括：

1. 监听屏幕旋转方向
1. 视图控制器中旋转方向的设置
1. 屏幕旋转方向下的视图处理
1. 强制横屏的处理

# 监听屏幕旋转方向

## UIDeviceOrientation：设备方向

1、 iOS定义了七种设备方向

``` bash
public enum UIDeviceOrientation : Int {
    case unknown // 未知方向，可能是设备)斜置
    case portrait //  设备直立
    case portraitUpsideDown // 设备直立，上下顛倒
    case landscapeLeft // 设备向左横置
    case landscapeRight // 设备向右橫置
    case faceUp // 设备朝上平躺
    case faceDown // 设备朝下平躺
}

```

2、 在iPhone没有锁定竖屏的情况下，获取设备方向。

``` bash
let orientation = UIDevice.current.orientation
```

> 注意：如果iPhone锁定了竖屏，该方法会返回 `unknown`

3、当设备变化的时候，会发出通知，我们可以通过监听 `NSNotification.Name.UIDeviceOrientationDidChange` 获取通知状态。

> 注意：如果iPhone锁定了竖屏，该通知失效

## UIInterfaceOrientation：界面方向

1、iOS定义了以下五种界面方向

``` bash
public enum UIInterfaceOrientation : Int {
    case unknown
    case portrait
    case portraitUpsideDown
    case landscapeLeft
    case landscapeRight
}
```

> 注意： 这里的界面左右是和设备的左右相反的，这是因为将设备旋转到左侧需要将内容旋转到右侧，也就是说 `UIInterfaceOrientation.landscapeLeft = UIDeviceOrientation.landscapeRight`

2、 读取界面方向

``` bash
let orientation = UIApplication.shared.statusBarOrientation
```

3、当界面变化的时候，会发出通知，我们可以通过监听 `NSNotification.Name.UIApplicationDidChangeStatusBarOrientation` 获取通知状态。

> 注意：手机锁定竖屏的时候，强制转换界面方向的时候，依然会收到通知

# 视图控制器中旋转方向的设置

## 全面禁止横竖屏的操作（简单，但不推荐的做法）

方法一：在项目的General-->Deployment Info-->Device Orientation中，只勾选Portrait(竖屏)

{% asset_img image1.png Xcode中设置 %}

方法二：在Appdelegate中实现

``` bash
func application(_ application: UIApplication, supportedInterfaceOrientationsFor window: UIWindow?) -> UIInterfaceOrientationMask {
        return .portrait
    }
```

## 项目中代码控制视图控制器的旋转

1、首先设置APP支持多个方向

{% asset_img image2.png Xcode中设置 %}

2、控制器中重写关键函数

在这里我们可以创建一个基类，在基类中统一处理，可以方便之后使用。

> 注意：这里主要提供思路，可以发散思维，不要拘泥于这种做法

关键函数有三个，有一个需要注意的地方，就是控制器的层级问题，如果使用了 `NavigationController` 和 `TabBarController` 也要重写子类并重写这三个函数。

直接上代码：

``` bash
// UIViewController 的基类
class BaseViewController: UIViewController {

    /// 是否自动旋转，这里默认不自动旋转
    override var shouldAutorotate: Bool {
        return false
    }

    /// 返回支持的方向
    override var supportedInterfaceOrientations: UIInterfaceOrientationMask {
        return .portrait
    }

    /// 由模态推出的视图控制器 优先支持的屏幕方向
    override var preferredInterfaceOrientationForPresentation: UIInterfaceOrientation {
        return .portrait
    }

}
```

``` bash
// UINavigationController 的基类
class BaseNavigationController: UINavigationController {

    override var shouldAutorotate: Bool {

        guard let VC = topViewController else {
            return false
        }
        return VC.shouldAutorotate
    }

    override var supportedInterfaceOrientations: UIInterfaceOrientationMask {

        guard let VC = topViewController else {
            return .portrait
        }
        return VC.supportedInterfaceOrientations
    }

    override var preferredInterfaceOrientationForPresentation: UIInterfaceOrientation {

        guard let VC = topViewController else {
            return .portrait
        }
        return VC.preferredInterfaceOrientationForPresentation
    }

}
```

``` bash
// UITabBarController 的基类
class BaseTabBarController: UITabBarController {

    override var shouldAutorotate: Bool {

        guard let VC = selectedViewController else {
            return false
        }
        return VC.shouldAutorotate
    }

    override var supportedInterfaceOrientations: UIInterfaceOrientationMask {

        guard let VC = selectedViewController else {
            return .portrait
        }
        return VC.supportedInterfaceOrientations
    }

    override var preferredInterfaceOrientationForPresentation: UIInterfaceOrientation {

        guard let VC = selectedViewController else {
            return .portrait
        }
        return VC.preferredInterfaceOrientationForPresentation
    }

}
```

## 屏幕旋转后的页面布局处理

1、在iOS 8之后，当屏幕旋转的时候， `UIScreen.main.bounds` 也发生了改变。也就是说横屏时候的屏幕宽度 其实是竖屏的时候屏幕的高度，所以在使用 `UIScreen.main.bounds.height` 的宏或者自定义函数的时候一定要注意。
使用屏幕宽高的时候，这样取值：

``` bash
let screenMax = max(UIScreen.main.bounds.height, UIScreen.main.bounds.width)
let screenMin = min(UIScreen.main.bounds.height, UIScreen.main.bounds.width)
```

2、监听通知，处理逻辑

监听 `NSNotification.Name.UIApplicationDidChangeStatusBarOrientation` 获取屏幕旋转的动作，做进一步的逻辑处理，实例：

``` bash
override func viewDidLoad() {
        super.viewDidLoad()

        NotificationCenter.default.addObserver(self, selector: #selector(handleStatusBarOrientationChange(_:)), name: .UIApplicationDidChangeStatusBarOrientation, object: nil)

    }

    deinit {
        NotificationCenter.default.removeObserver(self, name: .UIApplicationDidChangeStatusBarOrientation, object: nil)
    }

func handleStatusBarOrientationChange(_ userInfo: NSNotification) {

        let interfaceOrientation = UIApplication.shared.statusBarOrientation

        switch interfaceOrientation {

        case .portrait, .unknown, .portraitUpsideDown:
            table?.frame = CGRect.init(x: 0, y: 0, width: width, height: height)
            isPortrait = true

        case .landscapeRight, .landscapeLeft:
            table?.frame = CGRect.init(x: 0, y: 0, width: height, height: width)
            isPortrait = false

        }

    }
```

# 强制横屏

APP中某些页面，比如视频播放页，会要求一出现就横屏，或者点击横竖屏切换。

1、模态弹出 `ViewController` 情况下 强制横屏的设置

``` bash
let VC = PresentLandscapeViewController()
self.present(VC, animated: true, completion: nil)
```

在需要弹出的 `PresentLandscapeViewController` 中重写关键函数

``` bash
class PresentLandscapeViewController: BaseViewController {

    override var shouldAutorotate: Bool {
        return true
    }

    override var supportedInterfaceOrientations: UIInterfaceOrientationMask {
        return .landscapeLeft
    }

    override var preferredInterfaceOrientationForPresentation: UIInterfaceOrientation {
        return .landscapeLeft
    }

}

```

2、控制器中强制横竖屏的设置，适用于：push 一个 `ViewController` 或者页面内按钮控制

``` bash
func changeOrientation() {

        if isPortrait {
            isStatusBarPortrait = false
            UIDevice.current.setValue(UIInterfaceOrientation.landscapeLeft.rawValue, forKey: "orientation")
            UIApplication.shared.statusBarOrientation = .landscapeLeft
        }
        else {
            isStatusBarPortrait = true
            UIDevice.current.setValue(UIInterfaceOrientation.portrait.rawValue, forKey: "orientation")
            UIApplication.shared.statusBarOrientation = .portrait
        }

override var shouldAutorotate: Bool {
        return true
    }

override var supportedInterfaceOrientations: UIInterfaceOrientationMask {
        return isStatusBarPortrait ? .portrait : .landscapeLeft
    }

override var preferredInterfaceOrientationForPresentation: UIInterfaceOrientation {
        return isStatusBarPortrait ? .portrait : .landscapeLeft
    }
```

> 注意：做强制横屏操作的时候 `shouldAutorotate` 一定要返回 `true`

# 实例Demo

[Github](https://github.com/huxiaoyang/ScreenOrientations)