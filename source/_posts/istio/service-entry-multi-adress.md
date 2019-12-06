---
title: "istio ServiceEntry 多 Adresses 时 Sidecar 拿到的路由不全"
date: 2019-12-05T16:46:27+08:00
draft: false
author: "哗啦啦mesh团队"
authorlink: "https://github.com/ilylia"
summary: "本文分析了 istio ServiceEntry 多 Adresses 时 Sidecar 拿到的路由不全的原因。"
tags: ["istio", "sidecar", "service entry"]
categories: ["istio"]
keywords: ["service mesh","服务网格","istio","sidecar"]
---

## 概述

从网格中访问外部服务时，为了管理流量，需要为外部服务配置 `ServiceEntry`，如：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc
spec:
  hosts:
  - svc1.domain.com
  - svc2.domain.com
  ports:
  - number: 80
    name: http
    protocol: HTTP
  location: MESH_EXTERNAL
```

但是有些服务并没有配置域名，不怕，有 `IP` 也行：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-ip
spec:
  hosts:
  - svc.ip.cluster.local #not used
  addresses:
  - "172.16.0.1/32"
  - "172.16.0.2/32"
  ports:
  - number: 80
    name: http
    protocol: HTTP
  location: MESH_EXTERNAL
```

但是，上面的配置有个坑：

```bash
$ istioctl pc r sleep-755578fcb5-f5nqp.bar --name 80 -o json
[
    {
        "name": "8098",
        "virtualHosts": [
            {
                "name": "pre.external.cc.hualala.com:8098",
                "domains": [
                    "svc.ip.cluster.local",
                    "172.16.0.2"
                ],
                "routes": ...
            },
            {
                "name": "allow_any",
                "domains": [
                    "*"
                ],
                "routes": ...
            }
        ]
    }
]
```

可以看到，在 sidecar 里获取到的 route 中，只有 1 个 ip 了！这样一来，来自另外一个 ip 的流量还是被归到 `PassthroughCluster` 中，并且无法管理。

下面简单分析下原因。

<!--more-->

## 原因

sidecar 的 route 直接来源是 pilot 组件。

首先，注意这个结构体：

```go
// pilot/pkg/model/service.go:54
type Service struct {
    ...

    Hostname host.Name `json:"hostname"`
    Address string `json:"address,omitempty"`

    ...
}
```

所有的服务对象，如 k8s 的 Service，istio 的 ServiceEntry，都会转化为这个结构。

因此，上例中的 ServiceEntry 会变成 2 个 Service 对象：

- `Hostname` 为 `svc.ip.cluster.local`，`Address` 为 `172.16.0.1`
- `Hostname` 为 `svc.ip.cluster.local`，`Address` 为 `172.16.0.2`

继续翻翻代码：

```go
// pilot/pkg/model/sidecar.go:219, ConvertToSidecarScope
    // Assign namespace dependencies
    out.namespaceDependencies = make(map[string]struct{})
    for _, listener := range out.EgressListeners {
        for _, s := range listener.services {
            // TODO: port merging when each listener generates a partial service
            if _, found := servicesAdded[string(s.Hostname)]; !found {
                servicesAdded[string(s.Hostname)] = struct{}{}
                out.services = append(out.services, s)
                out.namespaceDependencies[s.Attributes.Namespace] = struct{}{}
            }
        }
    }
```

```go
// pilot/pkg/networking/core/v1alpha3/httproute.go:329, ConfigGeneratorImpl.buildSidecarOutboundVirtualHosts
    nameToServiceMap := make(map[host.Name]*model.Service)
    for _, svc := range services {
        if listenerPort == 0 {
            // Take all ports when listen port is 0 (http_proxy or uds)
            // Expect virtualServices to resolve to right port
            nameToServiceMap[svc.Hostname] = svc
        } else if svcPort, exists := svc.Ports.GetByPort(listenerPort); exists {
            nameToServiceMap[svc.Hostname] = &model.Service{
                Hostname:     svc.Hostname,
                Address:      svc.Address,
                MeshExternal: svc.MeshExternal,
                Resolution:   svc.Resolution,
                Ports:        []*model.Port{svcPort},
                Attributes: model.ServiceAttributes{
                    ServiceRegistry: svc.Attributes.ServiceRegistry,
                },
            }
        }
    }

// pilot/pkg/networking/core/v1alpha3/httproute.go:329, ConfigGeneratorImpl.buildSidecarOutboundVirtualHosts
        for _, svc := range virtualHostWrapper.Services {
            name := domainName(string(svc.Hostname), virtualHostWrapper.Port)
            if _, found := uniques[name]; !found {
                uniques[name] = struct{}{}
                domains := generateVirtualHostDomains(svc, virtualHostWrapper.Port, node)
                virtualHosts = append(virtualHosts, &route.VirtualHost{
                    Name:    name,
                    Domains: domains,
                    Routes:  virtualHostWrapper.Routes,
                })
            } else {
                push.Add(model.DuplicatedDomains, name, node, fmt.Sprintf("duplicate domain from virtual service: %s", name))
            }
        }
```

看这几段代码，里面都是用 svc 的 `Hostname` 做 key 来去重。虽然还不确定这么做的目的，但是对于前面描述的场景里，那俩 Service 的 Hostname 相同，于是就只留下了一个，sidecar 通过 xDS 获取到的 route 里自然也就只剩 1 个了。

## 解决

目前来说，要解决这个问题，也很简单，把多个 addresses 拆开，每个 address 给一个单独的 host，互不冲突就行。
