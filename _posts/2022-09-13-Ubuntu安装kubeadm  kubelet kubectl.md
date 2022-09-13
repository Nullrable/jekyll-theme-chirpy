---
title: Ubuntu安装kubeadm kubelet kubectl
author: nhsoft.lsd
date: 2022-09-13
categories: [Kubernetes]
tags: [Kubernetes,kubeadm]
pin: false
---

# Ubuntu安装kubeadm kubelet kubectl

1. curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
2. vim /etc/apt/sources.list.d/kubernetes.list deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
3. apt-get update
4. apt-get install -y kubelet kubeadm kubect
5. 安装kubenetes核心组件
```shell
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.3
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.3
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.3
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.3
```
6. 替换docker镜像tag
```shell
   docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.3 k8s.gcr.io/kube-controller-manager:v1.17.3
   docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.3 k8s.gcr.io/kube-scheduler:v1.17.3
   docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.3 k8s.gcr.io/kube-proxy:v1.17.3
   docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
   docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
   docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5 k8s.gcr.io/coredns:1.6.5
   docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.3 k8s.gcr.io/kube-apiserver:v1.17.3
```
7. 【选项】如果之前安装不成功，执行一下命令重置
   `kubeadm reset`
8. 初始化master节点，`apiserver-advertise-address`为本地节点
     `kubeadm init --pod-network-cidr=192.168.16.0/20 --apiserver-advertise-address=10.0.2.5 --ignore-preflight-errors=Swap`
9. 【选项】如果上面提示swap需要关闭错误，操作以下命令
   `vim /etc/fstab`，然后将`/dev/mapper/cryptswap1 none swap sw 0 0`  进行注解
10. 记录下该句，重要；用于工作节点加入
   ```shell
    kubeadm join 10.0.2.5:6443 --token 6zhllo.ua4oxad15k1ahbgd \
    --discovery-token-ca-cert-hash sha256:160a884e3cf04c5afa86fc399d0e73eb2e904fed117b05113c0091045064bde9
   ```
11. 设置KUBECONFIG环境变量vim .bashrc
    `export KUBECONFIG='/etc/kubernetes/admin.conf'`


辅助命令

查看日志
journalctl -f -u kubelet.service

永久修改主机名
vi /etc/hostname

关闭swap
vim /etc/fstab

安装flannel 网络插件
yaml文件见该GitHub
https://github.com/coreos/flannel/find/master

放到本地后使用以下命令
kubectl apply -f kube-fannel.yml

//修改工作节点访问其他节点
1. 将master节点 scp admin.conf root@10.0.2.8:/etc/kubernetes/
2. 进入工作节点 vim .bashrc
3. 将该句加在最后一行 export KUBECONFIG='/etc/kubernetes/admin.conf'
4. 将参数生效 source .bashrc

查看命名空间kube-system的pods
kubectl get pods -n kube-system -o wide

查看pod日志
kubectl logs -f kubernetes-dashboard-74c96fd8ff-vcxrn -n kube-system


