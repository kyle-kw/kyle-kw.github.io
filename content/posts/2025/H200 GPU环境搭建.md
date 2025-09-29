+++
title = 'H200/A100 GPU环境离线搭建'
date = 2025-09-26T14:31:00+08:00
tags = ["H200", "CUDA", "Nvidia"]
+++

## 0. 基础环境

- 操作系统： RedHat 9.4
- 内存： 2T
- 磁盘： 14T
- GPU： H200 * 8

## 1. 文件准备

> 如果可以访问互联网可以跳过这一步。使用下面官网的教程依次安装。

### 1.1 [CUDA 12.8 runfile (local)](https://developer.nvidia.com/cuda-12-8-0-download-archive)

### 1.2 [nvidia-fabric-manager](https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/)

注意这个包名中的nvidia驱动版本需要和主机的nvidia驱动版本一致，可以使用CUDA中的nvidia驱动，以此为基础去下载`nvidia-fabric-manager`.

### 1.3 [Docker](https://docs.docker.com/engine/install/rhel/#install-from-a-package)

如果宿主机glic等相关依赖版本较低，安装包失败，可以安装二进制版本：[docker](https://docs.docker.com/engine/install/binaries/#install-static-binaries)、[docker-compose](https://github.com/docker/compose)

### 1.4 [NVIDIA Container Toolkit](https://github.com/NVIDIA/libnvidia-container/tree/gh-pages/stable/rpm/x86_64)

[官方文档](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

需要下载这四个包，下载最新、版本相同的包：

- `libnvidia-container1-x.x.x-1.x86_64.rpm`
- `libnvidia-container-tools-x.x.x-1.x86_64.rpm`
- `nvidia-container-toolkit-x.x.x-1.x86_64.rpm`
- `nvidia-container-toolkit-base-x.x.x-1.x86_64.rpm`

---

下面列出我安装时相关文件的版本：

```
cuda_12.8.0_570.86.10_linux.run
nvidia-fabric-manager-570.86.10-1.x86_64.rpm

containerd.io-1.7.25-3.1.el9.x86_64.rpm
docker-ce-28.0.1-1.el9.x86_64.rpm
docker-ce-cli-28.0.1-1.el9.x86_64.rpm
docker-buildx-plugin-0.21.1-1.el9.x86_64.rpm
docker-compose-plugin-2.33.1-1.el9.x86_64.rpm

libnvidia-container1-1.17.5-1.x86_64.rpm
libnvidia-container-tools-1.17.5-1.x86_64.rpm
nvidia-container-toolkit-1.17.5-1.x86_64.rpm
nvidia-container-toolkit-base-1.17.5-1.x86_64.rpm
```

以下全程是用root用户操作的，非root权限请使用sudo提权。


## 2. 开始安装

### 2.1 将上述所有文件打包传到服务器

### 2.2 安装Nvidia驱动和CUDA

使用cuda_12.8.0_570.86.10_linux.run安装nvidia驱动和cuda

```bash
sh cuda_12.8.0_570.86.10_linux.run
```

之后输入`accept`回车，在安装组件界面选择上Nvidia Driver(X表示选择)之后安装。

安装完成之后，编辑`~/.bashrc`文件(如果是其他终端，请使用其他rc文件)，添加

```
export PATH=/usr/local/cuda-12.8/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.8/lib64:$LD_LIBRARY_PATH
```

保存退出, 查看Nvidia驱动和CUDA是否安装成功。

```
source ~/.bashrc

nvidia-smi

nvcc --version
```

若安装过程中失败，大概率是缺少开发依赖包导致的，可以尝试安装开发依赖包之后，重新安装CUDA。建议装操作系统时安装开发依赖包。

### 2.3 安装 nvidia-fabric-manager

再次提醒此包名中的nvidia驱动版本和主机的nvidia驱动版本需要一致，不然会安装失败，导致CUDA用不了。

安装`nvidia-fabric-manager`，并启用：
```bash
dnf install nvidia-fabric-manager-570.86.10-1.x86_64.rpm
systemctl enable nvidia-fabricmanager
systemctl start nvidia-fabricmanager
systemctl status nvidia-fabricmanager
```

### 2.4 安装 docker 以及 docker compose

两种安装方式选择其中一个方式安装

1. 包安装：
```bash
dnf install ./containerd.io-1.7.25-3.1.el9.x86_64.rpm \
  ./docker-ce-28.0.1-1.el9.x86_64.rpm \
  ./docker-ce-cli-28.0.1-1.el9.x86_64.rpm \
  ./docker-buildx-plugin-0.21.1-1.el9.x86_64.rpm \
  ./docker-compose-plugin-2.33.1-1.el9.x86_64.rpm
```

2. 二进制安装
```bash
tar -zxvf ./docker-x.x.x.tgz
mv ./docker/* /usr/bin

chmod +x ./docker-compose-linux-x86_64
mv ./docker-compose-linux-x86_64 /usr/bin/docker-compose
```

添加Systemctld服务, 编辑`/use/lib/systemd/system/docker.service`，将以下内容复制进去
```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target nss-lookup.target docker.socket firewalld.service containerd.service time-set.target
Wants=network-online.target containerd.service
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutStartSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity

# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

# kill only the docker process, not all processes in the cgroup
KillMode=process
OOMScoreAdjust=-500

[Install]
WantedBy=multi-user.target
```

之后启用配置

```
systemctl enable docker
systemctl start docker
systemctl status docker
```

### 2.5 安装 NVIDIA Container Toolkit

包安装

```bash
dnf install ./libnvidia-container1-1.17.5-1.x86_64.rpm \
./libnvidia-container-tools-1.17.5-1.x86_64.rpm \
./nvidia-container-toolkit-1.17.5-1.x86_64.rpm \
./nvidia-container-toolkit-base-1.17.5-1.x86_64.rpm
```

安装完成之后配置docker，编辑`/etc/docker/daemon.json`(如果不存在则创建)

```json
{
  "runtimes": {
    "nvidia": {
      "args": [],
      "path": "nvidia-container-runtime"
    }
  },
  "default-runtime": "nvidia"
}
```

保存，重启docker服务: `systemctl restart docker`


至此整个环境就搭建好了。

![H200环境截图](/images/H200环境搭建截图.png)

## 3 附录

以下记录可能会用到的命令

### 3.1 卸载CUDA和Nvidia驱动

如果要更新CUDA，需要先把CUDA卸载。

步骤：
1. 卸载`cuda-uninstaller`
2. 卸载nvidia驱动`nvidia-uninstall`

注意重新安装nvidia驱动和cuda需要reboot重启机器。cuda版本变了注意修改`.bashrc`文件。

### 3.2 RedHat 包相关的命令

若缺少包，但找到不到下载地址，可以本地装一个相同版本的RedHat虚拟机，用于下载对应的包

- 查询包： `yum list nginx`
  - `dnf search linux-headers`
- 下载包： `dnf download nvidia-fabric-manager-570.86.10-1 --alldeps`
- 下载包以及相关依赖： `dnf download gcc --resolve --destdir=./data`
- 查询依赖： `yum deplist nvidia-container-runtime`
  - `repotrack nvidia-container-runtime` 


