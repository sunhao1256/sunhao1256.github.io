---
title: "K8s安装"
date: 2022-07-02T20:01:43+08:00
tags: ['k8s']
---

> 文章摘自https://k8s.easydoc.net/docs/dRiQjyTY/28366845/6GiNOzyZ/nd7yOvdY

# 💽安装 Kubernetes 集群

### 安装方式介绍

- **minikube**
  只是一个 K8S 集群模拟器，只有一个节点的集群，只为测试用，master 和 worker 都在一起
- **直接用云平台 Kubernetes**
  可视化搭建，只需简单几步就可以创建好一个集群。
  优点：安装简单，生态齐全，负载均衡器、存储等都给你配套好，简单操作就搞定
- **裸机安装（Bare Metal）**
  至少需要两台机器（主节点、工作节点个一台），需要自己安装 Kubernetes 组件，配置会稍微麻烦点。
  可以到各云厂商按时租用服务器，费用低，用完就销毁。
  缺点：配置麻烦，缺少生态支持，例如负载均衡器、云存储。

> 本文档课件需配套 [视频](https://www.bilibili.com/video/BV1Tg411P7EB?p=2) 一起学习

### minikube

安装非常简单，支持各种平台，[安装方法](https://minikube.sigs.k8s.io/docs/start/)

> 需要提前安装好 Docker

```
# 启动集群
minikube start
# 查看节点。kubectl 是一个用来跟 K8S 集群进行交互的命令行工具
kubectl get node
# 停止集群
minikube stop
# 清空集群
minikube delete --all
# 安装集群可视化 Web UI 控制台
minikube dashboard
```

### 云平台搭建

- [腾讯云 TKE](https://cloud.tencent.com/product/tke)（控制台搜索容器）
- 登录阿里云控制台 - 产品搜索 Kubernetes

### 裸机搭建（Bare Metal）

##### 主节点需要组件

- docker（也可以是其他容器运行时）
- kubectl 集群命令行交互工具
- kubeadm 集群初始化工具

##### 工作节点需要组件 [文档](https://kubernetes.io/zh/docs/concepts/overview/components/#node-components)

- docker（也可以是其他容器运行时）
- kubelet 管理 Pod 和容器，确保他们健康稳定运行。
- kube-proxy 网络代理，负责网络相关的工作

#### 开始安装

> 你也可以试下 [这个项目](https://github.com/lework/kainstall)，用脚本快速搭建 K8S 裸机集群
> 当然，为了更好的理解，你应该先手动搭建一次

```
# 每个节点分别设置对应主机名
hostnamectl set-hostname master
hostnamectl set-hostname node1
hostnamectl set-hostname node2
# 所有节点都修改 hosts
vim /etc/hosts
172.16.32.2 node1
172.16.32.6 node2
172.16.0.4 master
# 所有节点关闭 SELinux
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

> 所有节点确保防火墙关闭
> systemctl stop firewalld
> systemctl disable firewalld

添加安装源（所有节点）

```
# 添加 k8s 安装源
cat <<EOF > kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
mv kubernetes.repo /etc/yum.repos.d/

# 添加 Docker 安装源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

安装所需组件（所有节点）
`yum install -y kubelet-1.22.4 kubectl-1.22.4 kubeadm-1.22.4 docker-ce`

> 注意，据学员反馈，1.24 以上的版本会报错，跟教程有差异，所以建议大家指定版本号安装，版本号确保跟老师的一样

启动 kubelet、docker，并设置开机启动（所有节点）

```
systemctl enable kubelet
systemctl start kubelet
systemctl enable docker
systemctl start docker
```

修改 docker 配置（所有节点）

```
# kubernetes 官方推荐 docker 等使用 systemd 作为 cgroupdriver，否则 kubelet 启动不了
cat <<EOF > daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://ud6340vz.mirror.aliyuncs.com"]
}
EOF
mv daemon.json /etc/docker/

# 重启生效
systemctl daemon-reload
systemctl restart docker
```

用 [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) 初始化集群（仅在主节点跑），

```
# 初始化集群控制台 Control plane
# 失败了可以用 kubeadm reset 重置
kubeadm init --image-repository=registry.aliyuncs.com/google_containers

# 记得把 kubeadm join xxx 保存起来
# 忘记了重新获取：kubeadm token create --print-join-command

# 复制授权文件，以便 kubectl 可以有权限访问集群
# 如果你其他节点需要访问集群，需要从主节点复制这个文件过去其他节点
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 在其他机器上创建 ~/.kube/config 文件也能通过 kubectl 访问到集群
```

> 有兴趣了解 kubeadm init 具体做了什么的，可以 [查看文档](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)

把工作节点加入集群（只在工作节点跑）

```
kubeadm join 172.16.32.10:6443 --token xxx --discovery-token-ca-cert-hash xxx
```

安装网络插件，否则 node 是 NotReady 状态（主节点跑）

```
# 很有可能国内网络访问不到这个资源，你可以网上找找国内的源安装 flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

查看节点，要在主节点查看（其他节点有安装 kubectl 也可以查看）
![image.png](https://sjwx.easydoc.xyz/46901064/files/kw8x2qdf.png)

# 更新configmap

```yaml
kubectl create configmap foo --from-file foo.properties -o yaml --dry-run | kubectl apply -f -
```

# NodePort 从master节点访问会超时

