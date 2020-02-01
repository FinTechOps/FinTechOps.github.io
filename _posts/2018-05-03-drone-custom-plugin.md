---
layout: default
title: "DRONE编写自定义插件"
date: 2018-05-03 05:00:00 +0200
published: 2018-05-03 05:00:00 +0200
comments: true
categories: CICD
tags: [CICD]
author: snakechen
---


<p>
    When using Apache Camel with Quarkus as of today, we are limited number of Camel connectors. One important one being
    the Apache Kafka connector.
    The Kafka connector provided through the Smallrye Kafka Extension is available for Quarkus though. So in this
    article, I will show how to
    wire Smallrye Kafka connector and Camel together.
    We will also be using the camel-servlet component to reuse the undertow http endpoint provided by Quarkus and to
    spice things up a little we will use the XML DSL of Camel.
    All of this will be natively compiled in the end.
</p>



Docker 容器在启动的时候可以配置一个启动命令，这样我们就可以做到实例化插件容器的时候就执行插件脚本。
另外 Drone 运行插件的时候可以将本次构建相关的信息以及插件配置信息以环境变量的形式注入到插件容器中去，所有的语言都可以读取系统环境变量，这样就可以使用任意的语言来编写drone插件


# 原理

Docker 容器在启动的时候可以配置一个启动命令，这样我们就可以做到实例化插件容器的时候就执行插件脚本。
另外 Drone 运行插件的时候可以将本次构建相关的信息以及插件配置信息以环境变量的形式注入到插件容器中去，所有的语言都可以读取系统环境变量，这样就可以使用任意的语言来编写drone插件



-   插件配置会以 `PLUGIN_xxx` 的形式注入，例如 `PLUGIN_TOKEN` 。
-   构建相关的信息會会以 `CI_xxx` 或者 `DRONE_xxx` 的形式注入，例如 `DRONE_COMMIT` , `CI_COMMIT_AUTHOR_NAME` 。



# 编写一个shell插件[官文例子]


#### 1.编写script.sh

```
#!/bin/sh

[ -n PLUGIN_USER ] && PLUGIN_USER="${WEBHOOK_USER}"

curl \
  -X "${PLUGIN_METHOD}" \
  -d "${PLUGIN_BODY}" \
  -u "${PLUGIN_USER}" \
  ${PLUGIN_URL}

```

#### 2.编写Dockerfile

```
FROM alpine
ADD script.sh /bin/
RUN chmod +x /bin/script.sh
RUN apk -Uuv add curl ca-certificates
ENTRYPOINT ["/bin/script.sh"]
```

通过上面两步，一个基本的插件就编写好了，该插件的功能就是在执行时去通过curl访问一个url


#### 3.打包推送到仓库


```
docker build . -t i.harbor.dragonest.net/drone/curl:0.0.1
docker push i.harbor.dragonest.net/drone/curl:0.0.2
```

至此一个标准的插件就编写完成了。


# 应用插件

上面我们编写了一个shell的插件，下面来应用一下该插件


在你的项目工程中根目录的`.drone.yml`文件中，添加一个pipeline：


```
pipeline:
  curl:
    image: i.harbor.dragonest.net/drone/curl:0.0.1
    url: http://172.16.254.224:8888/
    method: GET
    body: |
      hello world
```

在当前的pipeline中，`url`就表示的是环境变量的`PLUGIN_URL`, `method`就表示的是环境变量的`PLUGIN_METHOD`, 以此类推。

我们的`http://172.16.254.224:8888/`服务访问时会返回一个json内容为：

```
{
"result": "this is a test"
}
```

##### drone控制台输出效果


<img src="/images/docker/drone-plugin-demo.png" width="100%"/>




**其他语言的插件编写和shell基本一样，你需要关注的只是环境变量的注入，非常简单**