---
layout: default
title: "prometheus搭建和配置"
date: 2020-02-04 10:00:00
published: 2020-02-04 10:00:00
comments: true
categories: 监控
tags: [monitor]
author: yangyang
---

<p>
   Prometheus是一个开源的监控和报警系统，作为CNCF的仅次于kubernetes的重要成员，prometheus已经成为使用容器技术的互联网公司搭建监控系统的首选。本文会介绍prometheus，一步步教你搭建prometheus。
</p>


<!--more-->
# 一 简介

## 1.1 优点

- 多维度的数据模型（通过指标名称和标签键值对标识）
- 灵活的查询语言
- 支持pull和push的采集方式
- 可通过服务发现或者静态配置发现目标（监控数据源）
- 支持多模式的画图和仪表盘，配和grafana可以绘制出丰富的图表

## 1.2 组件

- Prometheus server（抓取、存储时间序列数据）
- pushgateway（支持短生命周期的jobs，接收push的监控数据）（prometheus原生支持pull工作模式，为了兼容push工作模式）
- exporters（用于支持开源服务的监控数据采集，比如：HAProxy、StatsD、Graphite等）（也就是agent）
- alertmanager（处理警报）

## 1.3 架构

<img src="/assets/images/posts/prometheus/structure.png" width="100%"/>

# 二 搭建

## 2.1 搭建准备

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

### 2.1.1 服务器列表

name | ip |  type | os |
-|-|-|-
prometheus-server | 192.168.100.100 | prometheus-server | centos7 64位
node_exporter | 192.168.100.101 | agent | centos7 64位
consul01 | 192.168.100.102 | consul01 | centos7 64位
consul02 | 192.168.100.103 | consul02 | centos7 64位

### 2.1.2 环境准备

## 2.2 prometheus安装

   prometheus的安装非常简单，只需在github上根据自己服务器版本选择[下载release](https://github.com/prometheus/prometheus/releases)的安装包，解压即可。

### 下载并安装prometheus
```shell
# 本次教程选择安装在opt目录
[root@prometheus-server-192-168-100-100 ~]# cd /opt/
[root@prometheus-server-192-168-100-100 opt]# wget https://github.com/prometheus/prometheus/releases/download/v2.15.2/prometheus-2.15.2.linux-amd64.tar.gz
[root@prometheus-server-192-168-100-100 opt]# tar -zxf prometheus-2.15.2.linux-amd64.tar.gz
```
- 安装完成，启动prometheus
```shell
# 设置systemd
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# vi /usr/lib/systemd/system/prometheus.service
[Unit]
Description=Prometheus
After=local-fs.target network-online.target network.target
Wants=local-fs.target network-online.target network.target
[Service]
User=root
Group=root
Type=simple
ExecStart=/opt/prometheus-2.15.2.linux-amd64/prometheus --storage.tsdb.path=/data1 --storage.tsdb.retention.time=90d --config.file=/opt/prometheus-2.15.2.linux-amd64/prometheus.yml --web.listen-address=0.0.0.0:9090 --web.enable-admin-api --web.enable-lifecycle
# storage.tsdb.path: 监控数据存储位置
# storage.tsdb.retention.time: 数据存储天数,90d为90天
# config.file: 配置位置
[Install]
WantedBy=multi-user.target
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]#  systemctl daemon-reload
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# systemctl start prometheus
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# systemctl enable prometheus
Created symlink from /etc/systemd/system/multi-user.target.wants/prometheus.service to /usr/lib/systemd/system/prometheus.service.
# 查看端口是够正常监听
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# netstat -tunlp | grep prometheus
tcp6       0      0 :::9090                 :::*                    LISTEN      48558/prometheus 
```
- 在本地游览器中访问http://192.168.100.100:9090 查看prometheus的web界面

<img src="/assets/images/posts/prometheus/prometheus-web.png" width="100%"/>

### 下载并安装alertmanager

- 下载解压

```shell

[root@prometheus-server-192-168-100-100 ~]# cd /opt/
[root@prometheus-server-192-168-100-100 opt]# wget https://github.com/prometheus/alertmanager/releases/download/v0.20.0/alertmanager-0.20.0.linux-amd64.tar.gz
[root@prometheus-server-192-168-100-100 opt]# tar -zxf alertmanager-0.20.0.linux-amd64.tar.gz
[root@prometheus-server-192-168-100-100 opt]# vi /usr/lib/systemd/system/alertmanager.service
[Unit]
Description=Prometheus Alertmanager
After=local-fs.target network-online.target network.target
Wants=local-fs.target network-online.target network.target
[Service]
User=root
Group=root
Type=simple
ExecStart=/opt/alertmanager-0.20.0.linux-amd64/alertmanager --config.file=/opt/alertmanager-0.20.0.linux-amd64/alertmanager.yml
# config.file 配置文件路径
[Install]
WantedBy=multi-user.target
[root@prometheus-server-192-168-100-100 opt]# systemctl daemon-reload
[root@prometheus-server-192-168-100-100 opt]# systemctl start alertmanager
[root@prometheus-server-192-168-100-100 opt]# systemctl enable alertmanager
Created symlink from /etc/systemd/system/multi-user.target.wants/alertmanager.service to /usr/lib/systemd/system/alertmanager.service.
# 查看端口监听
[root@prometheus-server-192-168-100-100 opt]# netstat -tunlp | grep alertmanager
tcp6       0      0 :::9093                 :::*                    LISTEN      48681/alertmanager  
tcp6       0      0 :::9094                 :::*                    LISTEN      48681/alertmanager  
udp6       0      0 :::9094                 :::*                                48681/alertmanager 
```

- 在本地游览器中访问http://192.168.100.100:9093 查看alertmanager的web界面

<img src="/assets/images/posts/prometheus/alertmanager-web.png" width="100%"/>

## 2.3 exporter安装

   prometheus监控体系中，数据采集大多依赖exporter完成，不同组件的监控需要不同的exporter，比如监控服务器资源的[node_exporter](https://github.com/prometheus/node_exporter)、监控中间件的[zookeeper_exporter](https://github.com/carlpett/zookeeper_exporter)/[redis_exporter](https://github.com/oliver006/redis_exporter)/[elasticsearch_exporter](https://github.com/justwatchcom/elasticsearch_exporter)等、监控数据库的[mongodb_exporter](https://github.com/percona/mongodb_exporter)/[mysql_exporter](https://github.com/prometheus/mysqld_exporter)等、监控网络连通性的[blackbox_exporter](https://github.com/prometheus/blackbox_exporter)。
   
   如果有特殊监控需求，则需要按照prometheus exporter开发规范开发定制的exporter。

### 安装node_exporter并启动
- 在**每台服务器**上执行以下命令
```shell
[root@prometheus-server-192-168-100-100 ~]# cd /opt/
[root@prometheus-server-192-168-100-100 opt]# wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
[root@prometheus-server-192-168-100-100 opt]# tar -zxf node_exporter-0.18.1.linux-amd64.tar.gz
```
- 配置node_exporter system服务并启动
```shell
[root@prometheus-server-192-168-100-100 opt]# vi /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=Prometheus node exporter
After=local-fs.target network-online.target network.target
Wants=local-fs.target network-online.target network.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/opt/node_exporter-0.18.1.linux-amd64/node_exporter

[Install]
WantedBy=multi-user.target
[root@prometheus-server-192-168-100-100 opt]# systemctl daemon-reload
[root@prometheus-server-192-168-100-100 opt]# systemctl start node_exporter
[root@prometheus-server-192-168-100-100 opt]# systemctl enable node_exporter
Created symlink from /etc/systemd/system/multi-user.target.wants/node_exporter.service to /usr/lib/systemd/system/node_exporter.service.
# 查看服务启动监听的9100端口
[root@prometheus-server-192-168-100-100 opt]# netstat -tunlp | grep 9100
tcp6       0      0 :::9100                 :::*                    LISTEN      47034/node_exporter 
```

# 三 配置

## 3.1 配置prometheus
```yaml
# 全局配置
global:
  scrape_interval:     15s   # 多久 收集 一次数据
  evaluation_interval: 30s   # 多久评估一次 规则
  scrape_timeout:      10s   # 每次 收集数据的 超时时间
  # 当Prometheus和外部系统(联邦, 远程存储, Alertmanager)通信的时候，添加标签到任意的时间序列或者报警
  external_labels:
    monitor: codelab
    foo:     bar
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 127.0.0.1:9093 # 配置alertmanager

scrape_configs:
  - job_name: 'monitoring/prometheus' 
    static_configs:
    - targets: ['localhost:9090'] # 配置收集prometheus数据
  
  - job_name: 'monitoring/alertmanager'
    static_configs:
    - targets: ['localhost:9093'] # 配置收集alertmanager数据

  - job_name: 'monitoring/node_exporter'
    static_configs:
    - targets:
      - 192.168.100.100:9100
      - 192.168.100.101:9100
      - 192.168.100.102:9100
      - 192.168.100.103:9100
```


## 3.2 配置alertmanager

## 3.3 基于consul的自动发现

