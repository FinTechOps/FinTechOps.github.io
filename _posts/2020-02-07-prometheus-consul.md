---
layout: default
title: "prometheus搭建和配置 - 基于consul的自动注册（二）"
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
# 介绍

  搭建prometheus中可以发现，prometheus的管理大多基于配置修改，当我们监控的服务器或者组件变多时，就不可避免的出现配置复杂、维护困难的问题。prometheus支持多种方式的自动发现机制，比如：
- static_configs: 静态服务发现
- file_sd_configs: 文件服务发现
- dns_sd_configs: DNS 服务发现
- kubernetes_sd_configs: Kubernetes 服务发现
- consul_sd_configs: Consul 服务发现
  本文将介绍如何使用prometheu的consul_sd_configs发现功能基于consul实现监控的自动注册。

## consul介绍

Consul是一个分布式K/V的数据库，功能主要为服务化的系统提供服务注册、服务发现和配置管理的功能。

# 搭建

沿用上文的搭建环境[prometheus搭建和配置(一)](https://www.fintecer.com/posts/prometheus/)

## 搭建准备

### 服务器列表

| name | ip |  type | os |
|:-----:|:-----:|:-----:|:-----:|
|prometheus-server | 192.168.100.100 | prometheus-server | centos7 64位 |
|consul01 | 192.168.100.101 | consul01 | centos7 64位 | 
|consul02 | 192.168.100.102 | consul02 | centos7 64位 | 
|consul03 | 192.168.100.103 | consul03 | centos7 64位 |

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

## 搭建consul集群
在consul官网选择自己服务器对应版本的安装包，下载解压即可
### 下载

在consul01,consul02,consul03上执行以下命令
```shell
[root@consul01-192-168-100-101 ~]# cd /opt
[root@consul01-192-168-100-101 opt]# wget https://releases.hashicorp.com/consul/1.6.3/consul_1.6.3_linux_amd64.zip
--2020-02-08 16:22:23--  https://releases.hashicorp.com/consul/1.6.3/consul_1.6.3_linux_amd64.zip
Connecting to 192.168.100.1:10800... connected.
Proxy request sent, awaiting response... 200 OK
Length: 40150725 (38M) [application/zip]
Saving to: ‘consul_1.6.3_linux_amd64.zip.1’

100%[=======================================================================================>] 40,150,725  6.87MB/s   in 5.8s   

2020-02-08 16:22:30 (6.58 MB/s) - ‘consul_1.6.3_linux_amd64.zip.1’ saved [40150725/40150725]
[root@consul01-192-168-100-101 opt]#  unzip consul_1.6.3_linux_amd64.zip 
Archive:  consul_1.6.3_linux_amd64.zip
  inflating: consul                  
[root@consul01-192-168-100-101 opt]#
```

### 启动

在consul01上执行以下命令，写入systemd服务启动，并设置开机启动

```shell
[root@consul01-192-168-100-101 opt]# cat /usr/lib/systemd/system/consul.service
[Unit]
Description=Prometheus consul
After=local-fs.target network-online.target network.target
Wants=local-fs.target network-online.target network.target
[Service]
User=root
Group=root
Type=simple
ExecStart=/opt/consul agent -server -bootstrap-expect=3 -data-dir=/data/consul -client=0.0.0.0 -datacenter=test -ui
# data-dir 数据目录
# datacenter 数据中心名
[Install]
WantedBy=multi-user.target
[root@consul01-192-168-100-101 opt]# systemctl daemon-reload
[root@consul01-192-168-100-101 opt]# systemctl start consul
[root@consul01-192-168-100-101 opt]# systemctl enable consul
Created symlink from /etc/systemd/system/multi-user.target.wants/consul.service to /usr/lib/systemd/system/consul.service.
```
在consul02，consul03上执行以下命令，写入systemd服务启动，并设置开机启动

```shell
[root@consul02-192-168-100-102 opt]# cat /usr/lib/systemd/system/consul.service
[Unit]
Description=Prometheus consul
After=local-fs.target network-online.target network.target
Wants=local-fs.target network-online.target network.target
[Service]
User=root
Group=root
Type=simple
ExecStart=/opt/consul agent -server -bootstrap-expect=3 -data-dir=/data/consul -client=0.0.0.0 -datacenter=test -join=192.168.100.101
# data-dir 数据目录
# datacenter 数据中心名
# join 需要加入的server的ip，这里是192.168.100.101
[Install]
WantedBy=multi-user.target
[root@consul02-192-168-100-102 opt]# systemctl daemon-reload
[root@consul02-192-168-100-102 opt]# systemctl start consul
[root@consul02-192-168-100-102 opt]# systemctl enable consul
Created symlink from /etc/systemd/system/multi-user.target.wants/consul.service to /usr/lib/systemd/system/consul.service.
```

### 验证集群状态

游览器访问consul的web界面，http://192.168.100.101:8500

<img src="/assets/images/posts/prometheus-consul/prometheus-consul.png" width="100%"/>

可以看到三个节点已经形成了集群模式，接下来就可以配置prometheus和自动注册了。


## 配置自动发现

### 配置prometheus
Prometheus主要配置prometheus.yml中consul_sd_configs，配置如下

```yaml
# 全局配置
global:
  scrape_interval:     15s   # 多久 收集 一次数据
  evaluation_interval: 30s   # 多久评估一次 规则
  scrape_timeout:      10s   # 每次 收集数据的 超时时间

scrape_configs:
  - job_name: 'consul'
    consul_sd_configs:
      - server: '192.168.100.101:8500'
    relabel_configs:
      - source_labels: [__meta_consul_tags]
        regex: .*,prome,.*
        action: keep
      - source_labels: [__meta_consul_service]
        target_label: job
```

这里简单介绍一下relabel_configs，Prometheus 允许用户在采集任务设置中，通过relabel_configs来添加自定义的Relabeling的额过程，来对标签进行指定规则的重写。

```yaml
      - source_labels: [__meta_consul_tags]
        regex: .*,prome,.*
        action: keep
```
keep: 丢弃 source_labels 的值中没有匹配到 regex 正则表达式内容的 Target 实例,实现过滤__meta_consul_service中prome标签
```yaml
      - source_labels: [__meta_consul_service]
        target_label: job
```
这段配置是指将__meta_consul_service标签得值替换成job标签的值

重启生效
```shell
[root@prometheus-server-192-168-100-100 ~]# curl -XPOST http://127.0.0.1:9090/-/reload
```

## 注册

我们将consul01，consul02，consul03上的node_exporter注册到consul，然后prometheus就能自动发现，进行监控数据的采集了。

### 向consul注册服务

```shell
[root@prometheus-server-192-168-100-100 ~]# curl -X PUT -d '{"id": "consul01-192-168-100-101","name": "node_exporter","address": "192.168.100.101","port": 9100,"tags": ["test","prome","node_exporter"],"checks": [{"http": "http://192.168.100.101:9100/metrics","interval": "35s"}]}' http://192.168.100.101:8500/v1/agent/service/register
[root@prometheus-server-192-168-100-100 ~]# curl -X PUT -d '{"id": "consul01-192-168-100-102","name": "node_exporter","address": "192.168.100.102","port": 9100,"tags": ["test","prome","node_exporter"],"checks": [{"http": "http://192.168.100.102:9100/metrics","interval": "35s"}]}' http://192.168.100.101:8500/v1/agent/service/register
[rootprometheus-server-192-168-100-100 ~]# curl -X PUT -d '{"id": "consul01-192-168-100-103","name": "node_exporter","address": "192.168.100.103","port": 9100,"tags": ["test","prome","node_exporter"],"checks": [{"http": "http://192.168.100.103:9100/metrics","interval": "35s"}]}' http://192.168.100.101:8500/v1/agent/service/register

```

### 验证

首先我们到consul web页面http://192.168.100.101:8500 查看consul注册成功与否。

1.进入页面点击service查看注册上来的服务

<img src="/assets/images/posts/prometheus-consul/consul-register01.png" width="100%"/>

2.查看注册服务的详细情况

<img src="/assets/images/posts/prometheus-consul/consul-register02.png" width="100%"/>

然后我们到prometheus 的web页面查看自动发现成功与否，游览器访问http://192.168.100.100:9090/targets

<img src="/assets/images/posts/prometheus-consul/prometheus-register.png" width="100%"/>

注册成功


# 总结

本文介绍了如何使用consul集群的搭建方式，以及基于consul实现prometheus监控服务的自动注册。实际生产应用中，其实还需要做更多的事，比如监控安装自动化流程中，调用consul的注册接口将安装的监控服务注册上去，或者使用cmdb这种管理平台的数据来自动注册到consul，来实现真正的实现自动注册与发现。