---
title: "使用Filebeat+kafka+logstash+elasticsearch+kibana搭建日志系统"
date: 2019-10-18T20:52:27+08:00
draft: false
tags: ["elasticsearch", "filebeat", "logstash", "kibana", "kafka", "zookeeper", "日志", "elk"]
categories: ["ELK"]
keywords: ["elasticsearch", "filebeat", "logstash", "kibana", "kafka", "zookeeper", "日志", "elk"]
---

整理下使用Filebeat+kafka+logstash+elasticsearch+kibana来搭建日志系统， 加上zookeeper是6个组件来完成一套日志系统，这也是在线上业务正式使用的日志系统。 

## 组件说明

- filebeat 是收集日志的工具需要安装在每个被收集日志的节点上，也可以启动docker 镜像来部署此组件。在这里收集到的日志直接传送到kafka消息中间件。

- kafka 消息中间件 需要zookeeper配合使用

- logstash  在这里作为接收和处理日志，并把处理好的日志写入elasticsearch集群中

- elasticsearch 日志收放的数据库，这里使用了5个数据节点和3个master节点，master节点和数据节点分开，这个下面会详细说明。 elasticsearch 会开启x-pack功能

- kibana 作为日志展示组件，也可以使用比较优秀的grafana

- 组件版本信息

  - jdk 1.8

  - zookeeper v3.4.13 
  - kafka v2.1.1
  - kafka-manager v1.3.3.22
  - elasticsearch  v6.6.0
  - logstash v6.6.0
  - kibana v6.6.0

使用jdk1.8 这个需要先安装

## kafka + zookeeper

### zookeeper

- 下载

  ```shell
  cd /usr/local/src
  wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.13/zookeeper-3.4.13.tar.gz
  ```

- 创建所需要的用户及数据目录

  ```shell
  useradd zookeeper -s /sbin/nologin
  mkdir /opt/zookeeper/logs -pv
  chown zookeeper:zookeeper /opt/zookeeper/ -R
  ```

- 安装

  ```shell
  cd /usr/local/src
  tar xvf zookeeper-3.4.13.tar.gz -C /usr/local/
  chown zookeeper:zookeeper /usr/local/zookeeper-3.4.13 -R
  ln -sv /usr/local/zookeeper-3.4.13 /usr/local/zookeeper
  ```

- 配置

  ```shell
  cd /usr/local/zookeeper/conf
  cp zoo_sample.cfg zoo.cfg
  # 配置zoo.cfg
  tickTime=2000
  initLimit=10
  syncLimit=5
  dataDir=/opt/zookeeper
  dataLogDir=/opt/zookeeper/logs
  clientPort=2181
  # ---- 如果以下不配置启动的zookeeper则是单机模式
  # 集群
  initLimit=10
  syncLimit=5
  server.1=10.10.10.11:2888:3888
  server.2=10.10.10.12:2888:3888
  server.3=10.10.10.13:2888:3888
  server.4=10.10.10.14:2888:3888
  server.5=10.10.10.15:2888:3888
  ```

  **server.id=host:port:port **

  **id 被称为 Server ID, 用来标识服务器在集群中的序号。同时每台 ZooKeeper 服务器上, 都需要在数据目录(即 dataDir 指定的目录) 下创建一个 myid 文件, 该文件只有一行内容, 即对应于每台服务器的Server ID**

  创建myid

  ```shell
  cd /opt/zookeeper
  echo 1 > myid
  ```

- 增加服务脚本

  ```shell
  vim /etc/systemd/system/zookeeper.service
  
  [Unit]
  Description=zookeeper
  After=syslog.target network.target
  [Service]
  Type=forking
  Environment=ZOO_LOG_DIR=/opt/zookeeper/logs # 增加启动时日志输出位置
  Environment=JAVA_HOME=/opt/jdk/jdk1.8 # 指定jdk
  ExecStart=/usr/local/zookeeper/bin/zkServer.sh start
  ExecStop=/usr/local/zookeeper/bin/zkServer.sh stop
  Restart=always
  User=zookeeper
  Group=zookeeper
  [Install]
  WantedBy=multi-user.target
  ```

- 启动

  ```shell
  systemctl start zookeeper 
  # 设置开机启动
  systemctl enable zookeeper
  ```

  

### kafka

- 下载

  ```shell
  cd /usr/local/src
  wget http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.1.1/kafka_2.11-2.1.1.tgz
  ```

- 创建用户及数据目录

  ```shell
  useradd kafka -s /sbin/nologin
  mkdir /opt/kafka/kafka-logs -pv
  chown kafka.kafka /opt/kafka/kafka-logs -R
  ```

  

- 安装

  ```shell
  cd /usr/local/src
  tar xvf kafka_2.11-2.1.1.tgz -C /usr/local/
  chown kafka:kafka /usr/local/kafka_2.11-2.1.1 -R
  ln -sv /usr/local/kafka_2.11-2.1.1 /usr/local/kafka 
  ```

- 配置

  ```shell
  # cd /usr/local/kafka/config/
  # cat server.properties  在节点 10.10.10.11
  broker.id=1 # 集群中唯一id
  listeners=PLAINTEXT://10.10.10.11:9092 # 不同的节点需要不同的地址，需要注意
  num.network.threads=5
  num.io.threads=12
  socket.send.buffer.bytes=102400
  socket.receive.buffer.bytes=102400
  socket.request.max.bytes=104857600
  log.dirs=/opt/kafka/kafka-logs
  num.partitions=1
  num.recovery.threads.per.data.dir=1
  offsets.topic.replication.factor=1
  transaction.state.log.replication.factor=1
  transaction.state.log.min.isr=1
  log.retention.hours=168
  log.segment.bytes=1073741824
  log.retention.check.interval.ms=300000
  zookeeper.connect=10.10.10.11:2181,10.10.10.12:2181,10.10.10.13:2181,10.10.10.14:2181,10.10.10.15:2181
  zookeeper.connection.timeout.ms=6000
  group.initial.rebalance.delay.ms=0
  ```

- 增加服务器脚本

  ```shell
  # vim /etc/systemd/system/kafka.service
  [Unit]
  Description=kafka
  After=syslog.target network.target
  [Service]
  Type=forking
  Environment=JAVA_HOME=/opt/jdk/jdk1.8
  ExecStart=/usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties
  ExecStop=/usr/local/kafka/bin/kafka-server-stop.sh /usr/local/kafka/config/server.properties &
  Restart=always
  User=kafka
  Group=kafka
  [Install]
  WantedBy=multi-user.target
  ```

- 启动

  ```shell
  systemctl start kafka.service
  # 设置开机启动
  systemctl enable kafka.service
  ```

### kafka-manager

- 下载

  ```shell
  cd /usr/local/src
  wget https://github.com/yahoo/kafka-manager/archive/1.3.3.22.zip
  ```

- 编译安装

  ```shell
  unzip 1.3.3.22.zip
  cd kafka-manager-1.3.3.22/
  ./sbt clean dist # 编译
  等待编译完成 ... 需要外网
  ```

- 安装

  ```shell
  cp  /usr/local/src/kafka-manager-1.3.3.22/target/universal/kafka-manager-1.3.3.22.zip /usr/local
  unzip /usr/local/kafka-manager-1.3.3.22.zip
  ln -sv /usr/local/kafka-manager-1.3.3.22 /usr/local/kafka-manager
  ```

  

- 配置

  ```shell
  cd /usr/local/kafka-manager/conf
  # 配置kafka-manager.zkhost
  kafka-manager.zkhosts="10.10.10.11:2181,10.10.10.12:2181,10.10.10.13:2181,10.10.10.14:2181,10.10.10.15:2181"
  # 启用认证及用户名密码
  basicAuthentication.enabled=true
  basicAuthentication.username="kafkalog"
  basicAuthentication.password="logkafka"
  ```

  

- 启动

  ```shell
  cd /usr/local/kafka-manager
  nohup bin/kafka-manager -Dconfig=config/application.conf &
  ```

## ELK

这三个组件使用yum来安装

### elasticsearch

#### 安装

- 配置yum源

  ```shell
  # cat /etc/yum.repos.d/elasticsearch.repo 
  [elasticsearch-6.x]
  name=Elasticsearch repository for 6.x packages
  baseurl=https://artifacts.elastic.co/packages/6.x/yum
  gpgcheck=1
  gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
  enabled=1
  autorefresh=1
  type=rpm-md
  
  ```

  

- 安装elasticsearch

  ```shell
  yum install elasticsearch -y
  ```

  

- 配置elasticsearch

  - 默认配置

    ```yaml
    cluster.name: mycluster  # 集群名称
    node.name: es01  # 节点名称
    node.master: true
    node.data: true
    path.data: /data/elastic
    path.logs: /data/logs
    network.host: 10.10.10.11
    http.port: 9200
    discovery.zen.ping.unicast.hosts: ["10.10.10.8","10.10.10.9","10.10.10.10"]  # 加入集群
    discovery.zen.minimum_master_nodes: 3  # 主节点最小3个
    ```

    

  - master 节点

    ```yaml
    node.master: true
    node.data: false
    ```

    

  - data 节点

    ```shell
    node.master: data
    node.data: true
    ```

    

  - client 节点

    ```shell
    node.master: false
    node.data: false
    ```

  - 设置所谓的冷温热

    主要是配置 node.attr.box_type

    ```shell
    # node.attr.box_type: hot   # 设置为hot 作为热节点 也可以使用其他单词，这只是一个标记或标签
    ```

    数据写入不同的节点：

    ```shell
    定义数据库的index template中加入"routing.allocation.require.box_type": "hot"
    如果对数据没有定义index template或者模板里没有定义`"routing.allocation.require.box_type": "hot"` 则数据存放在所有节点上，此时如果对数据进行存储到指定的节点上则需要通过命令设置
    curl -XPUT -d '{"settings": {"index.routing.allocation.require.box_type": "hot" }}' http://youip:9200/you_index/_settings?pretty
    之后数据就会写入带有"node.attr.box_type: hot"的节点上
    通过上面的命令也就可以把指定的索引迁移到指定的数据节点上
    ```

    

- 启动

  ```shell
  systemctl start elasticsearch.service
  systemctl enbale elasticsearch.service
  ```



#### 启用x-pack

##### 设置密码

```shell
# 在elasticsearch的主配置文件中加入如下配置
xpack.security.enabled: true
# 重启集群 
systemctl restart elasticsearch
# 使用如下命令进行密码设置
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

##### 破解x-pack

 由于x-pack试用只有一年的时间，到期后如果不购买，整个elasticsearch集群就会打不开会导致整个日志系统不可用 

操作如下:

- 下载反编译工具

  ```
  下载 luyten 下载页面：https://github.com/deathmarine/Luyten/releases 软件下载下来，打开软件
  ```

  

- 开始破解

   把x-pack-core-6.6.0.jar 这个我安装在/usr/share/elasticsearch/modules/x-pack-core/x-pack-core-6.6.0.jar 丢进去，就能看到我们jar包的源代码了。 我们需要把2个文件提取出来进行修改。 org.elasticsearch.license.LicenseVerifier org.elasticsearch.xpack.core.XPackBuild 修改LicenseVerifier 

  ```java
  public class LicenseVerifier
  {
      public static boolean verifyLicense(final License license, final byte[] encryptedPublicKeyData) {
          return true;
      }
      public static boolean verifyLicense(final License license) {
          return true;
      }
  }
  ```

   修改XPackBuild 

  ```java
  static {
          final Path path = getElasticsearchCodebase();
          String shortHash = null;
          String date = null;
          Label_0109: {
              shortHash = "Unknown";
              date = "Unknown";
          }
          CURRENT = new XPackBuild(shortHash, date);
      }
  ```

   重新编译这两个文件 

  ```shell
  javac -cp "/usr/share/elasticsearch/modules/x-pack-core/x-pack-core-6.6.0.jar:/usr/share/elasticsearch/lib/*" LicenseVerifier.java
  javac -cp "/usr/share/elasticsearch/modules/x-pack-core/x-pack-core-6.6.0.jar:/usr/share/elasticsearch/lib/*" XPackBuild.java
  ```

   将编译好的文件重新压缩到jar中 

  ```shell
  jar xvf x-pack-core-6.6.0.jar
  # 覆盖后 压缩
  jar -cf x-pack-core-6.6.0.jar *
  ```

  此时，已破解完成，还需要导入授权文件即可

- 导入授权文件

   从官网申请licence https://license.elastic.co/registration 并修改licence 

  ``````json
  {
      "license":{
          "uid":"15651de2-0eeb-450b-8eec-4230e658a78",
          "type":"platinum", # 白金版 可以使用所有功能
          "issue_date_in_millis":1550102400000,
          "expiry_date_in_millis":3475121398000, # 2080年过期
          "max_nodes":100,
          "issued_to":"lishan zhou (b)",
          "issuer":"Web Form",
          "signature":"AAAAAwAAAA2y1EXkrzMmMPy5DXiwAAABmC9ZN0hjZDBGYnVyRXpCOW5Bb3FjZDAxOWpSbTVoMVZwUzRxVk1PSmkxaktJR
          "start_date_in_millis":1550102400000
      }
  }
  ``````

  上传到elasticsearch集群

  ```shell
  curl -u username:passwd -XPUT ‘http://es-ip:port/_xpack/license’ -H “Content-Type: application/json” -d @/tmp/license.json
  ```

   之后需要重启elasticsearch 重启后发现日志报错说是需要ssl/TLS打开 

- 打开ssl

  ```shell
  /usr/share/elasticsearch/bin/elasticsearch-certutil ca
  /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
  # 然后把ca文件复制到各个节点中， 在elasticsearch增加如下配置；
  xpack.security.transport.ssl.enabled: true
  xpack.security.transport.ssl.verification_mode: certificate
  xpack.security.transport.ssl.keystore.path: elastic-certificates.p12 # 这个相对于/etc/elasticsearch这个路径
  xpack.security.transport.ssl.truststore.path: elastic-certificates.p12 # 这个相对于/etc/elasticsearch这个路径
  
  # 再次重启发现在集群状态正常
  # 查看x-pack过期
  curl -XGET -u username:passwd 'http://10.10.10.11:9200/_license'
  ```

   以上整个elasticsearch集群就安装完成并实现了x-pack认证功能 

### logstash

- 安装 	

  ```shell
  yum install logstash -y 
  ```

- 配置文件说明

  ```shell
  ll /etc/logstash
  drwxrwxr-x 2 root root 20480 Dec 18 14:32 conf.d # 存放pipeline配置的目录，比如 h5-web.conf
  -rw-r--r-- 1 root root 1846 Dec 17 15:54 jvm.options # 配置logstash 启动时的内存大小及其他jvm相关的配置
  -rw-r--r-- 1 root root 4568 Dec 7 05:07 log4j2.properties # logstash 日志相关的配置
  -rw-r--r-- 1 root root 342 Dec 7 05:07 logstash-sample.conf # 一个默认的pipeline配置
  -rw-r--r-- 1 root root 8210 Dec 18 13:12 logstash.yml # 主配置文件
  -rw-r--r-- 1 root root 285 Dec 7 05:07 pipelines.yml # pipeline配置
  -rw------- 1 root root 1696 Dec 7 05:07 startup.options # logstash启动相关参数 
  ```

  主配置文件

  ```yaml
  node.name: es01
  path.data: /var/lib/logstash
  
  pipeline.workers: 48    # 按cpu core数
  pipeline.output.workers: 48  # 按cpu core数
  pipeline.batch.size: 9000  # 每提交到elasticsearch的数据 默认为125 适当的调整这个数据可能提高elasticsearch的写入效率
  pipeline.batch.delay: 10000  # 每次延迟多少时间(毫秒)提交，适当的调整这个数据可能提高elasticsearch的写入效率
  # pipeline.unsafe_shutdown: false
  
  config.support_escapes: true   # 不转义功能如 \n 就是字符串'\n'，而不是换行
  # 队列配置，保持默认即可
  # queue.type: memory
  # queue.max_events: 0
  #
  # queue.max_bytes: 1024mb
  # queue.checkpoint.acks: 1024
  # queue.checkpoint.writes: 1024
  # queue.checkpoint.interval: 1000
  # dead_letter_queue.enable: false
  # dead_letter_queue.max_bytes: 1024mb
  # path.dead_letter_queue:
  #
  # ------------ Metrics Settings --------------
  #
  # Bind address for the metrics REST endpoint
  # http.host: "127.0.0.1"
  # http.port: 9600-9700
  # ------------ Debugging Settings --------------
  #
  # Options for log.level:
  #   * fatal
  #   * error
  #   * warn
  #   * info (default)
  #   * debug
  #   * trace
  #
  # log.level: info
  path.logs: /var/log/logstash
  
  # 启动监控
  xpack.monitoring.enabled: true
  xpack.monitoring.elasticsearch.username: logstash_system
  xpack.monitoring.elasticsearch.password: passwd
  xpack.monitoring.elasticsearch.url: ["http://10.10.10.11:9200", "http://10.10.10.12:9200", "http://10.10.10.13:9200", "http://10.10.10.14:9200", "10.10.10.15:9200"]
  xpack.monitoring.elasticsearch.sniffing: true
  xpack.monitoring.collection.interval: 10s
  xpack.monitoring.collection.pipeline.details.enabled: true
  ```

  pipeline配置

  ```shell
  cat pipelines.yml
  
  - pipeline.id: main
    path.config: "/etc/logstash/conf.d/*.conf"
  ```

  配置从kafka从读取日志并写入elasticsearch，这里使用三个配置文件，分别为input.conf filter.conf output.conf 文件内容分别为:

  - input.conf

    ```
    input {
            kafka {
                    id => "you_id"
                    bootstrap_servers => "kafka1:9092,kafka2:9092"
                    topics => ["you_topics"]
                    client_id => "you_client_Id"
                    group_id => "you_group_id"
                    auto_offset_reset => "latest"
                    consumer_threads => 2
                    max_poll_interval_ms => "600000"
                    request_timeout_ms => "60000"
                    session_timeout_ms => "60000"
                    max_poll_records => "100"
                    type => "you_type"
            }       
    }
    ```

    

  - filter.conf

    ```
    filter {
            json {
                    source => "message"
            }
            mutate {
                    add_field => { "[@metadata][index_name]" => "%{[@metadata][topic]}" }
            }
            mutate {
                    lowercase => [ "[@metadata][index_name]" ]  # 索引名转为小写，因elasticsearch 中索引名称不支持大写字母
            }
        
            if [type] == "type_type" {
                    ruby {
                            code => "event.set('index_datetime', event.timestamp.time.localtime.strftime('%Y.%m.%d-%H'))"  # 定义时间格式，增加显示小时
            }
            } else if [type] == "you_type_type" {
                    ruby {
                            code => "event.set('index_datetime', event.timestamp.time.localtime.strftime('%Y.%m.%d-%H'))"
                    }
            } else if [type] == "you_type" {
                    ruby {
                            code => "event.set('index_datetime', event.timestamp.time.localtime.strftime('%Y.%m.%d-%H'))"
                    }
            } else {
                    mutate {
                            add_field => {"hostname" => "%{[host][name]}"}
                            add_field => {"from_ip" => "%{[fields][ip]}"}
                            # 增加自己的字段
                    }
            }
    
            mutate {
                    remove_field => ["beat", "host", "message", "input", "prospector", "offset", "agent", "fields"]  # 清理不需要的字段 
            }
    }
    
    ```

    

  - output.conf

    ```
    output {
            if [type] == "you_type" {
                    elasticsearch {
                            hosts => ["10.10.10.11:9200","10.10.10.12:9200","10.10.10.13:9200","10.10.10.14:9200","10.10.10.15:9200"]
                            index => "索引名称1-%{index_datetime}"
                            user => "username"
                            password => "passwd"
                    }
            } else if [type] == "your_type_type" {
            	 elasticsearch {
                            hosts => ["10.10.10.11:9200","10.10.10.12:9200","10.10.10.13:9200","10.10.10.14:9200","10.10.10.15:9200"]
                            index => "索引名称2-%{index_datetime}"
                            user => "username"
                            password => "passwd"
                    }
            } else {
            	elasticsearch {
                            hosts => ["10.10.10.11:9200","10.10.10.12:9200","10.10.10.13:9200","10.10.10.14:9200","10.10.10.15:9200"]
                            index => "%{[@metadata][index_name]}-%{+YYYY.MM.dd}"
                            user => "username"
                            password => "passwd"
                    }
            }
    }
    
    ```

  - 一些说明

     上面的配置中带有type字段是为了不同的应用，使用if判断然后可以写入不同的索引中，还有一种更简洁的用法，就是使用%{[@metadata][topic]}。 `%{[@metadata][topic]}` 在logstash1.5版本开始，有一个特殊的字段，叫做@metadata。@metadata包含的内容不会作为事件的一部分输出。详情查看 [logstash input kafka配置](https://www.elastic.co/guide/en/logstash/6.6/plugins-inputs-kafka.html#_metadata_fields) 

    

- 启动

  ```shell
  systemctl start logstash
  systemctl enable logstash
  ```

### kibana

安装

```shell
yum install kibana -y 
```

配置较为简单省略

## Filebeat

### 安装filebeat

```shell
yum install filebeat -y
```



### 收集日志配置

主要收集日志为json格式的配置

```yaml
#=========================== Filebeat inputs =============================
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /opt/logs/spring/blgroup-message-api/blgroup-message-api_VM_10_201_35_204_rhel.log
    - /opt/logs/spring/blgroup-message-api/blgroup-message-api_error_VM_10_201_35_204_rhel.log
  # 使用JSON格式
  json.keys_under_root: true
  json.overwrite_keys: true
  # 增加字段
  fields:
    topic: you_topics
    ip: 10.10.10.5

#============================= Filebeat modules ===============================
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

setup.kibana:

#================================ Kafka Outputs =====================================
# Configure output to kafka
output.kafka:
  enabled: true
  hosts: ["10.10.10.11:9092", "10.10.10.12:9092", "10.10.10.13:9092", "10.10.10.14:9092", "10.10.10.105:9092"]
  # 不同的应用使用不同的topic
  topic: '%{[fields][topic]}'

#================================ Processors =====================================
# Configure processors to enhance or manipulate events generated by the beat.
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~


#================================ Logging =====================================
# Available log levels are: error, warning, info, debug
logging.level: warning
```



## docker-compose安装elasticsearch

 安装三个节点，第一个节点的9200端口开放出来 

```shell
mkdir elastic
cd elastic
```

 docker-compose.yml如下: 

```yaml
version: '2.2'
services:
  elastic1:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.5.4
    container_name: elastic1
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - node.name=elastic1
      - "discovery.zen.minimum_master_nodes=3" # 设置最小主节点个数
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" # 设置内存大小
      - xpack.security.enabled=true # 设置使用xpack
      - xpack.monitoring.enabled=true # 启动监控
      - xpack.monitoring.collection.enabled=true # 启用监控收集数据
    ulimits:
      memlock:
        soft: -1
        hard: -1

    volumes:
      - esdata1:/usr/share/elasticsearch/data

    ports:
      - 9200:9200 # 端口映射
    networks:
      - esnet

  elastic2:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.5.4
    container_name: elastic2
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - node.name=elastic2
      - "discovery.zen.minimum_master_nodes=3"
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=elastic1"
      - xpack.security.enabled=true
      - xpack.monitoring.enabled=true
      - xpack.monitoring.collection.enabled=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata2:/usr/share/elasticsearch/data
    depends_on:
      - elastic1
    networks:
      - esnet

  elastic3:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.5.4
    container_name: elastic3
    environment:
      - cluster.name=docker-cluster
      - node.name=elastic3
      - bootstrap.memory_lock=true
      - "discovery.zen.minimum_master_nodes=3"
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=elastic1"
      - xpack.security.enabled=true
      - xpack.monitoring.enabled=true
      - xpack.monitoring.collection.enabled=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata3:/usr/share/elasticsearch/data
    depends_on:
      - elastic1
    networks:
      - esnet
  kibana:
    image: docker.elastic.co/kibana/kibana:6.5.4
    container_name: kibana
    ports:
      - 5601:5601
    environment:
      SERVER_NAME: kibana
      ELASTICSEARCH_URL: http://10.10.10.2:9200
      ELASTICSEARCH_USERNAME: username
      ELASTICSEARCH_PASSWORD: you're password

    depends_on:
      - elastic1

volumes:
  esdata1:
    driver: local
  esdata2:
    driver: local
  esdata3:
    driver: local

networks:
  esnet:
```

执行命令:

```shell
docker-compose up -d
```

 开启 x-pack后需要执行以下命令, 来启用试用licence 

```shell
 curl -u username:passwd -H "Content-Type:application/json" -XPOST  http://127.0.0.1:9200/_xpack/license/start_trial?acknowledge=true
```

查看所运行的容器信息

```
# docker ps -a
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS                              NAMES
c5351bab0826        docker.elastic.co/elasticsearch/elasticsearch:6.5.4   "/usr/local/bin/dock…"   14 minutes ago      Up 14 minutes       9200/tcp, 9300/tcp                 elastic3
927b944eb914        docker.elastic.co/kibana/kibana:6.5.4                 "/usr/local/bin/kiba…"   14 minutes ago      Up 14 minutes       0.0.0.0:5601->5601/tcp             kibana
7e4cbb191213        docker.elastic.co/elasticsearch/elasticsearch:6.5.4   "/usr/local/bin/dock…"   14 minutes ago      Up 14 minutes       9200/tcp, 9300/tcp                 elastic2
a578c7df15e5        docker.elastic.co/elasticsearch/elasticsearch:6.5.4   "/usr/local/bin/dock…"   14 minutes ago      Up 14 minutes       0.0.0.0:9200->9200/tcp, 9300/tcp   elastic1

```



> 本文为原创文章，转载注明出处  
>
> 如果帮到你了，还请多支持

