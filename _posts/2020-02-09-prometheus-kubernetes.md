---
layout: default
title: "prometheus搭建和配置 - 监控kubernetes（三）"
date: 2020-02-07 10:00:00
published: 2020-02-07 10:00:00
comments: true
categories: 监控
tags: [monitor]
author: yangyang
---

<p>
   Prometheus是一个开源的监控和报警系统，作为CNCF的仅次于kubernetes的重要成员，prometheus已经成为使用容器技术的互联网公司搭建监控系统的首选。本文会介绍prometheus，一步步教你搭建prometheus。
</p>


<!--more-->

# 简介

前文中介绍了如何搭建prometheus监控和自动注册功能的实现。作为CNCF的一员的prometheus，监控kubernetes是prometheus的重要功能，通过配置prometheus可以监控比如Kubernetes指标、容器指标、节点资源指标以及应用程序指标等。prometheus监控kubernetes有两种部署方式，k8s内部署和k8s外部署，本文介绍的是k8s外独立部署监控集群。

## 监控组件

- Heapster：Heapster 是一个集群范围的监控和数据聚合工具，以 Pod 的形式运行在集群中。kubernetes 1.12后废弃由metrics-server代替

- cAdvisor：cAdvisor是Google开源的容器资源监控和性能分析工具，它是专门为容器而生，本身也支持 Docker 容器，在 Kubernetes 中，我们不需要单独去安装，cAdvisor 作为 kubelet 内置的一部分程序可以直接使用。

- Kube-state-metrics：kube-state-metrics通过监听 API Server 生成有关资源对象的状态指标，比如 Deployment、Node、Pod，需要注意的是 kube-state-metrics 只是简单提供一个 metrics 数据，并不会存储这些指标数据，所以我们可以使用 Prometheus 来抓取这些数据然后存储。

- metrics-server：metrics-server 也是一个集群范围内的资源数据聚合工具，是 Heapster 的替代品，同样的，metrics-server 也只是显示数据，并不提供数据存储服务。


# 搭建监控

## 搭建准备

### 服务器列表

| name | ip |  type | os |
|:--------:|:-------:|:-------:|:-------:|
|prometheus-server | 192.168.100.100 | prometheus-server | centos7 64位 |
|kubernetes | 192.168.100.101 | minikube | centos7 64位 | 


### 环境准备

每台服务器上执行以下命令准备环境
```shell
# 永久关闭selinux
[root@localhost ~]# sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config  
# 临时关闭
[root@localhost ~]# setenforce 0
[root@localhost ~]# getenforce
Permissive
#关闭防火墙
[root@localhost ~]# systemctl stop firewalld
# 关闭防火墙自启
[root@localhost ~]# systemctl enable firewalld
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.
```

### kubernetes安装

本文使用minikube构建快速的搭建kubernetes

1. 首先安装[docker](https://docs.docker.com/install/linux/docker-ce/centos/)
2. 安装使用[minikube构建kubernetes](https://kubernetes.io/docs/tasks/tools/install-minikube/)

## 配置prometheus

首先创建kubernetes的[token](https://www.jianshu.com/p/1c188189678c)和crt，本文使用minikube生成的default serviceaccount的token和crt,放在prometheus-server服务器上。

```shell
[root@prometheus-server-192-168-100-100 ~]# mkdir /root/serviceaccount
[root@prometheus-server-192-168-100-100 serviceaccount]# cd /root/serviceaccount
[root@prometheus-server-192-168-100-100 serviceaccount]# vi token
eyJhbGciOiJSUzI1NiIsImtpZCI6Ino1ajN6YWRDcl9Ubkc4bmljR3VGeGlHUE9uRUVFODdzSHZZb0oyeEpvTmcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLThscWI5Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlYzA1ZTY5NS0xZGZmLTQzZDctYTIzNi00N2I1NTVlYjZiNWEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.Twmr3jGCtURSud9d0lWsuiVhbeTP7tY4_YtVQdn0KIEGkd7U7gPofcvGTmvhfkH4Lz9XvMNUyjlJGKKqY58XLVrJ2tXZK65Mu3Sw3WQSQqydsAdeHR2H_n3_BfpFG2vlzpCEH0XAnHEIQ5twK_qdQ9A5y8nLqvomC7v_bkU2LqumgQKf_7USYvI_9D76gdp_4rNrwbjwp78NehD9OGHHt_gcGmG6vO9WIqKoBfH7w51yTDTG9fGJ3x7x-LyQ0R0vMk0iYcYMkkAClMbBCcmU3AK6yoklar0vPvk06bqc-IU6IKCG7kYwTNaUAHpt8R0Q05bxEvhl-wzHXS1JWV31rg
[root@prometheus-server-192-168-100-100 serviceaccount]# vi ca.crt
-----BEGIN CERTIFICATE-----
MIIC5zCCAc+gAwIBAgIBATANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwptaW5p
a3ViZUNBMB4XDTIwMDIwODA4MDYzOFoXDTMwMDIwNjA4MDYzOFowFTETMBEGA1UE
AxMKbWluaWt1YmVDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAK+X
n0ZvaxQ5cjVhDWK2AkudH3xZqxZdDj83uhbyxCPCYWiyFgEg0xhnLBedykUuD3NN
cV+SU+LSgPeYP7sJjtw/whRKp6C3ubJrQwSaTNq4l/kQ/Lq+KebwEkuTh+P5TkS7
ZMU4v7ZPPtym1Rn4xWLtBtHGDRTtmm5PdHG++gKDnO6tlB8uCWyuVt49+5Se96D+
ja2UFKvWG/415wDOGZvr+2eXRLXhEEBIRbU+P2HKWHMviI7RrBKuvFOXr7Dzz1nB
57CGgrhn0hoq0txr9O5InPkl2H0UFmMykpVt7nUXWOKIVO5yP/lW0uXFe79ealfq
3X7zCKFFPNPM2RbntHMCAwEAAaNCMEAwDgYDVR0PAQH/BAQDAgKkMB0GA1UdJQQW
MBQGCCsGAQUFBwMCBggrBgEFBQcDATAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3
DQEBCwUAA4IBAQCW32hrdpXq1l1l8ij0o4jlEFMZkTN99rCwH+JQDpJ97fXLaGT/
GXPXl2sTLdpDCyi22BHHabv87gQdRYQjh9/l9lWyVbr32ehT62HdYg5C3CdYZH7Z
koFpLY+FNnTgI71YpxLe4vZoY8OQtt7IHpAPPt4ZQhT/WnPmJZYk3Z1NMMCXYTJu
T86rzQetL29IEtpfSuxJBAXSEU688yvtXiQhMFDZSQZwmSYiih9F2etmQqSRr8H4
uKddyJz/3q1Yp7ZtROrbU8DxdN2ZSm4UNdBF0RUibSLkz2Cz7dPeqSj8GwT1Jj12
J70uDubyhR4fE+EyMV+GEu+YttriyBL8Ac6l
-----END CERTIFICATE-----
```

### 监控cAdvisor

kubernetes已经集成了cAdvisor监控组件，prometheus自动发现采集方式有两种：
- master上的kube-apiserver端口下可以调用:
    
    cAdvisor的metrics地址: /api/v1/nodes/[节点名称]/proxy/metrics/cadvisor

- 每个节点的kubelet的端口下可以调用:
    cAdvisor的metrics地址: /metrics/cadvisor

建议采用第二种数据采集调用方式，理由是方便node的服务发现和prometheus的target清晰展示。

下面为配置

```shell
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64] vi prometheus.yml
# 全局配置
global:
  scrape_interval:     15s   # 多久 收集 一次数据
  evaluation_interval: 30s   # 多久评估一次 规则
  scrape_timeout:      10s   # 每次 收集数据的 超时时间

scrape_configs:
  # 定义jobname
  - job_name: 'kubernetes-cadvisor'
    # kubernetes-cadvisor
    # 采集方式为https
    scheme: https
    # 采集的metrics接口
    metrics_path: /metrics/cadvisor
    # 采集是否跳过tls验证，建议跳过
    tls_config:
      insecure_skip_verify: true
    # 采集数据需要的token路径
    bearer_token_file: /root/serviceaccount/token
    # 自动发现kubernetes的配置
    kubernetes_sd_configs:
    # api_server地址
    - api_server: 'https://192.168.100.101:6443'
    # 发现的方式，因为要采集每个节点的kubelet端口
      role: node
      # 自动发现的token路径
      bearer_token_file: /root/serviceaccount/token
      # 自动发现的证书路径
      tls_config:
        ca_file: /root/serviceaccount/ca.crt
    # 采集到的数据的label的重写
    relabel_configs:
      # 以下配置的意义为目的默认label：__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name通过正则表达式Node;(.*)匹配后的第一个参数(${1})替换掉target_label: node
      - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
        separator: ;
        regex: Node;(.*)
        target_label: node
        replacement: ${1}
        action: replace
      - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
        separator: ;
        regex: Pod;(.*)
        target_label: pod
        replacement: ${1}
        action: replace
      - source_labels: [__meta_kubernetes_pod_name]
        separator: ;
        regex: pod_name
        target_label: pod
        replacement: $1
        action: replace
      - source_labels: [__meta_kubernetes_service_label_k8s_app]
        separator: ;
        regex: (.+)
        target_label: job
        replacement: ${1}
        action: replace
      - separator: ;
        regex: (.*)
        target_label: endpoint
        replacement: https-metrics
        action: replace
```
#### 验证

游览器打开http://192.168.100.100:9090/targets，可以看到cadvisor的监控已经自动发现并开始采集。

<img src="/assets/images/posts/prometheus-kubernetes/prometheus-cadvisor.png" width="100%"/>

打开graph界面http://192.168.100.100:9090/graph，查询cadvisor的指标container_cpu_system_seconds_total

<img src="/assets/images/posts/prometheus-kubernetes/prometheus-cadvisor01.png" width="100%"/>

### 监控kubelet

kubernetes部署在每个节点上kubelet组件都带有prometheus采集的metrics接口，prometheus自动发现采集方式有两种：
- master上的kube-apiserver端口下可以调用:
    
    kubelet的metrics地址: /api/v1/nodes/[节点名称]/proxy/metrics

- 每个节点的kubelet的端口下可以调用:
    kubelet的metrics地址: /metrics

建议采用第二种数据采集调用方式，理由是方便node的服务发现和prometheus的target清晰展示。

注意

- kubelet 需要开启下面两个参数

```
--authentication-token-webhook
--authorization-mode=Webhook
```
- prometheus配置的token需要有以下权限
```
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
```



下面为配置

```shell
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64] vi prometheus.yml
# 全局配置
global:
  scrape_interval:     15s   # 多久 收集 一次数据
  evaluation_interval: 30s   # 多久评估一次 规则
  scrape_timeout:      10s   # 每次 收集数据的 超时时间

scrape_configs:
  # 定义jobname
  - job_name: 'kubernetes-kubelet'
    honor_labels: true
    honor_timestamps: true
    # kubernetes-kubelet
    # 采集方式为https
    scheme: https
    # 采集的metrics接口
    metrics_path: /metrics
    # 采集是否跳过tls验证，建议跳过
    tls_config:
      insecure_skip_verify: true
    # 采集数据需要的token路径
    bearer_token_file: /root/serviceaccount/token
    # 自动发现kubernetes的配置
    kubernetes_sd_configs:
    # api_server地址
    - api_server: 'https://192.168.100.101:6443'
    # 发现的方式，因为要采集每个节点的kubelet端口
      role: node
      # 自动发现的token路径
      bearer_token_file: /root/serviceaccount/token
      # 自动发现的证书路径
      tls_config:
        ca_file: /root/serviceaccount/ca.crt
    relabel_configs:
    - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
      separator: ;
      regex: Node;(.*)
      target_label: node
      replacement: ${1}
      action: replace
    - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
      separator: ;
      regex: Pod;(.*)
      target_label: pod
      replacement: ${1}
      action: replace
    - source_labels: [__meta_kubernetes_pod_name]
      separator: ;
      regex: (.*)
      target_label: pod
      replacement: $1
      action: replace
    - source_labels: [__meta_kubernetes_service_label_k8s_app]
      separator: ;
      regex: (.+)
      target_label: job
      replacement: ${1}
      action: replace
    - separator: ;
      regex: (.*)
      target_label: endpoint
      replacement: https-metrics
      action: replace
```
#### 验证

游览器打开http://192.168.100.100:9090/targets，可以看到kubelet的监控已经自动发现并开始采集。

<img src="/assets/images/posts/prometheus-kubernetes/prometheus-kubelet01.png" width="100%"/>

打开graph界面http://192.168.100.100:9090/graph，查询kubelet的指标container_cpu_system_seconds_total

<img src="/assets/images/posts/prometheus-kubernetes/prometheus-kubelet02.png" width="100%"/>


未完待续