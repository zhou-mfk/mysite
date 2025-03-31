---
title: "VictoriaMetrics简单介绍与使用"
date: 2022-03-11T09:57:32+08:00
draft: false
tags: ["VictoriaMetrics", "Prometheus"]
categories: ["VictoriaMetrics"]
keywords: ["VictoriaMetrics", "Prometheus"]
---

VictoriaMetrics 是一种快速、经济高效且可扩展的监控解决方案和时间序列数据库 

**兼容 Prometheus及 PromQL**

 VictoriaMetrics 具有以下突出特点：

- 可以作为 Prometheus 的长期存储 

- 可以作为 Grafana 中 Prometheus 的替代品，因为它支持 Prometheus 查询 API 

- 可以用作 Grafana 中 Graphite 的替代品，因为它支持 Graphite API 

- 它具有易于设置和操作的特点：

  - VictoriaMetrics 由一个没有外部依赖的小型可执行文件组成 

  - 所有配置都是通过具有合理默认值的显式命令行标志完成的 

  - `-storageDataPath` 所有数据都存储在命令行标志指向的单个目录中。 

  - 可以使用vmbackup / vmrestore工具轻松快速地从即时快照备份到 S3 或 GCS

  - 它实现了基于 PromQL 的查询语言

  - MetricsQL，在 PromQL 之上提供了改进的功能

  - 它提供全局查询视图。多个 Prometheus 实例或任何其他数据源可能会将数据摄取到  VictoriaMetrics。稍后可以通过单个查询来查询此数据。

  -  数据摄取和数据查询提供了高性能和良好的纵向和横向可扩展性。性能比 InfluxDB 和 TimescaleDB 高 20 倍

  - 处理数百万个独特的时间序列时，使用的 RAM 比 InfluxDB 少 10 倍，比 Prometheus、Thanos 或 Cortex 少 7 倍

  -  针对高流失率的时间序列进行了优化

  - 提供了高数据压缩，因此与 TimescaleDB 相比，可以将多达 70 倍的数据点塞入有限的存储空间，与 Prometheus、Thanos 或 Cortex 相比，所需的存储空间减少多达 7 倍

  - 它针对具有高延迟 IO 和低 IOPS 的存储进行了优化

  -  单节点 VictoriaMetrics 可以替代使用竞争解决方案构建的中等规模集群。

  - 由于存储架构，它可以保护存储在非正常关机（即 OOM、硬件重置或`kill -9`）时免受数据损坏。 

  - 通过以下协议支持指标的抓取、拉取和回填

  - 从 Prometheus Export那里抓取的指标

  -  Prometheus 远程写入 API

  - 基于HTTP、TCP 和 UDP 的InfluxDB 线路协议

  - Graphite plaintext protocol

  - OpenTSDB 放置消息

  - HTTP OpenTSDB /api/put 请求

  - JSON 行格式

  - 任意 CSV 数据

  - Native binary format

  - 支持指标的重新标记

  - 可以通过串联限制器处理高基数问题和高流失率问题

  - 理想地适用于来自 APM、Kubernetes、物联网传感器、联网汽车、工业遥测、财务数据和各种企业工作负载的大量时间序列数据

  - 有开源集群版

    

## 组件

### 单机版

VictoriaMetrics

`-storageDataPath` 指定存储的路径

`-retentionPeriod` 保留存储的数据。旧数据会被自动删除。默认保留期为 1 个月。数据在删除之前会延迟一个月。例如1月1号的数据会到3月1号删除

默认端口为: `8428`

Prometheus.yml配置

```yaml
remote_write:
 - url: http://<victoriametrics-addr>:8428/api/v1/write
```

Prometheus 将传入的数据写入本地存储并并行复制到远程存储。这意味着`--storage.tsdb.retention.time`即使远程存储不可用，数据在本地存储中仍然可用

如果您计划从多个 Prometheus 实例向 VictoriaMetrics 发送数据，则将以下行添加到 Prometheus config global 部分:

```yaml
global:
 external_labels:
   datacenter: dc-123
```

这指示 Prometheus 在将`datacenter=dc-123`每个样本发送到远程存储之前为其添加标签。标签名称可以是任意的 -`datacenter`只是一个示例。标签值在 Prometheus 实例中必须是唯一的，因此可以按此标签过滤和分组时间序列

对于高负载的 Prometheus 实例（每秒 200k+ 样本），可以应用以下调优：

```yaml
remote_write:
 - url: http://<victoriametrics-addr>:8428/api/v1/write
   queue_config:
     max_samples_per_send: 10000
     capacity: 20000
     max_shards: 30

```

使用远程写入可将 Prometheus 的内存使用量增加约 25%。

如果您遇到 Prometheus 内存消耗过高的问题，请尝试降低`max_samples_per_send`和`capacity`参数, 请记住，这两个参数紧密相连。

### 实例验证

**单机版 docker-compose.yml**

准备目录

```shell
mkdir prom_config prom_data grafana_data victoria-metrics-data
```

```yaml
version: '3'
services:
    victoriametrics-server:
      container_name: victoriametrics
      image: victoriametrics/victoria-metrics:latest
      ports:
        - "8428:8428"
      user: root
      volumes:
        - /etc/localtime:/etc/localtime:ro
        - ./victoria-metrics-data:/victoria-metrics-data
      ulimits:
        nproc: 65535
        nofile:
          soft: 20000
          hard: 40000
 
    prometheus-server:
      image: prom/prometheus:latest
      container_name: prometheus
      hostname: prometheus
      restart: always
      # user: "1000:1000"
      user: root
      ports:
        - "9090:9090"
      volumes:
        - /etc/localtime:/etc/localtime:ro
        - ./prom_config:/etc/prometheus
        - ./prom_data:/prometheus
      command:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--storage.tsdb.path=/prometheus'
        - '--storage.tsdb.retention.time=600s'
        - '--storage.tsdb.no-lockfile'
        - '--log.level=info'
        - '--web.enable-lifecycle'
        - '--query.max-concurrency=100'
        - '--web.max-connections=1024'
        - '--log.format=json'
        - '--web.enable-admin-api'
      ulimits:
        nproc: 65535
        nofile:
          soft: 20000
          hard: 40000
#      command: ["--storage.tsdb.retention.time=10m"]
 
    grafana:
      container_name: grafana
      image: grafana/grafana:latest
      ports:
        - "3000:3000"
      volumes:
        - /etc/localtime:/etc/localtime:ro
        - ./grafana_data:/var/lib/grafana
      environment:
        - GF_SECURITY_ADMIN_PASSWORD=admin123
      user: root
```



prometheus.yml 配置

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
  # 增加扩展的标签
  external_labels:
    bl_monitor: os_monitor
 
 
# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093
 
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
 
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
 
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
 
    static_configs:
      - targets: ["localhost:9090", "you_address_ip:8428"]
 
# 设置victoriametrics
remote_write:
  - url: http://you_address_ip:8428/api/v1/write
```



### 集群版

`具有多租户功能`

`vminsert` 数据写入接口， 可以一个或多个 多个时需要前加http 负载均衡器 vmauth/nginx `默认端口8480`
`vmselect` 数据读取接口，可以一个或多个 多个时需要前加http 负载均衡器 vmauth/nginx `默认端口8481`
`vmstorge` 数据存储接口 最少两个节点， 多个节点，则需要在vmselect, vminsert中全部配置，新节点或删除节点时也需要修改 vmselect, vminsert 的配置  `默认端口8482`

`vmagent` 是类似 promtheus 的组件，可作为 promethues 的代替品

`vmalert` 是告警组件依赖 alertmanager

`vmauth` 是一个负载均衡器为 vminsert 和 vmselect

### 实例验证

集群版本docker-compose.yml

使用了 vmagent 代替了 Prometheus 在配置时需要指定租户

准备目录

```shell
mkdir grafanadata strgdata-1 strgdata-2 vmagentdata
```

```yaml

version: '3.5'
services:
  vmagent:
    container_name: vmagent
    image: victoriametrics/vmagent
    depends_on:
      - "vminsert"
    ports:
      - 8429:8429
    volumes:
      - ./vmagentdata:/vmagentdata
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--promscrape.config=/etc/prometheus/prometheus.yml'
      - '--remoteWrite.url=http://vminsert:8480/insert/0/prometheus/'
    restart: always
 
  grafana:
    container_name: grafana-1
    image: grafana/grafana:8.3.2
    depends_on:
      - "vmselect"
    ports:
      - 3001:3000
    restart: always
    volumes:
      - ./grafanadata:/var/lib/grafana
 
  vmstorage-1:
    container_name: vmstorage-1
    image: victoriametrics/vmstorage
    ports:
      - 8482
      - 8400
      - 8401
    volumes:
      - ./strgdata-1:/storage
    command:
      - '--storageDataPath=/storage'
    restart: always
  vmstorage-2:
    container_name: vmstorage-2
    image: victoriametrics/vmstorage
    ports:
      - 8482
      - 8400
      - 8401
    volumes:
      - ./strgdata-2:/storage
    command:
      - '--storageDataPath=/storage'
    restart: always
  vminsert:
    container_name: vminsert
    image: victoriametrics/vminsert
    depends_on:
      - "vmstorage-1"
      - "vmstorage-2"
    command:
      - '--storageNode=vmstorage-1:8400'
      - '--storageNode=vmstorage-2:8400'
    ports:
      - 8480
    restart: always
  vmselect:
    container_name: vmselect
    image: victoriametrics/vmselect
    depends_on:
      - "vmstorage-1"
      - "vmstorage-2"
    command:
      - '--storageNode=vmstorage-1:8401'
      - '--storageNode=vmstorage-2:8401'
    ports:
      - 8481:8481
    restart: always
 
  # vmalert:
  #   container_name: vmalert
  #   image: victoriametrics/vmalert
  #   depends_on:
  #     - "vmselect"
  #   ports:
  #     - 8880:8880
  #   volumes:
  #     - ./alerts.yml:/etc/alerts/alerts.yml
  #   command:
  #     - '--datasource.url=http://vmselect:8481/select/0/prometheus'
  #     - '--remoteRead.url=http://vmselect:8481/select/0/prometheus'
  #     - '--remoteWrite.url=http://vminsert:8480/insert/0/prometheus'
  #     - '--notifier.url=http://alertmanager:9093/'
  #     - '--rule=/etc/alerts/*.yml'
  #     # display source of alerts in grafana
  #     - '-external.url=http://127.0.0.1:3000' #grafana outside container
  #     # when copypaste the line below be aware of '$$' for escaping in '$expr'
  #     - '--external.alert.source=explore?orgId=1&left=["now-1h","now","VictoriaMetrics",{"expr":"{{$$expr|quotesEscape|crlfEscape|queryEscape}}"},{"mode":"Metrics"},{"ui":[true,true,true,"none"]}]'
  #   restart: always
 
  # alertmanager:
  #   container_name: alertmanager
  #   image:  prom/alertmanager
  #   volumes:
  #     - ./alertmanager.yml:/config/alertmanager.yml
  #   command:
  #     - '--config.file=/config/alertmanager.yml'
  #   ports:
  #     - 9093:9093
  #   restart: always
```



**prometheus.yml**

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
  # 增加扩展的标签
  external_labels:
    bl_monitor: os_monitor
 
 
# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093
 
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
 
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
 
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
 
    static_configs:
      - targets: ["vmagent:8429", "vmstorage-1:8482", "vmstorage-2:8482", "vmselect:8481", "vminsert:8480"]
 
  - job_name: "prometheus-consul"
    consul_sd_configs:
    - server: 'consul_address_ip:8500'
      datacenter: prometheus
      services: ['Linux']
    relabel_configs:
    - source_labels: ['__meta_consul_node']
      regex: '(1.1.11.*)'
      action: keep

```

