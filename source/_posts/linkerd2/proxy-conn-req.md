---
title: "linkerd2 proxy 连接请求处理 学习笔记"
date: 2019-10-25T16:58:27+08:00
draft: false
author: "哗啦啦mesh团队"
authorlink: "https://github.com/ilylia"
summary: "在本文章中，能粗略了解到在 linker2 的代理服务 proxy 组件中从收到连接到处理请求的流程逻辑。"
tags: ["linkerd", "proxy", "tower", "toiko"]
categories: ["linkerd"]
keywords: ["service mesh","服务网格","linkerd2","linkerd"]
---


proxy中用到了tower服务框架和toiko异步框架来实现，本文简单分析下proxy监听的端口从收到连接到处理请求的流程逻辑。

<!--more-->

## 入口

在inbound或outbound的spawn中：

```rust
    let proxy = Server::new(
        TransportLabels,
        transport_metrics,
        connect,
        server_stack,
        config.h2_settings,
        drain.clone(),
    );
```

定义了一个proxy，并用于后续接收连接的accept中处理连接对象

proxy为`linkerd2_app_core::proxy::server`对象，其`call`为（省略无关代码）：

```rust
    fn call(&mut self, connection: Connection) -> Self::Future {
        // accept_fut为对连接协议的解析任务
        let serve_fut = accept_fut.and_then(move |(proto, io)| match proto {
            Some(proto) => Either::B({
                // make_http即为proxy创建时的server_stack
                make_http
                    // 此处内部会调用server_stack的call接口，生成实际处理请求的Service
                    .make_service(source)
                    .and_then(move |http_svc| match proto {
                        Protocol::Http1 => Either::A({
                            // 与下类似，略
                        }),
                        Protocol::Http2 => Either::B({
                            let conn = http
                                ...
                                // 生成hyper::server::conn::Connection对象
                                .serve_connection(io, HyperServerSvc::new(http_svc));
                            drain
                                // 不停调用conn.poll()，即获取新的请求，并处理
                                .watch(conn, |conn| conn.graceful_shutdown())
                                .map(|_| ())
                                .map_err(Into::into)
                        }),
                    })
            }),
        });
    }
```

在`http.serve_connection`中：

```rust
        let either = match self.mode {
            ConnectionMode::H1Only | ConnectionMode::Fallback => {
                let mut conn = proto::Conn::new(io);
                // 请求分发
                let sd = proto::h1::dispatch::Server::new(service);
                Either::A(proto::h1::Dispatcher::new(sd, conn))
            }
            ConnectionMode::H2Only => {
                let rewind_io = Rewind::new(io);
                let h2 = proto::h2::Server::new(rewind_io, service, &self.h2_builder, self.exec.clone());
                Either::B(h2)
            }

        Connection {
            conn: Some(either),
            fallback: if self.mode == ConnectionMode::Fallback {
                Fallback::ToHttp2(self.h2_builder.clone(), self.exec.clone())
            } else {
                Fallback::Http1Only
            },
        }
```

而对`conn.poll`的调用，实际上也就是对`conn.conn.poll`的不停调用。`conn.conn`为`Either`，其`poll`就是其实际值的`poll`，也就是针对http1的`proto::h1::Dispatcher::poll`或针对http2的`proto::h2::Server::poll`。



### http1

调用堆栈：`poll` `->` `poll_catch` `->` `poll_inner` `->` `poll_loop`

`poll_loop`（省略无关）：

```rust
        // 防止这个任务一直占着cpu
        for _ in 0..16 {
            self.poll_read()?;
            self.poll_write()?;
            self.poll_flush()?;
        }
```

在`poll_read`中会先尝试读取请求头。在`poll_read_head`中：

```rust
        // 其内部会调用service.poll_ready()，判断服务栈是否准备好
        match self.dispatch.poll_ready() {
            Ok(Async::Ready(())) => (),
        }
        // 从连接中读到数据头
        match self.conn.read_head() {
            Ok(Async::Ready(Some((head, body_len, wants_upgrade)))) => {
                let mut body = match body_len {
                    DecodedLength::ZERO => Body::empty(),
                    other => {
                        let (tx, rx) = Body::new_channel(other.into_opt());
                        self.body_tx = Some(tx);
                        rx
                    },
                };
                // 将请求数据丢给dispatch
                // 其中body是一个通道接收端，在poll_read中读到请求体后会将请求体塞到通道发送端
                self.dispatch.recv_msg(Ok((head, body)))?;
                Ok(Async::Ready(()))
            }
```

`dispatch.recv_msg`：

```rust
    fn recv_msg(&mut self, msg: ::Result<(Self::RecvItem, Body)>) -> ::Result<()> {
        let (msg, body) = msg?;
        // 组装请求
        let mut req = Request::new(body);
        *req.method_mut() = msg.subject.0;
        *req.uri_mut() = msg.subject.1;
        *req.headers_mut() = msg.headers;
        *req.version_mut() = msg.version;
        // 调用服务栈处理请求，并将future任务存于in_flight，保证同时只处理一个请求
        self.in_flight = Some(self.service.call(req));
        Ok(())
    }
```

在`poll_write`中，会调用`dispatch.poll_msg`来获取请求回复，如果顺利拿到，则将回复数据通过conn送走，后续再通过`poll_flush`刷新io。

在`dispatch.poll_msg`中，从`in_flight`拿到任务future，调用其`poll`，拿到真实回复数据。

### http2

对于http2协议，需要先握手，成功后才会开始具体请求。在`poll`中，握手成功后，会创建`Serving`对象，然后调用其`poll_server`。

`poll_server`是一个无限循环，每个循环中，会先调用`service.poll_ready()`判断服务栈是否可用，就绪后

`poll_server`：

```rust
            loop {
                // 判断服务栈是否准备好
                match service.poll_ready() {
                    Ok(Async::Ready(())) => (),
                }
                // 从连接中读取请求数据
                if let Some((req, respond)) = self.conn.poll().map_err(::Error::new_h2) {
                    // 调用服务栈，并用返回的future任务创建H2Stream对象
                    let fut = H2Stream::new(service.call(req), respond);
                    // 丢到tokio框架去异步处理
                    exec.execute_h2stream(fut)?;
                }
            }
```

`H2Stream::poll`调用了`H2Stream::poll2`，在`poll2`中（部分省略）：

```rust
        loop {
            let next = match self.state {
                H2StreamState::Service(ref mut h) => {
                    // 从服务栈获取resp
                    let res = match h.poll() {
                        Ok(Async::Ready(r)) => r,
                    };

                    // 对resp head和body进行处理
                    let (head, mut body) = res.into_parts();
                    let mut res = ::http::Response::from_parts(head, ());
                    super::strip_connection_headers(res.headers_mut(), false);

                    // 定义宏
                    macro_rules! reply {
                        ($eos:expr) => ({
                            match self.reply.send_response(res, $eos) {
                                Ok(tx) => tx,
                            }
                        })
                    }

                    if let Some(full) = body.__hyper_full_data(FullDataArg(())).0 {
                        let mut body_tx = reply!(false);
                        let buf = SendBuf(Some(full));
                        body_tx
                            .send_data(buf, true)
                            .map_err(::Error::new_body_write)?;
                        return Ok(Async::Ready(()));
                    }

                    if !body.is_end_stream() {
                        let body_tx = reply!(false);
                        H2StreamState::Body(PipeToSendStream::new(body, body_tx))
                    } else {
                        reply!(true);
                        return Ok(Async::Ready(()));
                    }
                },
                H2StreamState::Body(ref mut pipe) => {
                    return pipe.poll();
                }
            };
            self.state = next;
        }
```



## 总结

以上相当于是框架的处理连接的逻辑，真正的业务（此处是指对请求数据的转发、路由、重试等等）逻辑处理都在服务栈中，也就是创建proxy对象的参数`server_stack`中。本文主要分析了各个业务服务service的3个相关接口是如何被调用的：

1. 由layer生成的stack的`call`，生成的future的结果就是其对应service
2. 各service的`poll_ready`，准备相关的数据，检查是否准备完毕
3. 各service的`call`，生成对应的任务future
4. 各任务future的`poll`，获取最终结果

通过本文，可比较清晰的了解别的分析笔记中各逻辑阶段的执行时机。

