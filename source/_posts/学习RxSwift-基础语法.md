---
title: RxSwift-基础语法
tags:
  - RxSwift
  - Swift
categories: RxSwift
---

# RxSwift

## Observable

### 创建

#### asObservable()

##### 相当于clone方法，也就是说必须先有Observable对象才能调用asObservable方法。

``` swift
var obs = Observable<String>.create { (observer) -> Disposable in
  observer.on(.Next("hahah"))
  observer.on(.Next("deasd"))
  observer.on(.Completed)
  return NopDisposable.instance
}

let observable = obs.asObservable()

observable.subscribeOn(MainScheduler.instance)
.subscribe{ event in
  print(event.debugDescription)
}
```

#### create

##### 最基本创建方式

``` swift
let disposeBag = DisposeBag()

//创建
let observable = Observable<String>.create { observerOfString -> Disposable in
	observerOfString.on(.next("😬"))
	observerOfString.on(.completed)
	return Disposables.create()
}

// 订阅
observable.subscribe { print($0) }
.disposed(by: disposeBag)

// 输出
output:
next(😬)
completed
``` 

#### never

##### 创建一个序列，不会终止也不会发出任何事件

``` swift
let disposeBag = DisposeBag()
let neverSequence = Observable<String>.never()

let neverSequenceSubscription = neverSequence
.subscribe { _ in
print("This will never be printed")
}

neverSequenceSubscription.disposed(by: disposeBag)
```

#### empty

##### 创建一个空的序列，只会发出一个完成事件

``` swift
let obs1 = Observable<String>.empty()
       
obs1.subscribe(
	onNext: {str in 
    	print(str)
     },
     onError: { (errorType) -> Void in
     	print(errorType)
     },
     onCompleted: { () -> Void in
     	print("complete")
     },
     onDisposed: {() -> Void in
     	print("dispose")
     }
)
        
output:    
completed
dispose 
``` 

#### just

##### 创建一个单个元素的序列

* 使用just方法不能将一组数据一起处理，只能一个一个处理

``` swift
let disposeBag = DisposeBag()
    
    Observable.just("🔴")
        .subscribe { event in
            print(event)
        }
        .disposed(by: disposeBag)

output:    
next(🔴)
completed
```

##### just方法是一个多态方法，允许在传入参数时候指定线程

* 指定当前线程完成subscribe相关事件

``` swift
Observable<String>
     .just("just with Scheduler", scheduler: CurrentThreadScheduler.instance)
     .subscribeNext({ (str) -> Void in
                print(str)
     })
     .dispose()
```

#### of

##### 使用固定数量的元素创建一个序列

* just的升级版，同样存在一个多态方法，可以带入线程控制

``` swift

let disposeBag = DisposeBag()
    
    Observable.of("🐶", "🐱", "🐭", "🐹")
        .subscribe(onNext: { element in
            print(element)
        })
        .disposed(by: disposeBag)
    
output:
🐶
🐱
🐭
🐹

```

#### from

##### 从一个序列创建一个可被观察的序列

* 就是将一个Array变成一个一个可观察序列

``` swift

let disposeBag = DisposeBag()
    
    Observable.from(["🐶", "🐱", "🐭", "🐹"])
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
output:
🐶
🐱
🐭
🐹

```

#### range

##### 创建一个发出一系列顺序整数然后终止的序列

*  Range方法其实方便版of方法，其功能和of差不多，我们只要输出start和count然后就能生成一组数据，让他们执行onNext

``` swift

let arr: [String] = ["ad", "cd", "ef", "gh"]
let disposeBag = DisposeBag()
    
    Observable.range(start: 1, count: 10)
        .subscribe { print($0) }
        .disposed(by: disposeBag)

output:
cd
ef
gh

```

#### repeatElement

##### 创建一个给予元素的无限序列

* repeatElement是一个无限循环的，它会一直循环onNext方法
* 这种循环是可以指定线程的
* 这里的take(3)表示只取前3个元素

``` swift

let disposeBag = DisposeBag()
    
    Observable.repeatElement("🔴")
        .take(3)
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)

output:
🔴
🔴
🔴

```

#### generate

##### 创建一个满足条件的序列

* generate方法是一个迭代器，它一直循环onNext事件，直到condition不满足要求退出。
* generate有四个参数：
	1. 最开始的循环变量
    2. 条件
    3. 迭代器，这个迭代器每次运行都会返回一个E类型，作为下一次是否执行onNext事件源，而是否正的要执行则看是否满足condition条件
    4. 调度器，可选参数，可以指定线程

* 下面的例子：初始变量是0，满足条件，执行onNext方法，同时通过迭代器"$0+1"生成一个1，继续满足条件执行onNext方法，直到不满足条件停止

``` swift

let disposeBag = DisposeBag()
    
    Observable.generate(
            initialState: 0,
            condition: { $0 < 3 },
            iterate: { $0 + 1 }
        )
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
        
output:
0
1
2

```


#### deferred

##### 只有当有订阅者订阅的时候才会去创建序列

* deferred不是第一步创建Observable，而是在subscriber的时候创建的
* 如果把后面两个订阅去掉的话，是不会有Creating输出的

``` swift

let disposeBag = DisposeBag()
    var count = 1
    
    let deferredSequence = Observable<String>.deferred {
        print("Creating \(count)")
        count += 1
        
        return Observable.create { observer in
            print("Emitting...")
            observer.onNext("🐶")
            observer.onNext("🐱")
            observer.onNext("🐵")
            return Disposables.create()
        }
    }
    
    deferredSequence
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    deferredSequence
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
        
output:
Creating 1
Emitting...
🐶
🐱
🐵
Creating 2
Emitting...
🐶
🐱
🐵

```

#### error

##### 创建一个没有元素并以错误终止的序列

* error方法是返回一个只能调用onError方法的Observable序列。其中的onNext和OnComleted方法是不会执行的

``` swift

Observable<String>
            .error(RxError.Timeout)
            .subscribe(
                onNext: { (str) -> Void in
                   print(str)
                   print("onNext")
                },
                onError: { (error)-> Void in
                    print(error)
                },
                onCompleted: { () -> Void in
                    print("onCompleted")
                },
                onDisposed: { () -> Void in
                    print("dispose")
                })
             .dispose()
             
output:
Sequence timeout
dispose

```

#### doOn

##### 在每个事件发出之后，可以调用其它处理，然后返回原事件

* 相当于一个拦截器，但是只能拦截不能修改
* 可以针对不同的事件类型单独拦截

``` swift

let disposeBag = DisposeBag()
    
    Observable.of("🍎", "🍐", "🍊", "🍋")
        .do(onNext: { print("Intercepted:", $0) }, onError: { print("Intercepted error:", $0) }, onCompleted: { print("Completed")  })
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
        
output:
Intercepted: 🍎
🍎
Intercepted: 🍐
🍐
Intercepted: 🍊
🍊
Intercepted: 🍋
🍋
Completed

```

### 操作

#### 合并操作

##### startWith

###### 在序列触发值之前插入一个多个元素的特殊序列

* 最后插入的元素数组在最前面

``` swift

let disposeBag = DisposeBag()
    
Observable.of("🐶", "🐱", "🐭", "🐹")
    .startWith("1")
    .startWith("2")
    .startWith("3", "🅰️", "🅱️")
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

output:
3
🅰️
🅱️
2
1
🐶
🐱
🐭
🐹

```

##### merge

###### 把多个序列合并成单个序列，并按照事件触发的先后顺序，依次发射值。

* 当其中某个序列发生了错误就会立即把错误发送到合并的序列并终止

``` swift

let disposeBag = DisposeBag()
    
let subject1 = PublishSubject<String>()
let subject2 = PublishSubject<String>()

Observable.of(subject1, subject2)
    .merge()
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
subject1.onNext("🅰️")
subject1.onNext("🅱️")
subject2.onNext("①")
subject2.onNext("②")
subject1.onNext("🆎")
subject2.onNext("③")

output:
🅰️
🅱️
①
②
🆎
③

```

##### zip

###### 把多个序列组合成到一起并触发一个值

* 但只有每一个序列都发射了一个值之后才会组合成一个新的值并发出来

``` swift

let disposeBag = DisposeBag()
    
let stringSubject = PublishSubject<String>()
let intSubject = PublishSubject<Int>()

Observable.zip(stringSubject, intSubject) { stringElement, intElement in
    "\(stringElement) \(intElement)"
    }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
stringSubject.onNext("🅰️")
stringSubject.onNext("🅱️")

intSubject.onNext(1)
intSubject.onNext(2)

stringSubject.onNext("🆎")
intSubject.onNext(3)

output:
🅰️ 1
🅱️ 2
🆎 3

```

##### combineLatest

###### 获取两个序列的最新值，并通过某个函数对其进行处理，处理完之后返回一个新的发射值

``` swift

let disposeBag = DisposeBag()
    
let stringSubject = PublishSubject<String>()
let intSubject = PublishSubject<Int>()

Observable.combineLatest(stringSubject, intSubject) { stringElement, intElement in
        "\(stringElement) \(intElement)"
    }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
stringSubject.onNext("🅰️")

stringSubject.onNext("🅱️")
intSubject.onNext(1)

intSubject.onNext(2)

stringSubject.onNext("🆎")

output:
🅱️ 1
🅱️ 2
🆎 2

```

* combineLatest还有一个变体可以接受一个数组，或者任何其他可被观察序列的集合。但是要求这些可被观察序列元素是同一类型

``` swift

let disposeBag = DisposeBag()

let stringObservable = Observable.just("❤️")

let fruitObservable = Observable.from(["🍎", "🍐", "🍊"])

let animalObservable = Observable.of("🐶", "🐱", "🐭", "🐹")

Observable.combineLatest([stringObservable, fruitObservable, animalObservable]) {
        "\($0[0]) \($0[1]) \($0[2])"
    }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
❤️ 🍎 🐶
❤️ 🍐 🐶
❤️ 🍐 🐱
❤️ 🍊 🐱
❤️ 🍊 🐭
❤️ 🍊 🐹

```

##### switchLatest

###### 每当一个新的序列发射时，原来序列将被丢弃

``` swift

let disposeBag = DisposeBag()
    
let subject1 = BehaviorSubject(value: "⚽️")
let subject2 = BehaviorSubject(value: "🍎")
let variable = Variable(subject1)
    
variable.asObservable()
    .switchLatest()
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
subject1.onNext("🏈")
subject1.onNext("🏀")

variable.value = subject2   //这是发射了一个新的序列，而不是直接覆盖原来的序列

subject1.onNext("⚾️")  //这里发射的值被忽略了

subject2.onNext("🍐")

output:
⚽️
🏈
🏀
🍎
🍐

```

##### sample

###### 为当前事件关联一个目标事件

* 当收到目标事件，就会从源序列取一个最新的事件，发送到序列，如果两次目标事件之间没有源序列的事件，则不发射值

``` swift

let source = PublishSubject<Int>()
let target = PublishSubject<String>()

let subscription = source
    .sample(target)
    .subscribe { event in
        print(event)
}

source.onNext(1)
target.onNext("A")  //获取最新的source

source.onNext(2)

source.onNext(3)
target.onNext("B")  //获取最新的source

target.onNext("C")  //没有最新的source，不发射

output:
next(1)
next(3)

```

#### 转换操作

##### map

###### 转换其中的每一个元素

``` swift

let disposeBag = DisposeBag()

Observable.of(1, 2, 3)
    .map { $0 * $0 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
1
4
9

```

##### faltMap

###### 把当前序列的元素转换成一个新的序列

* 把当前序列的元素转换成一个新的序列，并把他们合并成一个序列，这个在我们的一个可被观察者序列本身又会触发一个序列的时候非常有用，比如发送一个新的网络请求

``` swift

let disposeBag = DisposeBag()
    
struct Player {
    var score: Variable<Int>
}

let 👦🏻 = Player(score: Variable(80))
let 👧🏼 = Player(score: Variable(90))

let player = Variable(👦🏻)

player.asObservable()
    .flatMap { $0.score.asObservable() } 
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
👦🏻.score.value = 85

player.value = 👧🏼

👦🏻.score.value = 95 // Will be printed when using flatMap, but will not be printed when using flatMapLatest

👧🏼.score.value = 100

output:
80
85
90
95
100

```


##### flatMapLatest

###### 收到一个新的序列的时候，会丢弃原有的序列

* 和faltMap不同的是，flatMapLatest在收到一个新的序列的时候，会丢弃原有的序列

* 那么我们把上面的代码里面的flatMap改成flatMapLatest，试验一下

##### scan

###### 给予一个初始值，依次对每个元素进行操作，最后返回操作的结果

``` swift

let disposeBag = DisposeBag()
    
Observable.of(10, 100, 1000)
    .scan(1) { aggregateValue, newValue in
        aggregateValue + newValue
    }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
11
111
1111

```

#### 过滤方法

##### filter

###### 过滤序列中符合指定条件的值

``` swift

let disposeBag = DisposeBag()
    
Observable.of(
    "🐱", "🐰", "🐶",
    "🐸", "🐱", "🐰",
    "🐹", "🐸", "🐱")
    .filter {
        $0 == "🐱"
    }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
🐱
🐱
🐱

```

##### distinctUntilChanged

###### 过滤掉连续发射的重复元素。

``` swift

let disposeBag = DisposeBag()
    
Observable.of("🐱", "🐷", "🐱", "🐱", "🐱", "🐵", "🐱")
    .distinctUntilChanged()
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
🐱
🐷
🐱
🐵
🐱

```

##### elementAt

###### 只发送指定位置的值

``` swift

let disposeBag = DisposeBag()
    
Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
    .elementAt(3)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
🐸

```


##### single

###### 发送单个元素，或者满足条件的第一个元素

* 如果有多个元素或者没有元素都会抛出错误

``` swift
// 错误示例:

Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .single()
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
        
output:
🐱
Received unhandled error: /var/folders/h3/8n169g610_g69k50z7g_q4gc0000gp/T/./lldb/77046/playground61.swift:69:__lldb_expr_61 -> Sequence contains more than one element.

```

* 如果这里只有一个元素，则不会报错

``` swift
Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .single { $0 == "🐸" }
        .subscribe { print($0) }
        .disposed(by: disposeBag)
    
Observable.of("🐱", "🐰", "🐶", "🐱", "🐰", "🐶")
    .single { $0 == "🐰" }
    .subscribe { print($0) }
    .disposed(by: disposeBag)
    
Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
    .single { $0 == "🔵" }
    .subscribe { print($0) }
    .disposed(by: disposeBag)
    
output:
next(🐸)
completed

next(🐰)
error(Sequence contains more than one element.)

error(Sequence doesn't contain any elements.)


```

##### take

###### 获取序列前多少个值

``` swift

let disposeBag = DisposeBag()
    
Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
    .take(3)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
}

output:
🐱
🐰
🐶

```

##### takeLast

###### 获取序列后多少个值

``` swift

let disposeBag = DisposeBag()
    
Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
    .takeLast(3)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
🐸
🐷
🐵

```

##### takeWhile

###### 获取满足条件的值，直到条件变成false

* 发射值直到条件变成false，变成false后，后面满足条件的值也不会发射

``` swift

let disposeBag = DisposeBag()
    
Observable.of(1, 2, 3, 4, 5, 6，1，2)
    .takeWhile { $0 < 4 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
1
2
3

```

##### takeUntil

###### 发射原序列，直到新的序列发射了一个值

``` swift

let disposeBag = DisposeBag()
    
let sourceSequence = PublishSubject<String>()
let referenceSequence = PublishSubject<String>()

sourceSequence
    .takeUntil(referenceSequence)
    .subscribe { print($0) }
    .disposed(by: disposeBag)
    
sourceSequence.onNext("🐱")
sourceSequence.onNext("🐰")
sourceSequence.onNext("🐶")

referenceSequence.onNext("🔴")

sourceSequence.onNext("🐸")
sourceSequence.onNext("🐷")
sourceSequence.onNext("🐵")

output:
next(🐱)
next(🐰)
next(🐶)
completed

```

##### skip

###### 跳过开头指定个数的值

``` swift
let disposeBag = DisposeBag()
    
Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
    .skip(2)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
🐶
🐸
🐷
🐵

```

##### skipWhile

###### 跳过满足条件的值，直到条件变成false

* 跳过满足条件的值到条件变成false，变成false后，后面满足条件的值也不会跳过

``` swift

let disposeBag = DisposeBag()
    
Observable.of(1, 2, 3, 4, 5, 6, 1, 2)
    .skipWhile { $0 < 4 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
4
5
6
1
2

```

##### skipWhileWithIndex

###### 和skipWhile类似，只不过带上了index

``` swift

let disposeBag = DisposeBag()
    
Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
    .skipWhileWithIndex { element, index in
        index < 3
    }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
output:
🐸
🐷
🐵

```

##### skipUntil

###### 和takeUntil相反，跳过原序列，直到新序列发射了一个值

``` swift

let disposeBag = DisposeBag()
    
let sourceSequence = PublishSubject<String>()
let referenceSequence = PublishSubject<String>()

sourceSequence
    .skipUntil(referenceSequence)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
    
sourceSequence.onNext("🐱")
sourceSequence.onNext("🐰")
sourceSequence.onNext("🐶")
referenceSequence.onNext("🔴")
sourceSequence.onNext("🐸")
sourceSequence.onNext("🐷")
sourceSequence.onNext("🐵")
output:
🐸
🐷
🐵

```

#### 聚合操作

##### toArray

###### 把一个序列转成一个数组，然后作为新的一个值发射

``` swift

let disposeBag = DisposeBag()
    
Observable.range(start: 1, count: 10)
    .toArray()
    .subscribe { print($0) }
    .disposed(by: disposeBag)

output:
next([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
completed

```

##### reduce

###### 给一个初始值，然后和序列里的每个值进行运行，最后返回一个结果，然后把结果作为单个值发射出去

``` swift

let disposeBag = DisposeBag()
    
Observable.of(10, 100, 1000)
    .reduce(1, accumulator: +)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

output:
1111

```

##### concat

###### 串联多个序列，下一个序列必须等前一个序列完成才会发射出来

``` swift

let disposeBag = DisposeBag()
    
let subject1 = BehaviorSubject(value: "🍎")
let subject2 = BehaviorSubject(value: "🐶")

let variable = Variable(subject1)

variable.asObservable()
    .concat()
    .subscribe { print($0) }
    .disposed(by: disposeBag)
    
subject1.onNext("🍐")
subject1.onNext("🍊")

variable.value = subject2

subject2.onNext("I would be ignored")
subject2.onNext("🐱")

subject1.onCompleted()

subject2.onNext("🐭")

output:
next(🍎)
next(🍐)
next(🍊)
next(🐱)
next(🐭)

```

#### 连接操作

##### publish

###### 把一个序列转成一个可连接的序列

**http://www.alonemonkey.com/2017/03/24/rxswift-part-three/**

##### replay

###### 把源序列转换可连接的序列，并会给新的订阅者发送之前bufferSize个的值

``` swift

let intSequence = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
        .replay(5)

print("1 订阅")

_ = intSequence
    .subscribe(onNext: { 
    print("Subscription 1:, Event: \($0)") })

delay(1) { _ = intSequence.connect() }

delay(4) {
    print("4 订阅")
    _ = intSequence
        .subscribe(onNext: { print("Subscription 2:, Event: \($0)") })
}

delay(5) {
    print("5 订阅")
    _ = intSequence
        .subscribe(onNext: { print("Subscription 3:, Event: \($0)") })
}

output:
1 订阅
Subscription 1:, Event: 0
Subscription 1:, Event: 1
4 订阅
Subscription 2:, Event: 0
Subscription 2:, Event: 1     //之前发射的值
Subscription 1:, Event: 2
Subscription 2:, Event: 2
5 订阅
Subscription 3:, Event: 0
Subscription 3:, Event: 1
Subscription 3:, Event: 2     //之前发射的值
Subscription 1:, Event: 3
Subscription 2:, Event: 3
Subscription 3:, Event: 3

```

##### multicast

###### 传入一个Subject，每当序列发射都会触发这个Subject的发射

``` swift

let subject = PublishSubject<Int>()
    
_ = subject
    .subscribe(onNext: { print("Subject: \($0)") })
    
let intSequence = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
    .multicast(subject)

_ = intSequence
    .subscribe(onNext: { print("\tSubscription 1:, Event: \($0)") })

delay(2) { _ = intSequence.connect() }

delay(4) {
    _ = intSequence
        .subscribe(onNext: { print("\tSubscription 2:, Event: \($0)") })
}

delay(6) {
    _ = intSequence
        .subscribe(onNext: { print("\tSubscription 3:, Event: \($0)") })
}

output:
Subject: 0
	Subscription 1:, Event: 0
Subject: 1
	Subscription 1:, Event: 1
	Subscription 2:, Event: 1
Subject: 2
	Subscription 1:, Event: 2
	Subscription 2:, Event: 2
Subject: 3
	Subscription 1:, Event: 3
	Subscription 2:, Event: 3
	Subscription 3:, Event: 3

```

#### 错误操作

##### catchErrorJustReturn

###### 捕获到错误的时候，返回指定的值，然后终止

``` swift

let disposeBag = DisposeBag()
    
let sequenceThatFails = PublishSubject<String>()

sequenceThatFails
    .catchErrorJustReturn("😊")
    .subscribe { print($0) }
    .disposed(by: disposeBag)

sequenceThatFails.onNext("😬")
sequenceThatFails.onNext("😨")
sequenceThatFails.onNext("😡")
sequenceThatFails.onNext("🔴")
sequenceThatFails.onError(TestError.test)

output:
next(😬)
next(😨)
next(😡)
next(🔴)
next(😊)
completed

```

##### catchError

###### 捕获一个错误值，然后切换到新的序列

``` swift

let disposeBag = DisposeBag()  

let sequenceThatFails = PublishSubject<String>()

let recoverySequence = PublishSubject<String>()

sequenceThatFails
    .catchError {
        print("Error:", $0)
        return recoverySequence
    }
    .subscribe { print($0) }
    .disposed(by: disposeBag)

sequenceThatFails.onNext("😬")
sequenceThatFails.onNext("😨")
sequenceThatFails.onNext("😡")
sequenceThatFails.onNext("🔴")
sequenceThatFails.onError(TestError.test)
recoverySequence.onNext("😊")

output:
next(😬)
next(😨)
next(😡)
next(🔴)
Error: test
next(😊)

```

##### retry

###### 捕获到错误的时候，重新订阅该序列

* retry(_:) 表示最多重试多少次。 retry(3)

``` swift

let disposeBag = DisposeBag()
    var count = 1
    
    let sequenceThatErrors = Observable<String>.create { observer in
        observer.onNext("🍎")
        observer.onNext("🍐")
        observer.onNext("🍊")
        
        if count == 1 {
            observer.onError(TestError.test)
            print("Error encountered")
            count += 1
        }
        
        observer.onNext("🐶")
        observer.onNext("🐱")
        observer.onNext("🐭")
        observer.onCompleted()
        
        return Disposables.create()
    }
    
    sequenceThatErrors
        .retry()
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)

output:
🍎
🍐
🍊
Error encountered
🍎
🍐
🍊
🐶
🐱
🐭

```

#### 调试操作

##### debug

###### 打印所有的订阅者、事件、和处理

``` swift

let disposeBag = DisposeBag()

var count = 1

let sequenceThatErrors = Observable<String>.create { observer in
    observer.onNext("🍎")
    observer.onNext("🍐")
    observer.onNext("🍊")
    
    if count < 5 {
        observer.onError(TestError.test)
        print("Error encountered")
        count += 1
    }
    
    observer.onNext("🐶")
    observer.onNext("🐱")
    observer.onNext("🐭")
    observer.onCompleted()
    
    return Disposables.create()
}

sequenceThatErrors
    .retry(3)
    .debug()
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

output:
2017-03-25 12:38:47.241: playground157.swift:42 (__lldb_expr_157) -> subscribed
2017-03-25 12:38:47.246: playground157.swift:42 (__lldb_expr_157) -> Event next(🍎)
🍎
2017-03-25 12:38:47.247: playground157.swift:42 (__lldb_expr_157) -> Event next(🍐)
🍐
2017-03-25 12:38:47.248: playground157.swift:42 (__lldb_expr_157) -> Event next(🍊)
🍊
Error encountered
2017-03-25 12:38:47.250: playground157.swift:42 (__lldb_expr_157) -> Event next(🍎)
🍎
2017-03-25 12:38:47.250: playground157.swift:42 (__lldb_expr_157) -> Event next(🍐)
🍐
2017-03-25 12:38:47.251: playground157.swift:42 (__lldb_expr_157) -> Event next(🍊)
🍊
Error encountered
2017-03-25 12:38:47.252: playground157.swift:42 (__lldb_expr_157) -> Event next(🍎)
🍎
2017-03-25 12:38:47.253: playground157.swift:42 (__lldb_expr_157) -> Event next(🍐)
🍐
2017-03-25 12:38:47.253: playground157.swift:42 (__lldb_expr_157) -> Event next(🍊)
🍊
Error encountered
2017-03-25 12:38:47.255: playground157.swift:42 (__lldb_expr_157) -> Event error(test)
Received unhandled error: /var/folders/h3/8n169g610_g69k50z7g_q4gc0000gp/T/./lldb/26516/playground157.swift:43:__lldb_expr_157 -> test
2017-03-25 12:38:47.283: playground157.swift:42 (__lldb_expr_157) -> isDisposed

```

##### RxSwift.Resources.total

###### 提供所有Rx申请资源的数量

* 在检查内存泄露的时候非常有用

``` swift

print(RxSwift.Resources.total)
    
let disposeBag = DisposeBag()

print(RxSwift.Resources.total)

let variable = Variable("🍎")

let subscription1 = variable.asObservable().subscribe(onNext: { print($0) })

print(RxSwift.Resources.total)

let subscription2 = variable.asObservable().subscribe(onNext: { print($0) })

print(RxSwift.Resources.total)

subscription1.dispose()

print(RxSwift.Resources.total)

subscription2.dispose()

print(RxSwift.Resources.total)

output:
0
2
🍎
8
🍎
10
9
8

```

## Subscribe

可以使用subscribe(onNext:) 或者 subscribe(_:)，通过前面一种方式来订阅某个事件，可以只订阅其中某个，而不用全部订阅

```` swift
someObservable.subscribe(
    onNext: { print("Element:", $0) },
    onError: { print("Error:", $0) },
    onCompleted: { print("Completed") },
    onDisposed: { print("Disposed") }
)
````

### .onNext(element)

### .onError(error)

### .onCompleted()

## Subject

1、Subject就相当于一个桥梁或者代理，它既可以作为一个observer也可以作为一个Observable
2、PublishSubject, ReplaySubject, and BehaviorSubject不会自动发送完成事件当他们被回收时

### PublishSubject

#### 只会发送给订阅者订阅之后的事件，之前发生的事件将不会发送

* 订阅者2只能收到订阅后传过来的事件
* 如果要保证所有事件都能被订阅到，可以使用Create主动创建或使用ReplaySubject
* 如果被观察者因为错误被终止，PublishSubject只会发出一个错误的通知

``` swift
let disposeBag = DisposeBag()
let subject = PublishSubject<String>()

subject.addObserver("1").disposed(by: disposeBag)
subject.onNext("🐶")
subject.onNext("🐱")

subject.addObserver("2").disposed(by: disposeBag)
subject.onNext("🅰️")
subject.onNext("🅱️")
    
output:
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
```

### ReplaySubject

#### 不管订阅者什么时候订阅的都可以把所有发生过的事件发送给订阅者

``` swift
let disposeBag = DisposeBag()
let subject = ReplaySubject<String>.createUnbounded()

subject.addObserver("1").disposed(by: disposeBag)
subject.onNext("🐶")
subject.onNext("🐱")

subject.addObserver("2").disposed(by: disposeBag)
subject.onNext("🅰️")
subject.onNext("🅱️")

output:
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 2 Event: next(🐶)
Subscription: 2 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)

// 当然你也指定重发事件的缓冲区大小，比如上面的例子如果这样创建：
let subject = ReplaySubject<String>.create(bufferSize: 1)
// 指定缓冲区大小为1，那么订阅者2就不会收到🐶了
```

### BehaviorSubject

#### 广播所有事件给订阅者，对于新的订阅者，广播最近的一个事件或者默认值

``` swift
let disposeBag = DisposeBag()
let subject = BehaviorSubject(value: "🔴")

subject.addObserver("1").disposed(by: disposeBag)
subject.onNext("🐶")
subject.onNext("🐱")

subject.addObserver("2").disposed(by: disposeBag)
subject.onNext("🅰️")
subject.onNext("🅱️")

subject.addObserver("3").disposed(by: disposeBag)
subject.onNext("🍐")
subject.onNext("🍊")

output:
Subscription: 1 Event: next(🔴)
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 2 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
Subscription: 3 Event: next(🅱️)
Subscription: 1 Event: next(🍐)
Subscription: 2 Event: next(🍐)
Subscription: 3 Event: next(🍐)
Subscription: 1 Event: next(🍊)
Subscription: 2 Event: next(🍊)
Subscription: 3 Event: next(🍊)
```

### Variable

#### Variable是BehaviorSubject的封装

* 它和BehaviorSubject不同之处在于，不能向Variable发送.Complete和.Error，它会在生命周期结束被释放的时候自动发送.Complete

``` swift
let disposeBag = DisposeBag()
let variable = Variable("🔴")

variable.asObservable().addObserver("1").disposed(by: disposeBag)
variable.value = "🐶"
variable.value = "🐱"

variable.asObservable().addObserver("2").disposed(by: disposeBag)
variable.value = "🅰️"
variable.value = "🅱️"

output:
Subscription: 1 Event: next(🔴)
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 2 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
Subscription: 1 Event: completed
Subscription: 2 Event: completed
```
