---
title: "linkerd2 ServiceProfile 学习笔记"
date: 2019-10-30T16:58:27+08:00
draft: false
author: "哗啦啦mesh团队"
authorlink: "https://github.com/ilylia"
summary: "linkerd2 ServiceProfile 模版解析。"
tags: ["linkerd", "service profile"]
categories: ["linkerd"]
keywords: ["service mesh","服务网格","linkerd2","linkerd"]
---

在linkerd2.1中引入了service profile（简称sp）概念。通过配置sp，可以在原本服务的基础上配置linkerd的行为。目前版本的sp实现了基于路由的metrics，及对路由级别的请求的重试、超时判断。将来，像速率限制、熔断等功能也会基于sp实现。



## Template

```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  # service fqdn
  name: testSvc.testDemo.svc.cluster.local
  namespace: testDemo
spec:
  # 一个 service profile 定义若干路由.
  # Linkerd 能够根据路由聚合如请求量、延迟、成功率等指标.
  routes:
  - name: '/authors/{id}'

    # 每个路由都必须定义一个条件.  匹配该条件的请求会被认为属于该路由.
    # 如果一个请求匹配多个路由的条件，就取第1个.
    condition:
      # 最简单的条件就是请求路径的正则表达式.
      pathRegex: '/authors/\d+'

      # 请求方法.
      method: POST

      # 如果设置了条件的多个字段，那么它们必须同时匹配该请求.
      # 这等价于使用 'all' 条件:
      # all:
      # - pathRegex: '/authors/\d+'
      # - method: POST

      # 'all', 'any', 和 'not' 可以组合使用.
      # any:
      # - all:
      #   - method: POST
      #   - pathRegex: '/authors/\d+'
      # - all:
      #   - not:
      #       method: DELETE
      #   - pathRegex: /info.txt

    # 一个路由可被标记为 可重试 的.
    # 这意味着对属于该路由的请求的重试总是安全的，于是proxy将会在可能的情况下对失败的请求进行重试
    # isRetryable: true

    # 一个路由可以定义若干 response classes, 用于将回复分类.
    responseClasses:

    # 每个 response class 必须定义一个条件.
    # 所有满足该条件的回复会被分到这一类中.
    - condition:
        # 最简单的条件是状态码的范围.
        status:
          min: 500
          max: 599

        # 只指定 min 或 max 表示仅匹配该值.
        # status:
        #   min: 404 # 表示只匹配404.

        # 'all', 'any', 和 'not' 可以组合使用.
        # all:
        # - status:
        #     min: 500
        #     max: 599
        # - not:
        #     status:
        #       min: 503

      # 该分类代表成功还是失败.
      isFailure: true

    # 路由可以定义请求超时.
    # 任何匹配该路由的请求，如果超过该值，均会被取消.
    # 未指定时，默认值为10秒.
    # timeout: 250ms

  # 一个 service profile 还可以定义重试策略
  # 它可以根据原始请求量的比例来指定重试的最大次数
  # retryBudget:
  #   retryRatio 是相对原始请求量的最大重试次数比例
  #   如 0.2 表示最多增加20%的请求负载
  #   retryRatio: 0.2

  #   这是对 retryRatio 的补充，定义每秒重试次数
  #   当请求几率很低时，它执行重试
  #   minRetriesPerSecond: 10

  #   This duration indicates for how long requests should be considered for the
  #   purposes of calculating the retryRatio.  A higher value considers a larger
  #   window and therefore allows burstier retries.
  #   ttl: 10s
```



