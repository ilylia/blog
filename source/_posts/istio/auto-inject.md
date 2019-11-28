---
title: "istio sidecar 自动注入研究"
date: 2019-11-28T18:46:27+08:00
draft: false
author: "哗啦啦mesh团队"
authorlink: "https://github.com/ilylia"
summary: "本文分析了 istio 的 sidecar 自动注入的原理及相关配置。"
tags: ["istio", "sidecar", "inject", "admission"]
categories: ["istio"]
keywords: ["service mesh","服务网格","istio","sidecar"]
---

## 概述

研究 istio 在 k8s 中的自动注入的原理以及相关配置。

通常情况下，我们一般会直接在命名空间上添加标签 `istio-injection=enabled`，就启用了整个命名空间的自动注入功能。
这个功能是如何实现的？如果我不想启用整个命名空间的自动注入，只想开启个别 pod 的自动注入，需要怎么做？

下面，我们来从原理上分析一下自动注入的实现过程，了解下相关的配置。

## 流程

### kubernetes

在k8s上，判断一个webhook是否需要调用，在 `k8s.io/apiserver/pkg/admission/plugin/webhook/generic/webhook.go` 文件中 `Webhook.ShouldCallHook` 方法中：

```go
    var matches bool
    for _, r := range h.Rules {
        m := rules.Matcher{Rule: r, Attr: attr}
        if m.Matches() {
            matches = true
            break
        }
    }
    if !matches {
        return false, nil
    }

    return a.namespaceMatcher.MatchNamespaceSelector(h, attr)
```

对比 istio 默认设置的 k8s CRD `MutatingWebhookConfiguration` 配置中的相关部分：

```yml
webhooks:
- namespaceSelector:
    matchLabels:
      istio-injection: enabled
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    resources:
    - pods
    scope: '*'
```

可见，前面一段 `rules` 相关的逻辑判断，主要是过滤一些不关心的请求，此处相当于只保留 pod 的创建请求。

然后会判断命名空间的规则，此处是判断标签，是否有 key 为 `istio-injection` 且 value 为 `enabled`，只有符合该标签的命名空间中的 pod 的创建，才会调用 `istio-sidecar-injector` 这个 webhook。

### istio-sidecar-injector

webhook 请求过来后，执行到 `pkg/kube/inject/inject.go` 文件的 `injectRequired` 函数中：

```go
    if podSpec.HostNetwork {
        return false
    }

    // skip special kubernetes system namespaces
    for _, namespace := range ignored {
        if metadata.Namespace == namespace {
            return false
        }
    }

...

    var useDefault bool
    var inject bool
    // sidecar.istio.io/inject
    switch strings.ToLower(annos[annotation.SidecarInject.Name]) {
    case "y", "yes", "true", "on":
        inject = true
    case "":
        useDefault = true
    }

    if useDefault {
        // NeverInject check
        if NeverInject {
            inject = false
            useDefault = false
        }
    }
    if useDefault {
        // AlwaysInject check
        if AlwaysInject {
            inject = true
            useDefault = false
        }
    }

    var required bool
    switch config.Policy {
    default: // InjectionPolicyOff
        // invalid value
        required = false
    case InjectionPolicyDisabled:
        if useDefault {
            required = false
        } else {
            required = inject
        }
    case InjectionPolicyEnabled:
        if useDefault {
            required = true
        } else {
            required = inject
        }
    }

...
```

进入逻辑，首先判断是否使用 `HostNetwork`，然后看命名空间是否在忽略的列表（即 `kube-system` 和 `kube-public`，k8s 系统所在）中，通过这两个判断，筛去一些不适合注入的 pod。

然后检查 pod 是否配置了 `sidecar.istio.io/inject` 注解为 `true`（或 `yes` 等其它yml认为是`真`的值）：

- 为 `true`：注入
- 未设置：按默认继续判断

  - 判断 `NeverInject`：如果命中就不注入
  - 判断 `AlwaysInject`：如果命中就注入
  - 判断 `policy` 配置

    - `enabled`：注入
    - `disabled`：不注入
    - 其它值：不注入

- 其它：认为为 `false`，不注入

## 结论

由此，可得出结论：

1. 命名空间的标签控制是否调用 webhook
2. pod的注解控制是否注入
3. pod未设置时，NeverInject、AlwaysInject、policy 配合决定是否注入

对于需要开启命名空间的自动注入，然后特殊pod不自动时，可以直接走默认的设置，然后给命名空间加标签来实现（官方示例的做法）。

对于不开启命名空间自动注入，只针对特殊pod自动注入时，就需要如下调整：

1. 修改 `MutatingWebhookConfiguration` 配置：

    ```bash
    $ kubectl edit mutatingwebhookconfiguration istio-sidecar-injector
    ```

    并编辑内容：

    ```yml
    namespaceSelector:
      matchExpressions:
      - key: istio-injection
        operator: NotIn
        values:
        - disabled
    ```

    意为对标签值不为 `disabled` 的命名空间都调用 webhook。

1. 命名空间不加 `istio-injection` 标签
1. 给 pod 的 deplotment 添加注解：

    ```yml
    spec:
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "true"
    ```

    意为针对该 pod，需要自动注入。
