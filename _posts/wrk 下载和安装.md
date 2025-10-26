在国内环境下载 `wrk` 时，由于 GitHub 访问可能不稳定，可通过 **国内镜像源** 或 **直接下载源码包** 两种方式解决。以下是适配国内网络的下载方法：

### 一、通过 Gitee 镜像仓库下载（推荐）
Gitee 是国内代码托管平台，提供 `wrk` 的镜像仓库，下载速度快且稳定：

#### 1. 安装依赖（同之前步骤，确保已安装）
```bash
# Alibaba Cloud Linux/CentOS
sudo yum install -y gcc openssl-devel git

# Ubuntu/Debian
sudo apt install -y build-essential libssl-dev git
```

#### 2. 从 Gitee 克隆镜像仓库
```bash
# 克隆 Gitee 上的 wrk 镜像（替代 GitHub）
git clone https://gitee.com/mirrors/wrk.git
cd wrk
```

#### 3. 编译安装
```bash
make  # 编译生成可执行文件
sudo cp wrk /usr/local/bin/  # 全局可用
```


### 二、直接下载源码包（无 git 或镜像访问问题时）
若服务器无法使用 `git`，可直接下载源码压缩包：

#### 1. 下载源码包（通过 Gitee 镜像）
```bash
# 下载最新版源码包（Gitee 镜像）
wget https://gitee.com/mirrors/wrk/repository/archive/master.zip -O wrk-master.zip

# 解压
unzip wrk-master.zip
cd wrk-master  # 进入解压后的目录
```

#### 2. 编译安装
```bash
make
sudo cp wrk /usr/local/bin/
```


### 三、验证安装
```bash
wrk --version
```
输出类似 `wrk 4.2.0 [epoll] Copyright (C) 2012 Will Glozer` 即成功。


### 国内下载注意事项
1. 若 `wget` 或 `git` 仍慢，可临时切换国内 DNS（如 `114.114.114.114` 或阿里云 DNS `223.5.5.5`）。
2. 部分服务器可能需要安装 `unzip`（解压工具）：
   ```bash
   # CentOS/Alibaba Cloud Linux
   sudo yum install -y unzip
   # Ubuntu/Debian
   sudo apt install -y unzip
   ```

通过以上方法，可在国内环境快速获取 `wrk` 源码并完成安装，后续使用方式与之前完全一致。
