如何在低配服务器（阿里云服务器1核2g）下降低docker-mysql内存占用(仅限个人学习使用，企业用户忽略)



进入Docker容器内
```
docker exec -it mysql   /bin/bash
```
安装vim
```
apt-get update
```
```
apt-get install vim
```

进入配置文件目录下
```
cd /etc/mysql
```
修改配置文件
```
vi my.cnf
```

增加配置
```
performance_schema_max_table_instances=400 #设置效果不明显
table_definition_cache=400
performance_schema=off #效果明显
table_open_cache=64
innodb_buffer_pool_chunk_size=64M #效果不明显
innodb_buffer_pool_size=64M #效果不明显
```

默认配置运行内存占用大约400m，设置后运行100m左右

docker stats:


<p align="left">
    <a href="https://tva1.sinaimg.cn/large/008eGmZEly1gmwfbx54zzj30th0260t3.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/008eGmZEly1gmwfbx54zzj30th0260t3.jpg" width=""/>
    </a>
</p> 

