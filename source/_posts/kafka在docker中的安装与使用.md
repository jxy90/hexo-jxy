title: kafka在docker中的安装与使用(Windows)
author: 江小渔
tags: []
categories: []
date: 2019-07-05 10:26:00
---
# 下载
- 下载镜像
```
docker pull wurstmeister/zookeeper
docker pull wurstmeister/kafka
```

# 启动
- 启动zookeeper容器
```
docker run -d --name zookeeper -p 2181:2181 -t wurstmeister/zookeeper
```

- 启动kafka容器
```
docker run -d --name kafka --publish 9092:9092 --link zookeeper --env KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 --env KAFKA_ADVERTISED_HOST_NAME=192.168.59.101 --env KAFKA_ADVERTISED_PORT=9092 wurstmeister/kafka:latest
```
- 192.168.59.101 改为宿主机器的IP地址，如果不这么设置，可能会导致在别的机器上访问不到kafka。

#### 进入kafka容器(管理员权限CMD运行)
```
docker exec -it kafka /bin/bash
cd opt
cd kafka_2.11-0.10.1.0
ls
```

#### 测试kafka
- 
```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic mykafka
```
- 输入字符
![upload successful](/images/pasted-2.png)
- Ctrl+C 退出
- bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mykafka --from-beginning
- 之前输入的内容进行了输出
![upload successful](/images/pasted-3.png)
- exit退出container
- over

#### 参考: 
- https://www.cnblogs.com/yxlblogs/p/10115672.html
- https://blog.csdn.net/snowcity1231/article/details/54946857