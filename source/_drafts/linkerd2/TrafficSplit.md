---
title: "linkerd2 TrafficSplit 模版解析"
date: 2019-10-30T16:58:27+08:00
draft: false
author: "哗啦啦mesh团队"
authorlink: "https://github.com/ilylia"
summary: "linkerd2 TrafficSplit 模版解析。"
tags: ["linkerd", "traffic split"]
categories: ["linkerd"]
keywords: ["service mesh","服务网格","linkerd2","linkerd"]
---



## Template:

```yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: my-weights
spec:
  # 客户端连接的service.
  service: numbers
  # 实际的后端service.
  backends:
  - service: one
    # 权重, 1 = 1000m
    weight: 10m
  - service: two
    weight: 100m
  - service: three
    weight: 1500m
```





## Examples:

### service

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: reviews-v1
  labels:
    app: reviews
    version: v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: reviews
        version: v1
    spec:
      containers:
      - name: reviews
        image: istio/examples-bookinfo-reviews-v1:1.10.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
apiVersion: v1
kind: Service
metadata:
  name: reviews-v1
  labels:
    app: reviews
    service: reviews
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: reviews
    version: v1
```

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: reviews-v2
  labels:
    app: reviews
    version: v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: reviews
        version: v2
    spec:
      containers:
      - name: reviews
        image: istio/examples-bookinfo-reviews-v2:1.10.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
apiVersion: v1
kind: Service
metadata:
  name: reviews-v2
  labels:
    app: reviews
    service: reviews
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: reviews
    version: v2
```



###  TrafficSplit

```yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: reviews-rollout
spec:
  service: reviews
  backends:
  - service: reviews-v1
    weight: "900m"
  - service: reviews-v2
    weight: "100m"
```

