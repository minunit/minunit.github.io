+++
author = "yangkai"
title = "kubernetes metrics"
date = "2025-01-19"
description = "指导k8s集群环境搭建, metrics nodes使用率"
categories = [
    "kubernetes"
]
tags = [
    "kubernetes","k8s","metrics"
]
+++

Note: 使用metrics来进行K8S Node的使用状态，包括CPU/内存等。

通过Helm安装metrics，并修改为国内镜像
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm pull metrics-server/metrics-server
```

修改value.yaml
```yaml -> value.yaml
image:
  repository: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server #registry.k8s.io/metrics-server/metrics-server
defaultArgs:
  - --cert-dir=/tmp
  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
  - --kubelet-use-node-status-port
  - --kubelet-insecure-tls   # 跳过tls检查
  - --metric-resolution=15s
  
addonResizer:
  enabled: false
  image:
    repository: registry.cn-beijing.aliyuncs.com/minminmsn/addon-resizer #registry.k8s.io/autoscaling/addon-resizer
    tag: 1.8.4 #1.8.21
```

```bash
helm install metrics-server --namespace metrics-server --create-namespace .
helm upgrade metrics-server --namespace metrics-server .
kubectl
```