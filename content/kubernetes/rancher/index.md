+++
author = "yangkai"
title = "kubernetes rancher"
date = "2025-01-19"
description = "指导k8s集群环境搭建"
categories = [
    "kubernetes"
]
tags = [
    "kubernetes","k8s","Rancher"
]
+++


参考文档: 
1. Rancher文档: https://ranchermanager.docs.rancher.com/zh/getting-started/quick-start-guides/deploy-rancher-manager/helm-cli
2. Rancher文档: https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster
3. Rancher权威指南: https://forums.rancher.cn/t/rancher-4-lb/1681/4

Rancher的安装很简单，但一般是卡在证书这个位置，我本地是用的是公网证书，所以不需要自己生成私有证书

Note: 私有证书的部署，参考权威指南即可.

```bash
## {slef.demain.com}是自己的域名
## 设置证书
kubectl -n cattle-system create secret tls tls-rancher-ingress \
--cert=scs1733813642003_{slef.demain.com}_server.crt \
--key=scs1733813642003_{slef.demain.com}_server.key
### 添加rancher helm repo
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
### 安装rancher,username 默认值为 admin
helm install rancher rancher-stable/rancher \
--create-namespace \
--namespace cattle-system \
--set hostname={slef.demain.com} \
--set bootstrapPassword=admin@1230 \
--set replicas=1 \
--set ingress.tls.source=secret 
##   --set privateCA=true 如果是私有证书，才需要加这个参数
```

问题：公网域名如何在局域网中使用，我这里的方案是: 华为云EC + Nginx + FRP(内网穿透)，这仅仅只能用于学习，真正的生产都会有自己LB，以及公开的服务IP