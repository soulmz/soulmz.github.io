---
layout:     post   				    
title:      集群外的服务提供 k8s集群内使用
subtitle:   k8s exposed External Services
date:       2020-04-07		
author:     soulmz				
header-img: img/backgroup/77642582_p0.jpg	
catalog: true 						
tags:								
    - Kubernetes 
---

Kubernetes 集群外部服务，如何提供给集群内部使用？

正确姿势是: 使用 Service + EndPoint 形式，暴露给 Pod 的提供服务。

> 请注意：由于服务本身并不是K8S集群内部的Pod，因此配置过程中并不需要 **labelSector**  ，如果你加了 labelSector ，Endpoint 的IP 会被清理。

```yaml
apiVersion: v1
kind:  Endpoints
metadata:
  name: nginx-export
subsets:
  - addresses:
      - ip: "192.168.8.59"
    ports:
      - port: 80
        name: tcp-static
        protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-export
spec:
  ports:
    - port: 80
      name: tcp-static
      protocol: TCP
  type: ClusterIP
---
```



参考我早期的文章:

[kubernetes的服务发现与负载均衡(Service)](/2019/07/23/kubernetes的服务发现与负载均衡(Service))

