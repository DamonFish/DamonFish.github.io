---
layout: post
title: "iOS网络请求IP直连的SNI问题"
subtitle: ""
author: "DamonFish"
header-style: text
tags:
  - 网络
  - iOS
---

## 问题说明

在iOS开发中，当我们请求server的API时，假定server地址是baidu1.com.
Https的连接过程中， server会返回baidu1.com的公钥证书，我们可以通过公钥证书验证server是否是我们要访问的baidu1.com。

但是，如果server支持SNI， 那么server可以对应多个域名，比如baidu1，baidu2, baidu3...

正常情况下， 我们访问baidu1.com， server知道我们要访问的是baidu1.com，会返回baidu1.com的公钥证书。
但是如果是IP直连，server无法知道我们访问它对应的哪一个域名， 可能返回返回baidu2.com，baidu3.com...的证书.

那么，IP直连时，我们可以指定域名让server知道么？

**不太行**. iOS上层网络库NSURLConnection/NSURLSession没有提供接口进行SNI字段的配置，因此需要Socket层级的底层网络库例如CFNetwork，来实现IP直连网络请求适配方案。
吐槽一句， Android的OKHttp很方便支持......

## 常见解决方案

* CFNetwork
* curl

CFNetwork是一个iOS的底层库，这里有一个kCFStreamPropertySSLSettings进行设置握手的host，这样在握手的时候就有host了

``` c
// 建立inputstream，并注入ssl相关信息
CFReadStreamRef readStream = CFReadStreamCreateForHTTPRequest(kCFAllocatorDefault, cfrequest);
inputStream = (__bridge_transfer NSInputStream *)readStream;

// 配置SNI字段 stream.property[kCFStreamPropertySSLSettings][kCFStreamSSLPeerName] = originalHost
NSString *host = [curRequest.allHTTPHeaderFields objectForKey:@"Host"];

// 可以选择使用SSL或者TLS1.2，目前CFNetwork不支持HTTP2.0.
[inputStream setProperty:(__bridge id)CFSTR("kCFStreamSocketSecurityLevelTLSv1_2") forKey:(__bridge id)kCFStreamPropertySocketSecurityLevel];

NSDictionary *sslProperties = [[NSDictionary alloc] initWithObjectsAndKeys:host, (__bridge id) kCFStreamSSLPeerName, nil];
[inputStream setProperty:sslProperties forKey:(__bridge_transfer NSString *) kCFStreamPropertySSLSettings];
[inputStream setDelegate:self];
```

curl这个大家比较熟悉了， c级别的网络底层库，mac和linux系统都自带这个库； iOS有比较旧的第三方库，不推荐使用。

阿里和字节都提供HttpDNS服务， 他们也都提到了这两个方案。

阿里给出的方案说明仅有相关文章， 文章的主要内容来自<https://github.com/ChenYilong/iOSBlog/issues/13，> 该文章比较旧了，这位有名的业界大佬也早已移民。
字节给出了CFNetwork实现的代码，[地址](https://www.volcengine.com/docs/6758/174385), 但是建议商用上架之前还是好好掂量一番。

## 其他解决方案(Workaround)

根据问题的原因说明，我们可以想到两个直接方案：

1. 不验证server返回的公钥证书
2. 验证server可能返回的所有公钥证书

先别笑，这两个方案看似无脑粗暴，理论上讲却釜底抽薪， 是可行的。

### 不验证server返回的公钥证书

方案1验证不了server的身份， 可能会有中间人攻击。不过，安全通常是取舍。
如果接口对安全要求没那么高， IP直连就是为了解决DNS问题，是完全可以考虑的。
很多小公司本身在网络实现上就没有实现公钥验证，没有实现AFNetworking或Alamofire的 SSL pinning功能。

直接在didReceive challenge: URLAuthenticationChallenge收到证书挑战的方法中返回信任证书

``` c
- (void)URLSession:(NSURLSession *)session task:(nonnull NSURLSessionTask *)task didReceiveChallenge:(nonnull NSURLAuthenticationChallenge *)challenge completionHandler:(nonnull void (^)(NSURLSessionAuthChallengeDisposition, NSURLCredential * _Nullable))completionHandler {
    NSParameterAssert(task == self.task);
    NSParameterAssert(challenge != nil);
    NSParameterAssert(completionHandler != nil);
    NSParameterAssert([NSThread currentThread] == self.clientThread);
    
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    NSURLCredential *credential = nil;
    
    // 收到证书挑战，先判断需要认证的方法是不是需要认证服务器证书
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        // 返回该证书，信任这个证书
        disposition = NSURLSessionAuthChallengeUseCredential;
        credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
    } else {
        disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    }
    
    if (completionHandler) {
        completionHandler(disposition, credential);
    }
}
```

> **最新补充** 注意! iOS17更新：iOS17加强了安全，使用上面的信任之后，依然报ssl错误，错误码为-1200； 需要在info.plist中添加Allow Arbitrary Loads并置为YES，表示支持任意请求。

### 验证server可能返回的所有公钥证书

这个是我们曾经用过的方案。

先回顾一下历史， 国内最早提供HttpDNS服务的是阿里还有鹅厂， 当时不是SDK，只是提供一个url， 发送请求返回host对应的IP， 然后用IP直连请求。
IP的缓存需要自己处理，当时用的是HappyDNS，七牛提供的一个第三方DNS相关的库， 最近看了下， HappyDNS已经超过10年了，还在更新！

我们通过NSURLProtocl实现IP直连的时候比较早，开始是NSURLConnection，后面才更新到NSURLSession，当时资料很少， 阿里没有提供SDK，所以[ChenYilong](https://github.com/ChenYilong)还没有写那篇文章。

这里在回顾下SNI的概念：
```
SNI（Server Name Indication）是为了解决一个服务器使用多个域名和证书的SSL/TLS扩展。它的工作原理如下：
在连接到服务器建立SSL链接之前先发送要访问站点的域名（Hostname）。
服务器根据这个域名返回一个合适的证书。
目前，大多数操作系统和浏览器都已经很好地支持SNI扩展。
上述过程中，当客户端使用“IP直连”时，请求URL中的host会被替换成解析出来的IP，导致服务器获取到的域名为解析后的IP，无法找到匹配的证书，只能返回默认的证书或者不返回，所以会出现SSL/TLS握手不成功的错误。
```

当时不能无脑信任证书，我们的server虽然支持SNI，但是对应的域名只有两个。 我们理所当然的想到， 将sever返回的公钥证书和这两个同时匹配，有一个通过，就可以认为server身份验证通过。

**要求**： server默认返回的是预期域名中的一个。
这一点可能需要和server的童鞋沟通一下。 类似图片地址之类的，使用了CDN服务的请求可能不能这样处理。

对应App中一般的API请求，这个方案当时是可行的，是可以正常运作的； 也许有我不知道的SNI相关的盲区，知道的童鞋麻烦提点一下，谢谢。
从结果上看，这个方案是运行良好的。 最近拿现在公司的server试了下，也是OK的。

不过在实施这个方案前，最后和server认真的对一下，最好有容错机制。 毕竟过去太久，这个方案的很多细节已不记得了。

## 最后

iOS14以后，可以考虑加密DNS， 而不使用HttpDNS做IP直连的方案。
