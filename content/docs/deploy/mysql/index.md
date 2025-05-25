---
title: MySQL
draft: false
weight: 2
---

# MySQL
安装版本：MySQL 8.0.20

部署方式：一主二从

运行方式：docker-compose

## 各节点安装
### 创建MySQL目录
```
# 放置所有mysql相关文件 比如my.cnf
sudo mkdir /DATA/mysql
# 放置mysql数据文件 也作为mysql
sudo mkdir /DATA/mysql/mysql
```

### 创建MySQL用户并设置权限
```
# 创建mysql用户 设置为不可登陆系统 并设置用户的主目录为/DATA/mysql/mysql
sudo useradd -r -s /sbin/nologin -d /DATA/mysql/mysql mysql
# 指定mysql用户的主目录为/DATA/mysql/mysql
sudo usermod -d /DATA/mysql/mysql mysql
# 递归地将/DATA/mysql/mysql目录及其所有子目录和文件的所有者和所属组设置为mysql用户和mysql组
sudo chown -R mysql:mysql /DATA/mysql/mysql
# 递归地将/DATA/mysql/mysql目录及其子目录设和文件的权限设置为755
# 755：所有者有读写和执行权限，组用户和其他用户有读和执行权限
sudo chmod -R 755 /DATA/mysql/mysql
# 查找/etc/passwd文件中包含mysql的行 /etc/passwd是系统用户信息文件，包含所有用户的基本信息
grep mysql /etc/passwd

```

操作失误时的可选操作
```
# 删除用户及其主目录 没有r不删除主目录
sudo userdel -r mysql
# 手动删除主目录
sudo rm -rf /DATA/mysql
# 检查 
grep mysql /etc/passwd
# 删除用户组
sudo groupdel mysql
# 检查用户组是否删除
grep mysql /etc/group
```

### docker-compose文件
```
# 创建docker-compose文件在/DATA下，或者追加在已有文件中
vim docker-compose.yml
```
docker-compose.yml
```
version: '3'  # 使用docker-compose版本3
services:  # 定义服务
  mysql8:  # 定义一个名为mysql8的服务
    image: mysql:8.0.20  # 使用MySQL 8.0.20镜像
    container_name: mysql8  # 指定容器名称为mysql8
    restart: always  # 系统重启时自动重新启动
    cap_add:
      - SYS_NICE
    ports:  # 定义容器和主机之间的端口映射
      - "33060:3306"  # 将容器的3306端口映射到主机的33060端口
    environment:  # 定义环境变量
      MYSQL_ROOT_PASSWORD: "password"  # 设置root用户的密码
    volumes:  # 定义数据卷
      - /DATA/mysql/mysql:/var/lib/mysql # 挂载数据目录
      - /DATA/mysql/my.cnf:/etc/mysql/my.cnf # 挂载my.cnf
    healthcheck:
      test: ["CMD", "mysqladmin" ,"ping", "-h", "localhost", "-u", "user", "-ppassword"]  # 设置容器健康检查命令
      interval: 20s  # 每隔20秒进行一次健康检查
      timeout: 10s  # 健康检查超时时间为10秒
      retries: 3  # 健康检查失败时重试次数为3次

```
### 服务器间文件复制
```
# scp [源文件路径] [目标服务器用户名]@[目标服务器IP]:[目标路径]
scp -P [ssh port] /home/user/file.txt user@192.168.1.2:/home/user/
# scp [源服务器用户名]@[源服务器IP]:[源文件路径] [目标路径]
scp -P [ssh port] user@192.168.1.2:/home/user/file.txt /home/user/

```
授权文件
```
sudo chmod 777 /DATA/docker-compose.yml
```

### my.cnf文件
```
sudo vim /DATA/mysql/my.cnf
```
my.cnf
```
[mysqld]
# 指定数据目录
datadir         = /var/lib/mysql
# 安全文件前缀
secure-file-priv = NULL
# 不进行域名解析
skip-name-resolve
# 默认存储引擎
default-storage-engine = InnoDB
# 服务器监听的ip地址
bind-address = 0.0.0.0
# 服务器监听的端口号
port = 3306

# 连接配置
# 最大连接数
max_connections = 2000
# 单个用户最大连接数
max_user_connections = 1000  
# 最大连接错误数
max_connect_errors = 4000  
# 空闲连接的超时时间
wait_timeout = 300  
# 交互式连接的超时时间 数据库管理工具的连接 那些手动执行SQL的可能长时间空闲
interactive_timeout = 600  
 # 最大允许的数据包大小 避免一次查出太多数据打崩数据库
max_allowed_packet = 32M 

# 日志设置
# 使用短格式记录日志
log-short-format = 1  
# 启用慢查询日志
slow_query_log 
# 慢查询的时间阈值 2s及以上的查询被作为慢查询记录
long_query_time = 2  

[mysqldump]
quick  # 快速导出数据
max_allowed_packet = 16M  # 最大允许的数据包大小

[mysqlhotcopy]
interactive-timeout = 3600  # 交互式超时时间，超时时间设置为 1 小时

```
授权文件
```
# 所有者读写，组成员和其他用户可读不可写，完全不可执行
sudo chmod 644 /DATA/mysql/my.cnf
```

### docker compose运行镜像
```
sudo docker-compose up -d mysql8
```

### docker compose停止镜像
```
sudo docker-compose stop mysql8
```

### docker compose重启镜像
```
sudo docker-compose restart mysql8
```

### 日志查看
```
sudo docker logs mysql8
```

### 防火墙配置
```
sudo ufw allow 33060/tcp
```

### root用户限制为局域网访问
```
# 进入docker
sudo docker exec -it mysql8 /bin/bash

# 登录mysql
mysql -u root -p your_password
# 使用mysql数据库
use mysql;
# 删除'root'@'%' 不删除'root'@'localhost' 保留本机访问权限
DROP USER 'root'@'%';
# 创建'root'@'192.168.1.*' 建立局域网访问权限
CREATE USER 'root'@'192.168.1.%' IDENTIFIED BY '*******';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'192.168.1.%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

```

## 主从配置
### 主节点配置
#### my.cnf配置
```
sudo vim /DATA/mysql/my.cnf
```
要在[mysqld]部分添加
```
# 主从配置 - 主节点
# MySQL服务id 各节点不同
server_id = 1010
# 开启二进制日志
log-bin = mysql-bin
# 日志格式
binlog_format = ROW

```

#### 重启MySQL服务
```
# 要在所需docker-compose.yml文件同文件夹下执行
sudo docker-compose restart mysql8
```

#### 检查server-id配置确保生效
```
# 连接数据库执行
SHOW VARIABLES LIKE 'server_id';
```

#### 创建复制用户并赋予权限
```
CREATE USER 'repl'@'192.168.1.%' IDENTIFIED  WITH mysql_native_password BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.1.%';
FLUSH PRIVILEGES;

```
#### 查看主服务器的二进制日志信息并记录
```
SHOW master status
# 记录File和Position值
```

### 从节点配置
#### my.cnf配置
```
sudo vim /DATA/mysql/my.cnf
```
要在[mysqld]部分添加
```
# 主从配置 - 从节点
# MySQL服务标识id 每个节点不同 大于0
server_id = 1020
# 从节点不建议开启二进制日志
# 从节点设置为只读 然后在应用层做数据源切换 读写分离
# 仅read-only， root用户还是可以建表，普通用户只读
# 5.7.8后添加了super_read_only root用户也不可以执行读以外的操作
super_read_only = 1
```

#### 重启MySQL服务
```
sudo docker-compose restart mysql8
```

#### 配置从服务器连接主服务器
登录从服务器的MySQL数据库执行
```
CHANGE MASTER TO
    MASTER_HOST = '192.168.1.x',  # 主服务器的 IP 地址
    MASTER_PORT = '33060',        # 主服务器MySQL端口
    MASTER_USER = 'repl',           # 复制用户的用户名
    MASTER_PASSWORD = 'repl_password',  # 复制用户的密码
    MASTER_LOG_FILE = 'mysql-bin.00000x',  # 主服务器的二进制日志文件名
    MASTER_LOG_POS = 1234;          # 主服务器的二进制日志位置
# CHANGE MASTER TO之后，有任何SQL执行都会导致MASTER_LOG_POS变化
# 需要再次查看Master Status，停止复制修改后，再开始复制
```
MASTER_LOG_FILE   和   MASTER_LOG_POS   的值从主服务器的   SHOW MASTER STATUS;   命令中获取

#### 启动从服务器的复制进程
```
START SLAVE;
```

#### 停止从服务器的复制进程
```
STOP SLAVE;
```

#### 检查从服务器的复制状态
```
SHOW SLAVE STATUS;
# 确保Slave_IO_Running和Slave_SQL_Running都是Yes 
```

#### 验证主从复制
主服务器写数据，查看从服务器是否变更

## 一些常见操作

### 配置时主节点常用命令
```
SHOW VARIABLES LIKE 'server_id';

SHOW master status;

SHOW GRANTS FOR 'repl'@'192.168.1.%';

DROP USER 'repl'@'192.168.1.%'
CREATE USER 'repl'@'192.168.1.%' IDENTIFIED  WITH mysql_native_password BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.1.%';
FLUSH PRIVILEGES;
```

### 配置时从节点常用命令
```
SHOW VARIABLES LIKE 'server_id';

CHANGE MASTER TO
    MASTER_HOST = '192.168.1.11',
    MASTER_PORT = 33060,
    MASTER_USER = 'repl',
    MASTER_PASSWORD = 'repl_password',
    MASTER_LOG_FILE = 'mysql-bin.000001',
    MASTER_LOG_POS = 1739;

STOP SLAVE;

START SLAVE;

SHOW SLAVE STATUS;
```

### 登陆的是从节点，查找主节点IP
```
SHOW SLAVE STATUS;
# Master_Host 和 Master_Port
```

### 二进制日志格式的区别
1. ROW（5.7之后官方默认的选择）：
基于行的复制，记录的是每行的变化，可读性差，文件大，传输慢，但可以应对各种情况的影响
- 对于   INSERT   操作，记录插入的每一行数据
- 对于   UPDATE   操作，记录修改前和修改后的每一行数据
- 对于   DELETE   操作，记录被删除的每一行数据
绝对的数据一致性保证
2. STATEMENT：最初的MySQL主从复制模式，不推荐选择
记录的是SQL，文件小传输快
- 遇到非确定函数如NOW RAND UUID等，主从库执行的结果会不同
- 遇到存储过程和触发器可能无法正确复制
- 遇到用户自定义函数，可能会在主从库产生不同的效果
- 某些场景下 LAST_INSERT_ID()可能在主从库上值不一致
3. MIXED：混合模式，一些情况下可能有人选择
- 普通SQL记录为STATEMENT形式，有可读性
- 遇到非确定性函数等转换为ROW格式记录
- 复杂场景下如存储过程、触发器，可能会导致数据不一致

如果对数据一致性要求极高，且对日志大小和性能的影响可以接受，建议使用   ROW   格式。↑

如果需要在性能和一致性之间取得平衡，且希望减少日志大小和管理复杂性，建议使用   MIXED   格式

完全不要用STATEMENT格式


### 一主多从和多主多从的配置区别
- 多主多从：
主节点之间互为主从，可能出现数据冲突需要在应用层避免，或者引入冲突解决机制（如conflict-resolution 插件）

从节点要设置每个主节点为主节点，为每个主节点单独配置复制通道，便于从多个主节点获取数据（多源复制）