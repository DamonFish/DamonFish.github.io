---
layout: post
title: "weak-strong dance需注意的点"
subtitle: ""
author: "DamonFish"
# header-img: "img/post-bg-infinity.jpg"
# header-mask: 0.3
header-style: text
mathjax: true
tags:
  - iOS开发
---

## 概念

weak-strong dance用于处理闭包导致的循环引用。
比如self持有了block，block同时持有了self，相互引用，无法释放造成内存泄漏。

关于循环引用,这里主要讲在实际使用中需要注意的几点。

## 不是所有block里的self都需要weak

一般只有产生循环引用的场景，需要weak-strong dance。
下面的三种场景是不会产生循环引用的.

1. 如果block是非逃逸的，block不会持有self， 比如map，filter，reduce等block操作。这种block是立即执行的代码块，无需强持有self等到“后续”使用；

2. block是逃逸的，但是self不持有block，不需要。 例如系统相关方法的block， GCD的block或者UIView的动画block；

3. 单例持有block，block持有self， 这种情况后续单独说；

## 单例和 weak-strong dance

self事实上并不真的持有单例， self并不对单例负责。
内存管理基本原则： 谁开辟，谁释放。 谁开辟，谁管理。
block是单例开辟的，应该由单例管理并释放，与self无关。

下面的例子即使self不持有block，但仍需要对self 进行weak-strong dance，不然会内存泄漏，
     代码来自比较老的版本的极验登录。
self持有单例（隐式）， 单例持有block， block持有self， 形成了循环引用。

``` swift
// 一键登录进入授权页面
OneLoginManager.shared.requestToken(with: self, viewModel: viewModel) { [weak self] result in
}
```

同样是单例，SDWebImageManager 就不需要 weak-strong dance

``` c
[SDWebImageManager sharedManager loadImageWithURL:url
                options:SDWebImageRetryFailed | SDWebImageAvoidDecodeImage
               progress:nil
              completed:^(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, SDImageCacheType cacheType, BOOL finished, NSURL * _Nullable imageURL) {
                  [self doSomething];
         }];
```

AFNetwoking中的我们一般封装一个请求的单例继承自AFHTTPSessionManager，请求的回调block中也不需要 weak-strong dance。

所以一键登录例子中的Manager应该管理block的释放，不应需要 weak-strong dance的，其写法有缺陷。

如果你去查看其他无需 weak-strong dance的源码，你会发现清除block的代码，类似：

``` c
self.completionBlock = nil;


if (self.completionBlock) {
 LOTAnimationCompletionBlock completion = self.completionBlock;
 self.completionBlock = nil;
 completion(complete);
}
```

自己开辟的block自己释放。

## weak-strong dance 来加快self释放

前面讲到了， 非self持有的block， 没有循环引用，一般无需对self进行weak-strong dance。
但是下面一个场景：
进入到某viewController后，请求了一个接口。然后迅速退出了该viewController。

``` c
override func viewDidLoad() {
  super.viewDidLoad()

  // 没有使用weak-strong dance，退出viewController后， self会等到请求回调block执行后才会释放。
  ResourceManager.shared().fetch() { (result, downloadError) in
  // 做一些繁重的任务
   self.doSomeCompletethings()
 }
}
```

退出viewController后，因为self被block持有了， 请求和回调中繁重的任务还是继续执行， 浪费资源。
这里虽然没有循环引用，但self的释放被延后了。可借助weak-strong dance 让self尽快释放。

解决方法：
不强制持有self， 从而取消请求回调的执行。

   网络请求发送后，实际上是无法真正cancel的，我们能cancel掉回调，cancel掉这个请求的task。
   cancel方法应在dealloc中调用， 但是这里block持有self，dealloc执行时请求回调已结束。

   如果是这种情况， block不应强持有self。 请求回调执行时，self已经释放，会取消请求回调的后续执行。

``` swift
// 没有使用weak-strong dance，退出viewController后， self会等到请求回调block执行后才会释放。
ResourceManager.shared().fetch() { [weak self] (result, downloadError) in
   guard let self = self else { return }
   // 做一些繁重的任务
   self.doSomeComplicated()
}
```

添加weak-strong dance后， 会在判断self是否存在时候返回，不会执行doSomeComplicated

当然，这并不完美。
比较好的方式是在合适的地方，由ResourceManager自己取消整个任务。

```swift
// 在退出时或者其他合适的地方调用
ResourceManager.shared().cancelFetch()
ResourceManager.shared().fetchCompletion = nil
```

显式的取消是比较麻烦的。
如果你有兴趣了解自动取消，比如取消请求，取消观察之类的。
请参阅KVOController， 或者RXSwift（Moya）disposeBag的相关实现方式。

## 多层嵌套的 weak-strong dance （补充）

多层block嵌套的时候注意 weak-strong dance不要漏掉了

``` swift
self.taskOne { [weak self] in
   guard let self = self else { return }
   
   // 下面写了N行
   //
   //
   //
   
   // taskTwo的weak-strong dance 不小心漏掉了
   self.taskTwo {
      self.doSomething()
   }
}
```

你可能觉得这不太可能， 新版本的Swift 5.8的写法， 是能省略掉self的。
看下面的写法， 漏掉weak-strong dance可能就看上去就更自然。

``` swift
self.taskOne { [weak self] in
   guard let self else { return }
   
   // 下面写了N行
   //
   //
   //
   taskTwo {
       
   }
}
```
