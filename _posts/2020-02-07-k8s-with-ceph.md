---
layout: default
title: "配置k8s使用ceph块存储服务"
date: 2020-02-07 21:38:00
published: 2020-02-07 21:38:00
comments: true
categories: CICD
tags: [cicd]
author: ykfq
---

<p>
上一篇介绍了如何自己搭建一套 ceph 分布式存储集群，本篇介绍如何将 ceph 的块存储配置给 kubernetes 作为动态存储服务。
</p>

<!--more-->


## 前言

我们使用 ceph 提供两种类型的存储服务：
- Ceph fs: 提供给 cvm 用作共享存储
- Ceph rbd: 提供给 k8s 用作动态申请共享存储

本文详细介绍第二部分。

要在 k8s 集群中使用 `ceph rbd` 服务，k8s 集群节点需要安装 ceph 以提供 rbd 命令支持，分以下场景：
- 以二进制方式部署的 k8s 集群, 在集群安装 ceph 服务即可
- 以容器化部署的 k8s 集群，需要在 `kube-control-manager` pod 中集成 rbd 命令支持，可选方案有
  - 在 k8s 集群节点安装 ceph 服务，并将 rbd 以 volume 方式挂载到 pod 内部
  - 以 `kube-control-manager` 镜像启动一个容器并安装 ceph 服务，commit 成新镜像并重新运行pod
  - 使用开源项目 `rbd-provisioner` 在 k8s 内部署独立的服务来提供 rbd 命令支持

由于我们当前使用的是容器化部署，因此需要使用场景二，而场景二的前两种方案需要修改已有pod，具有入侵性，不推荐使用，最终选择使用 `rbd-provisioner` 方案。

## 部署 rbd-provisioner

- 创建 RBAC 资源

在 k8s 中运行 rbd-provisioner, 应该设置 rbac 权限。以下是运行 rbd-provisioner 需要创建的资源：

1.) ServiceAccout  
2.) Role  
3.) RoleBinding  
4.) ClusterRole  
5.) ClusterRoleBinding  


将其纳入一个 yaml 文件，方便一键创建： 

`vim rbd-provisioner-rbac.yaml`

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-provisioner
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rbd-provisioner
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rbd-provisioner
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rbd-provisioner
subjects:
- kind: ServiceAccount
  name: rbd-provisioner
  namespace: kube-system

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["kube-dns","coredns"]
    verbs: ["list", "get"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
roleRef:
  kind: ClusterRole
  name: rbd-provisioner
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: rbd-provisioner
    namespace: kube-system
```

ClusterRole 和 ClusterRoleBinding 无需指定 namespace，Cluster 级别权限会直接创建到 kube-system.



- 创建 Deployment

使用自有 docker 镜像仓库：

`vim rbd-provisioner-deployment.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: rbd-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      containers:
      - name: rbd-provisioner
        image: "hub.itsfun.tk/external_storage/rbd-provisioner:latest"
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/rbd
      serviceAccount: rbd-provisioner
```


## 如何申请共享存储

上面我们只是创建好了 `rbd-provisioner` , `kube-control-manager` 可以使用它通过 rbd 命令来创建 storageclass 了。

要让 pod 能够正常使用 ceph rbd 提供的共享存储服务，需要创建如下资源：
- Secret
- StorageClass
- PersistentVolumeClaim
- Pod/Deployment/StafulSet/DaemonSet

前两种资源只需在首次申请时创建，以后就可以直接使用了。 pvc 是创建任意需要共享存储的服务时都要创建的，并会绑定到对应的服务去使用。

- 创建 Secret
`vim ceph-rbd-secret.yaml`  
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: kube-system
data:
  key: DOPET2p6RmRPNFUrSkJBQUdDK25NelFtaDRkVWlGNkNBQTVLbGc9PQ==
type: kubernetes.io/rbd

---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-kube-secret
  namespace: kube-system
data:
  key: DOPCcHZ6WmRhRzdoTVJBQVk1dERXL3NxcS9Jc3hvVUsyWGN6dnc9PQ==
type: kubernetes.io/rbd
```

这里创建了2个 secret。两个 Secret 的 value 都是 ceph 集群中对应用户密钥的 base64 编码。需要在 ceph 集群节点上通过下面的方式获取：  
```bash
[root@ceph01:~]# ceph auth get-key client.admin | base64
DOPET2p6RmRPNFUrSkJBQUdDK25NelFtaDRkVWlGNkNBQTVLbGc9PQ==
[root@ceph01:~]#
[root@ceph01:~]# ceph auth get-key client.kube | base64
DOPCcHZ6WmRhRzdoTVJBQVk1dERXL3NxcS9Jc3hvVUsyWGN6dnc9PQ==
```


- 创建 StorageClass  

`vim ceph-rbd-sc.yaml`
```yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: ceph-kube-sc
  namespace: kube-system
  annotations:
     storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: ceph.com/rbd
parameters:
  monitors: 10.200.200.20:6789,10.200.200.21:6789,10.200.200.22:6789
  adminId: admin
  adminSecretName: ceph-admin-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-kube-secret
  userSecretNamespace: kube-system
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

注意：`provisioner` 需要指定为 `ceph.com/rbd` 。

`monitors`: ceph 集群 monitor 节点，多节点以逗号分隔  
`adminId/userId`: ceph 中对应的用户名  
`SecretName`: 上面创建的 Secret   
`SecretNamespace`: 上面创建的 Secret 所在的 namespace  

其它选项默认即可。


- 创建 PVC

`vim ceph-rbd-pvc.yml`
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-pod-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```


- 创建 pod

`vim ceph-pod-test.yaml`  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod-test01
  namespace: default
spec:
  containers:
  - name: ceph-busybox
    image: busybox
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-vol1
      mountPath: /usr/share/busybox
      readOnly: false
  volumes:
  - name: ceph-vol1
    persistentVolumeClaim:
      claimName: ceph-pod-test-pvc
```

注意：`Pod/Deployment/StafulSet/DaemonSet` 需要和它所申请的 pvc 在相同 namespace.

最后，批量创建上述服务：
```bash 
kubectl apply -f *.yaml
```

最后，确认下资源申请是否生效：

pod 正常 Running 没有发生 restart 并且 pod 中已经有 pvc 相关配置：
```bash
root@jn.set01.app.k8s.master01:~# kubectl -n test get pod ceph-pod-test01 -o yaml | grep -A3 volumes:
  volumes:
  - name: ceph-vol1
    persistentVolumeClaim:
      claimName: ceph-pod-test-pvc
```



## StorageClass 的维护和使用

当我们只有一个 StorageClass 时，它将是默认的 StorageClass，新创建的 pvc 在没有指定 StorageClass 的情况下，都会默认从这里申请资源。

如果我们有不同类型的存储设备，比如 普通硬盘或者 SSD，我就可以以此创建不同的 StorageClass，供不同用途使用。

- pvc 指定 StorageClass  

用 storageClassName 来指定：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
```


- 切换默认的 StorageClass  

```bash
kubectl get storageclass
NAME                 PROVISIONER               AGE
standard (default)   kubernetes.io/gce-pd      1d
gold                 kubernetes.io/gce-pd      1d
```

可以看到当前 standard 为默认 sc。  
现在我们将其设置非默认值并将另一个改成默认 sc：
```bash
kubectl patch storageclass <your-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass <your-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```



## 参考文档

- [RBD Volume Provisioner for Kubernetes 1.5+](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd)
- [使用rbd-provisioner提供rbd持久化存储](https://jimmysong.io/kubernetes-handbook/practice/rbd-provisioner.html)
- [Change the default StorageClass](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class)
- [PersistentVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)