---
title: "Pushgateway的使用案例"
date: 2020-08-27T22:38:40+08:00
draft: false
tags: ["prometheus", "pushgateway", "监控", "python", "requests"]
categories: ["prometheus"]
keywords: ["prometheus", "pushgateway", "监控", "python", "requests"]
---

**参考:** [pushgatewy GITHUB](https://github.com/prometheus/pushgateway)

我们都知道prometheus是主动的向被监控端拉取监控数据的，如果有需要向prometheus推送数据只能通过pushgateway的方式。

我们可以使用Prometheus监控到们大部分的应用和主机，但是部有会一些监控数据是无法通过在主机节点上部署一个exporter就能解决的。

比如我这边在工作中就遇到监控oracle的相关数据时就比较很不好监控，一些监控所需要的数据需要通过sql语句才能获得到。所以这边使用pushgate可以很好的解决我的问题。

### 下载和安装

```bash
# 下载
cd /usr/local/src
wget https://github.com/prometheus/pushgateway/releases/download/v1.2.0/pushgateway-1.2.0.linux-amd64.tar.gz
# 安装
tar xvf pushgateway-1.2.0.linux-amd64.tar.gz -C /usr/local
ln -sv /usr/local/pushgateway-1.2.0.linux-amd64 /usr/local/pushgateway
```

### 启动和配置

#### 启动和服务

启动

```shell
#可以直接启动
/usr/local/pushgateway 
# 默认是在前台启动 监听的端口为9091
```

pushgateway命令

```shell
usage: pushgateway [<flags>]

The Pushgateway

Flags:
  -h, --help                    
      --web.listen-address=":9091"    监控的地址和端口
      --web.telemetry-path="/metrics"   获得metrics的路径
      --web.external-url=        The URL under which the Pushgateway is externally reachable.
      --web.route-prefix=""      Prefix for the internal routes of web endpoints. Defaults to the path of --web.external-url.
      --web.enable-lifecycle    可以通过http来关闭pushgateway
      --web.enable-admin-api    启动admin api
      --persistence.file=""      指定存储的文件 默认存储在内存中,如果pushgateway异常,重启则数据不会被保留
      --persistence.interval=5m  指定存储的文件更新的频率
      --push.disable-consistency-check   不要检查推送的度量标准的一致性 这个参数比较危险 
      --log.level=info         日志级别
      --log.format=logfmt        Output format of log messages. One of: [logfmt, json]
      --version                  Show application version.
```



使用systemctl来启动，脚本如下:

```shell
$ vim /usr/lib/systemd/system/pushgateway.service
[Unit]
Description=pushgateway
After=network.target
[Service]
LimitCORE=infinity
LimitNOFILE=65535
LimitNPROC=65535
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/pushgateway/pushgateway --web.enable-lifecycle --web.enable-admin-api --persistence.file=/usr/local/pushgateway/metrics_data --persistence.interval=3m
Restart=on-failure
[Install]
WantedBy=multi-user.target

```

上面启动脚本我们指定了 本地存储位置和更新时间

启动

```shell
systemctl daemon-reload 
systemctl start pushgateway.service
systemctl status pushgateway.service  # 查看状态
 systemctl enable pushgateway.service  # 开机自动启动
journalctl -xf -u pushgateway.service # 查看日志

```



**也可以使用docker来启动 详细参数github地址**

### 数据推送

pushgateway 提供了数据推送接口 即 http://:9091/metrics   通过PUT 或 POST方法推送数据 DELETE可删除数据 

格式如下:

`http://:9091/metrics/job/<JOB_NAME>{/<LABEL_NAME>/<LABEL_VALUE>}`

具体可以参考官网或其他文章。这里给出prometheus与pushgateway的配置

```yaml
  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['192.168.1.100:9091']
        labels:
          group: 'pushgateway'
```

` honor_labels: true`  如果不配置则为false 。 此配置的说明 官方有很详细的说明: [About the job and instance labels](https://github.com/prometheus/pushgateway#about-the-job-and-instance-labels) 和prometheus的 [scrape_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)

当不配置时或为false时 我们通过api 推送我们的监控数据，如下所示:

```shell
echo "test_metric 123456" | curl --data-binary @- http://192.168.1.100:9091/metrics/job/test_job
```

不过我们会发现，除了 `test_metric` 外，同时还新增了 `push_time_seconds`  和 `push_failure_time_seconds` 两个指标，这两个是 PushGateway 系统自动生成的相关指标

我们prometheus查询的结果：

`test_metric{exported_job="test_job",instance="pushgateway",job="pushgateway"}`

为了使用推送的上来的job和instance 不被 prometheus本身监控pushgateway的标签数据覆盖则需要设置

`honor_labels: true`

以上都是或网络上介绍的都是curl的命令来推送数据的，下面来使用reqeusts来推送数据。

### 使用python的requests来推送数据

```python
# -*- coding: utf-8 -*-
import requests

PUSH_GATEWAY_URL = 'http://192.168.1.100:9091/metrics'


def push_metrics(metric_name, value, job, instance, labels: dict, type_str: str = None):
    """
    把数据push到prometheus上进行监控
    :param metric_name: 监控项名称
    :param value: 值 只能为数字
    :param job: 对应的job
    :param instance: 实例名，一般为ip地址
    :param labels: 标签
    :param type_str: 监控数据说明可为空 --> # TYPE some_metric counter/gauge
    :return:
    """
    headers = {
        'X-Requested-With': 'Python requests',
        'Content-type': 'text/xml'
    }
    # 组合访问地址
    url = f"{PUSH_GATEWAY_URL}/job/{job}/instance/{instance}"
    if labels:
        for k, v in labels.items():
            url += f"/{k}/{v}"

    payload = f"{type_str}\n{metric_name} {value}\n"

    response = requests.request("POST", url, headers=headers, data=payload)
    if response.status_code == 200:
        return True
    else:
        return False


def delete_metrics(job, instance, labels: dict):
    """
    删除指定的job和实例的数据
    :param job: 对应的job
    :param instance: 实例名，一般为ip地址
    :param labels: 标签
    :return:
    """

    headers = {
        'X-Requested-With': 'Python requests',
        'Content-type': 'text/xml'
    }

    # 组合访问地址
    url = f"{PUSH_GATEWAY_URL}/job/{job}/instance/{instance}"
    if labels:
        for k, v in labels.items():
            url += f"/{k}/{v}"

    response = requests.request("DELETE", url, headers=headers)
    if response.status_code == 202:
        return True
    return False

```

还可以python 的prometheus的prometheus_client  这个涉猎的不多。如果需要的可自行在官方文档上学习。



---



