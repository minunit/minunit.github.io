+++
author = "yangkai"
title = "kubernetes集群环境搭建"
date = "2025-01-18"
description = "指导k8s集群环境搭建"
categories = [
    "kubernetes"
]
tags = [
    "kubernetes","k8s"
]
+++

## 前提准备
### 虚拟机准备
Note: 这里可以置后配置，先配置其中一台，当完成安装Docker/Docker CRI之后，最后再复制虚拟机，避免重复操作。

1. 虚拟机(Centos7)搭建，网络模式网桥(NAT模式差别不大，这里主要是网桥)
2. 配置虚拟host，这一步不是必须的，不过可以配置上
```bash
sudo cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.31.10 infra0  # etcd/haproxy/keepalive  1C2G
192.168.31.11 infra1  # etcd/haporxy/keepalive  1C2G
192.168.31.12 infra2  # etcd 1C2G
192.168.31.20 k8s-master # k8s control manager 2C4G
192.168.31.21 k8s-master-2 # k8s control manager 2C4G
192.168.31.30 k8s-node-30 # k8s node 2C4G
192.168.31.31 k8s-node-31 # k8s node 2C4G
192.168.31.32 k8s-node-32 # k8s node 2C4G
EOF
```
### 关闭防火墙
```bash
sudo systemctl status firewalld
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```
### 设置时钟, 保持同步
服务器之间的时间同步很重要，如果时间不同步会出现很多奇怪的异常，比如Calico网络异常。
```bash
sudo yum install -y chrony
sudo systemctl start chronyd
sudo systemctl enable chronyd
sudo timedatectl set-ntp true

sed -i '1i server ntp1.aliyun.com iburst' /etc/chrony.conf
sed -i '1i server ntp2.aliyun.com iburst' /etc/chrony.conf
sed -i '1i server ntp3.aliyun.com iburst' /etc/chrony.conf
sed -i '1i server time.windows.com iburst' /etc/chrony.conf
sudo systemctl restart chronyd
```
### 配置IPV4
1. 编辑/etc/yum.conf，添加ip_resolve=4
```bash
vim /etc/yum.conf
ip_resolve=4 ## 添加以下内容
```
2. 启用 IPv4 数据包转发
```bash
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
modprobe br_netfilter
echo "br_netfilter" > /etc/modules-load.d/br_netfilter.conf
systemctl restart systemd-modules-load.service
```
### 关闭swap
编辑 /etc/fstab 文件, 注释/dev/mapper/centos-swap
```bash
vim /etc/fstab
# 将其注释掉，在行首加 #：
# /dev/mapper/centos-swap swap swap defaults 0 0
#保存并退出后，运行以下命令以重新挂载文件系统：
sudo mount -a

vim /etc/sysctl.conf
#添加以下行
vm.swappiness = 0
# 重启
sysctl --system
```
### 将 SELinux 设置为permissive 模式
这些说明适用于 Kubernetes 1.30 (Set SELinux in permissive mode (effectively disabling it))
```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

警告(来自k8s官网)：
setenforce 0通过运行并有效禁用SELinux，将其设置为宽容模式sed .
这是允许容器访问主机文件系统所必需的；例如，某些集群网络插件需要这样做。您必须这样做，直到 kubelet 中的 SELinux 支持得到改进。
如果您知道如何配置 SELinux，则可以保持其启用状态，但它可能需要 kubeadm 不支持的设置。

### 添加 Kubernetesyum存储库
Note: 这里的aliyun仓库是新地址，老地址只能下载老版本

```bash
# This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.30/rpm/repodata/repomd.xml.key
EOF

sudo yum clean all
sudo yum makecache
```

### 安装 kubelet、kubeadm 和 kubectl
`yum install kubelet kubeadm kubectl --disableexcludes=kubernetes`

## 安装Docker
```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://download.docker.com/linux/centos/docker-ce.repo #（中央仓库）
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo #（阿里仓库）

yum install docker-ce docker-ce-cli containerd.io docker-compose -y

cat > /etc/docker/daemon.json <<EOF
{
 "registry-mirrors": [
        "https://*****.mirror.swr.myhuaweicloud.com",
        "https://*****.mirror.aliyuncs.com"
 ],
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  }
}
EOF

systemctl start docker
systemctl enable docker

docker pull registry.aliyuncs.com/google_containers/pause:3.9
docker tag registry.aliyuncs.com/google_containers/pause:3.9 registry.k8s.io/pause:3.9
```
## 安装Docker CRI
Note: 由于kubernetes 新版本不再默认支持docker cri，所以需要手动安装。国内还是使用docker比较多，所以这里直接安装Docker CRI


```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.15/cri-dockerd-0.3.15-3.el7.x86_64.rpm
yum localinstall cri-dockerd-0.3.14-3.el7.x86_64.rpm 
## 安装后，需要将其默认的pause镜像，修改成国内镜像，不然会拉不下来镜像（Note: 网络可达的情况不会需要配置）
sudo sed -ri 's@^(.*fd://).*$@\1 --pod-infra-container-image registry.aliyuncs.com/google_containers/pause@' /usr/lib/systemd/system/cri-docker.service
sudo sed -ri 's@^(.*fd://).*$@\1 --pod-infra-container-image registry.k8s.io/pause@' /usr/lib/systemd/system/cri-docker.service

sudo systemctl daemon-reload && sudo systemctl restart cri-docker && sudo systemctl enable cri-docker
```


**Node: 此时需要复制虚拟机了，上述属于公共配置，下面就需要进行其他配置了**

## etcd集群搭建
### 下载etcd
```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.5.17/etcd-v3.5.17-linux-amd64.tar.gz
tar -xvf etcd-v3.5.17-linux-amd64.tar.gz
mv etcd-v3.5.17-linux-amd64 etcd
```
### 修改hostname
```bash
hostnamectl set-hostname infra0 # 192.168.31.10
hostnamectl set-hostname infra1 # 192.168.31.11
hostnamectl set-hostname infra2 # 192.168.31.12
```
### 配置etcd 集群
#### 192.168.31.10
在 192.168.31.10 (k8s-master) 上：
```bash
cat > /root/etcd/etcd.conf <<EOF
name: 'infra0'
data-dir: '/root/etcd/data'
wal-log: '/root/etcd/wal'
listen-peer-urls: 'http://192.168.31.10:2380'
listen-client-urls: 'http://192.168.31.10:2379,http://127.0.0.1:2379'
initial-advertise-peer-urls: 'http://192.168.31.10:2380'
advertise-client-urls: 'http://192.168.31.10:2379'
initial-cluster: 'infra0=http://192.168.31.10:2380,infra1=http://192.168.31.11:2380,infra2=http://192.168.31.12:2380'
initial-cluster-token: 'etcd'
initial-cluster-state: 'new'
EOF

cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd service
After=network.target

[Service]
Type=simple
ExecStart=/root/etcd/etcd --config-file /root/etcd/etcd.conf
WorkingDirectory=/root/etcd
StandardOutput=append:/root/etcd/log/etcd.log
StandardError=append:/root/etcd/log/etcd.log
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

#### 192.168.31.11
Note: etcd service配置同192.168.31.10一样，不在列出。
```bash
cat > /root/etcd/etcd.conf <<EOF
name: 'infra1'
data-dir: '/root/etcd/data'
wal-log: '/root/etcd/wal'
listen-peer-urls: 'http://192.168.31.11:2380'
listen-client-urls: 'http://192.168.31.11:2379,http://127.0.0.1:2379'
initial-advertise-peer-urls: 'http://192.168.31.11:2380'
advertise-client-urls: 'http://192.168.31.11:2379'
initial-cluster: 'infra0=http://192.168.31.10:2380,infra1=http://192.168.31.11:2380,infra2=http://192.168.31.12:2380'
initial-cluster-token: 'etcd'
initial-cluster-state: 'new'
EOF
nohup /root/etcd/etcd --config-file /root/etcd/etcd.conf > /root/etcd/log/etcd.log 2>&1 &
```
#### 192.168.31.12
Note: etcd service配置同192.168.31.10一样，不在列出。
```bash
cat > /root/etcd/etcd.conf <<EOF
name: 'infra2'
data-dir: '/root/etcd/data'
wal-log: '/root/etcd/wal'
listen-peer-urls: 'http://192.168.31.12:2380'
listen-client-urls: 'http://192.168.31.12:2379,http://127.0.0.1:2379'
initial-advertise-peer-urls: 'http://192.168.31.12:2380'
advertise-client-urls: 'http://192.168.31.12:2379'
initial-cluster: 'infra0=http://192.168.31.10:2380,infra1=http://192.168.31.11:2380,infra2=http://192.168.31.12:2380'
initial-cluster-token: 'etcd'
initial-cluster-state: 'new'
EOF
```
### 启动并验证 etcd 服务
```bash
# 启动 etcd 服务
sudo systemctl start etcd
sudo systemctl enable etcd
# 验证
/root/etcd/etcdctl --endpoints=http://192.168.31.10:2379 member list --write-out=table 
/root/etcd/etcdctl --endpoints=http://192.168.31.10:2379,http://192.168.31.11:2379,http://192.168.31.12:2379 endpoint status --write-out=table 
```

## 搭建Haproxy/Keepalive
Haproxy和Keepalive配合，作为K8S Apiservice的负载均衡
### 搭建Haproxy
```bash
## 下载源码并编译
wget https://github.com/haproxy/haproxy/archive/refs/tags/v3.1.0.tar.gz
sudo yum -y install make gcc pcre-devel bzip2-devel openssl-devel
make TARGET=linux31 USE_OPENSSL=1 ADDLIB=-lz ;
make install PREFIX=/usr/local/haproxy
ln -s /usr/local/haproxy/sbin/haproxy /usr/sbin/
sudo groupadd haproxy
sudo useradd -g haproxy haproxy
## 配置haproxy.cfg, 将流量转发到对应192.168.31.20/21机器上
cat > /usr/local/haproxy/haproxy.cfg  <<EOF
global
    log /dev/log    local0
    log /dev/log    local1 notice
    #chroot /usr/sbin/haproxy
  #  stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 2000

defaults
    log     global
    mode    tcp                       # 确保使用 TCP 模式
    option  tcplog                    # 使用 TCP 日志记录
    option  dontlognull
    timeout connect 5000ms            # 连接超时
    timeout client  50000ms           # 客户端超时
    timeout server  50000ms           # 服务端超时

frontend kubernetes-api
    bind *:6443
    default_backend kube-apiserver
    option tcplog                     # 为此 frontend 启用 TCP 日志

backend kube-apiserver
    balance roundrobin                # 负载均衡算法
    option tcp-check                  # 启用 TCP 检查
    server kube-api-1 192.168.31.20:6443 check
    server kube-api-2 192.168.31.21:6443 check
EOF

## 配置haproxy service，通过systemed管理
cat > /usr/lib/systemd/system/haproxy.service <<EOF
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target
Documentation=man:haproxy
Documentation=file:/root/haproxy-3.1.0/doc/configuration.txt
[Service]
ExecStartPre=/usr/sbin/haproxy -f /usr/local/haproxy/haproxy.cfg -c -q
ExecStart=/usr/sbin/haproxy -Ws -f /usr/local/haproxy/haproxy.cfg -p /run/haproxy.pid
ExecReload=/bin/kill -USR2 $MAINPID
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable haproxy
systemctl start haproxy
```

### 搭建keepalive
```bash
yum install keepalived -y
cat > /etc/keepalived/keepalived.conf <<EOF
vrrp_instance VI_1 {
    state MASTER
    interface ens33 #和本机保持一致
    virtual_router_id 51
    priority 101
    advert_int 1
    virtual_ipaddress {
        192.168.100.100
    }
}
EOF
# 启动 Keepalived
sudo systemctl start keepalived
# 设置开机自启
sudo systemctl enable keepalived
sudo systemctl status keepalived
```
## 部署K8S集群
Note: 这里通过yml文件的方式进行部署
### 初始化K8S
```bash
### 配置kubeadm-config.yaml
## 官方文档 https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/
cat > /etc/kubernetes/kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  name: k8s-master
  criSocket: unix:/var/run/cri-dockerd.sock
  taints: []
  imagePullPolicy: IfNotPresent
localAPIEndpoint:
  advertiseAddress: 192.168.31.20
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
etcd:
   external:
     endpoints:
     - http://192.168.31.10:2379
     - http://192.168.31.11:2379
     - http://192.168.31.12:2379
networking:
  serviceSubnet: 10.96.0.0/16
  podSubnet: 10.244.0.0/24
  dnsDomain: cluster.local
kubernetesVersion: v1.30.0
controlPlaneEndpoint: 192.168.31.100:6443
certificatesDir: /etc/kubernetes/pki
imageRepository: registry.aliyuncs.com/google_containers
clusterName: kubernetes
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

## 初始化节点
kubeadm init --config=/etc/kubernetes/kubeadm-config.yaml
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

sudo kubeadm init phase upload-certs --upload-certs --config kubeadm-config.yaml
```
Note: 结果这里会告诉我们如何加入control-plane，以及node节点
```test
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

kubeadm join 192.168.31.100:6443 --token ph2mh3.5s04nehxqumvia6u \
--discovery-token-ca-cert-hash sha256:683dfa2d87508e63862d6bd24676eb773126a670c96c10a71134f8c5ba3450a6 \
--control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.31.100:6443 --token ph2mh3.5s04nehxqumvia6u \
--discovery-token-ca-cert-hash sha256:683dfa2d87508e63862d6bd24676eb773126a670c96c10a71134f8c5ba3450a6 
```
### 加入控制平面集群
Note: 在日志中的命令中，加入--cri-socket unix:///var/run/cri-dockerd.sock，指定具体的cri
```bash
kubeadm join 192.168.31.100:6443 --token ph2mh3.5s04nehxqumvia6u \
--discovery-token-ca-cert-hash sha256:683dfa2d87508e63862d6bd24676eb773126a670c96c10a71134f8c5ba3450a6 \
--cri-socket unix:///var/run/cri-dockerd.sock \
--control-plane
```
### 加入Node节点
```bash 
kubeadm join 192.168.31.100:6443 --token ph2mh3.5s04nehxqumvia6u \
--discovery-token-ca-cert-hash sha256:683dfa2d87508e63862d6bd24676eb773126a670c96c10a71134f8c5ba3450a6 
# 这里如果Node节点想要使用kubelet命令，可以做以下操作
scp /etc/kubernetes/admin.conf root@192.168.31.30:/etc/kubernetes
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```
### 查看节点状态
Note: 由于电脑性能有限，这里并没有只有一个control plan

``` bash
  [root@k8s-master ~]# kubectl get node
  NAME          STATUS      ROLES           AGE   VERSION
  k8s-master    NotReady    control-plane   0     v1.30.8
  k8s-node-30   NotReady    <none>          0     v1.30.8
  k8s-node-31   NotReady    <none>          0     v1.30.8
```

此时会发现节点属于NotReady的状态，是由于并没有安装网络，所以会出现NotReady，安装好网络即可

## 安装Calico网络插件
```bash
wget https://github.com/projectcalico/calico/releases/download/v3.29.1/release-v3.29.1.tgz
tar xvf release-v3.29.1.tgz
# 修改docker镜像, 如果网络可达docker.io可不做修改
sed -i 's#docker.io/##g' calico.yaml
```
这里需要加入自动选择IP的方式，通过ens33的网卡来进行
```bash
# 在 - name: CALICO_IPV4POOL_IPIP 上面加入
- name: IP_AUTODETECTION_METHOD
  value: interface=ens33
```

等安装完成后，查看节点状态，即可发现已经变成Ready.
``` bash
  [root@k8s-master ~]# kubectl get node
  NAME          STATUS    ROLES         AGE   VERSION
  k8s-master    Ready    control-plane   0     v1.30.8
  k8s-node-30   Ready    <none>          0     v1.30.8
  k8s-node-31   Ready    <none>          0     v1.30.8
```
至此k8s集群安装完成，但并不满足真正生产的使用条件，后续还需要安装一些服务
1. Ingress/Gateway对外开放服务
2. 安装Rancher，k8s管理平台，通过页面进行管理
3. gitlab + jenkines 进行自动化部署到K8S
4. harbor 完成私有化镜像
5. ELK 完成日志监控
6. Prometheus 监控K8S集群，以及服务状态，各种报警等

### 安装calicoctl
Note: 这里不是必需的，有需要可以安装
```bash
curl -O -L https://github.com/projectcalico/calico/releases/download/v3.29.1/calicoctl-linux-amd64
chmod +x calicoctl-linux-amd64
sudo mv calicoctl-linux-amd64 /usr/local/bin/calicoctl
```
