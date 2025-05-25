---
title: Redis
draft: false
weight: 3
---

# Redis
安装版本：7.0.4

部署方式：一主二从三哨兵

节点分配：每节点一个Redis-Server，一个Sentinel

运行方式：docker-compose


## 目录和用户权限
### 创建Redis和Sentinel目录
```
# 存放所有Redis相关文件
sudo mkdir /DATA/redis
# 存放Redis和Sentinel配置文件
sudo mkdir /DATA/redis/config
# 存放Sentinel配置和文件
sudo mkdir /DATA/redis/config/sentinel
# 存放挂载docker内运行Redis的数据目录
sudo mkdir /DATA/redis/data

```

### 创建redis用户并设置工作目录
```
# 创建 redis 用户 设置为不可登陆系统 并设置用户的主目录为/DATA/redis/data
sudo useradd -r -s /sbin/nologin -d /DATA/redis/data redis
# 指定 redis 用户的主目录为/DATA/redis/data
sudo usermod -d /DATA/redis/data redis
# 递归地将/DATA/redis/data目录及其所有子目录和文件的所有者和所属组设置为redis用户和redis组
sudo chown -R redis:redis /DATA/redis/data
# 递归地将/DATA/redis/data目录及其子目录设和文件的权限设置为755
# 755：所有者有读写和执行权限，组用户和其他用户有读和执行权限
sudo chmod -R 755 /DATA/redis/data
# 查找/etc/passwd文件中包含mysql的行 /etc/passwd是系统用户信息文件，包含所有用户的基本信息
grep redis /etc/passwd

```

## 主节点配置
### redis.conf
```
sudo vim /DATA/redis/config/redis.conf


# 监听端口
port 63790
# 访问密码
requirepass password
# 数据库数量 使用cluster模式时只会有一个database即DB0
databases 16


# 绑定本机的网络接口（网卡） 绑定的是网卡的IP地址
# 0.0.0.0 监听所有 默认127.0.0.1
bind 0.0.0.0

# 默认开启
# 如果没有设置密码和且没有设置bind，只允许本机访问
protected-mode yes

# 单位秒，timeout时间内客户端没有数据交互，关闭连接
timeout 60

# 客户端同时连接的最大数量 默认10000
# 达到最大值时关闭新连接并返回max number of clients reached
maxclients 1000

# 内存管理 
# 最大内存，推荐最大设置为6GB
# 不要设置过大内存，防止执行RDB内存快照文件或者AOF重写时因为数据太大阻塞太长时间
maxmemory 2GB

# 内存淘汰策略 默认noeviction
# noeviction -> 不删除任何 key，内存满了直接返回报错
# 默认情况下slave节点会忽略maxmemory配置，除非被提升为master
# 只有master会执行内存淘汰策略，master删除key后会发送DEL指令给slave
maxmemory-policy noeviction

# 过期key滞留在内存的比例 默认值为1 表示10%
# 设置的越小，一次淘汰周期需要消耗的CPU更多 需要删除更多的过期数据
active-expire-effort 1

# 持久化
# AOF持久化开启
appendonly yes

# AOF 持久化模式，默认为 "always"。可以是 always、everysec 或 no
# always：每个写操作都立即同步到磁盘，最费性能
# everysec：每秒钟同步一次到磁盘，折中的选择
# no：完全依赖操作系统的行为，可能会丢失数据，但性能最高
appendfsync everysec

# AOF-RDB混合持久化
# 配置成yes必须先开启AOF AOF重写生成的文件将同时包含RDB和AOF格式内容
# 推荐开启
aof-use-rdb-preamble yes

# 性能监控
# 慢查询日志 执行时间只是命令阶段的时间，不包括建立连接发送回复等
# slow log 仅保存在内存中，效率很高
# 执行时间大于多少微秒的查询进行记录 1s = 1,000,000微秒 默认10000
slowlog-log-slower-than 10000

# 最多保存多少条慢查询日志 slowlog本身是FIFO 默认128
slowlog-max-len 128

```

### sentinel.conf
```
sudo vim /DATA/redis/config/sentinel/sentinel.conf

# 哨兵端口
port 26379

# 监控的redis主节点的ip port
# master-name 自定义
# quorum 多少个sentinel主观认为master失联，认为客观上master失联
# sentinel monitor <master-name> <ip> <redis-port> <quorum>
sentinel monitor mymaster 192.168.1.11 63790 2

# redis实例的密码 主从的访问密码必须要一样
sentinel auth-pass mymaster password

# 指定多少毫秒之后主节点没有应答哨兵
# 此时哨兵主观上认为主节点下线
# 默认30秒
# sentinel down-after-milliseconds <master-name> <milliseconds>
sentinel down-after-milliseconds mymaster 30000

# 设置故障转移时，从节点同步新主节点数据的并发数量
# 值越小，对主节点的压力越小，但同步速度可能较慢
# sentinel parallel-syncs <master-name> <numslaves>
sentinel parallel-syncs mymaster 1

# 设置故障转移的超时时间（单位：毫秒）
# 如果故障转移在这个时间内没有完成，则认为失败
sentinel failover-timeout mymaster 180000

# 配置哨兵自身的ip 避免走自动检测给出其他哨兵访问不到的地址
sentinel announce-ip 192.168.1.11
sentinel announce-port 36379
```

### docker-compose.yml
```
sudo vim /DATA/docker-compose.yml

version: '3'  # 使用docker-compose版本3
services:  # 定义服务
    redis7:
      image: redis:7.0.4
      container_name: redis7
      user: "996:986"
      restart: always
      ports:
        - 63790:63790
      environment:
        TZ: "Asia/Shanghai"
      volumes:
        - /DATA/redis/config/redis.conf:/etc/redis/redis.conf
        - /DATA/redis/data:/data
      command: ["redis-server", "/etc/redis/redis.conf"]
    sentinel:
      image: redis:7.0.4
      container_name: sentinel
      restart: always
      ports:
        - 36379:26379
      volumes:
        - /DATA/redis/config/sentinel:/etc/redis/config/sentinel
      environment:
        TZ: "Asia/Shanghai"
      command: ["redis-sentinel", "/etc/redis/config/sentinel/sentinel.conf"]
```

### docker-compose.yml语法验证
```
docker-compose config
```

## 子节点配置
### redis.conf
```
sudo vim /DATA/redis/config/redis.conf

# 监听端口 sentinel不知道外面映射啥端口，只好把内外端口设置一样
port 63790
# 访问密码
requirepass password
# 数据库数量 使用cluster模式时只会有一个database即DB0
databases 16


# 绑定本机的网络接口（网卡） 绑定的是网卡的IP地址
# 0.0.0.0 监听所有 默认127.0.0.1
bind 0.0.0.0

# 默认开启
# 如果没有设置密码和且没有设置bind，只允许本机访问
protected-mode yes

# 单位秒，timeout时间内客户端没有数据交互，关闭连接
timeout 60

# 客户端同时连接的最大数量 默认10000
# 达到最大值时关闭新连接并返回max number of clients reached
maxclients 1000

# 内存管理 
# 最大内存，推荐最大设置为6GB
# 不要设置过大内存，防止执行RDB内存快照文件或者AOF重写时因为数据太大阻塞太长时间
maxmemory 2GB

# 内存淘汰策略 默认noeviction
# noeviction -> 不删除任何 key，内存满了直接返回报错
# 默认情况下slave节点会忽略maxmemory配置，除非被提升为master
# 只有master会执行内存淘汰策略，master删除key后会发送DEL指令给slave
maxmemory-policy noeviction

# 过期key滞留在内存的比例 默认值为1 表示10%
# 设置的越小，一次淘汰周期需要消耗的CPU更多 需要删除更多的过期数据
active-expire-effort 1

# 持久化
# AOF持久化开启
appendonly yes

# AOF 持久化模式，默认为 "always"。可以是 always、everysec 或 no
# always：每个写操作都立即同步到磁盘，最费性能
# everysec：每秒钟同步一次到磁盘，折中的选择
# no：完全依赖操作系统的行为，可能会丢失数据，但性能最高
appendfsync everysec

# AOF-RDB混合持久化
# 配置成yes必须先开启AOF AOF重写生成的文件将同时包含RDB和AOF格式内容
# 推荐开启
aof-use-rdb-preamble yes

# 性能监控
# 慢查询日志 执行时间只是命令阶段的时间，不包括建立连接发送回复等
# slow log 仅保存在内存中，效率很高
# 执行时间大于多少微秒的查询进行记录 1s = 1,000,000微秒 默认10000
slowlog-log-slower-than 10000

# 最多保存多少条慢查询日志 slowlog本身是FIFO 默认128
slowlog-max-len 128


# 主从复制
# replicaof <masterip> <masterport>将当前实例成为master的从节点
replicaof 192.168.1.11 63790

# master节点的requiepass
masterauth **********

# 从节点只读，默认为yes，建议保留默认配置
replica-read-only yes

# slave每10s Ping一次master
repl-ping-replica-period 10

# slave与master之间的复制超时时间，默认60s
repl-timeout 60

# slave优先级 哨兵使用 默认100
# master节点挂掉，哨兵选择priority最小的slave节点作为新的master
replica-priority 100

```

### sentinel.conf
```
sudo scp -P 2222 quanta@192.168.1.11:/DATA/redis/config/sentinel.conf /DATA/redis/config/sentinel.conf
```

修改配置sentinel ip
```
sudo vim /DATA/redis/config/sentinel/sentinel.conf

# 配置哨兵自身的ip 避免走自动检测给出其他哨兵访问不到的地址
sentinel announce-ip 192.168.1.1x
sentinel announce-port 36379
```

### docker-compose.yml
```
sudo vim /DATA/docker-compose.yml

version: '3'  # 使用docker-compose版本3
services:  # 定义服务
  redis7:
    image: redis:7.0.4
    container_name: redis7
    restart: always
    ports:
      - 63811:6379
    # 指定时区，保证容器内时间正确
    environment:
      TZ: "Asia/Shanghai"
    volumes:
      # 映射配置文件和数据目录
      - /DATA/redis/redis-master.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-master:/data
    #sysctls:
    #  net.core.somaxconn: '511'
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
  sentinel:
    image: redis:7.0.4
    container_name: redis-sentinel-1
    restart: always
    ports:
      - 26379:26379
    volumes:
      - ./s1/:/usr/local/etc/redis/conf/
    # 指定时区，保证容器内时间正确
    environment:
      TZ: "Asia/Shanghai"
    # sysctls:
    #   net.core.somaxconn: '511'
    command: ["redis-sentinel", "/usr/local/etc/redis/conf/redis-sentinel-1.conf"]
```
### docker-compose.yml语法验证
```
docker-compose config
```

## 各节点启动
```
cd /DATA
sudo docker-compose up -d redis7
sudo docker-compose up -d sentinel

# 停止容器
sudo docker-compose stop redis7
sudo docker-compose stop sentinel
# 删除容器以清空容器的控制台日志
sudo docker rm redis7
sudo docker rm sentinel

```

### 打开防火墙端口
```
# 开放Redis端口给哨兵访问
sudo ufw allow 63790/tcp
sudo ufw deny 6379/tcp

# 开放哨兵端口
sudo ufw allow 36379/tcp
sudo ufw deny 26379/tcp
```

### 验证哨兵
```
# 登录sentinel docker
sudo docker exec -it sentinel /bin/bash

# 登录sentinel
redis-cli -p 26379

# 查看信息
# Sentinel部分 master行 显示了masterip:port slave数量 sentinel数量
info

# 看看哨兵认为目前有哪些主节点、从节点、哨兵节点
SENTINEL MASTER mymaster
SENTINEL SLAVES mymaster
SENTINEL SENTINELS mymaster

# 强制刷新主从信息
SENTINEL RESET mymaster

# sentinel会把一些信息在sentinel.conf里面记录下来
# 包括选举次数 每次选举结果节点信息等
# Sentinel重启时会根据sentinel.conf的记录
# 将上次选举的结果作为主节点
# 想要日志里面把选举次数、当前主节点清空，就要清理sentinel.conf
```

### 日志说明
```
1:X 17 Feb 2025 20:44:53.226 * Running mode=sentinel, port=26379.
1:X 17 Feb 2025 20:44:53.228 * Sentinel new configuration saved on disk
1:X 17 Feb 2025 20:44:53.228 # Sentinel ID is 6b8d5f11db078b21d5251456f5f7d5b4c117c3c9
# 主节点监控
1:X 17 Feb 2025 20:44:53.228 # +monitor master mymaster 192.168.1.11 63790 quorum 2
# 从节点监控
1:X 17 Feb 2025 20:45:03.273 * +slave slave 192.168.1.12:63790 192.168.1.12 63790 @ mymaster 192.168.1.11 63790
1:X 17 Feb 2025 20:45:03.277 * Sentinel new configuration saved on disk
# sentinel节点互相发现
1:X 17 Feb 2025 20:45:03.617 * +sentinel sentinel cb3cce637d7611c93785059ebd3dd6c4a5a5d6ad 192.168.1.12 36379 @ mymaster 192.168.1.11 63790
1:X 17 Feb 2025 20:45:03.620 * Sentinel new configuration saved on disk
# sentinel节点互相发现
1:X 17 Feb 2025 20:45:12.053 * +sentinel sentinel b4979ced54d276d73a9a16fb9ad1935e0e8a5e20 192.168.1.13 36379 @ mymaster 192.168.1.11 63790
1:X 17 Feb 2025 20:45:12.056 * Sentinel new configuration saved on disk
# 从节点监控
1:X 17 Feb 2025 20:45:13.285 * +slave slave 192.168.1.13:63790 192.168.1.13 63790 @ mymaster 192.168.1.11 63790

```

### 验证主从
```
# 主从节点登录redis
sudo docker exec -it redis7 /bin/bash
redis-cli -p 63790 -a **********
info replication


```

## 故障转移测试
### 停机操作
```
# 停止主节点
sudo docker-compose stop redis7
# 查看sentinel日志
sudo docker logs --tail 100 -f sentinel


```
### 日志说明
```
1:X 17 Feb 2025 20:45:13.290 * Sentinel new configuration saved on disk
# 主观下线
1:X 17 Feb 2025 21:06:53.259 # +sdown master mymaster 192.168.1.11 63790
# 两个节点投票 客观下线
1:X 17 Feb 2025 21:06:53.315 # +odown master mymaster 192.168.1.11 63790 #quorum 2/2
# 第一次选举
1:X 17 Feb 2025 21:06:53.315 # +new-epoch 1
# 尝试对主节点故障转移
1:X 17 Feb 2025 21:06:53.315 # +try-failover master mymaster 192.168.1.11 63790
1:X 17 Feb 2025 21:06:53.318 * Sentinel new configuration saved on disk
# 本哨兵投票
1:X 17 Feb 2025 21:06:53.318 # +vote-for-leader 6b8d5f11db078b21d5251456f5f7d5b4c117c3c9 1
# 其他哨兵投票
1:X 17 Feb 2025 21:06:53.324 # b4979ced54d276d73a9a16fb9ad1935e0e8a5e20 voted for 6b8d5f11db078b21d5251456f5f7d5b4c117c3c9 1
# 其他哨兵投票
1:X 17 Feb 2025 21:06:53.325 # cb3cce637d7611c93785059ebd3dd6c4a5a5d6ad voted for 6b8d5f11db078b21d5251456f5f7d5b4c117c3c9 1
# 为旧主节点的故障转移 选举领导者
1:X 17 Feb 2025 21:06:53.385 # +elected-leader master mymaster 192.168.1.11 63790
# 为旧主节点的故障转移 选择合适的从节点
1:X 17 Feb 2025 21:06:53.385 # +failover-state-select-slave master mymaster 192.168.1.11 63790
# 为旧主节点的故障转移 选择了合适的从节点13 从节点13属于旧主节点11
1:X 17 Feb 2025 21:06:53.452 # +selected-slave slave 192.168.1.13:63790 192.168.1.13 63790 @ mymaster 192.168.1.11 63790
# 向选中的节点发送slaveof no one命令，让它不再作为其他节点的从节点，为提升为主节点做准备
1:X 17 Feb 2025 21:06:53.452 * +failover-state-send-slaveof-noone slave 192.168.1.13:63790 192.168.1.13 63790 @ mymaster 192.168.1.11 63790
# 等待13节点完成提升为主节点的操作
1:X 17 Feb 2025 21:06:53.508 * +failover-state-wait-promotion slave 192.168.1.13:63790 192.168.1.13 63790 @ mymaster 192.168.1.11 63790
1:X 17 Feb 2025 21:06:54.305 * Sentinel new configuration saved on disk
# 13成功提升为主节点 之前是11的从节点
1:X 17 Feb 2025 21:06:54.305 # +promoted-slave slave 192.168.1.13:63790 192.168.1.13 63790 @ mymaster 192.168.1.11 63790
# 进入故障转移的重新配置从节点阶段 原主节点是11
1:X 17 Feb 2025 21:06:54.305 # +failover-state-reconf-slaves master mymaster 192.168.1.11 63790
# 向原先11的从节点12 发送了重新配置的命令
1:X 17 Feb 2025 21:06:54.369 * +slave-reconf-sent slave 192.168.1.12:63790 192.168.1.12 63790 @ mymaster 192.168.1.11 63790
# 从节点12正在更新自己的配置指向新的主节点
1:X 17 Feb 2025 21:06:54.517 * +slave-reconf-inprog slave 192.168.1.12:63790 192.168.1.12 63790 @ mymaster 192.168.1.11 63790
# 从节点12完成了重新配置操作 指向了新的主节点
1:X 17 Feb 2025 21:06:54.517 * +slave-reconf-done slave 192.168.1.12:63790 192.168.1.12 63790 @ mymaster 192.168.1.11 63790
# 旧节点11的故障转移操作结束
1:X 17 Feb 2025 21:06:54.583 # +failover-end master mymaster 192.168.1.11 63790
# 记录主节点切换完毕 原主节点11切换为新主节点13
1:X 17 Feb 2025 21:06:54.583 # +switch-master mymaster 192.168.1.11 63790 192.168.1.13 63790
# 记录12为从节点
1:X 17 Feb 2025 21:06:54.372 * +slave slave 192.168.1.12:63790 192.168.1.12 63790 @ mymaster 192.168.1.13 63790
# 记录11变为从节点 虽然下线了
1:X 17 Feb 2025 21:06:54.372 * +slave slave 192.168.1.11:63790 192.168.1.11 63790 @ mymaster 192.168.1.13 63790
1:X 17 Feb 2025 21:06:54.378 * Sentinel new configuration saved on disk
# 记录11节点主观下线 +是增加主观下线标记
1:X 17 Feb 2025 21:07:24.410 # +sdown slave 192.168.1.11:63790 192.168.1.11 63790 @ mymaster 192.168.1.13 63790
```

### 故障恢复
```
# 重启旧主节点
sudo docker-compose start redis7
# 查看sentinel日志
sudo docker logs --tail 100 -f sentinel

```

### 日志说明
```
# 移除11 的主观下线标记
1:X 17 Feb 2025 21:45:58.909 # -sdown slave 192.168.1.11:63790 192.168.1.11 63790 @ mymaster 192.168.1.13 63790
# 转换11为新主节点13的从节点
1:X 17 Feb 2025 21:46:08.884 * +convert-to-slave slave 192.168.1.11:63790 192.168.1.11 63790 @ mymaster 192.168.1.13 63790
```

### 服务发现和故障转移过程（哨兵选举）
服务发现
1. sentinel根据配置指定的master监控master并通信
2. master知道slave的信息
3. sentinel根据redis的sub/pub机制获取其他sentinel信息

故障转移
1. 主节点下线后，一段时间sentinel ping主节点不通，记录主节点主观下线，并通知其他sentinel
2. 认为主节点主观下线的sentinel个数超过quorm个数，也就是半数以上，标记主节点客观下线，开启故障转移流程
3. 各sentinel开始投票选择节点作为主节点，并选择一个sentinel协调故障转移流程
4. 获得选票过半的节点被选中准备提升为主节点
5. sentinel向被选中节点发送提升主节点命令，并等待其完成转换
6. 被选中节点提升为新主节点后，sentinel向其他节点发送转换新主节点的信息
7. 其他节点完成转换后，集群主节点状态转换完毕
8. 旧主节点被记录为新主节点的从节点
9. 旧主节点上线后，sentinel告知他将自己转换为新主节点的从节点