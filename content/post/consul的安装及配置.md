---
title: "Consul的安装及配置"
date: 2020-01-08T21:05:45+08:00
draft: false
tags: ["consul"]
categories: ["consul"]
keywords: ["consul"]
---

## 认识consul

[consul官方地址](https://www.consul.io/)

 ### 什么consul

Consul是HashiCorp公司推出的开源工具，用于实现分布式系统的服务发现与配置。

Consul由Go语言开发，部署起来非常容易，只需要极少的可执行程序和配置文件，具有绿色、轻量级的特点

Consul是分布式的、高可用的、 可横向扩展的

与之类似的产品有以下几种:

* etcd
* zookeeper
* eureka

### consul 架构

consul 是 server / agent 的架构 

* Consul servers：负责数据的存储和复制，servers之间选举出一个leader，虽然consul一个也可以工作，但是一般需要提供3至5个servers组成的servers集群
* Consul agent：所有提供服务的节点都需要运行Consul agent(使用和查询服务的不需要)，这个Agent主要负责服务本身和这个服务所在节点的健康检查

### consul的组件

* 服务发现（Service Discovery）
* 健康检查（Health Checking）
* KV存储（Key/Value Store）
* 多数据中心（Multi Datacenter）

## 安装及配置

由于部署非常简单，我这里直接使用docker-compose直接部署了，如果不了解docker 和docker-compose的请移步到[Docker Docs](https://docs.docker.com/get-started/)

```shel
# 准备目录
mkdir consul
cd consul
mkdir node1 node2 node3 client
# 这里准备部署三个server节点一个client节点
```

docker-compose.yml

```yaml
version: '3'
services:
  consul_node1:
    container_name: consul_node1
    image: consul:1.6.2
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./node1:/consul/data:rw
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    command: consul agent -data-dir=/consul/data -datacenter="prometheus" -bootstrap -server -node=consul_node1 -enable-script-checks

  consul_node2:
    container_name: consul_node2
    image: consul:1.6.2
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./node2:/consul/data:rw
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    links:
      - consul_node1
    depends_on:
      - consul_node1
    command: consul agent -data-dir=/consul/data -datacenter="prometheus" -server -node=consul_node2 -retry-join=consul_node1 -retry-interval=30s -enable-script-checks

  consul_node3:
    container_name: consul_node3
    image: consul:1.6.2
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./node3:/consul/data:rw
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    links:
      - consul_node1
    depends_on:
      - consul_node1
    command: consul agent -data-dir=/consul/data -datacenter="prometheus" -server -node=consul_node3 -retry-join=consul_node1 -retry-interval=30s -enable-script-checks

  consul_client:
    container_name: consul_client
    image: consul:1.6.2
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./client:/consul/data:rw
    ports:
      - "8500:8500"
      - "8600:8600"
      - "8300:8300"
      - "8301:8301"
      - "8302:8302"
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    links:
      - consul_node1
    depends_on:
      - consul_node1
    command: consul agent -data-dir=/consul/data -datacenter="prometheus" -node=consul_client -retry-join=consul_node1 -retry-interval=30s -ui -bind="0.0.0.0" -client="0.0.0.0" -enable-script-checks

```

执行安装部署命令:

```shell
docker-compose up   # 此命令在前台输出 如果想要在后台运行需要加 -d 指令 我们可以先在前台输出，如果没有问题再使用后台指令运行
# docker-compose up -d   后台运行docker-compose
```

 完成后，可以使用以下指令

```shell
# 进入一个节点
docker exec -it consul_node1 sh
# 执行查看节点成员信息
consul members
Node                      Address                       Status      Type          Build            Protocol  DC          Segment
consul_node1   192.168.80.2:8301    alive         server      1.6.2  2         prometheus         <all>
consul_node2   192.168.80.4:8301    alive         server      1.6.2  2         prometheus         <all>
consul_node3   192.168.80.5:8301    alive         server      1.6.2  2         prometheus         <all>
consul_client    192.168.80.3:8301    alive         client        1.6.2  2         prometheus         <default>

```

查看ui， 在浏览器中输入http://you_ip:8500/

![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20200108214501.png)

consul 点进去就可以看到consul的节点信息

![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20200108214404.png)



**consul 还有很多指令可到consul的官方站点学习**

## consul的使用方式

### 服务发现Service Discovery

[Consul serivces 文档](https://www.consul.io/docs/agent/services.html)

注册地址为: http://you_ip:8500/v1/catalog/register

注册时需要传入以下json格式数据

```json
{
    "ID": "6df55a0a-f8b5-4301-a3e3-ab41e537af5d",  # 一个uuid 
    "Address": "you ip ", # 节点的ip地址
    "Node": "you node name ",  # 要保证节点名称唯一
    "Service": {  # 要注册的服务信息
        "Service": "mysql",  #  服务的名称 
        "ID": "serviceid", # 服务的id 保证唯一
        "tags": ["mysql", "master" ],
        "port": 3306,
        "address": "service的ip"
    }
}
```

uuid 可以使用以下命令生成

```shell
cat /proc/sys/kernel/random/uuid
```

注册命令如下:

```shell
ajson=`cat a.json`
curl -s -m 5 -X PUT -d "$ajson" http://consul_ip:8500/v1/catalog/deregister
```

完成后在页面上就会出现mysql信息

![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20200108224510.png)

![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20200108224804.png)

此时并没有node check 和serice check  要想增加check 在启动时需要添加 `-enable-script-checks` 选项 docker-compose.yml中已启用了

参考[consul check](https://www.consul.io/docs/agent/checks.html)



删除注册的节点命令如下:

```shell
curl -X PUT -d '{"Node": "node name"}' http://consul_ip:8500/v1/catalog/deregister
```

节点有多个服务时需要删除其中一个服务时命令如下 :

```shell
curl -X PUT -d '{"Node": "node name","Service": {"ID": "y"}}' http://consul_ip:8500/v1/catalog/deregister
```



### KV存储

### 配置中心



**未完待续**