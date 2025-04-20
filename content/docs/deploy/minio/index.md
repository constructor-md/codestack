---
title: MinIO
draft: false
weight: 5
---

# MinIO
部署方式：单节点

运行方式：docker-compose

## 目录创建
```
sudo mkdir /DATA/minio /DATA/minio/data /DATA/minio/config
```

## docker-compose.yml
```
sudo vim /DATA/docker-compose.yml

services:
  minio:
    image: minio/minio
    restart: always
    mem_limit: 1G
    ports:
      - "9000:9000"
      - "19001:9001"
    container_name: minio
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=_admin123
      - MINIO_BROWSER_DEFAULT_LOCALE=zh_CN
    volumes:
      - /DATA/minio/data:/data
      - /DATA/minio/config:/root/.minio

```

>9000 是MInIO S3 API端口
>9001 是MinIO WebUI 控制台端口
>需要的节点才映射出来即可

## 启动
```
sudo docker-compose up -d minio
```

## 防火墙打开
```
sudo ufw allow 19001/tcp
```