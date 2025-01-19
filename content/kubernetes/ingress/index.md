+++
author = "yangkai"
title = "kubernetes ingress配置"
date = "2025-01-19"
description = "指导k8s集群环境搭建"
categories = [
    "kubernetes"
]
tags = [
    "kubernetes","k8s","ingress"
]
+++

## 安装helm
参考官方文档: https://helm.sh/zh/docs/intro/install/
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
## 安装ingress
相关文档:
1. Ingress Nginx Github: https://github.com/kubernetes/ingress-nginx
2. K8S Ingress Nginx文档: https://kubernetes.io/docs/concepts/services-networking/ingress/
3. Ingress Nginx文档: https://kubernetes.github.io/ingress-nginx/deploy/

Note: 通过Kubernate官方文档可以看出，目前官方更推荐Gateway的方式来进行服务对外暴露，我这里搭建的仍然是Ingress的方式，如果需要搭建Gateway，则参考官方文档 

### 下载Ingress Nginx
首先通过helm拉取下来对应的yaml，然后对其进行修改，否则国内安装会有问题

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm pull ingress-nginx/ingress-nginx
tar ingress-nginx-4.12.0.tgz
```
### 修改Ingress value.yaml
```bash
global:
   image:
    # -- Registry host to pull images from.
     registry: registry.cn-hangzhou.aliyuncs.com

# 修改镜像并注释(由于修改为国内，所以就注释掉digest/digestChroot，不进行校验镜像)
google_containers/nginx-ingress-controller
#digest: sha256:e6b8de175acda6ca913891f0f727bca4527e797d52688cbe9fec9040d6f6b6fa
#digestChroot: sha256:87c88e1c38a6c8d4483c8f70b69e2cca49853bb3ec3124b9b1be648edf139af3

# 修改镜像并注释
google_containers/kube-webhook-certgen
#digest: sha256:aaafd456bda110628b2d4ca6296f38731a3aaf0bf7581efae824a41c770a8fc4
# Deployment 修改为 DaemonSet， 并增加nodeSelect
   nodeSelector:
     kubernetes.io/os: linux
     ingress: "true"

# 已经有自己的cni网络，所以使用本身的网络
# Optionally change this to ClusterFirstWithHostNet in case you have 'hostNetwork: true'.
dnsPolicy: ClusterFirstWithHostNet
hostNetwork: true

# 不使用LoadBalancer方式, 这里不过多介绍LoadBalancer，一般会选择外部LB，这里没有，就使用本机IP
type: ClusterIp  #LoadBalancer

# 修改false，https验证问题
admissionWebhooks:
    enabled: false
```

### 安装Ingress到主节点
在配置文件中，对其部署增加了nodeSelector, 所以如果我想将其部署到k8s-master节点上，则需要对master节点进行打标签
```bash
## 给master标签: ingress: "true"
kubectl label node k8s-master ingress=true
## 部署Ingress
helm install ingress-nginx --namespace ingress-nginx --create-namespace .
```

## 测试Ingress
```bash
vim ~/ingress/nginx-po-ingress.yaml
kubectl apply -f ~/ingress/nginx-po-ingress.yaml
## 测试 是否正常，正常会返回nginx的html
curl --resolve ghosting.k8s.ingress:80:192.168.31.20 http://ghosting.k8s.ingress/nginx
```

nginx-po-ingress.yaml的配置如下
```yaml :nginx-po-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-po-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: ghosting.k8s.ingress
      http:
        paths:
          - pathType: Prefix
            path: /nginx
            backend:
              service:
                name: nginx
                port:
                  number: 80
```
