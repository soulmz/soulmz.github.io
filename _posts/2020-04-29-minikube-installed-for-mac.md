---
layout:     post   				    
title:      MacOS 安装 Minikube
subtitle:   Minikube Install On MacOS
date:       2020-04-29		
author:     soulmz				
header-img: img/backgroup/77642582_p0.jpg	
catalog: true 						
tags:								
  - Kubernetes 
  - Minikube
---



# Minikube

[Minikube Github](https://github.com/kubernetes/minikube)

Minikube 是 `kubernetes` 社区为了方便大家，快速上手学习 k8s 服务的工具。 `All In Single Node`

注: 本文将以 MacOS 方式来安装并介绍。

如其他操作系统，可以参考：

[Minikube - kubernetes 本地试验环境](https://yq.aliyun.com/articles/221687)

## Minikube 架构图

![image-20200429141829808](/img/in-post/image-20200429141829808.png)

用户通过 kubectl 来对 kubernetes 进行管理

Minikube 运行在  Hyperkit (VM) 虚拟机

## 前提条件

> 注意:   本文 采用 `Homebrew` 来管理包。

### 安装 Docker for Mac 客户端

```bash
brew cask install docker
```

> 注意: docker 切勿开启 kubernetes 功能 ，需要配置

![image-20200429141720025](/img/in-post/image-20200429141720025.png)

### 安装 hyperkit

```bash
brew install hyperkit
```

### 安装 Minikube & Kubectl

```bash
brew install kubernetes-cli minikube
```



以上安装完成后，可以使用 Minikube 进行安装 kubernetes 集群了

##  Minikube 安装 Kubernetes v1.17.3

> Minikube 默认 使用本地最好的驱动来创建 kubernetes 本地环境
>
> ```bash
> minikube start
> ```
>
> 但是，minikube 安装过程需要科学上网，否则国内环境基本是无法安装和下载。

**国内环境推荐使用此方式安装：**

```bash
$ minikube start --memory='6000mb' --cpus=4 --registry-mirror=https://{改成自己的加速节点}.mirror.aliyuncs.com --image-mirror-country='cn' --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.7.3.iso
```

- `--memory='6000mb'`: 为 minikube 虚拟机 分配内存数
- `--cpus=4` :  为minikube 虚拟机分配 cpu 数
- `--registry-mirror=https://{改成自己的加速节点}.mirror.aliyuncs.com`:  docker 国内加速镜像配置
- ` --image-mirror-country='cn'`:  将缺省利用 registry.cn-hangzhou.aliyuncs.com/google_containers 作为安装Kubernetes的容器镜像仓库
-  `--image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'`:  初始化k8s集群组件镜像 默认 `k8s.gcr.io` 仓库，这里修改成 `registry.cn-hangzhou.aliyuncs.com/google_containers` 
- `--iso-url=***`: 利用阿里云的镜像地址下载相应的 .iso 文件

**安装成功后的界面:**

![image-20200429144428388](/img/in-post/image-20200429144428388.png)

至此，已完成本地 K8S 的实验环境。

关于minikube 的相关操作手册，可以阅读[Minikube官方文档](https://minikube.sigs.k8s.io/docs/start/) 

##  参考资料

[Minikube - kubernetes 本地试验环境](https://yq.aliyun.com/articles/221687)

[Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)

[Minikube官方文档](https://minikube.sigs.k8s.io/docs/start/)









