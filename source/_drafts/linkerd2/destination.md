---
title: "linkerd2 destination 学习笔记"
date: 2019-10-30T16:58:27+08:00
draft: false
author: "哗啦啦mesh团队"
authorlink: "https://github.com/ilylia"
summary: "linker2 destination 功能。"
tags: ["linkerd", "destination"]
categories: ["linkerd"]
keywords: ["service mesh","服务网格","linkerd2","linkerd"]
---


destination组件用Golang开发而成。



## 初始化

启动后会从命令行参数中获取若干配置，接着调用`k8s.InitializeAPI`初始化k8s api对象，用于后续从k8s中获取数据。然后调用`destination.NewServer`构造grpc对象用以提供grpc服务。



## 运行

主要有2个接口：`Get` `GetProfile`



### `Get`

这是个单请求流回复的接口，用于实时更新指定目标地址`fqdn`的实际`endpoint`变化。

收到请求后：

1. 解析请求参数，根据`fqdn`找到需要监控的`service`
2. 创建一个`endpointTranslator`对象并订阅到初始化时根据k8s api创建的`EndpointsWatcher`上。

这样，k8s数据有相关变化时，就会触发`endpointTranslator`的`Add` `Remove` `NoEndpoints`接口，并组织成`destination.Update`结构发送至回复stream。



### `GetProfile`

这也是个单请求流回复的接口，用于实时更新指定目标地址`fqdn`的相关`ServiceProfile`变化。

收到请求后：

1. 解析请求参数，取得`service`
2. 创建一个`profileTranslator`对象用于处理sp数据
3. 创建一个`trafficSplitAdaptor`并订阅到`TrafficSplitWatcher`，用于监控[`TrafficSplit`](https://github.com/deislabs/smi-spec/blob/master/traffic-split.md)变化
4. 创建一个`ProfileID`并订阅到`ProfileWatcher`，用于监控`ServiceProfile`变化

当以上数据有变化时，数据会先到`trafficSplitAdaptor::publish`方法进行合并，然后到`profileTranslator`中转换成grpc接口结构发送至回复stream。

注：在proxy中，正是根据`ServiceProfile`数据进行流量路由管理的。

