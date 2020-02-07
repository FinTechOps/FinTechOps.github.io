---
layout: default
title: "prometheus搭建和配置"
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

## 优点

- 多维度的数据模型（通过指标名称和标签键值对标识）
- 灵活的查询语言
- 支持pull和push的采集方式
- 可通过服务发现或者静态配置发现目标（监控数据源）
- 支持多模式的画图和仪表盘，配和grafana可以绘制出丰富的图表

## 组件

- Prometheus server（抓取、存储时间序列数据）
- pushgateway（支持短生命周期的jobs，接收push的监控数据）（prometheus原生支持pull工作模式，为了兼容push工作模式）
- exporters（用于支持开源服务的监控数据采集，比如：HAProxy、StatsD、Graphite等）（也就是agent）
- alertmanager（处理警报）

## 架构

<img src="/assets/images/posts/prometheus/structure.png" width="100%"/>

# 搭建

## 搭建准备

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

### 服务器列表

name | ip |  type | os |
-|-|-|-
prometheus-server | 192.168.100.100 | prometheus-server | centos7 64位
node_exporter01 | 192.168.100.101 | agent | centos7 64位
node_exporter01 | 192.168.100.102 | consul01 | centos7 64位
node_exporter01 | 192.168.100.103 | consul02 | centos7 64位

### 环境准备

## prometheus安装

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


## exporter安装

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

# 配置

## 配置prometheus
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
- 调用prometheus的reload配置的接口使配置生效
```shell
[root@prometheus-server-192-168-100-100 ~]# cd /opt/prometheus-2.15.2.linux-amd64
# 检查配置是否配置正确
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# ./promtool check config ./prometheus.yml 
Checking ./prometheus.yml
  SUCCESS: 0 rule files found
# reload配置
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# curl -XPOST http://127.0.0.1:9090/-/reload
```
- 查看prometheus的web端，如图

<img src="/assets/images/posts/prometheus/prometheus-reload-conf.png" width="100%"/>


- 在web界面查询监控数据

<img src="/assets/images/posts/prometheus/prometheus-query-cpu.png" width="100%"/>

## 配置alertmanager

介绍了配置prometheus如何配置收集exporter的数据后，一个完整的监控系统告警也是必不可少的。alertmanager通过灵活的配置支持多种告警手段，下面介绍如何使用alertmanager配置邮件告警。

这里对高级功能不做介绍，以后另外介绍高级功能，比如：
- 分组：  可以通过route书的group_by来进行报警的分组，多条消息一起发送。
- 抑制： 重要报警抑制低级别报警。
- 静默： 故障静默，确保在接下来的时间内不会在收到同样报警信息

### alertmanager配置
```yaml
# 全局配置项
global: 
  resolve_timeout: 5m #处理超时时间，报警恢复的时候不是立马发送的，在接下来的这个时间内，如果没有此报警信息触发，才发送报警恢复消息。默认为5min，
  smtp_smarthost: '****' # 邮箱smtp服务器代理
  smtp_from: '****' # 发送邮箱名称
  smtp_auth_username: '****' # 邮箱名称
  smtp_auth_password: '****' #邮箱密码

# 定义路由树信息
route:
  group_by: ['alertname'] # 报警分组名称
  group_wait: 10s # 最初即第一次等待多久时间发送一组警报的通知
  group_interval: 10s # 在发送新警报前的等待时间
  repeat_interval: 1m # 发送重复警报的周期
  receiver: 'email' # 发送警报的接收者的名称，以下receivers name的名称

# 定义警报接收者信息
receivers:
  - name: 'email' # 警报
    email_configs: # 邮箱配置
    - to: '****'  # 接收警报的email配置
```
- 重启alertmanager

```shell
[root@prometheus-server-192-168-100-100 opt]# systemctl restart alertmanager
```

### rules配置

prometheus在收集到监控数据过后，会根据配置中的rules规则，触发告警并推送到alertmanager，alertmanager会根据自身的配置将告警发送出去，下面简单演示一下rules的配置方法。

需求: 内存剩余小于10%告警

- 首先需要知道node_exporter中，内存的剩余大小的键为：node_memory_MemAvailable_bytes，内存的总大小的key为：node_memory_MemTotal_bytes ，它们的比值*100就为内存使用率node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

- 以上表达式node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100，当小于10的时候需要告警

- 由此可以写出rules规则：

```yaml
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# mkdir rules
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# vi rules/node_exporter.yml
groups:
- name: NodeExporterAlert
  rules:
  - alert: OutOfMemory
    expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      # 表达式node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 查询出来的labels都可以在此引用:{{ $labels.* }}
      # 表达式触发后当前值为{{ $value }}
      summary: "Out of memory (instance {{ $labels.instance }})"
      description: "Node memory is filling up (< 10% left)\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
```
- 配置prometheus使rules生效

```yaml
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# vi prometheus.yml
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

# 将rules文件引入
rule_files:
  - 'rules/node_exporter.yml'

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

# 检查配置并生效
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# ./promtool check config ./prometheus.yml 
Checking ./prometheus.yml
  SUCCESS: 1 rule files found

Checking rules/node_exporter.yml
  SUCCESS: 1 rules found

[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# curl -XPOST http://127.0.0.1:9090/-/reload
[root@prometheus-server-192-168-100-100 prometheus-2.15.2.linux-amd64]# 
```

- 验证rules配置正常，游览器访问：http://192.168.100.100:9090/alerts

<img src="/assets/images/posts/prometheus/prometheus-rules.png" width="100%"/>