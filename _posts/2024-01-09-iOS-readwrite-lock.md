---
layout: post
title: "iOS 读写锁"
subtitle: "GCD 实现读写锁注意的坑"
author: "DamonFish"
header-img: ""
header-bg-css: "linear-gradient(to right, #24b94a, #38ef7d);"
tags:
  - 多线程
  - 锁
  - iOS
---

## 概念

读写锁用于解决读写问题。读操作可并发重入，写操作是互斥的。
这意味着多个线程可以同时读数据，但写数据时需要获得一个独占的锁。

能同时多个线程读，
只能同时一个线程写（修改）.
不能同时一个线程写，一个线程读。

**多读单写**
读写锁常见的用法是控制线程对内存中的某种数据结构的访问.

## 实现方式

1. pthread_rwlock， 专门的读写锁
2. dispatch_barrier_async

我们只讲第二种方式，dispatch_barrier_async.

## What's Barrier

GCD的并发队列有一个Barrier Block的概念，关于Barrier Block，Apple给出的解释如下:

> Will not run until all blocks submitted earlier have completed
Blocks submitted later will not run until barrier block has completed

简单讲就是， 通过dispatch_barrier_async向队列提交的任务A，A会等在它之前提交的任务完成后执行，
A执行中， 所有在它之后提交的任务都得等A完成后执行。
dispatch_barrier_async提交的任务像一个栅栏任务， 拦住了后续提交的任务。

仔细想想， dispatch_barrier_async的特性，决定了它能实现读写锁。

在 AFN 中，header的读写锁实现。

```
// 并发队列
self.requestHeaderModificationQueue = dispatch_queue_create("requestHeaderModificationQueue", DISPATCH_QUEUE_CONCURRENT);

// 同步方法，可以并发获取字典
- (NSDictionary *)HTTPRequestHeaders {
    NSDictionary__ block *value；
    dispatch_sync(self.requestHeaderModificationQueue， ^{
    value = 【NSDictionary dictionaryWithDictionary:self.mutableHTTPRequestHeaders];
    });
    return value；
}
// 异步方法，使用dispatch_barrier_async/dispatch_barrier_sync可以保证一次只有一个线程
- (void)setValue:(NSString *)value forHTTPHeaderField:(NSString *)field
    dispatch_barrier_async(self.requestHeaderModificationQueue， ^{
    [self . mutableHTTPRequestHeaders setValue:value forKey:field];
    });
}
// 同步方法，并发获取字典里的值
- (NSString *)valueForHTTPHeaderField:(NSString *)field {
    NSString __block *value；
    dispatch_sync(self.requestHeaderModificationQueue，^{
    value = [self.mutableHTTPRequestHeaders valueForKey:field];
    });
    return value；
}

```

## What's New? 锁不住！

一般的读写锁就讲的上面了，更多细节请自行Google.
但是实际使用上有些额外的值得考虑。

如果想锁住只是标量或普通的值类型， String, Int, Double, Bool, [String], [String: Int]，没有问题。

但是很多情况，想锁住的是Array或Dictionary这样的容器就得小心了。
**锁得了容器本身的读写，无法锁住容器内元素的读写**

例如dataSource是一个Person类实例的数组
```
  queue.async { [weak self] in
      guard let self else { return }
      // read
      self.firstPerson = dataSource.first
  }
  
  // 这是很容易就操作的
  // 在其他的线程修改 DataSource数组中第一个person
  self.firstPerson.name = nil
```

如果`firstPerson.name` 在其他线程修改，  
那么在读写锁所在线程：
读firstPerson，person的内容可能已经被修改。
写firstPerson，会产生线程冲突！

除了线程冲突错误，可能还会遇到一些访问以及被释放对象的错误, 因为被其他线程释放了。

## 解决方法

将容器本身修改，以及容器内元素的修改都放到写操作里。
先实现一个读写锁控制读写的属性封装器，下面的代码是修改GRDB的实现：

`ReadWriteBox.swift`
```
import Dispatch

/// A ReadWriteBox grants multiple readers and single-writer guarantees on a
/// value. It is backed by a concurrent DispatchQueue.
@propertyWrapper
final class ReadWriteBox<T> {
    private var _wrappedValue: T
    private var queue: DispatchQueue
    
    var wrappedValue: T {
        get { read { $0 } }
        set { update { $0 = newValue } }
    }
    
    var projectedValue: MJReadWriteBox<T> { self }
    
    init(wrappedValue: T) {
        _wrappedValue = wrappedValue
        queue = DispatchQueue(label: "Secent.ReadWriteBox", attributes: [.concurrent])
    }
    
    func read<U>(_ block: (T) throws -> U) rethrows -> U {
        try queue.sync {
            try block(_wrappedValue)
        }
    }
    
    func update<U>(_ block: @escaping (inout T) -> U) {
        queue.async(flags: [.barrier]) { [weak self] in
            guard let self else { return }
            _ = block(&self._wrappedValue)
        }
    }
    
    /// Sync update with a return value, attention deadlock!
    func syncUpdate<U>(_ block: (inout T) throws -> U) rethrows -> U {
        try queue.sync(flags: [.barrier]) {
            try block(&_wrappedValue)
        }
    }
}
```

写相关的操作如下。
```
@ReadWriteBox var dataSource = [Person]()

  // 写操作
  $dataSource.update { [weak self] allPersons in
      guard let self else { return }
      let firstPerson = allPersons.first
      firtstPerson.name = nil
  }

```

### 线程死锁

其实这种死锁并不常见，还是有必要说明一下.

```
let queue = DispatchQueue(label: "Secent.ReadWriteBox", attributes: [.concurrent])

for i in 0..<100 {
    print("test begin", i)
    DispatchQueue.global().async {
        self.queue.sync {
            print("read")
        }
        
        self.queue.async(flags: .barrier) {
            print("write")
        }
    }
}
```
执行上面的代码， App不会crash，但是queue所在的线程已经死锁。

> The problem is that you're scheduling 100 tasks to be executed concurrently. This will exceed the virtual thread limit (usually 64) –
and therefore you'll have 64 tasks sitting there waiting for their read or writes to finish,
but they can't finish because there's no more threads left to do them on.
If you reduce your loop down to 64, or make your queue a serial queue in order to bottleneck the tasks, the code will once again work. 
Although, this is a pretty contrived example. In reality, you would never have so many contested read and writes happening 
concurrently (this would be an indication of a more fundamental problem in your logic) – and even if you did, your writes should most probably 
be happening asynchronously with a dispatch_barrier_async.

实际上是一个线程池耗尽问题
sync和barrier共同造成的。

loop数量为100， 线程池已满（64）。
线程池已耗尽， 然后sync提交的任务在等待 barrier提交的任务的执行完， 但是barrier提交的任务已经分配不到线程执行了，永远无法执行完。