---
title: "Prometheus基于consul的自动发现"
date: 2019-12-27T13:03:41+08:00
draft: false
tags: ["prometheus", "监控", "consul"]
categories: ["prometheus"]
keywords: ["prometheus", "监控", "consul"]

---

## consul 集群

Consul是由HashiCorp开发的一个支持多数据中心的分布式服务发现和键值对存储服务的开源软件，被大量应用于基于微服务的软件架构当中。

### 安装（docker-compose实现）

docker-compose的安装请参考:

[docker-compose 安装及使用](https://docs.docker.com/compose/install/)

创建目录

```shell
mkdir consul
cd consul
mkdir node1 node2 node3 client
```

docker-compose.yml如下所示:

```ymal
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
    command: consul agent -data-dir=/consul/data -datacenter="prometheus" -bootstrap -server -node=consul_node1

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
    command: consul agent -data-dir=/consul/data -datacenter="prometheus" -server -node=consul_node2 -retry-join=consul_node1 -retry-interval=30s

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
    command: consul agent -data-dir=/consul/data -datacenter="prometheus" -server -node=consul_node3 -retry-join=consul_node1 -retry-interval=30s

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
    command: consul agent -data-dir=/consul/data -datacenter="prometheus" -node=consul_client -retry-join=consul_node1 -retry-interval=30s -ui -bind="0.0.0.0" -client="0.0.0.0"
```



启动consul集群

```shell
docker-compose up -d
```



## 与prometheus集成

修改prometheus.yml配置文件增加如下配置

```
  - job_name: 'consul_linux'
    consul_sd_configs:
    - server: '127.0.0.1:8500'
      datacenter: prometheus
      services: ['linux']
    relabel_configs:
    - source_labels: ['__meta_consul_service']
      regex: '(.*)'
      target_label: 'job'
      replacement: '$1'
    - source_labels: ['__meta_consul_node']
      regex: '(.*)'
      target_label: 'instance'
      replacement: '$1'
    - source_labels: ['__meta_consul_tags']
      regex: ',(.*),'
      target_label: 'group'
      replacement: '$1'
```



## Relabel

### prometheus 的Relabeling机制

在Prometheus所有的Target实例中，都包含一些默认的Metadata标签信息。可以通过Prometheus UI的Targets页面中查看这些实例的Metadata标签的内容：

![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20191227131728.png)

默认情况下，当Prometheus加载Target实例完成后，这些Target时候都会包含一些默认的标签

```
__address__：当前Target实例的访问地址<host>:<port>
__scheme__：采集目标服务访问地址的HTTP Scheme，HTTP或者HTTPS
__metrics_path__：采集目标服务访问地址的访问路径
__param_<name>：采集任务目标服务的中包含的请求参数
```



通过Consul动态发现的服务实例还会包含以下Metadata标签信息：

```
__meta_consul_address：consul地址
__meta_consul_dc：consul中服务所在的数据中心
__meta_consulmetadata：服务的metadata
__meta_consul_node：服务所在consul节点的信息
__meta_consul_service_address：服务访问地址
__meta_consul_service_id：服务ID
__meta_consul_service_port：服务端口
__meta_consul_service：服务名称
__meta_consul_tags：服务包含的标签信息
```



完整的relabel_config配置如下所示：

```
# The source labels select values from existing labels. Their content is concatenated
# using the configured separator and matched against the configured regular expression
# for the replace, keep, and drop actions.
[ source_labels: '[' <labelname> [, ...] ']' ]

# Separator placed between concatenated source label values.
[ separator: <string> | default = ; ]

# Label to which the resulting value is written in a replace action.
# It is mandatory for replace actions. Regex capture groups are available.
[ target_label: <labelname> ]

# Regular expression against which the extracted value is matched.
[ regex: <regex> | default = (.*) ]

# Modulus to take of the hash of the source label values.
[ modulus: <uint64> ]

# Replacement value against which a regex replace is performed if the
# regular expression matches. Regex capture groups are available.
[ replacement: <string> | default = $1 ]

# Action to perform based on regex matching.
[ action: <relabel_action> | default = replace ]
```

其中action定义了当前relabel_config对Metadata标签的处理方式，默认的action行为为replace。

replace行为会根据regex的配置匹配source_labels标签的值（多个source_label的值会按照separator进行拼接），并且将匹配到的值写入到target_label当中，如果有多个匹配组，则可以使用${1}, ${2}确定写入的内容。如果没匹配到任何内容则不对target_label进行重新。



其中action 包含以下:

```shell
replacement
keep
drop
labeldrop
labelkeep
labemap
hashmod
```

其中**keep and drop**  是用来过滤Target实例的

```
当action设置为keep时，Prometheus会丢弃source_labels的值中没有匹配到regex正则表达式内容的Target实例，而当action设置为drop时，则会丢弃那些source_labels的值匹配到regex正则表达式内容的Target实例。可以简单理解为keep用于选择，而drop用于排除。
```


