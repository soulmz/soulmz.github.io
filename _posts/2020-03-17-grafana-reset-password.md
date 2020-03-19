---
layout:     post   				    
title:      Grafana 使用 grafana-cli 重置密码
subtitle:   kubernetes grafana
date:       2020-03-17		
author:     soulmz				
header-img: img/backgroup/77642582_p0.jpg	
catalog: true 						
tags:								
    - kubernetes 
    - grafana
---

查看监控集群 grafana pod 名称

```bash
[root@k8s-m50 ~]# kubectl get po
NAME                                   READY   STATUS    RESTARTS   AGE
alertmanager-main-0                    2/2     Running   4          286d
alertmanager-main-1                    2/2     Running   0          10d
alertmanager-main-2                    2/2     Running   13         289d
grafana-794b4fc4dc-brm98               1/1     Running   0          136d
kube-state-metrics-549c57574d-tzlfs    4/4     Running   0          76d
node-exporter-47hqm                    2/2     Running   0          23h
node-exporter-9nggc                    2/2     Running   5          282d
node-exporter-bwm7r                    2/2     Running   14         358d
node-exporter-fqp7z                    2/2     Running   12         358d
node-exporter-fx2rt                    2/2     Running   2          260d
node-exporter-gmb6l                    2/2     Running   3          260d
node-exporter-lgpp2                    2/2     Running   7          287d
node-exporter-nhkkh                    2/2     Running   13         358d
node-exporter-nrdhd                    2/2     Running   7          291d
node-exporter-nwtsq                    2/2     Running   0          9d
node-exporter-p27zw                    2/2     Running   8          290d
node-exporter-q4wtf                    2/2     Running   10         318d
node-exporter-t99kc                    2/2     Running   3          261d
node-exporter-xn6h6                    2/2     Running   4          293d
prometheus-adapter-66fc7797fd-rfk8r    1/1     Running   2          289d
prometheus-k8s-0                       3/3     Running   1          9d
prometheus-k8s-1                       3/3     Running   7          289d
prometheus-operator-7cb68545c6-v9msw   1/1     Running   1          10d
```

进入Grafana 容器并使用 `grafana-cli` 命令重置密码

```bash
[root@k8s-m50 ~]# kubectl exec -it grafana-794b4fc4dc-brm98 bash
bash-5.0$ grafana-cli admin reset-admin-password
INFO[03-19|03:15:08] Connecting to DB                         logger=sqlstore dbtype=sqlite3
INFO[03-19|03:15:08] Starting DB migration                    logger=migrator

Error: New password is too short

NAME:
   Grafana cli admin reset-admin-password - reset-admin-password <new password>

USAGE:
   Grafana cli admin reset-admin-password [arguments...]
bash-5.0$ grafana-cli admin reset-admin-password admin
INFO[03-19|03:15:12] Connecting to DB                         logger=sqlstore dbtype=sqlite3
INFO[03-19|03:15:12] Starting DB migration                    logger=migrator

Admin password changed successfully ✔
```



至此，完成密码重置了。


