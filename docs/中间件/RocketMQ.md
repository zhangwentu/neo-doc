RocketMQ是分布式的消息队列，所以有各种高性能，高可用的部署方案。
 本文暂时不考虑高性能和高可用的部署方案。主要说明如果使用Docker部署RocketMQ。
 本文将在Docker单台宿主机上部署：

- nameserver ：消息服务器将结点及消息服务信息注册到该结点上，客户端也连接到该结点上，获取消息服务信息。
- broker：提供消息服务
- 通过rockermq-console对RocketMQ监控管理



启动Namesrv容器

```
docker run -d -p 9876:9876 -v /Users/zwt/work/docker/rocketmq/data/namesrv/logs:/root/logs -v /Users/zwt/work/docker/rocketmq/data/namesrv/store:/root/store --name rmqnamesrv -e "MAX_POSSIBLE_HEAP=100000000" rocketmqinc/rocketmq sh mqnamesrv
```

启动broker容器

```
新建配置文件broker.conf
内容如下：

brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
# 如果是本地程序调用云主机 mq，这个需要设置成 云主机 IP
brokerIP1=192.168.9.133
```



```
docker run -d -p 10911:10911 -p 10909:10909 -v  /Users/zwt/work/docker/rocketmq/data/broker/logs:/root/logs -v  /Users/zwt/work/docker/rocketmq/data/broker/store:/root/store -v  /Users/zwt/work/docker/rocketmq/conf/broker.conf:/opt/rocketmq/conf/broker.conf --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" -e "MAX_POSSIBLE_HEAP=200000000" rocketmqinc/rocketmq sh mqbroker -c /opt/rocketmq/conf/broker.conf
```

启动控制台

```
docker run -e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.9.133:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false" -p 8080:8080 -t pangliang/rocketmq-console-ng
```





配置docker-mopose(未实践)

```
version: '2'
services:
  namesrv:
    image: going/rocketmq-namesrv:4.2.0
    ports:
      - 9876:9876
    volumes:
      - "E:/rocketmq/namesrv/master/logs:/opt/logs"
      - "E:/rocketmq/namesrv/master/store:/opt/store"
  broker:
    image: going/rocketmq-broker:4.2.0
    ports:
      - 10909:10909
      - 10911:10911
    volumes:
      - "E:/rocketmq/broker/master-1/logs:/opt/logs"
      - "E:/rocketmq/broker/master-1/store:/opt/store"
    links:
      - namesrv:namesrv
  console:
    image: styletang/rocketmq-console-ng:latest
    ports:
     - "8080:8080"
    links:
     - namesrv:namesrv
    environment:
     JAVA_OPTS: -Drocketmq.config.namesrvAddr=namesrv:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false
  producer:
    image: going/rocketmq-producer:4.1.0
    links:
     - namesrv:namesrv
  consumer:
    image: going/rocketmq-consumer:4.1.0
    links:
     - namesrv:namesrv     

```

