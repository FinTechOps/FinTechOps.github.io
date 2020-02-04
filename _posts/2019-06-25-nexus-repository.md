---
layout: default
title: "使用nexus部署docker镜像仓库"
date: 2020-02-03 14:20:00
published: 2019-06-27 16:01:00
comments: true
categories: KUBERNETES
tags: [cicd]
author: ykfq
---

<p>
本文记录使用 nexus 3 部署 docker 镜像仓库及镜像代理功能。

使用 Kubespray 在安装 kubernetes 过程中，会直接从官方镜像仓库（如 hub.docker.com，gcr.io, quay.io 等）拉取镜像，同时还会从 github 下载 kubeadm、etcd、cni 等二进制文件，重度依赖访问外网，因此我们有必要先配置镜像代理服务器。
</p>


<!--more-->


## 安装 nexus 3
- 下载 nexus-3.20.1-01-unix.tar.gz   
  访问如下地址，随便填写一个邮箱，下载 nexus oss 二进制包：  
  [https://www.sonatype.com/download-oss-sonatype](https://www.sonatype.com/download-oss-sonatype)


- 解压并启动服务
  ```bash
  cd /data/
  tar xf /root/nexus-3.20.1-01-unix.tar.gz
  ln -sf nexus-3.19.1-01 nexus
  cd nexus && bin/nexus start
  ```
  以上会将服务运行在 `8081` 端口，默认账户名/密码：`admin/admin123`


## 配置 nexus 使用 http 代理

- 准备一个可以正常访问外网的代理  
  省略。

- 在 nexus 中配置代理设置  
  在 neuxs 的 System - HTTP 下，勾选 HTTP proxy 并填写上述端口和地址：  
  > HTTP proxy host：127.0.0.1  
  > HTTP proxy host：30888  

- 排除代理国内服务  
  有时候我们也想用 nexus 来整合国内的一些下载服务，以及我们自己的私有仓库，提供统一的入口。  
  如果还让这些仓库走代理，那就又慢又耗费 v.p.s 流量了。不用担心，Nexus 提供了在 HTTP 代理中排除域名的功能，只要将国内的域名添加到排除列表即可。



## 配置 Docker 仓库代理

- 创建 Blobs  
  在 Nexus 上创建一个 Docker 专用的 Blob，Nexus 会在磁盘上创建一个对应的目录并存储文件及索引。  
  这样，如果某天我们要在其它环境部署一套一样的代理，我们可以直接拷贝 Nexus 应用目录，以及这个 blob 存储就行了。  

  在 **设置 - Repository - Blob Stores** 下，新建一个 Blob，比如名为 `blob-docker`。

- 创建 Repositries  
  依次配置添加下面几个仓库地址的 Docker 镜像代理，类型选择 Proxy：

  * https://hub.docker.com
  * https://gcr.io
  * https://k8s.gcr.io
  * https://quay.io
  * https://elastic.co

  最后配置一个 Group 类型的 Docker 仓库，将前面代理类型仓库进行聚合。  
  这里需要为 Group 类型的仓库地址监听一个端口，用来提供 docker 服务，我们使用 8082 。然后为这个服务绑定一个域名，并配置 https 证书。

做完这些，我们就有一个自己的 Docker 镜像仓库地址了, 比如域名为：  
[https://hub.fintecer.com](https://hub.fintecer.com)



## 如何使用  

- 直接替换镜像地址中的域名，后面的路径保持不变，例如：  
  ```bash
  docker pull hub.fintecer.com/busybox
  docker pull hub.fintecer.com/nginx:1.16.0
  docker pull hub.fintecer.com/coreos/etcd:v3.2.26
  ```

- Docker 守护进程设置 registry-mirrors 参数  
  如果我们在 docker 的 daemon.json 文件指定了 `"registry-mirrors": ["https://hub.fintecer.com"],` 配置， 我们甚至可以不用域名就能直接拉取镜像了。  
  daemon.json 示例：  
  ```bash
  mkdir /etc/docker  
  cat > /etc/docker/daemon.json << EOF
  {
    "registry-mirrors": ["https://hub.fintecer.com"],
    "bip": "10.255.255.1/24",
    "ipv6": false
  }
  EOF

  # 重启服务
  systemctl daemon-reload
  systemctl restart docker
  ```

  最后使用如下方式下载镜像：  
  ```bash
  docker pull busybox
  docker pull nginx:1.16.0
  docker pull coreos/etcd:v3.2.26
  ```

当我们从 github 下载现成的 yaml 文件时，几乎无需修改镜像地址就能正常拉起服务了，非常方便。