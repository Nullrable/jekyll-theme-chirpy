---
title: Kubernetes环境安装
author: nhsoft.lsd
date: 2022-08-17
categories: [Kubernetes]
tags: [Kubernetes,Docker]
pin: false
order: 2
---

# 1. 安装Docker
* [Docker for Mac](https://docs.docker.com/desktop/install/mac-install/)
* [Docker for Windows](https://docs.docker.com/desktop/install/windows-install/)

# 2. 配置镜像加速服务
国内下载 Kubernetes 集群所需的镜像速度太慢，因此需要在 Preferences>>Docker Engine 中配置一下，注意主要增加这个参数即可`registry-mirrors`：
```yaml
{
  "insecure-registries": [],
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn"
  ],
  "experimental": true,
  "debug": true
}
```
我这边翻墙了，所以没配置代理
![](/assets/img/nhsoft_lsd/2022-08-19-img.png)

# 3. 安装Kubernetes
Docker设置中勾选上后这两个选项后，会自动下载

![](/assets/img/nhsoft_lsd/2022-08-19-img_1.png)

安装完以后`docker images`，会列出k8s相关的镜像

![](/assets/img/nhsoft_lsd/2022-08-19-img_2.png)

# 4. 验证Kubernetes是否安装成功
`kubectl get nodes`

![](/assets/img/nhsoft_lsd/2022-08-19-img_3.png)

`kubectl get pods -n kube-system`
![](/assets/img/nhsoft_lsd/2022-08-19-img_4.png)

# 5. 部署 Kubernetes Dashboard
## 5.1 安装Kubernetes Dashboard
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml

# 开启本机访问代理，默认监听 8001 端口
kubectl proxy
```
## 5.2 访问[登录页面](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)
![img_5.png](/assets/img/nhsoft_lsd/2022-08-19-img_5.png)

## 5.3 选择Token登陆

### 5.3.1 创建管理员用户
创建文件`dashboard-admin.yml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

### 5.3.2 [执行配置文件](/assets/img/nhsoft_lsd/dashboard-admin.yml)
```shell
# 创建 ServiceAccount kubernetes-dashboard-admin 并绑定集群管理员权限
kubectl apply -f dashboard-admin.yml

# 获取登陆 token
kubectl -n kubernetes-dashboard create token admin-user

# 清空账号
kubectl -n kubernetes-dashboard delete serviceaccount admin-user
kubectl -n kubernetes-dashboard delete clusterrolebinding admin-user
```
![](/assets/img/nhsoft_lsd/2022-08-19-img_9.png)

### 5.3.3 启动本地代理
`kubectl proxy`
![](/assets/img/nhsoft_lsd/2022-08-19-img_8.png)

## 5.3.4 填入Token，并登陆
![](/assets/img/nhsoft_lsd/2022-08-19-img_7.png)


[官方文档](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md)
