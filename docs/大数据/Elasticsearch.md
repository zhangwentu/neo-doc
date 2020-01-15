

# 全文搜索引擎入门——Elasticsearch 

- [一、安装](## 一、安装) 
  - [ 1.安装Es](### 1.安装Es) 
  - [ 2.安装分词器ik](### 2.安装分词器ik)
  - [ 3.安装kibana](### 3.安装kibana)
  - [4.测试一下](### 4.测试一下)
- [二、应用]()
  - [1.](### 1.Test)

## 一、安装

通过docker 安装一个***ES集群***

### 1.安装Es

**拉取镜像**：

```
➜  ~ docker pull elasticsearch:7.2.0 
```

如果只想启动一个单个节点只需要：

```
➜  ~ docker run --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -d elasticsearch:7.2.0
```

然后就可以检查一下es是否已经启动 通过命令行或者浏览器访问都可以：

`curl http://localhost:9200`

可以看到返回了数据：

```
{
  "name" : "es-master",
  "cluster_name" : "es-cluster",
  "cluster_uuid" : "Sst0V4ZIT4Ku44i0snbNXw",
  "version" : {
    "number" : "7.2.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "508c38a",
    "build_date" : "2019-06-20T15:54:18.811730Z",
    "build_snapshot" : false,
    "lucene_version" : "8.0.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

但是我们是抱着学习的态度使用es的，还是尝试一下集群吧。

***目标：3个节点的es集群，1主2从***

我选择使用docker-compose 来搭建集群，其实不用他也可以，需要配置多个节点，分别启动es节点，但是感觉不如这个方式清爽，每次启动只需要执行一次就可以了。

首先需要具有docker-compose 我是通过pip安装的，感觉这样比较简单

```
➜  ~ sudo pip3 install docker-compose 
```

检查一下是否安装成功：

```
➜  ~ docker-compose version

docker-compose version 1.25.1, build a82fef0
docker-py version: 4.1.0
CPython version: 3.7.5
OpenSSL version: OpenSSL 1.1.1d  10 Sep 2019
```

准备配置文件前可以先拉取一个elasticsearch-head可视化工具 ，目前官方版本只支持5.0 ，可以浏览和查看数据，可以类比于navicate相对于mysql。

```
➜  ~ docker pull mobz/elasticsearch-head:5
```

**准备配置文件** 

先新建一个文件夹 elasticsearch-config 用来存放需要的配置文件

文件路径如下：

```
➜  mkdir elasticsearch-config
➜  cd elasticsearch-config
➜  ...具体的添加内容见下文
➜  elasticsearch-config tree
.
├── docker-compose.yml
├── master
│   └── conf
│       └── elasticsearch.yml
├── node1
│   └── conf
│       └── elasticsearch.yml
└── node2
    └── conf
        └── elasticsearch.yml

6 directories, 4 files
```

```
es-master：master节点，确定分片位置，索引的新增、删除请求分配

es-node1：分片的 CRUD，以及搜索和整合操作

es-node2：分片的 CRUD，以及搜索和整合操作

es-head：es插件
```

docker-compose.yml：

```
version 是python版本 根据自己的版本指定
```

```
version: "3.7"
services:
     es-master:
       image:  elasticsearch:7.2.0
       container_name: es-master
       restart: always
       volumes:
         - ./master/data:/usr/share/elasticsearch/data:rw
         - ./master/conf/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
         - ./master/logs:/user/share/elasticsearch/logs:rw
         - .
       ports:
         - "9200:9200"
         - "9300:9300"

     es-node1:
       image:  elasticsearch:7.2.0
       container_name: es-node1
       restart: always
       volumes:
         - ./node1/data:/usr/share/elasticsearch/data:rw
         - ./node1/conf/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
         - ./node1/logs:/user/share/elasticsearch/logs:rw
     es-node2:
       image:  elasticsearch:7.2.0
       container_name: es-node2
       restart: always
       volumes:
         - ./node2/data:/usr/share/elasticsearch/data:rw
         - ./node2/conf/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
         - ./node2/logs:/user/share/elasticsearch/logs:rw
     es-head:
       image: mobz/elasticsearch-head:5
       container_name: es-head
       restart: always
       ports:
       - "9100:9100"
```

master/conf/elasticsearch.yml

```
bootstrap.memory_lock: false
cluster.name: es-cluster
node.name: es-master
node.master: true
node.data: false
network.host: 0.0.0.0
http.port: 9200
transport.tcp.port: 9300
discovery.zen.ping.unicast.hosts :  ["es-master:9300", "es-node1:9300", "es-node2:9300"]
discovery.zen.minimum_master_nodes: 1
cluster.initial_master_nodes: [“es-master”]

path.logs: /usr/share/elasticsearch/logs
#设置跨域 其他服务才能访问
http.cors.enabled: true
http.cors.allow-origin: "*"
xpack.security.audit.enabled: true       

```

node/conf/elasticsearch.yml

```
cluster.name: es-cluster
node.name: es-node1 #注意更换节点名字
node.master: false
node.data: true
network.host: 0.0.0.0
http.port: 9200
transport.tcp.port: 9300
discovery.zen.ping.unicast.hosts :  ["es-master:9300", "es-node1:9300", "es-node2:9300"]

path.logs: /usr/share/elasticsearch/logs
http.cors.enabled: true
http.cors.allow-origin: "*"
xpack.security.audit.enabled: true       

```

**执行**

```
➜  ~ docker-compose up  #开启
➜  ~ docker-compose down #关闭
```

PS：

如果出现错误 137 可能是你的docker 的memory 不足 默认2G 给个4G吧 不要抠门 毕竟2020年了

### 2.安装分词器ik

进入es的容器

```bash
➜  ~ docker exec -it 容器名 /bin/bash
```

执行命令安装ik ,下载下来解压即可

```ruby
➜  ~ cd plugins
➜  ~ elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.2.0/elasticsearch-analysis-ik-7.2.0.zip
➜  ~ mkdir ik
➜  ~ unzip -d ik elasticsearch-analysis-ik-7.2.0.zip
➜  ~ rm -f elasticsearch-analysis-ik-7.2.0.zip
```

但是我发现这样有的时候会很慢,而且需要每个节点部署，单个节点还行，但是我们现在是集群啊，所以我本地下载下来 

```
➜  ~ cd elasticsearch-config
➜  ~ mkdir plugins
➜  ~ cd plugins
➜  ~ wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.2.0/elasticsearch-analysis-ik-7.2.0.zip
➜  ~ unzip -d ik elasticsearch-analysis-ik-7.2.0.zip
```

然后使用在docker-compose.yml的volumes里增加配置

```
- ./plugins/ik:/usr/share/elasticsearch/plugins/ik
```



### 3.安装kibana

拉取镜像

**单节点时可以用如下方式：**

```
➜  ~ docker pull kibana:7.2.0
➜  ~ docker run --name kibana --link 容器名:别名 -p 5601:5601 -d kibana:7.2.0
➜  ~ docker start kibana
```

启动以后可以打开浏览器输入http://localhost:5601就可以打开kibana的界面了

**当通过docker-compose启动时也需要把这段加到docker-compose.yml中：**

```
     kibana:
       image: kibana:7.2.0
       container_name: es-kibana
       restart: always
       volumes:
         - ./kibana/kibana.yml:/usr/share/kibana/config/kibana.yml:rw

       ports:
        - 5601:5601
```

**kibana.yml**

```
#
# ** THIS IS AN AUTO-GENERATED FILE **
#
 
# Default Kibana configuration for docker target
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://es-master:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true
i18n.locale: zh-CN

```



### 4.测试一下

装好了kibana 顺便测一下ik

执行

```
POST _analyze
{
  "analyzer":"ik_smart",
  "text":"美国科学家利用从青蛙胚胎中提取的活细胞，创造出了第一个毫米级“活体可编程机器人”。"
}
```

 结果

```
{
  "tokens" : [
    {
      "token" : "美国",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "科学家",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "利用",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "从",
      "start_offset" : 7,
      "end_offset" : 8,
      "type" : "CN_CHAR",
      "position" : 3
    },
    {
      "token" : "青蛙",
      "start_offset" : 8,
      "end_offset" : 10,
      "type" : "CN_WORD",
      "position" : 4
    },
    {
      "token" : "胚胎",
      "start_offset" : 10,
      "end_offset" : 12,
      "type" : "CN_WORD",
      "position" : 5
    },
    {
      "token" : "中",
      "start_offset" : 12,
      "end_offset" : 13,
      "type" : "CN_CHAR",
      "position" : 6
    },
    {
      "token" : "提取",
      "start_offset" : 13,
      "end_offset" : 15,
      "type" : "CN_WORD",
      "position" : 7
    },
    {
      "token" : "的",
      "start_offset" : 15,
      "end_offset" : 16,
      "type" : "CN_CHAR",
      "position" : 8
    },
    {
      "token" : "活",
      "start_offset" : 16,
      "end_offset" : 17,
      "type" : "CN_CHAR",
      "position" : 9
    },
    {
      "token" : "细胞",
      "start_offset" : 17,
      "end_offset" : 19,
      "type" : "CN_WORD",
      "position" : 10
    },
    {
      "token" : "创造",
      "start_offset" : 20,
      "end_offset" : 22,
      "type" : "CN_WORD",
      "position" : 11
    },
    {
      "token" : "出了",
      "start_offset" : 22,
      "end_offset" : 24,
      "type" : "CN_WORD",
      "position" : 12
    },
    {
      "token" : "第一个",
      "start_offset" : 24,
      "end_offset" : 27,
      "type" : "CN_WORD",
      "position" : 13
    },
    {
      "token" : "毫米",
      "start_offset" : 27,
      "end_offset" : 29,
      "type" : "CN_WORD",
      "position" : 14
    },
    {
      "token" : "级",
      "start_offset" : 29,
      "end_offset" : 30,
      "type" : "CN_CHAR",
      "position" : 15
    },
    {
      "token" : "活体",
      "start_offset" : 31,
      "end_offset" : 33,
      "type" : "CN_WORD",
      "position" : 16
    },
    {
      "token" : "可编程",
      "start_offset" : 33,
      "end_offset" : 36,
      "type" : "CN_WORD",
      "position" : 17
    },
    {
      "token" : "机器人",
      "start_offset" : 36,
      "end_offset" : 39,
      "type" : "CN_WORD",
      "position" : 18
    }
  ]
}

```

