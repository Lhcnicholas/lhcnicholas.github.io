---
title: Mac上用Docker安装Redis集群
tags:
  - Redis
  - Docker
date: 2023-08-09 16:56:27
---


### 前言

前一阵闲来无事，想自己装个Redis集群玩玩，结果发现还挺费劲，中间遇到了不少麻烦，遂记录一下。



### Docker Compose配置

使用的是Docker Compose，配置文件如下:

```yaml
version: "3"

networks:
  redis-cluster:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

services:
  node1: 
    image: redis
    container_name: redis-cluster-1
    ports: 
      - 6371:6371
      - 16371:16371
    volumes:
      - ./compose/redis/redis-cluster.conf:/etc/redis.conf
    command: ["redis-server", "/etc/redis.conf"]
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.10
  node2: 
    image: redis
    container_name: redis-cluster-2
    ports: 
      - 6372:6372
      - 16372:16372
    volumes:
      - ./compose/redis/redis-cluster-2.conf:/etc/redis.conf
    command: ["redis-server", "/etc/redis.conf"]
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.2
  node3: 
    image: redis
    container_name: redis-cluster-3
    ports: 
      - 6373:6373
      - 16373:16373
    volumes:
      - ./compose/redis/redis-cluster-3.conf:/etc/redis.conf
    command: ["redis-server", "/etc/redis.conf"]
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.3
  node4: 
    image: redis
    container_name: redis-cluster-4
    ports: 
      - 6374:6374
      - 16374:16374
    volumes:
      - ./compose/redis/redis-cluster-4.conf:/etc/redis.conf
    command: ["redis-server", "/etc/redis.conf"]
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.4
  node5: 
    image: redis
    container_name: redis-cluster-5
    ports: 
      - 6375:6375
      - 16375:16375
    volumes:
      - ./compose/redis/redis-cluster-5.conf:/etc/redis.conf
    command: ["redis-server", "/etc/redis.conf"]
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.5
  node6: 
    image: redis
    container_name: redis-cluster-6
    ports: 
      - 6376:6376
      - 16376:16376
    volumes:
      - ./compose/redis/redis-cluster-6.conf:/etc/redis.conf
    command: ["redis-server", "/etc/redis.conf"]
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.6

```

要注意，我们要保证所有的redis节点都在同一个网络里，保证互相访问畅通。并且，这里我们要使用静态IP，因为后面创建集群的时候，它不支持使用节点名称访问，只能用IP。

### Redis配置

Redis集群模式下，Redis配置文件也是需要做一些修改的。

先看配置：

```sh
# 端口号（因为要映射到宿主机，端口不能重复）
port 6371～6376
# 开启集群模式
cluster-enabled yes

# 集群配置文件名，随便取个名字就好
cluster-config-file nodes.conf

# Docker NAT网络模式下，需要配置Redis集群节点对外暴露的IP和Port
# 这里我用的是宿主机IP，如果用Redis集群局域网IP的话，那这个集群就只能内部玩了，外面是无法正常使用的。
cluster-announce-ip 10.1.62.101
cluster-announce-port 6376
```

### 创建集群

各个Redis节点都启动成功后，随便进入一个Redis节点中，执行以下命令：

```sh
➜  yaml $ docker exec -it redis-cluster-1 bash
root@a7875167255d:/data# redis-cli --cluster create 10.1.62.101:6371 10.1.62.101:6372 10.1.62.101:6373
>>> Performing hash slots allocation on 3 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
M: 827efbbd222514e9310690c6d0c8013ca473935b 10.1.62.101:6371
   slots:[0-5460] (5461 slots) master
M: edd79c53980d8f56581a156d54db0ffad5f69bdb 10.1.62.101:6372
   slots:[5461-10922] (5462 slots) master
M: 70e470e76a1fb102a37da6507358cca6f276924a 10.1.62.101:6373
   slots:[10923-16383] (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 10.1.62.101:6371)
M: 827efbbd222514e9310690c6d0c8013ca473935b 10.1.62.101:6371
   slots:[0-5460] (5461 slots) master
M: edd79c53980d8f56581a156d54db0ffad5f69bdb 10.1.62.101:6372
   slots:[5461-10922] (5462 slots) master
M: 70e470e76a1fb102a37da6507358cca6f276924a 10.1.62.101:6373
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

当你看到` [OK] All 16384 slots covered.` 的时候，就说明咱们的集群创建成功了。

### 试一试

```sh
root@a7875167255d:/data# redis-cli -c -p 6371
127.0.0.1:6371> set name nic
-> Redirected to slot [5798] located at 10.1.62.101:6372
OK
10.1.62.101:6372> set age 18
-> Redirected to slot [741] located at 10.1.62.101:6371
OK
10.1.62.101:6371> keys *
1) "age"
```

