---
title: ElasticSearch
draft: false
weight: 4
---

# ElasticSearch
安装版本：8.15.0

部署方式：三节点

运行方式：docker-compose

## Linux系统设置
```
# 修改内核参数
# 设置每个进程最多拥有的最大内存映射区域数量 默认65536对ES来说不足
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 创建目录
```
sudo mkdir /DATA/es /DATA/es/data /DATA/es/logs /DATA/es/plugins /DATA/es/config

```

## 权限设置
```
sudo chmod -R 777 /DATA/es/data /DATA/es/logs /DATA/es/plugins /DATA/es/config
```

## 各单节点启动获取官方配置
```
sudo vim /DATA/docker-compose.yml

version: '3'
services:
  es:
      image: docker.elastic.co/elasticsearch/elasticsearch:8.15.0
      container_name: es
      restart: always
      environment:
        - "ES_JAVA_OPTS=-Xms4g -Xmx4g" # 宿主机最大内存的一半 再留点给其他应用
        - "ELASTIC_PASSWORD=bucunzaide2333"
        - "TZ=Asia/Shanghai"
      ports:
        - "19200:19200"
        - "19300:19300"
      volumes:
        - /DATA/es/data:/usr/share/elasticsearch/data
        - /DATA/es/logs:/usr/share/elasticsearch/logs
        - /etc/hosts:/etc/hosts
        - es_config:/usr/share/elasticsearch/config
        - /DATA/es/plugins:/usr/share/elasticsearch/plugins
      ulimits:
        # mmap 映射内存不限制
        memlock:
          soft: -1                  
          hard: -1
        # 文件描述符打开个数修改    
        nofile:                                          
          soft: 65535                             
          hard: 65535


# 末尾添加 作用是不要让宿主机空目录覆盖
volumes:
      es_config:
        driver: local
        driver_opts:
          type: none
          device: /DATA/es/config
          o: bind


```

> #mmap 映射内存不限制
>
> ES底层lucene底层使用mmap映射磁盘文件
>
> memlock本意是限制内存映射的大小，默认为64KB，不够，修改为不用限制大小
>
> #文件描述符打开个数修改
>
> Elasticsearch 需要打开大量的文件来处理索引数据、日志文件以及网络连接
>
> 如果文件描述符的数量不足，导致“Too many open files” 和频繁的文件关闭和重新打开操作，从而降低性能
>
> Elasticsearch 官方推荐将   nofile   的   soft   和   hard   值设置为 65535 或更高
>
> soft  （软限制）：是当前生效的限制值，用户可以动态调整，但不能超过   hard   限制。
>
> hard  （硬限制）：是系统允许的最大值，用户无法超过这个限制

```
# 启动 三个节点的es
sudo docker-compose up -d es

# 验证节点状态
curl --cacert /DATA/es/config/certs/http_ca.crt -u elastic:bucunzaide2333 https://localhost:19200/_cluster/health?pretty


```

## 生成和复制节点间传输层证书
```
# 进入一个节点的 docker
sudo docker exec -it es /bin/bash

./bin/elasticsearch-certutil ca
# Please enter the desired output file [elastic-stack-ca.p12]:
# 直接空格默认文件名和路径
# Enter password for elastic-stack-ca.p12 :
# 设置密码 2333

# 用刚生成的CA证书生成节点证书
# elastic-stack-ca.p12在/usr/share/elasticsearch 目录下 
# 在这个目录执行，elastic-certificates.p12也生成在这个目录下
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
# Enter password for CA (elastic-stack-ca.p12) : 
# 输入CA证书的密码 密码 2333
# Enter password for elastic-certificates.p12 :
# 输入节点证书的密码，设置密码 2333

# 生成的证书都移动到config/certs目录下
mv elastic-certificates.p12 config/certs/
mv elastic-stack-ca.p12 config/certs/

exit

# 各个节点都需要这两个文件 复制到各节点
scp -P 2222 quanta@192.168.1.11:/DATA/es/config/certs/elastic-certificates.p12 /DATA/es/config/certs
scp -P 2222 quanta@192.168.1.11:/DATA/es/config/certs/elastic-stack-ca.p12 /DATA/es/config/certs


# 密码加入各节点的信任库 覆盖原本的设置y
# 提示输入节点证书密码 2333
sudo docker exec -it es /bin/bash
./bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
./bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password

exit

# https证书保持各节点自己的 各类客户端访问不使用https
# 检查https是否可以访问 curl访问-k跳过证书验证 浏览器https高级继续 （不认自签名证书）
curl -k --cacert /DATA/es/config/certs/http_ca.crt -u elastic:bucunzaide2333 https://localhost:19200/_cluster/health?pretty


```

## 修改各节点 elasticsearch.yml 文件
```
sudo vim /DATA/es/config/elasticsearch.yml

# 集群名称 每个节点相同 根据他生成 cluster_uuid
# cluster_uuid 相同的加入同一个集群，修改后只能删除data目录重来
cluster.name: "es-cluster"
# 节点名称 每个节点不同
node.name: "es-node1"
# 节点主机名 在/etc/hosts换节点ip用
network.publish_host: dataserver1
network.host: 0.0.0.0
transport.port: 19300
http.port: 19200

#----------------------- BEGIN SECURITY AUTO CONFIGURATION -----------------------
#
# The following settings, TLS certificates, and keys have been automatically      
# generated to configure Elasticsearch security features on 28-02-2025 09:24:45
#
# --------------------------------------------------------------------------------

# Enable security features
xpack.security.enabled: true

xpack.security.enrollment.enabled: true

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
# 关闭https访问
xpack.security.http.ssl:
  enabled: false
  keystore.path: certs/http.p12

# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/elastic-certificates.p12
  truststore.path: certs/elastic-certificates.p12


# Create a new cluster with the current node only
# Additional nodes can still join the cluster later
cluster.initial_master_nodes: ["es-node1", "es-node2", "es-node3"]
discovery.seed_hosts: ["dataserver1", "dataserver2", "dataserver3"]
#----------------------- END SECURITY AUTO CONFIGURATION -------------------------



# 节点间复制并修改节点名和publish_host
scp -P 2222 user@ip:/DATA/es/config/elasticsearch.yml /DATA/es/config/elasticsearch.yml
sudo vim /DATA/es/config/elasticsearch.yml

# 修改各节点防火墙
sudo ufw allow 19200/tcp 
sudo ufw allow 19300/tcp
sudo ufw delete allow 9200/tcp 
sudo ufw delete allow 9300/tcp
# 重启各节点
sudo docker-compose down es
# 删除 data目录下的内容 因为修改了cluster_name
sudo rm -rf /DATA/es/data/*
sudo docker-compose up -d es
```

## 启动并验证
```
sudo docker-compose up -d es

# 验证节点状态
curl -u elastic:bucunzaide2333 http://localhost:19200/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  #  green  ：所有主分片和副本分片都已分配，集群状态良好。
  #  yellow  ：所有主分片已分配，但部分副本分片未分配。
  #  red  ：部分主分片未分配，集群状态不佳
  "status" : "green",
  # 本次查询是否超时
  "timed_out" : false,
  # 集群节点数量
  "number_of_nodes" : 3,
  # 集群数据节点数量
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
curl -u elastic:bucunzaide2333 http://localhost:19200/_cat/nodes?pretty
192.168.1.13 13 71 2 0.07 0.31 0.22 cdfhilmrstw - es-node3
192.168.1.12 14 71 2 0.24 0.38 0.26 cdfhilmrstw - es-node2
192.168.1.11 18 53 2 0.07 0.30 0.22 cdfhilmrstw * es-node1
```
## 设置内置用户和密码
```
sudo docker exec -it es /bin/bash

./bin/elasticsearch-reset-password -i -u kibana_system

# 多节点验证是否可以登录
curl -u kibana_system:bucunzaide2333 http://localhost:19200/_cat/nodes?pretty

```
> 用户和密码单独索引在security
> 数据目录挂载出来的情况下可以在docker重启后沿用

## 安装kibana(单节点)
### 目录和权限
```
sudo mkdir -p /DATA/kibana/config /DATA/kibana/data
sudo chmod -R 777 /DATA/kibana/config /DATA/kibana/data
```

### 启动kibana
```
sudo vim /DATA/docker-compose.yml

version: '3'
services:
  kibana:
      image: kibana:8.15.0
      container_name: kibana
      restart: always
      environment:
        - "TZ=Asia/Shanghai"
      ports:
        - "15601:15601"
      volumes:
      - kibana_config:/usr/share/kibana/config
      - /DATA/kibana/data:/usr/share/kibana/data
      - /etc/hosts:/etc/hosts


volumes:
      kibana_config:
        driver: local
        driver_opts:
          type: none
          device: /DATA/kibana/config
          o: bind

sudo docker-compose up -d kibana
```

### 修改 kibana.yml 配置文件
```
sudo vim /DATA/kibana/config/kibana.yml

server.host: "0.0.0.0"
server.port: 15601
server.shutdownTimeout: "5s"
elasticsearch.hosts: [ "http://dataserver1:19200" ]
elasticsearch.username: "kibana_system"
elasticsearch.password: "bucunzaide2333"
i18n.locale: "en"
monitoring.ui.container.elasticsearch.enabled: true


sudo docker-compose restart kibana
```
### 打开防火墙并验证访问
```
sudo ufw allow 15601/tcp

http://192.168.1.11:15601/

# 在 dev_tools 中尝试查询
# es集群健康状态
GET _cat/health?v
# es集群节点个数
GET _cat/nodes?v
# es 集群索引
GET _cat/indices?v
# 查看插件列表
GET /_cat/plugins?v
```

## 安装IK分词器(每个节点都要装)
```
# 安装 每个节点都需要装
sudo docker exec -it es /bin/bash
bin/elasticsearch-plugin install https://get.infini.cloud/elasticsearch/analysis-ik/8.15.0
exit
# 重启es
sudo docker-compose restart es

# 移除 ik 分词器
./bin/elasticsearch-plugin remove analysis-ik

# kibana 查看插件列表
GET /_cat/plugins?v
name     component   version
es-node2 analysis-ik 8.15.0
es-node1 analysis-ik 8.15.0
es-node3 analysis-ik 8.15.0
```

### 测试IK分词器
ik分词器提供了两种分词方式
```
# ik_smart 智能分词
# 会做最粗粒度的拆分, 比如会将"中华国歌"分成如下"中华", “国歌”
# 适合 Phrase 查询
POST /_analyze
{
  "analyzer": "ik_smart",
  "text": "我爱北京天安门"
}

# result
# 我 爱 北京 天安门
{
  "tokens": [
    {
      "token": "我",
      "start_offset": 0,
      "end_offset": 1,
      "type": "CN_CHAR",
      "position": 0
    },
    {
      "token": "爱",
      "start_offset": 1,
      "end_offset": 2,
      "type": "CN_CHAR",
      "position": 1
    },
    {
      "token": "北京",
      "start_offset": 2,
      "end_offset": 4,
      "type": "CN_WORD",
      "position": 2
    },
    {
      "token": "天安门",
      "start_offset": 4,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 3
    }
  ]
}


# ik_max_word 是最细粒度的分词模式，会尽可能多地输出分词结果 穷举所有可能 中文语法下的
# 适合 Term Query 查询
POST /_analyze
{
  "analyzer": "ik_max_word",
  "text": "我爱北京天安门"
}

# result
# 我 爱 北京 天安门 天安 门
{
  "tokens": [
    {
      "token": "我",
      "start_offset": 0,
      "end_offset": 1,
      "type": "CN_CHAR",
      "position": 0
    },
    {
      "token": "爱",
      "start_offset": 1,
      "end_offset": 2,
      "type": "CN_CHAR",
      "position": 1
    },
    {
      "token": "北京",
      "start_offset": 2,
      "end_offset": 4,
      "type": "CN_WORD",
      "position": 2
    },
    {
      "token": "天安门",
      "start_offset": 4,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 3
    },
    {
      "token": "天安",
      "start_offset": 4,
      "end_offset": 6,
      "type": "CN_WORD",
      "position": 4
    },
    {
      "token": "门",
      "start_offset": 6,
      "end_offset": 7,
      "type": "CN_CHAR",
      "position": 5
    }
  ]
}

```
