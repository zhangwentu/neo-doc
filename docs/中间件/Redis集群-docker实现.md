



一、简介

redis主要有三种集群模式

**1. 主从复制**

同Mysql主从复制的原因一样，Redis虽然读取写入的速度都特别快，但是也会产生读压力特别大的情况。为了分担读压力，Redis支持主从复制，读写分离。一个Master可以有多个Slaves。

优点

- 数据备份
- 读写分离，提高服务器性能

缺点

- 不能自动故障恢复,RedisHA系统（需要开发）
- 无法实现动态扩容

<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm70d5d7vxj30xq0i80tu.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm70d5d7vxj30xq0i80tu.jpg" width=""/>
    </a>
</p> 


**2. 哨兵机制**



Redis Sentinel是社区版本推出的原生`高可用`解决方案，其部署架构主要包括两部分：Redis Sentinel集群和Redis数据集群。

其中Redis Sentinel集群是由若干Sentinel节点组成的分布式集群，可以实现故障发现、故障自动转移、配置中心和客户端通知。Redis Sentinel的节点数量要满足2n+1（n>=1）的奇数个。

优点

- 自动化故障恢复

缺点

- Redis 数据节点中 slave 节点作为备份节点不提供服务
- 无法实现动态扩容

<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm705izsoxj30xv0u078x.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm705izsoxj30xv0u078x.jpg" width=""/>
    </a>
</p> 


**3. Redis-Cluster**



Redis Cluster是社区版推出的Redis分布式集群解决方案，主要解决Redis分布式方面的需求，比如，当遇到单机内存，并发和流量等瓶颈的时候，Redis Cluster能起到很好的负载均衡的目的。

Redis Cluster着眼于`提高并发量`。

群集至少需要3主3从，且每个实例使用不同的配置文件。

在redis-cluster架构中，`redis-master节点一般用于接收读写，而redis-slave节点则一般只用于备份`， 其与对应的master拥有相同的slot集合，若某个redis-master意外失效，则再将其对应的slave进行升级为临时redis-master。

在redis的官方文档中，对redis-cluster架构上，有这样的说明：在cluster架构下，默认的，一般redis-master用于接收读写，而redis-slave则用于备份，`当有请求是在向slave发起时，会直接重定向到对应key所在的master来处理`。 但如果不介意读取的是redis-cluster中有可能过期的数据并且对写请求不感兴趣时，则亦可通过`readonly`命令，将slave设置成可读，然后通过slave获取相关的key，达到读写分离。具体可以参阅redis[官方文档](https://redis.io/commands/readonly)等相关内容

优点

- 解决分布式负载均衡的问题。具体解决方案是分片/虚拟槽slot。
- 可实现动态扩容
- P2P模式，无中心化

缺点

- 为了性能提升，客户端需要缓存路由表信息
- Slave在集群中充当“冷备”，不能缓解读压力

<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm705tidgej317y0o4k0r.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm705tidgej317y0o4k0r.jpg" width=""/>
    </a>
</p> 


二、单点搭建
1.拉取镜像：

```
docker pull redis:latest
```
2.查看镜像:
```
[root@XXX ~]# docker images
REPOSITORY               TAG     IMAGE ID      CREATED      SIZE
docker.io/library/redis  latest  ef47f3b6dc11  2 weeks ago  108 MB
```
3.启动镜像:
```
docker run -itd --name redis -p 6379:63799 redis
```
参数说明 ：
- **-p 6379:63799**：映射容器服务的 6379 端口到宿主机的 63799 端口。
- 外部可以直接通过宿主机ip:63799 访问到 Redis 的服务。

查看进程:

```
[root@XXX ~]# docker ps -a
CONTAINER ID  IMAGE                           COMMAND       CREATED        STATUS            PORTS                    NAMES
1c99dca0cd93  docker.io/library/redis:latest  redis-server  3 seconds ago  Up 2 seconds ago  0.0.0.0:6379->63799/tcp  redis
```
4.进入容器:
```
[root@10-9-6-129 ~]# docker exec -it redis /bin/bash
root@1c99dca0cd93:/data# redis-cli
127.0.0.1:6379>
```
三、**Redis Sentinel** 哨兵模式搭建(一主两从三哨兵)

1.镜像拉取同上

2.编写配置文件
分别建立两个文件夹 用来放置服务和哨兵
redis-server  redis-sentinel 
在服务文件夹中分别新建四个配置文件(三个redis-config 和一个docker-compose.yml)
redis-master.conf 
```
# bind 127.0.0.1

# 启用保护模式
# 即在没有使用bind指令绑定具体地址时
# 或在没有设定密码时
# Redis将拒绝来自外部的连接
# protected-mode yes

# 监听端口
port 6379

# 启动时不打印logo
# 这个不重要，想看logo就打开它
always-show-logo no

# 设定密码认证
requirepass 123456

# 禁用KEYS命令
# 一方面 KEYS * 命令可以列出所有的键，会影响数据安全
# 另一方面 KEYS 命令会阻塞数据库，在数据库中存储了大量数据时，该命令会消耗很长时间
# 期间对Redis的访问也会被阻塞，而当锁释放的一瞬间，大量请求涌入Redis，会造成Redis直接崩溃
rename-command KEYS ""

# 此外还应禁止 FLUSHALL 和 FLUSHDB 命令
# 这两个命令会清空数据，并且不会失败
```
redis-slave1.conf
```
# bind 127.0.0.1

# 启用保护模式
# 即在没有使用bind指令绑定具体地址时
# 或在没有设定密码时
# Redis将拒绝来自外部的连接
# protected-mode yes

# 监听端口
port 6380

# 启动时不打印logo
# 这个不重要，想看logo就打开它
always-show-logo no

# 设定密码认证
requirepass 123456

# 禁用KEYS命令
# 一方面 KEYS * 命令可以列出所有的键，会影响数据安全
# 另一方面 KEYS 命令会阻塞数据库，在数据库中存储了大量数据时，该命令会消耗很长时间
# 期间对Redis的访问也会被阻塞，而当锁释放的一瞬间，大量请求涌入Redis，会造成Redis直接崩溃
rename-command KEYS ""

# 此外还应禁止 FLUSHALL 和 FLUSHDB 命令
# 这两个命令会清空数据，并且不会失败

# 配置master节点信息
# 格式：
#slaveof <masterip> <masterport>
# 此处masterip所指定的redis-server-master是运行master节点的容器名
# Docker容器间可以使用容器名代替实际的IP地址来通信
slaveof 127.0.0.1 6379

# 设定连接主节点所使用的密码
masterauth "123456"
```
redis-slave2.conf
```
# bind 127.0.0.1

# 启用保护模式
# 即在没有使用bind指令绑定具体地址时
# 或在没有设定密码时
# Redis将拒绝来自外部的连接
# protected-mode yes

# 监听端口
port 6381

# 启动时不打印logo
# 这个不重要，想看logo就打开它
always-show-logo no

# 设定密码认证
requirepass 123456

# 禁用KEYS命令
# 一方面 KEYS * 命令可以列出所有的键，会影响数据安全
# 另一方面 KEYS 命令会阻塞数据库，在数据库中存储了大量数据时，该命令会消耗很长时间
# 期间对Redis的访问也会被阻塞，而当锁释放的一瞬间，大量请求涌入Redis，会造成Redis直接崩溃
rename-command KEYS ""

# 此外还应禁止 FLUSHALL 和 FLUSHDB 命令
# 这两个命令会清空数据，并且不会失败

# 配置master节点信息
# 格式：
#slaveof <masterip> <masterport>
# 此处masterip所指定的redis-server-master是运行master节点的容器名
# Docker容器间可以使用容器名代替实际的IP地址来通信
slaveof 127.0.0.1 6379

# 设定连接主节点所使用的密码
masterauth "123456"
```
docker-compose-redis-server.yml
```
---

version: '3'

services:
  # 主节点的容器
  redis-server-master:
    image: redis
    container_name: redis-server-master
    restart: always
    # 为了规避Docker中端口映射可能带来的问题
    # 这里选择使用host网络
    network_mode: host
    # 指定时区，保证容器内时间正确
    environment:
      TZ: "Asia/Shanghai"
    volumes:
      # 映射配置文件和数据目录
      - ./redis-master.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-master:/data
    sysctls:
      # 必要的内核参数
      net.core.somaxconn: '511'
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
  # 从节点1的容器
  redis-server-slave-1:
    image: redis
    container_name: redis-server-slave-1
    restart: always
    network_mode: host
    depends_on:
      - redis-server-master
    environment:
      TZ: "Asia/Shanghai"
    volumes:
      - ./redis-slave1.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-slave-1:/data
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
  # 从节点2的容器
  redis-server-slave-2:
    image: redis
    container_name: redis-server-slave-2
    restart: always
    network_mode: host
    depends_on:
      - redis-server-master
    environment:
      TZ: "Asia/Shanghai"
    volumes:
      - ./redis-slave2.conf:/usr/local/etc/redis/redis.conf
      - ./data/redis-slave-2:/data
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
```
redis-sentinel-1.conf
```
# bind 127.0.0.1

# 哨兵的端口号
# 因为各个哨兵节点会运行在单独的Docker容器中
# 所以无需担心端口重复使用
# 如果需要在单机
port 26379

# 设定密码认证
requirepass 123456

# 配置哨兵的监控参数
# 格式：sentinel monitor <master-name> <ip> <redis-port> <quorum>
# master-name是为这个被监控的master起的名字
# ip是被监控的master的IP或主机名。因为Docker容器之间可以使用容器名访问，所以这里写master节点的容器名
# redis-port是被监控节点所监听的端口号
# quorom设定了当几个哨兵判定这个节点失效后，才认为这个节点真的失效了
sentinel monitor local-master 127.0.0.1 6379 2

# 连接主节点的密码
# 格式：sentinel auth-pass <master-name> <password>
sentinel auth-pass local-master 123456

# master在连续多长时间无法响应PING指令后，就会主观判定节点下线，默认是30秒
# 格式：sentinel down-after-milliseconds <master-name> <milliseconds>
sentinel down-after-milliseconds local-master 30000
```
redis-sentinel-2.conf
```
# bind 127.0.0.1

# 哨兵的端口号
# 因为各个哨兵节点会运行在单独的Docker容器中
# 所以无需担心端口重复使用
# 如果需要在单机
port 26380

# 设定密码认证
requirepass 123456

# 配置哨兵的监控参数
# 格式：sentinel monitor <master-name> <ip> <redis-port> <quorum>
# master-name是为这个被监控的master起的名字
# ip是被监控的master的IP或主机名。因为Docker容器之间可以使用容器名访问，所以这里写master节点的容器名
# redis-port是被监控节点所监听的端口号
# quorom设定了当几个哨兵判定这个节点失效后，才认为这个节点真的失效了
sentinel monitor local-master 127.0.0.1 6379 2

# 连接主节点的密码
# 格式：sentinel auth-pass <master-name> <password>
sentinel auth-pass local-master 123456

# master在连续多长时间无法响应PING指令后，就会主观判定节点下线，默认是30秒
# 格式：sentinel down-after-milliseconds <master-name> <milliseconds>
sentinel down-after-milliseconds local-master 30000
```
redis-sentinel-3.conf
```
# bind 127.0.0.1

# 哨兵的端口号
# 因为各个哨兵节点会运行在单独的Docker容器中
# 所以无需担心端口重复使用
# 如果需要在单机
port 26381

# 设定密码认证
requirepass 123456

# 配置哨兵的监控参数
# 格式：sentinel monitor <master-name> <ip> <redis-port> <quorum>
# master-name是为这个被监控的master起的名字
# ip是被监控的master的IP或主机名。因为Docker容器之间可以使用容器名访问，所以这里写master节点的容器名
# redis-port是被监控节点所监听的端口号
# quorom设定了当几个哨兵判定这个节点失效后，才认为这个节点真的失效了
sentinel monitor local-master 127.0.0.1 6379 2

# 连接主节点的密码
# 格式：sentinel auth-pass <master-name> <password>
sentinel auth-pass local-master 123456

# master在连续多长时间无法响应PING指令后，就会主观判定节点下线，默认是30秒
# 格式：sentinel down-after-milliseconds <master-name> <milliseconds>
sentinel down-after-milliseconds local-master 30000
```

docker-compose-redis-sentinel.yml
```
---

version: '3'

services:
  redis-sentinel-1:
    image: redis
    container_name: redis-sentinel-1
    restart: always
    # 为了规避Docker中端口映射可能带来的问题
    # 这里选择使用host网络
    network_mode: host
    volumes:
      - ./redis-sentinel-1.conf:/usr/local/etc/redis/redis-sentinel.conf
    # 指定时区，保证容器内时间正确
    environment:
      TZ: "Asia/Shanghai"
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-sentinel", "/usr/local/etc/redis/redis-sentinel.conf"]
  redis-sentinel-2:
    image: redis
    container_name: redis-sentinel-2
    restart: always
    network_mode: host
    volumes:
      - ./redis-sentinel-2.conf:/usr/local/etc/redis/redis-sentinel.conf
    environment:
      TZ: "Asia/Shanghai"
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-sentinel", "/usr/local/etc/redis/redis-sentinel.conf"]
  redis-sentinel-3:
    image: redis
    container_name: redis-sentinel-3
    restart: always
    network_mode: host
    volumes:
      - ./redis-sentinel-3.conf:/usr/local/etc/redis/redis-sentinel.conf
    environment:
      TZ: "Asia/Shanghai"
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-sentinel", "/usr/local/etc/redis/redis-sentinel.conf"]
```

目录结构如下
<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm75ll63kyj30d709udgd.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm75ll63kyj30d709udgd.jpg" width=""/>
    </a>
</p> 

3.启动服务
```
docker-compose  -f redis-sentinel/docker-compose-redis-sentinel.yml up -d
docker-compose  -f redis-server/docker-compose-redis-sentinel.yml up -d
```
<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm75owc1opj30zh03rmy1.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm75owc1opj30zh03rmy1.jpg" width=""/>
    </a>
</p> 

4.测试服务
进入容器服务
执行 info replication 查看服务状态
<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm75tahxu3j30j608egmu.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm75tahxu3j30j608egmu.jpg" width=""/>
    </a>
</p> 
### 容灾演练

现在我们杀掉主节点 redis-server-master
自动切换了节点2为主节点
<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm762k0u56j30lh0dowgi.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm762k0u56j30lh0dowgi.jpg" width=""/>
    </a>
</p> 


四、**Redis Cluster**  集群模式搭建 (三主三从)


1.镜像拉取同上

2.编写配置文件


redis config配置文件 redis-cluster.tmpl :
```
#  redis端口
port ${PORT}
# 关闭保护模式
protected-mode no
# 开启集群
cluster-enabled yes
# 集群节点配置
cluster-config-file nodes.conf
# 超时
cluster-node-timeout 5000
# 集群节点IP host模式为宿主机IP
cluster-announce-ip {改为宿主机的IP 56.1.1.1}
# 集群节点端口 7001 - 7006
cluster-announce-port ${PORT}
cluster-announce-bus-port 1${PORT}
# 开启 appendonly 备份模式
appendonly yes
# 每秒钟备份
appendfsync everysec
# 对aof文件进行压缩时，是否执行同步操作
no-appendfsync-on-rewrite no
# 当目前aof文件大小超过上一次重写时的aof文件大小的100%时会再次进行重写
auto-aof-rewrite-percentage 100
# 重写前AOF文件的大小最小值 默认 64mb
auto-aof-rewrite-min-size 64mb
```
生成文件夹脚本redis-cluster-config.sh:
```
for port in `seq 7001 7006`; do \
  mkdir -p ./redis-cluster/${port}/conf \
  && PORT=${port} envsubst < ./redis-cluster.tmpl > ./redis-cluster/${port}/conf/redis.conf \
  && mkdir -p ./redis-cluster/${port}/data; \
done
```

编写compose配置 docker-compose-redis-cluster.yml 

```
version: '3.7'

services:
  redis7001:
    image: 'redis'
    container_name: redis7001
    command:
      ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./redis-cluster/7001/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-cluster/7001/data:/data
    ports:
      - "7001:7001"
      - "17001:17001"
    environment:
      # 设置时区为上海，否则时间会有问题
      - TZ=Asia/Shanghai


  redis7002:
    image: 'redis'
    container_name: redis7002
    command:
      ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./redis-cluster/7002/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-cluster/7002/data:/data
    ports:
      - "7002:7002"
      - "17002:17002"
    environment:
      # 设置时区为上海，否则时间会有问题
      - TZ=Asia/Shanghai


  redis7003:
    image: 'redis'
    container_name: redis7003
    command:
      ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./redis-cluster/7003/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-cluster/7003/data:/data
    ports:
      - "7003:7003"
      - "17003:17003"
    environment:
      # 设置时区为上海，否则时间会有问题
      - TZ=Asia/Shanghai


  redis7004:
    image: 'redis'
    container_name: redis7004
    command:
      ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./redis-cluster/7004/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-cluster/7004/data:/data
    ports:
      - "7004:7004"
      - "17004:17004"
    environment:
      # 设置时区为上海，否则时间会有问题
      - TZ=Asia/Shanghai


  redis7005:
    image: 'redis'
    container_name: redis7005
    command:
      ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./redis-cluster/7005/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-cluster/7005/data:/data
    ports:
      - "7005:7005"
      - "17005:17005"
    environment:
      # 设置时区为上海，否则时间会有问题
      - TZ=Asia/Shanghai


  redis7006:
    image: 'redis'
    container_name: redis7006
    command:
      ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./redis-cluster/7006/conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./redis-cluster/7006/data:/data
    ports:
      - "7006:7006"
      - "17006:17006"
    environment:
      # 设置时区为上海，否则时间会有问题
      - TZ=Asia/Shanghai
```
用脚本(redis-cluster-config.sh)根据配置文件(redis-cluster.tmpl) 生成文件夹目录

<p align="center"> <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm6utpvrl6j30990k60tx.jpg" target="_blank">
<img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm6utpvrl6j30990k60tx.jpg" width=""/>
</a></p> 
3.启动服务

```
docker-compose  -f docker-compose-redis-cluster.yml up -d
```

<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm6w58d1b6j30hy03hmxj.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm6w58d1b6j30hy03hmxj.jpg" width=""/>
    </a>
</p> 
<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm6w426l3aj30yp03p75d.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm6w426l3aj30yp03p75d.jpg" width=""/>
    </a>
</p> 


4.集群配置

本机IP：117.50.82.218

```
docker exec -it redis7001 redis-cli -p 7001 --cluster create 117.50.82.218:7001 117.50.82.218:7002 117.50.82.218:7003 117.50.82.218:7004 117.50.82.218:7005 117.50.82.218:7006 --cluster-replicas 1
```
<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm6wsp7wd0j318c0o2wk4.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm6wsp7wd0j318c0o2wk4.jpg" width=""/>
    </a>
</p> 


5.查看集群状态

进入redis服务
```

docker exec -it redis7001 /bin/bash 

redis-cli -p 7001
```

执行 info replication 查看服务状态

执行 cluster nodes  及 cluster info 查看集群状态

<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm6z6bteuyj30s50ibdj3.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm6z6bteuyj30s50ibdj3.jpg" width=""/>
    </a>
</p> 
使用测试

进行读写需要在进入redis-cli的时候加-c参数 
由于Redis Cluster会根据key进行hash运算，然后将key分散到不同slots，test的hash运算结果在redis7002节点上的slots中。所以我们操作redis7001写操作会自动路由到7002 如果过没有加-c 会无法存储

<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm6z9htkw0j30dj03zjrn.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm6z9htkw0j30dj03zjrn.jpg" width=""/>
    </a>
</p> 
### 容灾演练

现在我们杀掉主节点redis7001
```
docker stop redis7001
```
发现节点redis7004已经接替了它的位置。
<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm6zgrbgf3j30ri04waba.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm6zgrbgf3j30ri04waba.jpg" width=""/>
    </a>
</p> 
再试着启动7001
```
docker start redis7001
```

它自动作为slave挂载到7004

<p align="left">
    <a href="https://tva1.sinaimg.cn/large/0081Kckwly1gm6zldeagaj30sy06375w.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0081Kckwly1gm6zldeagaj30sy06375w.jpg" width=""/>
    </a>
</p> 


