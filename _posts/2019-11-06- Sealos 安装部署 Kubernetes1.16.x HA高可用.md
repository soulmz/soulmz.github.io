---
layout:     post   				    
title:      kubernetes v1.16.x 部署 - by sealos 
subtitle:   kubernetes sealos lvs负载均衡 
date:       2019-11-06				
author:     soulmz 					
header-img: img/backgroup/77642582_p0.jpg	
catalog: true 						
tags:								
    - kubernetes Sealos
---



# Sealos 部署 Kubernetes1.16.x HA高可用

## sealos 官方介绍

> kubernetes高可用安装工具，一条命令，离线安装，包含所有依赖，内核负载不依赖haproxy keepalived,纯golang开发,99年证书

[Sealos - Github](https://github.com/fanux/sealos)

[离线包地址](http://store.lameleg.com/)

---

## sealos 原理

![sealos-design](/img/in-post/sealos-ha-design.png)

## 环境


| 主机           | Public - IP / Private - IP | 作用                         | 系统      | 备注                                        |
| -------------- | -------------------------- | ---------------------------- | --------- | ------------------------------------------- |
| hz-k8s-master1 | 191.168.8.57               | k8s-Master | centos7.7 / 5.x kernel |                                     |
| hz-k8s-master2 | 191.168.8.58               | k8s-Master | centos7.7 / 5.x kernel |                                             |
| hz-k8s-master3 | 191.168.8.59               | k8s-Master | centos7.7 / 5.x kernel |                                             |
| hz-k8s-node1   | 191.168.8.60               | k8s-node | centos7.7 / 5.x kernel |               |
| hz-k8s-node2   | 191.168.8.61               | k8s-node | centos7.7 / 5.x kernel |               |
| Hz-k8s-node3   | 191.168.8.62               | k8s-node | centos7.7 / 5.x kernel |               |


## 服务器配置

> 所有节点都需要设置 并以 root 用户

`sealos` 支持一键安装。但，依然并不满足我的需求。


### hosts 配置

```bash

sudo cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.57 hz-k8s-master1
192.168.8.58 hz-k8s-master2
192.168.8.59 hz-k8s-master3
192.168.8.60 hz-k8s-node1
192.168.8.61 hz-k8s-node2
192.168.8.62 hz-k8s-node3
EOF

[root@hz-k8s-master1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.57 hz-k8s-master1
192.168.8.58 hz-k8s-master2
192.168.8.59 hz-k8s-master3
192.168.8.60 hz-k8s-node1
192.168.8.61 hz-k8s-node2
192.168.8.62 hz-k8s-node3
```

### 配置 NTP 同步

```bash
yum install -y ntpdate
ntpdate -u cn.ntp.org.cn
#如果想定时执行ntpdate进行时间同步，可以通过crontab来进行：
crontab -e
#输入以下内容，每小时的第19分钟做一次时间同步：
05 * * * * /usr/sbin/ntpdate cn.ntp.org.cn >> /var/log/ntpdate.log
```



### `hz-k8s-master1` 节点 ssh 免密登录

```bash
[root@hz-k8s-master1 ~]# ssh-keygen
[root@hz-k8s-master1 ~]# ssh-copy-id root@hz-k8s-master{1..3}
[root@hz-k8s-master1 ~]# ssh-copy-id root@hz-k8s-node{1..3}

# 配置 .ssh/config
[root@hz-k8s-master1 ~]# cat .ssh/config
Host *
    ServerAliveInterval 60
    ServerAliveInterval 10
    TCPKeepAlive yes
    ControlPersist yes
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p

Host hz-k8s-master1
Hostname 192.168.8.57
User root

Host hz-k8s-master2
Hostname 192.168.8.58
User root

Host hz-k8s-master3
Hostname 192.168.8.59
User root

Host hz-k8s-node1
Hostname 192.168.8.60
User root

Host hz-k8s-node2
Hostname 192.168.8.61
User root

Host hz-k8s-node3
Hostname 192.168.8.62
User root

[root@hz-k8s-master1 ~]# chmod 600 .ssh/config
[root@hz-k8s-master1 ~]# systemctl restart sshd
```

### 关闭防火墙

```bash
yum install firewalld -y
systemctl stop firewalld.service
systemctl disable firewalld.service
```

### 关闭selinux

```bash
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
setenforce 0
```

### 安装Docker

```bash
 
 # 查看docker版本 
 yum list docker-ce --showduplicates|sort -r
 
 # 删除旧docker版本
 sudo yum -y remove docker \
                   docker-client \
                   docker-client-latest \
                   docker-common \
                   docker-latest \
                   docker-latest-logrotate \
                   docker-logrotate \
                   docker-selinux \
                   docker-engine-selinux \
                   docker-engine \
                   docker-ce \
                   docker-ce-cli \
                   docker-ce-selinux
 
 yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 # 指定安装 18.06.1
 yum -y install docker-ce-18.06.3.ce
 # 安装最新
  yum -y install docker-ce
 # docker提示
 yum install -y epel-release bash-completion && cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
 
 mkdir -p /etc/docker/

 cat > /etc/docker/daemon.json <<EOF
 {
   "registry-mirrors": ["https://hub-mirror.c.163.com", "https://docker.mirrors.ustc.edu.cn"],
   "exec-opts": ["native.cgroupdriver=systemd"],
   "storage-driver": "overlay2",
   "storage-opts": [
     "overlay2.override_kernel_check=true"
   ],
   "max-concurrent-downloads": 20,
   "log-driver": "json-file",
   "log-opts": {
     "max-size": "100m",
     "max-file": "3"
   }
 }
 EOF
 
 systemctl daemon-reload && systemctl enable docker
 # 编辑systemctl的Docker启动文件
 sed -i "13i ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT" /usr/lib/systemd/system/docker.service
 systemctl daemon-reload && systemctl start docker
```

## Sealos 部署 Kubernetes

需要先从 [离线包地址](http://store.lameleg.com/) 下载好包地址。并上传到 master1 节点。

```bash
# 安装 sealos
wget https://github.com/fanux/sealos/releases/download/v3.0.1/sealos && \
    chmod +x sealos && mv sealos /usr/bin 
# 初始化 k8s 集群 使用 1.16.2 版本
sealos init \
	  --master 192.168.8.57 \
	  --master 192.168.8.58 \
	  --master 192.168.8.59 \
	  --node 192.168.8.60 \
	  --node 192.168.8.61 \
	  --node 192.168.8.62 \
	  --pkg-url /root/kube1.16.2.tar.gz \
	  --version v1.16.2

```

安装完成后，查看下服务。

```bash
[root@hz-k8s-master1 ~]# kubectl -n kube-system get all -owide
NAME                                           READY   STATUS    RESTARTS   AGE     IP                NODE             NOMINATED NODE   READINESS GATES
pod/calico-kube-controllers-564b6667d7-qvs8m   1/1     Running   0          6h20m   100.104.193.194   hz-k8s-master1   <none>           <none>
pod/calico-node-866th                          1/1     Running   0          6h20m   192.168.8.57      hz-k8s-master1   <none>           <none>
pod/calico-node-8zf4q                          1/1     Running   0          6h20m   192.168.8.58      hz-k8s-master2   <none>           <none>
pod/calico-node-lxmdx                          1/1     Running   0          6h18m   192.168.8.62      hz-k8s-node3     <none>           <none>
pod/calico-node-mxtpq                          1/1     Running   0          6h18m   192.168.8.61      hz-k8s-node2     <none>           <none>
pod/calico-node-p7sg8                          1/1     Running   0          6h18m   192.168.8.60      hz-k8s-node1     <none>           <none>
pod/calico-node-vq6tg                          1/1     Running   0          6h19m   192.168.8.59      hz-k8s-master3   <none>           <none>
pod/coredns-5644d7b6d9-9mc4w                   1/1     Running   0          6h20m   100.104.193.195   hz-k8s-master1   <none>           <none>
pod/coredns-5644d7b6d9-mn765                   1/1     Running   0          6h20m   100.104.193.193   hz-k8s-master1   <none>           <none>
pod/etcd-hz-k8s-master1                        1/1     Running   0          6h18m   192.168.8.57      hz-k8s-master1   <none>           <none>
pod/etcd-hz-k8s-master2                        1/1     Running   0          6h20m   192.168.8.58      hz-k8s-master2   <none>           <none>
pod/etcd-hz-k8s-master3                        1/1     Running   0          6h19m   192.168.8.59      hz-k8s-master3   <none>           <none>
pod/kube-apiserver-hz-k8s-master1              1/1     Running   0          6h19m   192.168.8.57      hz-k8s-master1   <none>           <none>
pod/kube-apiserver-hz-k8s-master2              1/1     Running   0          6h20m   192.168.8.58      hz-k8s-master2   <none>           <none>
pod/kube-apiserver-hz-k8s-master3              1/1     Running   0          6h19m   192.168.8.59      hz-k8s-master3   <none>           <none>
pod/kube-controller-manager-hz-k8s-master1     1/1     Running   1          6h19m   192.168.8.57      hz-k8s-master1   <none>           <none>
pod/kube-controller-manager-hz-k8s-master2     1/1     Running   0          6h20m   192.168.8.58      hz-k8s-master2   <none>           <none>
pod/kube-controller-manager-hz-k8s-master3     1/1     Running   0          6h19m   192.168.8.59      hz-k8s-master3   <none>           <none>
pod/kube-proxy-hjn7f                           1/1     Running   0          6h18m   192.168.8.60      hz-k8s-node1     <none>           <none>
pod/kube-proxy-j7k42                           1/1     Running   0          6h19m   192.168.8.59      hz-k8s-master3   <none>           <none>
pod/kube-proxy-ndcjw                           1/1     Running   0          6h20m   192.168.8.58      hz-k8s-master2   <none>           <none>
pod/kube-proxy-qfkcn                           1/1     Running   0          6h18m   192.168.8.61      hz-k8s-node2     <none>           <none>
pod/kube-proxy-t9dlc                           1/1     Running   0          6h20m   192.168.8.57      hz-k8s-master1   <none>           <none>
pod/kube-proxy-wvctj                           1/1     Running   0          6h18m   192.168.8.62      hz-k8s-node3     <none>           <none>
pod/kube-scheduler-hz-k8s-master1              1/1     Running   1          6h19m   192.168.8.57      hz-k8s-master1   <none>           <none>
pod/kube-scheduler-hz-k8s-master2              1/1     Running   0          6h20m   192.168.8.58      hz-k8s-master2   <none>           <none>
pod/kube-scheduler-hz-k8s-master3              1/1     Running   0          6h19m   192.168.8.59      hz-k8s-master3   <none>           <none>
pod/kube-sealyun-lvscare-hz-k8s-node1          1/1     Running   0          6h18m   192.168.8.60      hz-k8s-node1     <none>           <none>
pod/kube-sealyun-lvscare-hz-k8s-node2          1/1     Running   0          6h18m   192.168.8.61      hz-k8s-node2     <none>           <none>
pod/kube-sealyun-lvscare-hz-k8s-node3          1/1     Running   0          6h18m   192.168.8.62      hz-k8s-node3     <none>           <none>

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE     SELECTOR
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   6h20m   k8s-app=kube-dns

NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE     CONTAINERS    IMAGES                          SELECTOR
daemonset.apps/calico-node   6         6         6       6            6           beta.kubernetes.io/os=linux   6h20m   calico-node   calico/node:v3.8.2              k8s-app=calico-node
daemonset.apps/kube-proxy    6         6         6       6            6           beta.kubernetes.io/os=linux   6h20m   kube-proxy    k8s.gcr.io/kube-proxy:v1.16.2   k8s-app=kube-proxy

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS                IMAGES                           SELECTOR
deployment.apps/calico-kube-controllers   1/1     1            1           6h20m   calico-kube-controllers   calico/kube-controllers:v3.8.2   k8s-app=calico-kube-controllers
deployment.apps/coredns                   2/2     2            2           6h20m   coredns                   k8s.gcr.io/coredns:1.6.2         k8s-app=kube-dns

NAME                                                 DESIRED   CURRENT   READY   AGE     CONTAINERS                IMAGES                           SELECTOR
replicaset.apps/calico-kube-controllers-564b6667d7   1         1         1       6h20m   calico-kube-controllers   calico/kube-controllers:v3.8.2   k8s-app=calico-kube-controllers,pod-template-hash=564b6667d7
replicaset.apps/coredns-5644d7b6d9                   2         2         2       6h20m   coredns                   k8s.gcr.io/coredns:1.6.2         k8s-app=kube-dns,pod-template-hash=5644d7b6d9
```

这里可以看到 多了三个 `static pod` ，此三个 `static pod` 部署在 node 节点，用`IPVS`来维护

- master 节点的 `apiserver.cluster.local` hosts 记录都会加一条指向自己的IP地址

```bash
[root@hz-k8s-master1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.57 hz-k8s-master1
192.168.8.58 hz-k8s-master2
192.168.8.59 hz-k8s-master3
192.168.8.60 hz-k8s-node1
192.168.8.61 hz-k8s-node2
192.168.8.62 hz-k8s-node3
192.168.8.57 apiserver.cluster.local
```

- node 节点 `apiserver.cluster.local` hosts 记录会指向`10.103.97.2` IP地址，此地址将由 IPVS 来维护管理。

```bash
[root@hz-k8s-node1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.57 hz-k8s-master1
192.168.8.58 hz-k8s-master2
192.168.8.59 hz-k8s-master3
192.168.8.60 hz-k8s-node1
192.168.8.61 hz-k8s-node2
192.168.8.62 hz-k8s-node3
10.103.97.2 apiserver.cluster.local
```

所有的node节点，都会创建一个`static pod` `kube-sealyun-lvscare-{hostsname}` 进行 IPVS 维护。

## 测试k8s集群正常

- 首先验证kube-apiserver, kube-controller-manager, kube-scheduler, pod network 是否正常：

```shell
[root@master1 ~]# kubectl create deployment nginx --image=nginx:alpine
deployment.apps/nginx created
[root@master1 ~]# kubectl get pods -l app=nginx -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE
nginx-65d5c4f7cc-fgj2f   1/1     Running   0          52s   10.244.6.2   master1   <none>
nginx-65d5c4f7cc-lq585   1/1     Running   0          23s   10.244.8.2   master2   <none>
[root@master1 ~]# kubectl scale deployment nginx --replicas=2
deployment.extensions/nginx scaled
```

- 再验证一下kube-proxy是否正常：

```shell
[root@master1 ~]# kubectl expose deployment nginx --port=80 --type=NodePort
service/nginx exposed
# 查看暴露的服务端口
[root@master1 ~]# kubectl get services nginx
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.98.173.217   <none>        80:30879/TCP   9s


# 可以通过任意 NodeIP:Port 在集群外部访问这个服务，本示例中部署的2台集群IP分别是10.10.110.101和10.10.110.102、10.10.110.103、10.10.110.104
curl http://10.10.110.101:30879
curl http://10.10.110.102:30879
curl http://10.10.110.103:30879

```

- 最后验证一下dns, pod network是否正常：

```shell
[root@master1 ~]# kubectl run -it curl --image=radial/busyboxplus:curl
kubectl run --generator=deployment/apps.v1beta1 is DEPRECATED and will be removed in a future version. Use kubectl create instead.
If you don't see a command prompt, try pressing enter.
# 解析nginx(pod)的
[ root@curl-5cc7b478b6-ftmcl:/ ]$ nslookup nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.98.173.217 nginx.default.svc.cluster.local

#测试IP
[ root@curl-5cc7b478b6-ftmcl:/ ]$ curl http://10.98.173.217
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

# 测试dns
[ root@curl-5cc7b478b6-ftmcl:/ ]$ curl http://nginx.default.svc.cluster.local
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

验证通过，集群搭建成功，接下来我们就可以参考官方文档来部署其他服务，愉快的玩耍了。


## Sealos 升级 kubernetes

如果使用 `sealos` 的话，依然必须使用 它的离线包 进行升级。因 `kubeadm` 二进制文件有修改过证书代码。

同时，需要把离线包，分发到所有节点上并解压。覆盖 `kubectl kubeadm kubelet ` 等二进制文件

```bash
## 下载 kube1.16.2.tar 离线包并解压
tar -zxvf kube1.16.2.tar
## 拷贝 kubeadm 和 kubectl 
cp kube/bin/{kubeadm,kubectl} /usr/bin/
## 拷贝 docker Image 镜像
docker load -i kube/images/images.tar
## 看下升级计划
kubeadm upgrade plan
## 尝试升级 看看是否成功
kubeadm upgrade apply v1.16.2 --dry-run
## 升级
kubeadm upgrade apply v1.16.2

## 注意 这里需要 中断 kubelet 服务，并更新 kubelet ，再启动。
systemctl stop kubelet
cp kube/bin/kubelet /usr/bin/
systemctl start kubelet

## 升级worker 节点
....

```

我看到作者，已经开始开发 sealos upgrape 方式了，相信在不久将来就可以用上了。😁

## 卸载 kubernetes 集群

这里清除的同时，会把离线包也一起清除。因此，先备份一份。

```bash
# 备份
cp -r kube1.16.2.tar.gz kube1.16.2.tar.gz_bak
# 清除集群节点
sealos clean \
    --master 192.168.8.57 \
    --master 192.168.8.58 \
    --master 192.168.8.59 \
    --node 192.168.8.60 \
    --node 192.168.8.61 \
    --node 192.168.8.62
```

这里的操作并不合理，如果节点过多，那么就需要一直增加。希望作者后续完善一下。

## 总结

`sealos` 确实做到了开箱即用。

只需要，创建好几台虚拟机，下载好离线包并执行 `sealos init` 命令即可完成一个k8s集群，不管是 多主多从 还是 一主多从。(官方有视频)

并且这里采用的是 `lvs` 来进行负载，可以说是非常轻巧的方式，并不需要使用 haproxy 、keepalived 来进行负载。减少了复杂度，这点值得肯定。(可以直接部署在公有云，如果采用 keepalived 则不行了。)

这里需要注意两点：

- 如果 内核 一开始使用 3.10 后续升级成 5.x 会导致 k8s 的服务不可用。
- sealos 离线包脚本，默认使用的 cgroupfs ，如果手动将 docker 和 kubelet 改成 systemd 同样会出现 k8s 服务不可用。

本人并未深入去了解相关问题，有哪位朋友知道解决方案，希望可以告知下哈。