---
title: "Python与Rabbitmq"
date: 2020-04-17T21:13:30+08:00
draft: false
tags: ["python", "rabbitmq", "kombu"]
categories: ["Python"]
keywords: ["python", "rabbitmq", "kombu"]
---

## 使用docker安装rabbitmq

拉取镜像

```shell
docker pull rabbitmq:management # 直接使用带web管理的
```

生成容器

```shell
mkdir /home/rabbitmq_data
docker run -itd --name rabbitmq -p 5672:5672 -p 15672:15672 -v /home/rabbitmq_data:/var/lib/rabbitmq -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin rabbitmq:management
```

* RABBITMQ_DEFAULT_USER 用户名
* RABBITMQ_DEFAULT_PASS 密码

访问web端

![](https://raw.githubusercontent.com/zhou-mfk/blogimages/master/img/20200416211044.png)

## kombu模块

[kombu官方站点](https://kombu.readthedocs.io/)

使用pip安装

```shell
pip install kombu
```



### 生产者

```python
from kombu import Connection


def kombu_production(key, body, key_timeout=None):
    """
    rabbitmq 生产者
    :param key: 队列名
    :param body: 队列数据
    :param key_timeout: type:int 单位:s 超时时间 默认不过期
    :return:
    """
    MQ_URL = "amqp://admin:admin@192.168.1.100:5672//"
    connection = Connection(MQ_URL)
    channel = connection.Producer(serializer='json')
    channel.publish(body=body, routing_key=key, expiration=key_timeout)
    connection.release()
    
```

### 消费者

```python
from kombu import Connection, Queue


class Consumer(object):
    """任务队列消费者"""

    def __init__(self, queue_name):
        """
        :param queue_name: 队列名称
        """
        self.queue_name = queue_name
        self.connection = Connection('amqp://admin:admin@192.168.1.100:5672//')
        
    def run(self):
        """开始消费队列"""
        while True:
            try:
                with self.connection.Consumer(
                        queues=[Queue(self.queue_name, routing_key=self.queue_name)],
                        accept=['pickle', 'json'],
                        callbacks=[self.task_callbacks],
                        prefetch_count=1  # 每次取得一个消息
                ):
                    self.connection.drain_events(timeout=10)
            except socket.timeout as e:
                logger.error(f'运行时异常 {self.queue_name} {e}')
                break
            except Exception as e:
                logger.error(f'运行时异常 {self.queue_name} {e}')
                break

    @staticmethod
    def task_callbacks(body, message):
        """任务回调函数(重写)"""
        message.requeue()
        

class MyConsumer(Consumer):
    """
    自定义的一个消费者
    """
    
    def task_callbacks(body, message):
        if body:
            print(body)
        return message.ack
    
    
if __name__ == "__main__":
    my_consumer = MyConsumer('MyQueue')
    my_consumer.run()
```



以上即是python 与 rabbitmq 相关的代码片段

---