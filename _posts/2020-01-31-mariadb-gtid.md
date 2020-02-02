---
layout: default
title: "MariaDB升级复制模式为GTID"
date: 2020-01-31 05:00:00 +0200
published: 2020-01-31 05:00:00 +0200
comments: true
categories: 数据库
tags: [database]
author: snakechen
---


<p>
我们在使用公有云Mariadb集群的时候，有时候由于业务需要，需要自建数据库从库，比如大数据拉取，比如短时数据库的快照状态的停留；
我们使用传统的复制模式建立自建从库的时候，当Mariadb集群发生Failover后会出现binlog的位置改变，从而导致自建从库被迫断开，此时我们需要将传统的复制模式改为GTID模式，
当Mariadb集群发生Failover时从库则不会断开。
</p>

<!--more-->




#### 1、[从库]从库在查看复制状态，注意复制延迟情况

```
SHOW SLAVE STATUS\G
- Relay_Master_Log_File: mysql-bin.000027
- Exec_Master_Log_Pos: 310094849
- Master_Log_File: mysql-bin.000027
- Read_Master_Log_Pos: 310094849
```

#### 2、[从库]在从库执行命令，停止当前复制

```
STOP SLAVE;
```

#### 3、[从库]查看并记录中继日志执行的位置信息

```
SHOW SLAVE STATUS\G
- Relay_Master_Log_File: mysql-bin.000027
- Exec_Master_Log_Pos: 310094849
```
#### 4、[主库]在主库查看从库执行位置对应的GTID位置

```
SELECT BINLOG_GTID_POS('mysql-bin.000027', 300050472);
+------------------------------------------------+
| BINLOG_GTID_POS('mysql-bin.000027', 300050472) |
+------------------------------------------------+
| 192-1681233-35567846                           |
+------------------------------------------------+

```
#### 5、[从库]回到从库设置GTID的开始位置，即第4步中从主库查询到的位置信息

```
SET GLOBAL gtid_slave_pos = '192-1681233-35567846';
```

#### 6、[从库]重新设置主库信息

```
CHANGE MASTER TO MASTER_HOST='192.168.1.233', # 主机
                 MASTER_PORT=3306,            # 端口
                 MASTER_USER='replicator',    # 用户
                 MASTER_PASSWORD='replpass',  # 密码
                 MASTER_USE_GTID=slave_pos;   # 位置
```

#### 7、[从库]启动复制

```
START SLAVE;

```