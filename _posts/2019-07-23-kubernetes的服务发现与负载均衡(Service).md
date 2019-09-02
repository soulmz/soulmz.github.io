---
layout:     post   				    
title:      kubernetes service 服务发现
subtitle:   kubernetes 服务发现
date:       2019-07-23 				
author:     soulmz 					
header-img: img/post-bg-2015.jpg 	
catalog: true 						
tags:								
    - kubernetes
---



# kubernetes的服务发现与负载均衡(Service)

示例图: 

![kubernetes-service](/img/kubernetes-service.png)



---

## Service的作用

- **服务发现:** 由于 `Kubernetes` 的调度机制，在 `kubernetes` 中，`Pod` 的 `IP` 不是固定的。如果其他程序需要访问这个 `Pod`，要怎么知道这个 `Pod` 的 `IP`呢？
- **负载均衡:** 由于 `Deployment` 管理着多个 `Pod` 的副本，如果其他程序需要访问这些 `Pod`，显然需要一个 `Proxy` 为这些 `Pod` 做负载均衡。
- **外部路由:** 如果应用程序在 `kubernetes` 外部，如何访问 `kubernetes` 内部的 `Pod` 呢？ `kubernetes` 提供了 `Service` 功能，用来解决这些问题。

## 服务发现与负载均衡

`Service` 通常会和 `Deployment` 结合在一起使用，首先通过 `Deployment` 部署应用程序，然后再使用 `Service` 为应用程序提供服务发现、负载均衡和外部路由的功能。



### Service 提供了两种服务发现的方式:

- 环境变量
- DNS

#### 一、环境变量

我们可以通过，容器内的 env 查看所有的环境变量。

首先，创建一个新的Pod

```bash
#可直接引用创建
$ cat curl-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: curl
    image: tutum/curl
    command:
      - sleep
      - "3600"
    ports:
    - containerPort: 80
$ kubectl apply -f curl-pod.yaml
```

然后进入这个Pod，查看它的环境变量。可以看到，当kubernetes创建这个Pod时，会自动注入这些环境变量: 

```bash
# kubectl exec -it curl bash
root@curl:/# env | grep NGINX
NGINX_SERVICE_PORT_80_TCP_PORT=80
NGINX_SERVICE_PORT_80_TCP_PROTO=tcp
NGINX_SERVICE_SERVICE_PORT_TCP_80_80_CHJH2=80
NGINX_SERVICE_SERVICE_HOST=192.168.255.152
NGINX_SERVICE_PORT=tcp://192.168.255.152:80
NGINX_SERVICE_PORT_80_TCP=tcp://192.168.255.152:80
NGINX_SERVICE_SERVICE_PORT=80
NGINX_SERVICE_PORT_80_TCP_ADDR=192.168.255.152
```

因此，在curl这个 Pod 中，可以通过访问这些环境变量，从而访问nginx-service。

---

#### 二、DNS解析

依然，进入 Curl Pod 里面，可以通过DNS解析，访问服务。

```bash
root@curl:/# curl http://nginx-service.default:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

>  注意:  由于nginx-service的 namespace 是**default**，因此它的 DNS 域名是nginx-service.default。
>
> 如果需要访问 别的 namespace ，只需要把 default 替换成 当前命名空间。
>
> 如果不带 namespace 默认查找的是当前命名空间下的服务名。

---

### 负载均衡

当服务通过 Service 进行服务的调用，Service 会自动将接收到的流量转发给它代理的 两个 Pod。(默认策略是轮询)



---



## 外部路由

kubernetes 集群内部需要访问集群外部服务，两种方式:

- IP + NodePort (通常情况)
- 提供 kubernetes 的外部Endpoint，使用 Service 代理。( DNS解析 )



kubernetes 集群的通讯总结:

1. 集群内部
   - PodA  访问到 PodB ，需要通过 **DNS + ClusterIP** (Service服务名)  负载均衡 (推荐方案)
   - PodA 访问有状态服务(**Eureka、zk、redis**) 需要通过 **HeadlessService** 返回具体 PodC 的所有 EndPoint 返回给 PodA 。(此方式需解决，分布式存储、开发人员本地联调等相关问题)
2. 集群内 -> 集群外
   - PodA 需要访问 MySQL 实例，通过 **IP + Port 访问**。
   - PodA 通过 **DNS + 空Service 代理 EndPoint** 连接 MySQL 实例。达到应用本身只需要通过服务名访问，无需关心IP + Port (推荐方案)
3. 集群外 -> 集群内
   - 通过外部的 **反向代理 (nginx)** 来访问集群内部。
   - 通过集群内部提供 **反向代理 (ingress)** 来访问集群内部。(推荐方案)
