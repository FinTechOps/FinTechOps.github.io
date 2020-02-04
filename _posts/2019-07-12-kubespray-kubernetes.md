---
layout: default
title: "使用kubespray部署高可用kubernetes集群"
date: 2020-02-03 15:20:00
published: 2019-07-12 10:01:00
comments: true
categories: KUBERNETES
tags: [kubernetes]
author: ykfq
---

<p>
本文记录使用 kubespray v2.10.4 部署 kubernetes: 1.13.8 过程，网络组件使用 flannel。
</p>

<!--more-->


## 背景

想要快速安装 k8s，必须事先准备相应镜像。如果我们每次都安装相同版本，使用 `docker pull + docker tag` 的方式比较合适。  

但是，如果我们要安装不同版本呢，或者随着时间推移，我们需要升级新版，对于每个 k8s 要部署的版本，我们要维护一系列组件对应的版本，挨个tag无疑是一件痛苦的事情。

这时候我们可以选择国内的一些 docker 镜像代理，但是多数只支持代理 dockerhub，如果是 gcr.io、quay.io 等镜像，就显得无能为力。

好在前面我们已经使用 nexus3 来部署了一套docker的镜像仓库，集成了代理以及内部私服功能，现在我们就可以利用这个代理，使用 kubespray 快速、批量的部署 Kubernetes 集群了。


## Kubespray 简介

kubespray 是一个 Ansible 项目，支持主流的 k8s 版本和众多插件，可用于部署高可用的 k8s 集群。

**Deploy a Production Ready Kubernetes Cluster**

- Can be deployed on **AWS, GCE, Azure, OpenStack, vSphere or Baremetal**
- **Highly available** cluster
- **Composable** (Choice of the network plugin for instance)
- Supports most popular **Linux distributions**
- **Continuous integration tests**


## 当前组件及版本

- kubespray v2.10.4  

- Core  
  - kubernetes v1.14.3  
  - etcd v3.2.26  
  - docker v18.06  

- Network Plugin  
  - calico v3.4.0  
  - flanneld v0.11.0  

- Application  
  - coredns v1.5.0  
  - ingress-nginx v0.21.0  
  - cephfs-provisioner v2.1.0-k8s1.11  
  - rbd-provisioner v2.1.1-k8s1.11  


## 初始化集群系统

建议在部署 kubernetes 集群之前，将各个系统进行初始化和配置调优。


## 使用 kubespray 安装 kubernetes   

由于 Kubespray 对 ansible 有版本要求，我们先按照 kubespray 的 requirements 来安装依赖。

### 下载kubespray  

```bash
curl -LSO https://github.com/kubernetes-sigs/kubespray/archive/v2.10.4.tar.gz
tar xf v2.10.4.tar.gz && mv kubespray-2.10.4 kubespray
```

### 安装依赖  

```bash
cd kubespray
yum -y install python36 python-pip
pip install -r requirements.txt --index-url=https://mirrors.ustc.edu.cn/pypi/web/simple
```

### 定制优化 ansible 配置  

- 生成新的 inventory  
  ```bash
  cp -rfp inventory/sample inventory/mycluster
  ```

- 使用 inventory 生成器创建 hosts.yml  
  ```bash
  declare -a IPS=(192.168.12.1 192.168.12.2 192.168.12.3 192.168.12.4 192.168.12.5)
  export HOST_PREFIX=k8s.node
  export CONFIG_FILE=inventory/mycluster/hosts.yml 
  python3 contrib/inventory_builder/inventory.py ${IPS[@]}
  
  # 调整主机名格式
  k8s.master0x
  k8s.etcd0x
  k8s.node0x
  ```

- 增加YAML变量文件覆盖docker镜像地址  
  `vim inventory/mycluster/group_vars/k8s-cluster/vars.yml`  
  ```yaml
  # image repo from roles/download/defaults/main.yml
  kube_image_repo: "google_ontainers"
  etcd_image_repo: "coreos/etcd"
  flannel_image_repo: "coreos/flannel"
  flannel_cni_image_repo: "coreos/flannel-cni"
  calico_node_image_repo: "calico/node"
  calico_cni_image_repo: "calico/cni"
  calico_policy_image_repo: "calico/kube-controllers"
  calico_rr_image_repo: "calico/routereflector"
  calico_typha_image_repo: "calico/typha"
  pod_infra_image_repo: "google_containers/pause-{{ image_arch }}"
  install_socat_image_repo: "xueshanf/install-socat"
  netcheck_agent_image_repo: "l23network/k8s-netchecker-agent"
  netcheck_server_image_repo: "l23network/k8s-netchecker-server"
  weave_kube_image_repo: "weaveworks/weave-kube"
  weave_npc_image_repo: "weaveworks/weave-npc"
  contiv_image_repo: "contiv/netplugin"
  contiv_init_image_repo: "contiv/netplugin-init"
  contiv_auth_proxy_image_repo: "contiv/auth_proxy"
  contiv_etcd_init_image_repo: "ferest/etcd-initer"
  contiv_ovs_image_repo: "contiv/ovs"
  cilium_image_repo: "cilium/cilium"
  cilium_init_image_repo: "library/busybox"
  kube_router_image_repo: "cloudnativelabs/kube-router"
  multus_image_repo: "nfvpe/multus"
  nginx_image_repo: nginx
  haproxy_image_repo: haproxy
  coredns_image_repo: "coredns/coredns"
  nodelocaldns_image_repo: "google_containers/k8s-dns-node-cache"
  dnsautoscaler_image_repo: "google_containers/cluster-proportional-autoscaler-{{ image_arch }}"
  test_image_repo: busybox
  busybox_image_repo: busybox
  helm_image_repo: "lachlanevenson/k8s-helm"
  tiller_image_repo: "kubernetes-helm/tiller"
  registry_image_repo: "registry"
  registry_proxy_image_repo: "google_containers/kube-registry-proxy"
  metrics_server_image_repo: "google_containers/metrics-server-amd64"
  local_volume_provisioner_image_repo: "external_storage/local-volume-provisioner"
  cephfs_provisioner_image_repo: "external_storage/cephfs-provisioner"
  rbd_provisioner_image_repo: "external_storage/rbd-provisioner"
  local_path_provisioner_image_repo: "rancher/local-path-provisioner"
  ingress_nginx_controller_image_repo: "kubernetes-ingress-controller/nginx-ingress-controller"
  cert_manager_controller_image_repo: "jetstack/cert-manager-controller"
  addon_resizer_image_repo: "google_containers/addon-resizer"
  dashboard_image_repo: "google_containers/kubernetes-dashboard-{{ image_arch }}"
  
  # roles/kubernetes-apps/helm/defaults/main.yml
  helm_stable_repo_url: https://mirror.azure.cn/kubernetes/charts
  
  # roles/download/defaults/main.yml
  # 需要在nexus 上配置代理下载github raw 文件
  kubeadm_download_url: "https://nexus.fintecer.com/repository/raw/{{ kubeadm_version }}/bin/linux/{{ image_arch }}/kubeadm"
  hyperkube_download_url: "https://nexus.fintecer.com/repository/raw/{{ kube_version }}/bin/linux/{{ image_arch }}/hyperkube"
  etcd_download_url: "https://nexus.fintecer.com/repository/raw/{{ etcd_version }}/etcd-{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
  cni_download_url: "https://nexus.fintecer.com/repository/raw/{{ cni_version }}/cni-plugins-{{ image_arch }}-{{ cni_version }}.tgz"
  calicoctl_download_url: "https://nexus.fintecer.com/repository/raw/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
  
  # roles/container-engine/docker/defaults/main.yml
  docker_rh_repo_base_url: "https://mirrors.cloud.tencent.com/docker-ce/linux/centos/$releasever/$basearch/stable"

  # default is /var/lib/docker
  #docker_daemon_graph: "/data/docker"
  
  docker_insecure_registries:
    - 192.168.12.100:8888
  
  # Add other registry,example China registry mirror.
  docker_registry_mirrors:
    - https://hub.fintecer.com
  
  # Add vips to certs
  supplementary_addresses_in_ssl_keys:
  - 192.168.12.10

  # Addons
  helm_enabled: true
  
  # Kubernetes internal network for services, unused block of space.
  kube_service_addresses: 172.18.224.0/19
  
  # This network must be unused in your network infrastructure!
  kube_pods_subnet: 172.18.192.0/19
  
  # Set kube_version
  kube_version: v1.14.3
  
  ## Choose network plugin (cilium, calico, contiv, weave or flannel)
  kube_network_plugin: flannel
  #
  ## Kubernetes configuration dirs and system namespace, default: /etc/kubernetes
  #kube_config_dir: /opt/kubernetes # 已取消，有些模版文件内写死了路径，修改到其它地方会有问题
  #
  ## Etcd configuration dirs, default: /etc/ssl/etcd
  #etcd_config_dir: /opt/etcd # 已取消，有些模版文件内写死了路径，修改到其它地方会有问题
  #
  #kube_cert_compat_dir: /opt/kubernetes/pki
  #kube_cert_compat_dir: /opt/kubernetes/pki
  
  ## Limits for kube components
  #kube_controller_memory_limit: 512M
  #kube_controller_cpu_limit: 250m
  #kube_controller_memory_requests: 100M
  #kube_controller_cpu_requests: 100m
  kube_controller_node_monitor_grace_period: 30s
  #kube_controller_node_monitor_period: 5s
  kube_controller_pod_eviction_timeout: 10s
  #kube_controller_terminated_pod_gc_threshold: 12500
  #kube_scheduler_memory_limit: 512M
  #kube_scheduler_cpu_limit: 250m
  #kube_scheduler_memory_requests: 170M
  #kube_scheduler_cpu_requests: 80m
  #kube_apiserver_memory_limit: 2000M
  #kube_apiserver_cpu_limit: 800m
  #kube_apiserver_memory_requests: 256M
  #kube_apiserver_cpu_requests: 100m
  kube_apiserver_request_timeout: "30s"
  
  ## Auto rotate the kubelet client certificates by requesting new certificates from the kube-apiserver when the certificate expiration approaches.
  kubelet_rotate_certificates: true 
  ```

### 使用 kubespray 部署 kubernetes
 
  ```bash
  ansible-playbook -i inventory/mycluster/hosts.yml cluster.yml

  # 失败重试
  ansible-playbook -i inventory/mycluster/hosts.yml cluster.yml --limit @cluster.retry
  
  # 重置集群并重新安装
  ansible-playbook -i inventory/mycluster/hosts.yml reset.yml
  ansible-playbook -i inventory/mycluster/hosts.yml cluster.yml
  ```

### 安装完成  

  集群信息如下：
  ```console
  kubectl get nodes -o wide
  NAME            STATUS   ROLES    AGE      VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
  k8s.master01    Ready    master   25min    v1.14.3   192.168.12.1   <none>        CentOS Linux 7 (Core)   3.10.0-693.el7.x86_64   docker://18.9.5
  k8s.master02    Ready    master   25min    v1.14.3   192.168.12.2   <none>        CentOS Linux 7 (Core)   3.10.0-693.el7.x86_64   docker://18.9.5
  k8s.node01      Ready    <none>   25min    v1.14.3   192.168.12.3   <none>        CentOS Linux 7 (Core)   3.10.0-693.el7.x86_64   docker://18.9.5
  k8s.node02      Ready    <none>   25min    v1.14.3   192.168.12.4   <none>        CentOS Linux 7 (Core)   3.10.0-693.el7.x86_64   docker://18.9.5
  k8s.node03      Ready    <none>   25min    v1.14.3   192.168.12.5   <none>        CentOS Linux 7 (Core)   3.10.0-693.el7.x86_64   docker://18.9.5
  ```

## 安装 nginx-ingress

我们在使用 kubespray 部署 k8s 时打开了参数`helm_enabled: true`，它会同时为我们部署好 helm 服务，这样就可以使用 helm 来安装和管理 chart 应用了。

这里我们就用 helm 来安装 nginx-ingress。

nginx-ingress 的chart 提供了许多内置的参数，我们可以在安装时进行覆盖，比如镜像地址，调度到固定节点等。  
nginx-ingress 的 chart 文档参考这里：[nginx-ingress](https://github.com/helm/charts/tree/master/stable/nginx-ingress)

下面是直接安装 nginx-ingress 的步骤：

- 使用 helm 指定参数来安装

```bash
# 给 ingress node 节点设置标签并添加污点
kubectl label node k8s.ingress01 k8s.ingress02 ingress-node=yes
kubectl taint nodes k8s.ingress01 k8s.ingress02 ingress-node=yes:NoSchedule

# 安装 ingress 同时设置 ingress 容忍污点并且仅调度到 ingress node
helm install stable/nginx-ingress \
  --name xw-ingress \
  --namespace nginx-ingress \
  --set controller.image.repository=kubernetes-ingress-controller/nginx-ingress-controller \
  --set controller.stats.enabled=true \
  --set controller.nodeSelector.ingress-node=yes \
  --set controller.hostNetwork=true \
  --set controller.tolerations[0].key="ingress-node",controller.tolerations[0].operator="Equal",controller.tolerations[0].value="yes",controller.tolerations[0].effect="NoSchedule" \
  --set defaultBackend.image.repository=google_containers/defaultbackend \
  --set defaultBackend.image.tag=1.4 \
  --set defaultBackend.nodeSelector.ingress-node=yes \
  --set defaultBackend.tolerations[0].key="ingress-node",controller.tolerations[0].operator="Equal",controller.tolerations[0].value="yes",controller.tolerations[0].effect="NoSchedule"
```

- 使用 helm 指定配置文件来安装  

上面的命令也可以保存为下面的 yaml 文件并使用如下命令安装：  

```yaml
cat > nginx-ingress-values.yaml << EOF
controller:
  image:
    repository: kubernetes-ingress-controller/nginx-ingress-controller
  hostNetwork: true
  tolerations:
    - key: "ingress-node"
      operator: "Equal"
      value: "yes"
      effect: "NoSchedule"
  nodeSelector:
    ingress-node: "yes"
  replicaCount: 2
  stats:
    enabled: false
  metrics:
    enabled: true
defaultBackend:
  image:
    repository: google_containers/defaultbackend
    tag: "1.4"
  tolerations:
    - key: "ingress-node"
      operator: "Equal"
      value: "yes"
      effect: "NoSchedule"
  nodeSelector: 
    ingress-node: "yes"
EOF

# 安装
helm install stable/nginx-ingress --name xw-ingress \
  --namespace nginx-ingress -f nginx-ingress-values.yaml
```


## 组件升级

kubernetes 版本迭代很快，建议不要选择最近发布的主版本，而是选择上一个主版本的补丁版，它有较多的 hotfix 更新，我们可以少踩坑。 

升级包含2种类型：

### 全量升级  
  运行 cluster.yaml playbook 并指定组件版本，可以使用这种方式全量更新集群组件，但不够安全，不会考虑节点宕机问题。
  ```bash
  ansible-playbook cluster.yml -i inventory/mycluster/hosts.yml -e kube_version=v1.14.10
  ```


### 优雅升级   
  运行 upgrade-cluster.yml playbook 并指定组件版本，可以使用这种方式**滚动**更新集群组件，会逐个驱逐节点pod并完成升级。
  ```bash
  ansible-playbook upgrade-cluster.yml -b -i inventory/mycluster/hosts.yml -e kube_version=v1.14.10
  ```


## 节点删除与添加

- 添加节点：支持添加 master、worker、etcd 节点
- 删除节点：只支持删除 worker 节点

### 添加节点

添加节点有2种方式：  
- 如果是要扩容 `master`、`worker`、`etcd`，则需要重新跑 `cluster.yml` 这个playbook  
  修改`inventory/mycluster/hosts.yml` 添加新增节点，并重新执行**整个 `cluster.yml` playbook**：  
  ```bash
  ansible-playbook -i inventory/mycluster/hosts.ini cluster.yml 
  ```

- 如想要快速扩容 `worker`，则可以直接跑`scale.yml`这个playbook  
  修改`inventory/mycluster/hosts.yml`，释掉原有 node 并添加新的 node（保留 master 和 etcd），并使用 `scale.yml` playbook 扩容 node 节点： 
  ```bash
  ansible-playbook -i inventory/mycluster/hosts.ini scale.yml
  ```

  以上方式可以实现最小化变更，结合下面的节点移除功能，可以实现集群 **node节点** 的自动伸缩（autoscaling）。  


### 移除节点


移除节点**只支持**移除 `worker`节点, 有3种方式： 

- 将计划**移除的节点**写到 `inventory/mycluster/hosts.yml` 中，并跑一次`remove-node.yml` playbook  
  ```bash
  ansible-playbook -i inventory/mycluster/hosts.ini remove-node.yml
  ```


- 使用完整的`inventory`并指定`--extra-vars "node=<nodename>,<nodename2>"`参数并执行 `remove-node.yml` playbook  
  ```bash
  ansible-playbook -i inventory/mycluster/hosts.ini remove-node.yml \
  --extra-vars "node=nodename,nodename2"
  ```


- 使用完整的`inventory`并指定`--limit nodename,nodename2` 参数执行 `remove-node.yml` playbook  

  ```bash
  ansible-playbook -i inventory/mycluster/hosts.ini remove-node.yml \
    --limit nodename,nodename2"
  ```

最后，我们还需要使用 kubectl 手动移除 node 节点  

  ```bash
  kubectl delete node node5
  ```

## 参考文档  

- 移除节点： [Adding nodes](https://github.com/kubernetes-sigs/kubespray/blob/v2.10.4/docs/getting-started.md#adding-nodes)
- 移除节点： [Remove nodes](https://github.com/kubernetes-sigs/kubespray/blob/v2.10.4/docs/getting-started.md#remove-nodes)