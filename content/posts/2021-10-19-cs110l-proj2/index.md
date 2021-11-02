---
title: "CS110L Proj-2: BalanceBeam 反向代理"
date: 2021-10-19T12:42:33+08:00
description: 非常非常迷你的 Nginx 精简版
tags:
- CS110L
- Rust
toc: true
---

实现一个简易的负载均衡代理，主要是熟悉 Rust 的异步 IO 用法

> 我使用的 rust 是 1.54 版本，在 2020 版的代码中，使用的是 clap 的  3.0.0-beta.1 版本，这个版本在 1.54 下编译不通过。在更换 clap 版本为 3.0.0-beta.5 并按照库文档修改之后，编译通过。
>
> 并且 2020 版的初始代码的第二个测试无法通过测试，不知道是否是 clap 版本变化的原因，要做对应修改。
>
> 上述两处变化对应的代码修改可以在[这个提交](https://github.com/zianglei/cs110l-spr-2020-starter-code/commit/bc4fbae4ba385ebbdc6043517da2f778e31cb244)中查看

## Milestone 1

第一个变动是使用线程来处理各个连接。在 handle_connection 处使用 thread::spawn 创建线程，每个线程调用 handle_connection 处理连接。由于各个线程需要共享 ProxyState，因此需要使用 Arc 创建多线程恭喜的不可变引用。

```rust
    for stream in listener.incoming() {
        if let Ok(stream) = stream {
            // Handle the connection!
            let state_ref = state.clone();
            thread::spawn(move || {
                handle_connection(stream, &*state_ref);
            });
        }
    }
```

如果使用 [ThreadPool](https://docs.rs/threadpool/1.8.1/threadpool/)，就按照其中的文档创建 pool 并执行即可

## Milestone 2

Milestone 2 主要是用 tokio 异步框架代替传统的阻塞 IO。需要做的只是将 TcpStream、TcpListener 等替换成 tokio 框架提供的即可，然后将处理函数变成异步函数。

整体不需要做很多改动，可以看出 tokio 框架提供了很好的与原生类似的接口，方便切换。

## Milestone 3

Milestone 3 需要在检测到当前 upstream server 请求失败的时候，将该 server 标记为失效，重新向其他 server 请求。如果所有 upstream server 都失效，则返回错误。

Handout 提供了三种实现方式，我选择了使用 tokio 读写锁的方式实现，并在 ProxyState 中添加了一个向量存储server 是否有效的标志位，又加上一个计数成员统计当前有效的 server 数量。每一个 server 失败，都会将计数减一，直到减为 0，返回错误。

注意在使用读写锁的时候，需要将读锁先使用 drop 释放掉之后再执行写锁

## Milestone 4

Milestone 4 引入了主动检测机制，背景是有很多服务器并不是自身挂了，它们依然能够与客户端建立连接，但是有可能它们背后的其他服务器（例如数据库服务器）挂了，导致这些服务器不能拿到客户端请求的数据，从而返回错误。

通常这些服务器会拥有“检测路径”，当有请求访问这些路径的时候，服务器就会做一个快速的测试，比如快速访问数据库服务器，检测服务器是否失效，如果正常运行，返回给客户端 HTTP 200，失效则返回错误。

这种主动检测的好处在于当服务器由于负载过大或者重启等原因暂时失效，过一段时间又正常工作时，load balancer 能够检测到这种情况，从而将标记为失效的服务器重新恢复。

在实现的时候可以使用 tokio::timer 提供的定时机制，起一个异步函数，间隔 active_health_check_interval 与所有 upstream server 都建立一次连接，发起一次请求。

发起请求可以使用提供的 request 接口

```rust
let request = http::Request::builder()
    .method(http::Method::GET)
    .uri(active_check_health_path)
    .header("Host", upstream)
    .body(Vec::new())
    .unwrap();
```

注意使用 tokio::time::interval 的时候，interval.tick 第一次调用会立即触发。由于在之前的测试中有请求数量限制的要求，如果我们加上 active health check，那么判断 upstream server 是否活跃的请求也会被计算在内，导致前面检测不通过。

由于前面的检测都会将活跃检测间隔设置为默认值 10s， 因此我们可以先等待 10s 之后再进行检测，此时正常的测试流程早已经走完，因此可以正常通过之前的测试，就是要注意 interval.tick 第一次调用会立即触发超时，因此要首先调用一次。

```rust
async fn active_health_check() {
  let mut interval = tokio::time::interval(...);
  interval.tick().await;
	loop {
    interval.tick().await;
    do_active_health_check();
  }
}
```

## Milestone 5

Milestone 5 需要做 Rate Limit，即速率限制，创建的速率限制算法可以参考[这篇文章](https://konghq.com/blog/how-to-design-a-scalable-rate-limiting-algorithm/)，此处只要求实现固定时间窗口的速率限制。

基本做法是在 ProxyState 中记录当前时间，然后每次调用 rate limiting 函数的时候就检查有没有超过固定时间窗口（此处是一分钟），超过之后就将所有计数器清空，否则将对应 client_ip 的计数值 +1，如果超过限制数量，就返回错误。

然后在 handle_connection 函数中调用该函数，当该函数返回错误时，生成 402 错误响应。
