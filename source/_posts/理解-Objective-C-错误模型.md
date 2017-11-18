---
title: 理解 Objective-C 错误模型
date: 2017-11-18 12:25:02
tags: 
  - 内存
categories: Objective-C
---

很多编程语言有"异常(exception)机制"，Objective-C也不例外，但是我们在自己使用Objective-C写代码或者阅读其它Objective-C代码的时候，很少看见有"异常"代码。

为什么呢？

# Exception

"自动引用计数"（ARC）在默认情况下并不是『异常安全的』。具体来说，这意味着：如果抛出异常，那么本应在作用域结尾释放的对象现在却不能释放了。

那我们可以使ARC『异常安全』吗？

可以，只需要设置编译器的标志，就可以生成『异常安全』的代码，不过这会引入一些副作用，会产生额外的代码，即使在不抛出异常的时候，也照样要执行这部分代码。这就无形中增加了程序的开销。

> 需要打开的编译器标志是 `-fobjc-arc-exceptions`

也许有人会说，那如果我们不使用ARC呢？

即使不使用ARC，也很难写出在抛出异常时不导致内存泄露代码。

比如下面这段代码：

``` objc
id someResource = /*......*/;
if (/*.. check for error .*/) {
    @throw [NSException exceptionWithName:@"NSExceptionName"
                            reason:@"There are an error"
                          userInfo:nil];
}
[someResource doSomething];
[someResource release];
```

在抛出异常后，资源并不会被释放。也许我们可以在抛出异常之前先释放`someResource`。我们当然可以这么做，但是这样会照成代码的执行路径很复杂，代码可读性很差，而且开发者经常会忘记在抛出异常之前先释放它，我们不能把希望寄托于程序员永远不犯错，这是不可能的。

> 所以，Objective-C语言通常采用的方式是：只在极其罕见的情况下抛出异常，异常抛出之后，无需考虑回复问题，而且应用此时应该退出。这就意味着不用再编写复杂的『异常安全』代码了。

异常只应该用于极其严重的错误，该错误不可挽回，一般会使程序`crash`，或者强制`crash`。

# Error

既然异常只用于处理严重错误，那么普通的错误怎么办呢？

在出现『非致命错误』时，Objective-C语言的编程范式为：令方法返回`nil/0`，或者使用`NSError`，以表明其中有错误发生。

比如说：

```objc
- (id)initWithValue:(id)value {
    self = [super init];
    if (self) {
        if (/** 参数导致对象不能被正常初始化 */) {
            self = nil;
        } else {
            /** 正常初始化 */
        }
    }
    return self;
}
```

调用者在发现返回 `nil` 的时候，意味着初始化方法并没有把实例创建好，于是便可确定其中发生了错误。

## NSError

NSError的语法更灵活，因为经由此对象，我们可以把错误原因返回给调用者。

NSError对象封装了三条信息：

- Error domain （错误范围，字符串类型）

  错误发生的范围，也就是产生错误的根源，通常用一个特有的全局变量来定义。比如，"处理URL的子系统"在从URL中解析或取得数据时如果出错了，那么就会使用`NSURLErrorDomain`来表明错误。

- Error code （错误码， 整型）

  独有的错误代码，用来指明某个范围内具体发生了何种错误。某个特定范围内可能会发生一系列相关错误，这些错误通常采用`enum`来定义。例如，当HTTP出错时，可能会把HTTP状态码设为错误码。

- User info （用户信息，字典类型）

  有关此错误的额外信息，其中或许包含一段 "本地化描述"，或许还包含有导致该错误发生的另外一个错误，经由此种消息，可将相关错误串成一条『错误链』

在设计API的时候，NSError的第一种常见的用法是通过委托（Delegate）来传递错误。有错误发生时，当前对象会把错误信息经由协议里的某个方法传递给其委托对象，例如，NSURLConnection在其委托协议NSURLConnectionDelegate之中就定义了如下方法

```objc
- (void)connection:(NSURLConnection *)connection
  didFailWithError:(NSError *)error;
```

当NSURLConnection出错后（比如远程服务器链接操作超时），就会调用此方法以处理相关错误。这个委托方法并不是非得实现不可：是不是必须处理此错误，可交由NSURLConnection类的用户来判断。这比抛出异常要好，因为调用者至少可以自己决定NSURLConnection是否回报此错误。

`NSError` 的另一种常见用法是：经由方法的 "输出参数" 返回给调用者。

例如：

```objc
- (BOOL)doSomething:(NSError **)error;
```

传递给方法的参数是个指针，而该指针本身又指向另一个指针，那个指针指`NSError`对象。或者也可以把它当成一个直接指向`NSError`对象的指针。这样一来，此方法不仅能有普通的返回值，而且还能经由 "输出参数" 把`NSError`对象回传给调用者。其用法如下：

```objc
NSError *error = nil
BOOL ret = [object doSomething:&error];
if (error) {
  // 这里处理错误
}
```

像这样的方法一般都会返回Boolean值，用以表示操作是成功还是失败。如果调用者不关注具体的错误信息，那么直接判断这个Boolean值就可以了。若是关注具体的错误信息，那就检查经由 "输出参数" 所返回的错误对象。在不想知道具体错误信息的时候，可以给error参数传入nil。例如：

```objc
BOOL ret = [object doSomething:nil];
if (ret) {
  // 操作
}
```

"输出参数" 很神奇，那么它具体是怎么工作的呢？

实际上，在使用ARC的时，编译器会把方法签名中的 `NSError **` 转换成 `NSError* __autoreleasing*`，也就是说，指针所指的对象会在方法执行完毕后自动释放。这个对象必须自动释放，因为 "doSomething:" 方法不能保证其调用者可以把此方法创建的 `NSError` 释放掉，所以必须加入autorelease。这就与大部分方法（以new、alloc、copy、mutableCopy开头的方法不在此列）的返回值所具备的语义相同了。

该方法通过下列方式把 `NSError` 对象传递到 "输出参数" 中：

```objc
- (BOOL)doSomething:(NSError **)error {
    /// 具体的能产生错误的操作

    if (/** 发生错误 */) {
        if (error) {
            *error = [NSError errorWithDomain:domain
                                         code:code
                                     userInfo:userInfo];
        }
        return NO;
    } else {
        return YES;
    }
}
```

这段代码以 `*error` 语法为 `error` "解引用"，也就是说，`error` 所指向的那个指针现在要指向一个新的 `NSError` 对象了。在解引用之前，必须先保证 `error` 参数不是 nil，因为空指针解引用会导致 "段错误（也就是越界访问）" 并使应用程序崩溃。调用者在不关心具体错误的时候，会给 `error` 参数传入nil，所以必须判断这种情况。

# Safe Exception

在使用MRC时，编写安全异常代码，应该是如下方式：

```objc
HXYSomeClass *object;
@try {
    object = [[HXYSomeClass alloc] init];
    [object doSomethingThatMayThrow];
}
@catch (...) {
    NSLog(@"这里有错误")
}
@finally {
    [object release];
}
```

无论是否发生异常，`@finally` 中的代码都会执行，且只执行一次。

在ARC环境下，我们不能手动执行`release`，所以没必要使用 `@finally` 也不能解决问题。虽然前面我们讲了，只在极端情况下应用因为异常状态而终止时才应抛出异常，因为这时候程序即将终止，那么是否会发生内存泄露已经无关紧要了。但是，如果你一意孤行，有错误就想要抛出异常，这时候就必须打开异常安全标志 `-fobjc-arc-exceptions`，这个标志是默认不打开的。

有一种情况是默认打开的，就是处于 `Objective-C++` 模式时，因为C++频繁使用异常，所以 `Objective-C++`程序员很可能也会使用异常。

但是此处依然建议不要在非极端情况下使用异常，如果你的代码里频出现异常，首先应考虑的是重构你的代码。用前面所讲的 `NSError` 传递错误信息来取代抛出异常。