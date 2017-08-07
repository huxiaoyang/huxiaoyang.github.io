---
title: iOS日志系统
date: 2017-08-07 13:33:26
tags:
  - 工具
  - iOS
categories: iOS
---

几乎每个iOS开发者，都会使用 `NSLog` 来进行项目调试，把信息打印在Xcode控制台，快速识别分析。

但是，`NSLog` 并不是一个太友好的Log工具，使用 `NSLog` 打印大量Log，如果Log很频繁，比如使用 `CADisplayLink` 每一帧都打印Log，会明显的拖慢程序的运行速度。

虽然我们可以通过预编译宏做一些处理：

``` swift
#ifndef __OPTIMIZE__

#define NSLog(FORMAT, ...) do{\
fprintf(stderr,"<%s : ", [[[NSString stringWithUTF8String:__FILE__] lastPathComponent] UTF8String]);\
fprintf(stderr,"%zd> ", __LINE__);\
fprintf(stderr,"%s\n", __func__);\
fprintf(stderr,"%s\n", [[NSString stringWithFormat:FORMAT, ##__VA_ARGS__] UTF8String]);\
fprintf(stderr,"------------ * ------------\n\n");\
} while(0)

#else

# define NSLog(...) {}

#endif
```

但是，很多情况下我们不建议直接使用 `NSLog`。具体的原因，可以查看这篇博客：

[NSLog效率低下的原因及尝试lldb断点打印Log](http://blog.sunnyxx.com/2014/04/22/objc_dig_nslog/)

# CocoaLumberjack

不使用 `NSLog` ，我们同时又有打Log的需求，怎么办？

推荐使用 [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack)

Demo和用法，推荐查看官方文档。不想看英文，可以往下看

## 简单介绍CocoaLumberjack

`CocoaLumberjack` 全局日志级别分如下几种：

``` objc
typedef NS_ENUM(NSUInteger, DDLogLevel){
    /**
     *  不输出日志
     */
    DDLogLevelOff       = 0,

    /**
     *  仅仅能看到Error相关的日志输出
     */
    DDLogLevelError     = (DDLogFlagError),

    /**
     *  能看到Error、Warn相关的日志输出
     */
    DDLogLevelWarning   = (DDLogLevelError   | DDLogFlagWarning),

    /**
     *  能够看到Error、Warn、Info相关的日志输出
     */
    DDLogLevelInfo      = (DDLogLevelWarning | DDLogFlagInfo),

    /**
     *  能够看到Error/Warn/Info/Debug相关的日志输出
     */
    DDLogLevelDebug     = (DDLogLevelInfo    | DDLogFlagDebug),

    /**
     * 能够看到所有级别的日志输出
     */
    DDLogLevelVerbose   = (DDLogLevelDebug   | DDLogFlagVerbose),

    /**
     *  能够看到所有级别的日志输出
     */
    DDLogLevelAll       = NSUIntegerMax
};

```

同时，`CocoaLumberjack` 还设定了具体Log级别

``` objc
typedef NS_OPTIONS(NSUInteger, DDLogFlag){
    /**
     *  0...00001 DDLogFlagError
     */
    DDLogFlagError      = (1 << 0),

    /**
     *  0...00010 DDLogFlagWarning
     */
    DDLogFlagWarning    = (1 << 1),

    /**
     *  0...00100 DDLogFlagInfo
     */
    DDLogFlagInfo       = (1 << 2),

    /**
     *  0...01000 DDLogFlagDebug
     */
    DDLogFlagDebug      = (1 << 3),

    /**
     *  0...10000 DDLogFlagVerbose
     */
    DDLogFlagVerbose    = (1 << 4)
};

```

> 注意：DDLogLevel 定义了全局的 log 等级，DDLogFlag 是我们打 log 时设定的 log 等级，CocoaLumberjack 会比较两者，如果 flag 低于 level，则不会打 log。

## 开始使用

使用 `Cocoapods` 或者其它方式将 `CocoaLumberjack` 工程引入项目。

然后我们在项目中来具体配置使用：

### 第一步

我们创建一个工具类 `HYLogManager`，用来管理集中配置。

``` objc
+ (void)setupLog {
    [DDLog addLogger:[DDTTYLogger sharedInstance]]; // TTY = Xcode console
    [DDLog addLogger:[DDASLLogger sharedInstance]]; // ASL = Apple System Logs

    DDFileLogger *fileLogger = [[DDFileLogger alloc] init]; // File Logger
    fileLogger.rollingFrequency = 60 * 60 * 24; // 24 hour rolling
    fileLogger.logFileManager.maximumNumberOfLogFiles = 7;
    [DDLog addLogger:fileLogger withLevel:DDLogLevelError];
}
```

然后在 `application:didFinishLaunchingWithOptions:` 方法中使用 `[HYLogManager setupLog]` 初始化。

### 第二步

使用宏来处理，指定日志的记录级别。可以在`PCH`文件或者我们创建的工具类 `HYLogManager.h` 中设置。

``` objc
//通过DEBUG模式设置全局日志等级，DEBUG时为Verbose，所有日志信息都可以打印，否则Error，只打印
#ifdef DEBUG
static const DDLogLevel ddLogLevel = DDLogLevelVerbose;
#else
static const DDLogLevel ddLogLevel = DDLogLevelError;
#endif
```

之后我们就可以在代码中使用：

``` objc
DDLogError(@"[Error]:%@", @"输出错误信息");//输出错误信息
DDLogWarn(@"[Warn]:%@", @"输出警告信息");//输出警告信息
DDLogInfo(@"[Info]:%@", @"输出描述信息");//输出描述信息
DDLogDebug(@"[Debug]:%@", @"输出调试信息");//输出调试信息
DDLogVerbose(@"[Verbose]:%@", @"输出详细信息");//输出详细信息
```

来替换 `NSLog`。

### 第三步

如果我们想要 `format` 日志，比如我们想要打印出文件名、函数名、行号等信息。

有两种方式：

1. 配置一个宏

    ``` objc
    #define HYLog(format, ...) DDLogError((@"[%@][Line:%d]:" format), [[NSString stringWithUTF8String:__FILE__] lastPathComponent], __LINE__, ## __VA_ARGS__)
    ```
    使用：`HYLog(@"User selected file:%@ withSize:%d", @"filePath", 123)`

    打印结果：`2017-08-07 14:42:41:268 BSKit[33893:4665914] [AppDelegate.m][Line:87]:User selected file:filePath withSize:123`

1. 实现 `DDLogFormatter` 协议

    ``` objc
    @interface TestFormatter : NSObject <DDLogFormatter>

    @end

    @implementation TestFormatter

    - (NSString *)formatLogMessage:(DDLogMessage *)logMessage
    {
        return [NSString stringWithFormat:@"%@ | %@ @ %@ | %@",
                [logMessage fileName], logMessage->_function, @(logMessage->_line), logMessage->_message];
    }
    ```
    然后，在 `HYLogManager` 中将初始化方法改为

    ``` objc
    + (void)setupLog {
        [[DDASLLogger sharedInstance] setLogFormatter:formatter];
        [[DDTTYLogger sharedInstance] setLogFormatter:formatter];

        [DDLog addLogger:[DDTTYLogger sharedInstance]]; // TTY = Xcode console
        [DDLog addLogger:[DDASLLogger sharedInstance]]; // ASL = Apple System Logs

        DDFileLogger *fileLogger = [[DDFileLogger alloc] init]; // File Logger
        fileLogger.rollingFrequency = 60 * 60 * 24; // 24 hour rolling
        fileLogger.logFileManager.maximumNumberOfLogFiles = 7;
        [DDLog addLogger:fileLogger withLevel:DDLogLevelError];
    }
    ```
    使用：`DDLogInfo(@"User selected file:%@ withSize:%d", @"filePath", 123)`

### 第四步

日志上传，很多时候我们上传的是 `crash` 信息，如果想上传用户的行为信息，也可以。

可以从 `DDLogFileManager protocol` 下面的两个协议中获取Log文件 `rolling` 时的通知，便可以在此时将Log文件上传：

``` objc
- (void)didArchiveLogFile:(NSString *)logFilePath;
- (void)didRollAndArchiveLogFile:(NSString *)logFilePath;
```

# Aspects

配置完后，我们要手机Log，如果需要统计用户行为用于分析。那么我们需要在工程项目的各个部分打Log。

我们不可能一个个去找方法添加代码，这样是体力活。

那么我们怎么优雅的添加Log呢？

答案是使用 [Aspects-iOS面向切面编程框架](https://github.com/steipete/Aspects)

## 安装

使用 `Cocoapods`

``` ruby
pod 'Aspects'
```

## 使用

`Aspects` 使用很简单，我们这里介绍的是如何写一个配置文件和一个解析器来集中处理Log

### 首先，创建配置文件

创建一个配置类 `BSAOPConfigureManager`，在这里配置项目中需要打Log的方法。

头文件：

``` objc
#define BSLoggingTrackedEvents @"com.XiaoYang.BSLoggingTrackedEvents"
#define BSLoggingEventName @"com.XiaoYang.BSLoggingEventName"
#define BSLoggingEventSelectorName @"com.XiaoYang.BSLoggingEventSelectorName"
#define BSLoggingEventHandlerBlock @"com.XiaoYang.BSLoggingEventHandlerBlock"



typedef void (^AspectHandlerBlock)(id<AspectInfo> aspectInfo);



@interface BSAOPConfigureManager : NSObject

+ (instancetype)shareInstance;

@property (nonatomic, strong) NSDictionary *configs;

@end
```

实现文件：

``` objc
@implementation BSAOPConfigureManager

+ (instancetype)shareInstance {
    static BSAOPConfigureManager *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[BSAOPConfigureManager alloc] init];
    });
    return instance;
}

- (instancetype)init {
    self = [super init];
    if (self) {

    }
    return self;
}

- (NSDictionary *)configs {
    if (!_configs) {
        _configs = @{
                    @"CollectionViewController": @{
                            BSLoggingTrackedEvents: @[
                                    @{
                                        BSLoggingEventName: @"button one clicked",
                                        BSLoggingEventSelectorName: @"buttonOneClicked:selectItemAtIndexPath:",
                                        BSLoggingEventHandlerBlock: ^(id<AspectInfo> aspectInfo, UICollectionView *collectionView, NSIndexPath *indexPath) {

                                            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
                                                NSLog(@"button one clicked");

                                                NSLog(@"class is %@", [aspectInfo.instance class]);

                                                NSLog(@"argment is %@ --- %zd", collectionView, indexPath.row);
                                            });

                                        },
                                        },
                                    @{
                                        BSLoggingEventName: @"button two clicked",
                                        BSLoggingEventSelectorName: @"buttonTwoClicked:",
                                        BSLoggingEventHandlerBlock: ^(id<AspectInfo> aspectInfo) {
                                            NSLog(@"button two clicked");
                                        },
                                        },
                                    @{
                                        BSLoggingEventName: @"button three clicked",
                                        BSLoggingEventSelectorName: @"buttonThreeClicked:",
                                        BSLoggingEventHandlerBlock: ^(id<AspectInfo> aspectInfo) {
                                            NSLog(@"button three clicked");
                                        },
                                        },
                                    ],
                            },
                    };
    }
    return _configs;
}

@end
```

### 其次，实现解析器

创建一个 `NSObject` 子类，重载它的 `load` 方法，实现最小的对业务方的“代码入侵” 。

``` objc
+ (void)load {
    dispatch_once(&onceToken, ^{
            [BSViewControllerIntercepter setupIntercepter];
        });
}

+ (void)setupIntercepter {
    // 拦截自定义事件
    // 我们配置的方法，有可能被更改，或者代码出错，导致该方法不存在。Aspect在hook方法时会返回「Unable to find selector」的error信息
    // 解析的过程可以放在异步线程，或者下一次loop中
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSDictionary *configs = [[BSAOPConfigureManager shareInstance] configs];
        for (NSString *className in configs) {

            Class clazz = NSClassFromString(className);
            NSDictionary *config = configs[className];

            if (config[BSLoggingTrackedEvents]) {
                for (NSDictionary *event in config[BSLoggingTrackedEvents]) {

                    SEL selector = NSSelectorFromString(event[BSLoggingEventSelectorName]);
                    AspectHandlerBlock block = event[BSLoggingEventHandlerBlock];

                    NSError *error;
                    id s = [clazz aspect_hookSelector:selector
                                          withOptions:AspectPositionAfter
                                           usingBlock:block
                                                error:&error];

                    if (error) {
                        NSLog(@"error is %@", error);
                    }

                    NSLog(@"%@", s);

                }
            }

        }
    });

}
```

# GCDWebServer

我们有了日志打印，但是我们怎么能够查看它呢，使用真机调试的时候，除了上传到服务端，还有什么快速查看的方式呢？

1. 如果 `CocoaLumberJack` 开启了 `DDASLLogger`。也就是初始化了这句代码 `[DDLog addLogger:[DDASLLogger sharedInstance]] `

    我们可以通过Xcode -> window -> devices -> 选中当前设备 -> view device Logs 查看所有当前设备的log

1. 但是如果我们为了安全考虑，不开启 `DDASLLogger` 服务。我们可以使用 `GCDWebServer`

## 介绍

`GCDWebServer` 是一个基于 GCD 的轻量级服务器框架，用于内嵌到 MacOS 或者 iOS 系统的应用中提供 HTTP1.1 的服务。使用 GCDWebServer 我们可以很轻松的在我们的应用中搭建一个 HTTP 服务器，让用户可以通过浏览器访问我们应用中的数据。或者使用 GCDWebServer 来实现一个无线U盘 App。

`GCDWebServer` 功能很强大，这里只简单使用它的共享功能。

## 安装

引入框架，依然使用 `Cocoapods` ，根据我们的需求，我使用

``` objc
pod 'GCDWebServer/WebUploader'
```

## 初始化

这里我们只想查看Log，所以在 `application:didFinishLaunchingWithOptions:` 中添加代码：

``` objc
#ifdef DEBUG
NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) firstObject];
    NSString *logPath = [documentsPath stringByAppendingPathComponent:@"Logs"];
    _uploader = [[GCDWebUploader alloc] initWithUploadDirectory:logPath];
#endif
```

运行之后，使用 `Chrome` 查看 `http://iphone.local/`，我们会看到 `.log` 文件，这就是我们的日志，下载查看即可

# 参考

 [iOS用CocoaLumberJack抓取crash日志上传](http://www.jianshu.com/p/ea1e6b210b27)

 [iOS异常捕获](http://www.iosxxx.com/blog/2015-08-29-iosyi-chang-bu-huo.html)

 [iOS收集崩溃信息一点思考](https://github.com/enoughpower/iOS_Notes/wiki/iOS%E6%94%B6%E9%9B%86%E5%B4%A9%E6%BA%83%E4%BF%A1%E6%81%AF%E4%B8%80%E7%82%B9%E6%80%9D%E8%80%83)