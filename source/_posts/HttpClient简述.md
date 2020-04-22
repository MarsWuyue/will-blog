---
title: HttpClient介绍和最佳实践
tag:
  - HttpClient
categories: HttpClient
---

## HttpClient简介

### 来源

- 在.NET 4.5中，可以直接通过NUGET安装
- System.Net.Http: 提供了HttpClient的基类和相关的类
- System.Net.Http.Formatting：基于System.Net.Http，添加了序列化、反序列化以及一些附加功能的支持

### 与WebRequest的区别和优势

- HttpClient的默认handler（HttpClientHandler and WebRequestHandler）都是基于HttpWebRequest的，但是HttpClient没有暴露出那么多的配置项，但是已经cover了基本的配置，如果你需要更多的配置控制权，那么需要使用HttpWebRequest
- 一个HttpClient实例可以用来发送多个请求
- HttpClient并不跟server或者host绑定，因此可以用一个HttpClient实例发送各种请求
- 我们可以将HttpClient封装成各种自定义的Client，用于访问不同的系统
- HttpClient是基于Task的，可以跟async/await进行很好的融合，实现异步request请求，代码更加简洁

## 最佳实践

### 如果你为每个请求都new一个HttpClient实例，那么在请求结束后，一定要Dispose

Dispose方法的作用是清理HttpClient的handlers，如果每次都new新的httpclient，又不对之前的实例进行清理，那么会造成资源过度占用。如果不调用Dispose，那么建议使用using

```c#
using (var httpClient = new HttpClient())
```

### 避免为每个请求新建实例

每次新建一个实例，会造成很多资源开销，比如在新建一个连接的时候，就需要三次握手，如果我们再去销毁实例，那么消耗的资源就更多了，这种资源消耗在测试环境下不容易发现，据官网统计，一个线上系统如果为每个请求都新建实例，销毁实例的操作大概要消耗5%-7%的CPU。

因此，建议只使用一个HttpClient实例，使他跟应用程序具有相同的生命周期，实例中的方法是线程安全的，可以放心使用。

### 合理设置Timeout

为了避免有些资源连接或者执行过慢导致请求延迟，建议合理设置timeout的值

### 尽可能的重用连接

为了减少新建连接的资源浪费，建议重用连接。
可以通过设置```httpClient.DefaultRequestHeaders.ConnectionClose = false```或者```HttpWebRequest.KeepAlive = true```的方式保证长连接。

***Note：如果请求是使用负载均衡的形式获得每次请求的url，那么不建议使用keep alive的形式，可以看下一条最佳实践***

### 不要永远hold住这些连接

对于上面说的通过负载均衡获取每次请求的url的情景，如果设置了keepAlive=true，那么如果这个客户端down掉，我们就很长一段时间都没办法知道，同时会导致我们的服务在这段时间内不可用。
那么，可以通过下面的配置，使连接每隔一段时间就失效重连，这样既解决了每个请求都需要重新频繁建立连接的问题，同时也可以避免keepAlive带来的服务长时间不可用的问题。
```c#
ServicePointManager.FindServicePoint(new Uri(url)).ConnectionLeaseTimeout = ConnectionLeaseTimeoutMs;
```

### 处理错误

HttpCLient在尽量不会通过抛出异常，因此我们可以通过Rsponse Status Code来判断下一步的操作，处理结果、重试或补偿等。
但是如果请求超时，HttpClient会抛出一个TaskCanceledException的异常，我们需要对这个异常进行处理