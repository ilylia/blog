---
title: "linkerd2 tap 学习笔记"
date: 2019-10-31T16:58:27+08:00
draft: false
author: "哗啦啦mesh团队"
authorlink: "https://github.com/ilylia"
summary: "在本文章中，能粗略了解到在 linker2 的 tap 组件中从收到连接到处理请求的流程逻辑。"
tags: ["linkerd", "tap"]
categories: ["linkerd"]
keywords: ["service mesh","服务网格","linkerd2","linkerd"]

---

tap组件用Golang开发而成，负责接受外部tap请求，并将请求转发至proxy组件，然后把拿到的数据回复回去。

<!--more-->



## 初始化

1. 从命令行参数中获取若干配置
2. `k8s.InitializeAPI`初始化k8s api对象，用于后续从k8s中获取数据
3. `tap.NewGrpcTapServer`构造`GRPCTapServer`对象
4. `tap.NewAPIServer`构造http server



## 路由

在`initRouter`中，初始化了若干路由：

- `GET /`

返回各接口map

- `GET /healthz`
- `GET /healthz/log`
- `GET /healthz/ping`

以上3个均为探活接口

- `GET /metrics`

返回prometheus格式指标数据

- `GET /openapi/v2`

返回各接口swagger

- `GET /version`

返回组件版本

- `GET /apis`
- `GET /apis/tap.linkerd.io`
- `GET /apis/tap.linkerd.io/v1alpha1`

以上返回APIGroup、APIResource相关信息

- `GET /apis/tap.linkerd.io/v1alpha1/watch/namespaces/{ns}`
- `GET /apis/tap.linkerd.io/v1alpha1/watch/namespaces/{ns}/{res-type}/{res-name}`

返回各接口map

- `POST /apis/tap.linkerd.io/v1alpha1/watch/namespaces/{ns}/tap`
- `POST /apis/tap.linkerd.io/v1alpha1/watch/namespaces/{ns}/{res-type}/{res-name}/tap`

核心功能，处理来自其它组件的tap请求



## tap请求

1. 从url解析资源对象
2. 从请求头获取user，`pkgK8s.ResourceAuthzForUser`判断k8s权限
3. `protohttp.HTTPRequestToProto`将请求body反序列化为grpc请求`public.TapByResourceRequest`
4. `protohttp.TapReqToURL`拼装tap请求url，与请求url对比验证
5. 将`http.ResponseWriter`伪装成一个grpc stream
6. 强行调用`GRPCTapServer::TapByResource`进入tap逻辑



`GRPCTapServer::TapByResource`：

1. 获取该资源类型所有实例
2. 遍历实例，找到对应的pod，排除掉未被linkerd2 mesh管理的和禁用tap的pod
3. 创建事件通道
4. 转换参数格式
5. 对上述pod，在协程中调用`GRPCTapServer::tapProxy`
6. 将事件通道中收到的事件推到grpc stream



`GRPCTapServer::tapProxy`：

1. 创建grpc client，并连接该pod中的`linkerd2_proxy`组件提供的tap的grpc服务
2. `client.Observe`开始获取tap event事件数据，并转换格式后推向事件通道


