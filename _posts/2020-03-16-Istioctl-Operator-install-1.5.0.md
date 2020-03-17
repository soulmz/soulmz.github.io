---
layout:     post   				    
title:      istioctl Opeartor 部署 istio-1.5.0
subtitle:   kubernetes istio 1.5.0
date:       2020-03-16				
author:     soulmz 					
header-img: img/backgroup/77642582_p0.jpg	
catalog: true 						
tags:								
    - kubernetes 
    - istio
---



# istioctl Opeartor 部署 istio-1.5.0

### 简介

### 服务网格

>  术语服务网格（Service Mesh）用于描述微服务之间的网络，以及通过此网络进行的服务之间的交互。随着服务数量和复杂度的增加，服务网格将变的难以理解和管理。
>
> 对服务网格的需求包括：服务发现、负载均衡、故障恢复、指标和监控，以及A/B测试、金丝雀发布、限速、访问控制、端对端身份验证等。

### Istio 是什么

>使用云平台给DevOps团队带来了额外的约束，为了Portability开发人员通常需要使用微服务架构，运维人员则需要管理非常多数量的服务。Istio能够连接、保护、控制、观察这些微服务。
>
>Istio是运行于分布式应用程序之上的透明（无代码入侵）服务网格，它同时也是一个平台，提供集成到其它日志、监控、策略系统的接口。
>
>Istio的实现原理是，为每个微服务部署一个Sidecar，代理微服务之间的所有网络通信。在此基础上你可以通过Istio的控制平面实现：
>
>1. 针对HTTP、gRPC、WebSocket、TCP流量的负载均衡
>2. 细粒度的流量控制行为，包括路由、重试、故障转移、故障注入（fault injection）
>3. 可拔插的策略层+配置API，实现访问控制、限速、配额
>4. 自动收集指标、日志，跟踪集群内所有流量，包括Ingress/Egress
>5. 基于强身份认证和授权来保护服务之间的通信

---

Istio 1.5.0 架构

> 1.5.0 版本开始，该版本最大的变化是将控制平面的所有组件合成了一个单体结构叫: `Istiod`

![640](/img/640.jpeg)

Istio 1.0 - istio1.4.x 的架构

> 从架构图可以看出，在 Istio 1.5 中，饱受诟病的 `Mixer` 终于被废弃了，新版本的 HTTP 遥测默认基于 in-proxy Stats filter，同时可使用 **WebAssembly**[1] 开发 `in-proxy` 扩展。

![istio-arch](/img/istio-arch-4437654.png)

## 部署

> 在部署 Istio 之前，首先需要确保 Kubernetes 集群（kubernetes 版本建议在 `1.14` 以上）已部署并配置好本地的 kubectl 客户端。

之前的部署方式，都是采用 `Helm Template` 方式生成模板，使用 `kubectl apply` 方式部署。

官方文档在 `1.4.x` 版本之后，开始推荐使用 `istioctl` + `Operator` 方式部署。

### 下载 最新的 Istio(本文最新版本 1.5.0)

```shell
curl -L https://istio.io/downloadIstio | sh -
```

下载完后，进去 `istio-1.5.0` 目录

```bash
➜  istio-1.5.0 git:(master) ✗ tree -L 1 ./
./
├── LICENSE
├── README.md
├── bin
├── install
├── manifest.yaml
├── samples
└── tools

4 directories, 4 files
```

### 配置 `Istioctl`

```bash
➜  istio-1.5.0 git:(master) ✗ cp bin/istioctl /usr/local/bin
```

### `Istioctl` 补全提示

将 istio 安装包内 tools 目录下的 istioctl.bash 文件拷贝到用户根目录下

``` bash
cp tools/istioctl.bash ~
```

 编辑 ~/.bash_profile 文件，在文件末尾添加如下内容：(zsh 的 改成 ~/.zshrc )

```bash
source ~/istioctl.bash
```

添加完毕后，加载配置使配置生效：

```bash
source ~/.bash_profile 
# zsh 的这样使用
source ~/.zshrc
```

输入 istioctl 然后按两次 tab 键，发现增强自动补全功能已经生效：

![image-20200316184810273](/img/image-20200316184810273.png)

### Istio CNI Plugin

当前实现将用户 pod 流量转发到 proxy 的默认方式是使用 privileged 权限的 `istio-init` 这个 init container 来做的（运行脚本写入 iptables），需要用到 `NET_ADMIN` capabilities。对 linux capabilities 不了解的同学可以参考我的 [Linux capabilities 系列](http://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484610&idx=1&sn=0f75f48b1651f03163bef421280c25f8&chksm=fbee440fcc99cd19c786acd3de00fee7914664171395013bb3fb4e1dfb1a84f618fc1a9b042e&scene=21#wechat_redirect)。

Istio CNI 插件的主要设计目标是消除这个 privileged 权限的 init container，换成利用 Kubernetes CNI 机制来实现相同功能的替代方案。具体的原理就是在 Kubernetes CNI 插件链末尾加上 Istio 的处理逻辑，在创建和销毁 pod 的这些 hook 点来针对 istio 的 pod 做网络配置：写入 iptables，让该 pod 所在的 network namespace 的网络流量转发到 proxy 进程。

详细内容请参考**官方文档**[6]。

使用 Istio CNI 插件来创建 sidecar iptables 规则肯定是未来的主流方式，不如我们现在就尝试使用这种方法。

### Kubernetes 关键插件（Critical Add-On Pods）

众所周知，Kubernetes 的核心组件都运行在 master 节点上，然而还有一些附加组件对整个集群来说也很关键，例如 DNS 和 metrics-server，这些被称为**关键插件**。一旦关键插件无法正常工作，整个集群就有可能会无法正常工作，所以 Kubernetes 通过优先级（PriorityClass）来保证关键插件的正常调度和运行。要想让某个应用变成 Kubernetes 的**关键插件**，只需要其 `priorityClassName` 设为 `system-cluster-critical` 或 `system-node-critical`，其中 `system-node-critical` 优先级最高。

> 注意：关键插件只能运行在 `kube-system` namespace 中！

### 初始化并安装 Operator 

```bash
istioctl operator init
```

或者

```bash
kubectl apply -f https://istio.io/operator.yaml
```

此时，`istioctl` 会创建一个 `istio-operator` 命名空间与 `istio-opeartor deployment`

```bash
➜  istio-1.5.0 git:(master) ✗ kubectl get ns
NAME                     STATUS   AGE
cattle-system            Active   3d15h
default                  Active   131d
gitlab-runner            Active   118d
istio-operator           Active   3d16h
istio-system             Active   3d16h
kube-node-lease          Active   131d
kube-public              Active   131d
kube-system              Active   131d
sb-jaeger-tracing-demo   Active   107d
➜  istio-1.5.0 git:(master) ✗ kubectl -n istio-operator get po -owide
NAME                              READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
istio-operator-7c69599466-4wrlp   1/1     Running   1          18h   100.83.184.194   hz-k8s-node2   <none>           <none>
```

### 创建 `istio-system` 和 部署 `istio 控制面板`

> 官方推荐的 default 配置 , 默认使用 demo 配置。 [Istio-配置内容](https://istio.io/docs/setup/additional-setup/config-profiles/) 
>
> 各配置差异如下: 

![image-20200317101947126](/img/image-20200317101947126.png)

>  这里我们采用 default 版本部署
>
> 创建一个 default 文件，并使用 default 配置

```bash
➜  istio-1.5.0 git:(master) ✗ kubectl create ns istio-system
➜  istio-1.5.0 git:(master) ✗ vim default.yaml
➜  istio-1.5.0 git:(master) ✗ cat default.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: default
➜  istio-1.5.0 git:(master) ✗ kubectl apply -f default.yaml
istiooperator.install.istio.io/example-istiocontrolplane created
```

> `istio-operator` 负责动态 `创建、修改` istio 组件。可以采用 查看创建日志过程
>
> ```bash
> kubectl logs -f -n istio-operator $(kubectl get pods -n istio-operator -lname=istio-operator -o jsonpath='{.items[0].metadata.name}')
> ```

分屏查看执行效果:

<img src="/img/image-20200317103503042.png" alt="image-20200317103503042" style="zoom:50%;" />

查看部署效果

```bash
➜  istio-1.5.0 git:(master) ✗ kubectl get all -n istio-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/istio-ingressgateway-78f757846c-xbbnv   1/1     Running   0          5h10m
pod/istiod-65c5b8df9d-7cc4g                 1/1     Running   0          5h10m
pod/prometheus-6fd77b7876-xpwlh             2/2     Running   0          5h10m

NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                      AGE
service/istio-ingressgateway   LoadBalancer   10.102.37.152    <pending>     15020:30390/TCP,80:31092/TCP,443:32081/TCP,15029:30966/TCP,15030:31200/TCP,15031:30480/TCP,15032:31115/TCP,15443:31899/TCP   5h10m
service/istio-pilot            ClusterIP      10.103.180.252   <none>        15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP                                                                     5h10m
service/istiod                 ClusterIP      10.108.160.250   <none>        15012/TCP,443/TCP                                                                                                            5h10m
service/prometheus             ClusterIP      10.98.39.69      <none>        9090/TCP                                                                                                                     5h10m

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingressgateway   1/1     1            1           5h10m
deployment.apps/istiod                 1/1     1            1           5h10m
deployment.apps/prometheus             1/1     1            1           5h10m

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/istio-ingressgateway-78f757846c   1         1         1       5h10m
replicaset.apps/istiod-65c5b8df9d                 1         1         1       5h10m
replicaset.apps/prometheus-6fd77b7876             1         1         1       5h10m

NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   <unknown>/80%   1         5         1          5h10m
horizontalpodautoscaler.autoscaling/istiod                 Deployment/istiod                 <unknown>/80%   1         5         1          5h10m
```

 修改 `default.yaml` 配置，修改如下：

```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: default
  components:
    base:
      enabled: true
    cni:
      enabled: true
      namespace: kube-system
    # Istio Gateway feature
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
        k8s:
          service:
            type: ClusterIP #change to NodePort, ClusterIP or LoadBalancer if need be
          replicaCount: 2
          tolerations:
            - key: node-role.kubernetes.io/master
              operator: Exists
              effect: NoSchedule
          nodeSelector:
            istio-gateway-ingress: enable
          resources:
            limits:
              cpu: 2000m
              memory: 2Gi
          strategy:
            type: RollingUpdate
            rollingUpdate:
              maxSurge: 0%
              maxUnavailable: 100%
          # 反亲和 配置
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchExpressions:
                      - key: app
                        operator: In
                        values:
                          - istio-ingressgateway
                  topologyKey: kubernetes.io/hostname
            hostNetwork: true
            dnsPolicy: ClusterFirstWithHostNet
    egressGateways:
      - name: istio-egressgateway
        enabled: true
        autoscaleEnabled: false
        k8s:
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 2000m
              memory: 1024Mi
          strategy:
            rollingUpdate:
              maxSurge: 1
              maxUnavailable: 0
  values:
  # 部署 istio-cni 插件
    cni:
      # 排除部分命名空间。
      excludeNamespaces:
        - istio-system
        - kube-system
        - monitoring
      logLevel: info
    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
      istio-ingressgateway:
        autoscaleEnabled: false
        debug: info
  # 组件
  addonComponents:
    kiali:
      enabled: true
    grafana:
      enabled: true
    tracing:
      enabled: true
```

- istio-ingressgateway 的 Service 默认类型为 `LoadBalancer`，需将其改为 `ClusterIP`。
- 为防止集群资源紧张，更新配置后无法创建新的 `Pod`，需将滚动更新策略改为先删除旧的，再创建新的。
- 将 istio-ingressgateway 调度到指定节点。
- 默认情况下除了 `istio-system` `namespace` 之外，istio cni 插件会监视其他所有 namespace 中的 Pod，然而这并不能满足我们的需求，更严谨的做法是让 istio CNI 插件至少忽略 `kube-system`、`istio-system` 这两个 namespace，如果你还有其他的特殊的 namespace，也应该加上，例如 `monitoring`。

部署完成后，查看各组件状态：

```bash
➜  istio-1.5.0 git:(master) ✗ kubectl apply -f default.yaml
istiooperator.install.istio.io/example-istiocontrolplane configured
➜  istio-1.5.0 git:(master) ✗ kubectl get all -n istio-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/grafana-5cc7f86765-h9g8g                1/1     Running   0          15m
pod/istio-egressgateway-7f8d56799-dqdhh     1/1     Running   0          15m
pod/istio-ingressgateway-75df755789-72wjw   1/1     Running   0          15m
pod/istio-ingressgateway-75df755789-kvqjr   1/1     Running   0          15m
pod/istio-tracing-8584b4d7f9-zxglc          1/1     Running   0          15m
pod/istiod-5cd6fbf5fc-vfhv8                 1/1     Running   0          15m
pod/kiali-76f556db6d-f8l88                  1/1     Running   0          15m
pod/prometheus-b8744885c-4lgh9              2/2     Running   0          15m

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
service/grafana                     ClusterIP   10.102.178.186   <none>        3000/TCP                                                                     15m
service/istio-egressgateway         ClusterIP   10.98.180.223    <none>        80/TCP,443/TCP,15443/TCP                                                     15m
service/istio-ingressgateway        ClusterIP   10.102.37.152    <none>        15020/TCP,80/TCP,443/TCP,15029/TCP,15030/TCP,15031/TCP,15032/TCP,15443/TCP   5h52m
service/istio-pilot                 ClusterIP   10.103.180.252   <none>        15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP                     5h52m
service/istiod                      ClusterIP   10.108.160.250   <none>        15012/TCP,443/TCP                                                            5h52m
service/jaeger-agent                ClusterIP   None             <none>        5775/UDP,6831/UDP,6832/UDP                                                   15m
service/jaeger-collector            ClusterIP   10.102.239.138   <none>        14267/TCP,14268/TCP,14250/TCP                                                15m
service/jaeger-collector-headless   ClusterIP   None             <none>        14250/TCP                                                                    15m
service/jaeger-query                ClusterIP   10.103.196.164   <none>        16686/TCP                                                                    15m
service/kiali                       ClusterIP   10.99.136.145    <none>        20001/TCP                                                                    15m
service/prometheus                  ClusterIP   10.98.39.69      <none>        9090/TCP                                                                     5h52m
service/tracing                     ClusterIP   10.99.107.47     <none>        80/TCP                                                                       15m
service/zipkin                      ClusterIP   10.109.139.203   <none>        9411/TCP                                                                     15m

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana                1/1     1            1           15m
deployment.apps/istio-egressgateway    1/1     1            1           15m
deployment.apps/istio-ingressgateway   2/2     2            2           5h52m
deployment.apps/istio-tracing          1/1     1            1           15m
deployment.apps/istiod                 1/1     1            1           5h52m
deployment.apps/kiali                  1/1     1            1           15m
deployment.apps/prometheus             1/1     1            1           5h52m

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-5cc7f86765                1         1         1       15m
replicaset.apps/istio-egressgateway-7f8d56799     1         1         1       15m
replicaset.apps/istio-ingressgateway-75df755789   2         2         2       15m
replicaset.apps/istio-ingressgateway-78f757846c   0         0         0       5h52m
replicaset.apps/istio-tracing-8584b4d7f9          1         1         1       15m
replicaset.apps/istiod-5cd6fbf5fc                 1         1         1       15m
replicaset.apps/istiod-65c5b8df9d                 0         0         0       5h52m
replicaset.apps/kiali-76f556db6d                  1         1         1       15m
replicaset.apps/prometheus-6fd77b7876             0         0         0       5h52m
replicaset.apps/prometheus-b8744885c              1         1         1       15m

NAME                                         REFERENCE           TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istiod   Deployment/istiod   <unknown>/80%   1         5         1          5h52m
```

`Istio-cni` 部署的时候，会产生短暂的`网络中断` ，检查节点配置是否正常：

```bash
[root@hz-k8s-master1 ~]# cat /etc/cni/net.d/10-calico.conflist
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "hz-k8s-master1",
      "mtu": 1440,
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {
        "portMappings": true
      }
    },
    # 此处
    {
      "cniVersion": "0.3.1",
      "name": "istio-cni",
      "type": "istio-cni",
      "log_level": "info",
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/ZZZ-istio-cni-kubeconfig",
        "cni_bin_dir": "/opt/cni/bin",
        "exclude_namespaces": [
          "istio-system",
          "kube-system",
          "monitoring"
        ]
      }
    }
  ]
}
```

`Istio-CNI` 部署采用 `daemonSet` (守护进程)方式：

```bash
➜  istio-1.5.0 git:(master) ✗ kubectl -n kube-system get pod -l k8s-app=istio-cni-node -owide
NAME                   READY   STATUS    RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
istio-cni-node-2q8xw   2/2     Running   0          4m9s    192.168.8.57   hz-k8s-master1   <none>           <none>
istio-cni-node-58qt2   2/2     Running   0          4m9s    192.168.8.62   hz-k8s-node3     <none>           <none>
istio-cni-node-7k8jv   2/2     Running   0          4m10s   192.168.8.63   hz-k8s-ci        <none>           <none>
istio-cni-node-gg4jz   2/2     Running   0          4m10s   192.168.8.59   hz-k8s-master3   <none>           <none>
istio-cni-node-jc2vz   2/2     Running   0          4m10s   192.168.8.61   hz-k8s-node2     <none>           <none>
istio-cni-node-ljjkf   2/2     Running   0          4m9s    192.168.8.58   hz-k8s-master2   <none>           <none>
istio-cni-node-zqkwj   2/2     Running   0          4m2s    192.168.8.60   hz-k8s-node1     <none>           <none>
```

部署至此已完成。



### 清除 Istio服务 与 istio-opeartor

```bash
$ kubectl delete -f default.yaml 
$ kubectl delete ns istio-operator --grace-period=0 --force
$ kubectl delete ns istio-system --grace-period=0 --force
```

## 参考资料

[https://istio.io/zh/docs/setup/upgrade/istioctl-upgrade/](https://istio.io/zh/docs/setup/upgrade/istioctl-upgrade/)

[九析带你轻松完爆服务网格 - istio 安装](https://blog.51cto.com/14625168/2474224)

[让我们来看看回到单体的 Istio 到底该怎么部署](https://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484837&idx=1&sn=a299f66fd43b3a59a8e430a0397acde4&chksm=fbee4568cc99cc7ee91bf35d87477e5f40ea0292f7161b16c1ba2b260be5a037d3380131f335&mpshare=1&scene=23&srcid=&sharer_sharetime=1583485220334&sharer_shareid=c649a7ff9c6c32ae94e59c9ebaeee9f1#rd)

