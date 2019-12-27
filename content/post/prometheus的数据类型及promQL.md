---
title: "Prometheus的数据类型及promQL"
date: 2019-12-10T16:52:08+08:00
draft: false
tags: ["prometheus", "监控"]
categories: ["prometheus"]
keywords: ["prometheus", "监控"]
---

> Prometheus通过指标名称（metrics name）以及对应的一组标签（labelset）唯一定义一条时间序列。指标名称反映了监控样本的基本标识，而label则在这个基本特征上为采集到的数据提供了多种特征维度。用户可以基于这些特征维度过滤，聚合，统计从而产生新的计算后的一条时间序列。

## Metrics类型

Prometheus定义了4中不同的指标类型(metric type)：

* Counter（计数器）
* Gauge（仪表盘）
* Histogram（直方图）
* Summary（摘要）

### Counter 只增不减的计数器

Counter类型的指标其工作方式和计数器一样，只增不减（除非系统发生重置）

如监控项中的网卡进出流量数据只增加发送或接收的量，这样就可以使用的内置rate() 或irate() 来计算进出的网络速度：

```shell
rate(net_bytes_recv{host="$host",interface=~"eth.*|bond.*"}[5m])
计算最近5m的网卡流量
```


### Gauge 可增可减仪表盘

Gauge类型的指标侧重于反应系统的当前状态。因此这类指标的样本数据可增可减的瞬时值。

这类的数据不可以用类似rate()的函数。 可以使用predict_linear()来预测未来数据趋势

如: 

```shell
predict_linear(node_filesystem_free{job="node"}[2h], 4 * 3600) < 0 
# 根据前两小时的数据预测试此后4小时的磁盘空间是否小于0
```

### 使用Histogram和Summary分析数据分布情况

Histogram和Summary主用用于统计和分析样本的分布情况

例如: prometheus 的查询持续时间情况：

```shell
# HELP prometheus_engine_query_duration_seconds Query timings
# TYPE prometheus_engine_query_duration_seconds summary
prometheus_engine_query_duration_seconds{slice="inner_eval",quantile="0.5"} 3.3634e-05
prometheus_engine_query_duration_seconds{slice="inner_eval",quantile="0.9"} 0.000148559
prometheus_engine_query_duration_seconds{slice="inner_eval",quantile="0.99"} 0.086480298
prometheus_engine_query_duration_seconds_sum{slice="inner_eval"} 2.8547534292935394e+06
prometheus_engine_query_duration_seconds_count{slice="inner_eval"} 1.260549577e+09
```

类型为Histogram的监控指标

```shell
# HELP prometheus_tsdb_compaction_chunk_range_seconds Final time range of chunks on their first compaction
# TYPE prometheus_tsdb_compaction_chunk_range_seconds histogram
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="100"} 836536
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="400"} 836536
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="1600"} 836536
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="6400"} 836536
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="25600"} 1.81803349e+08
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="102400"} 1.99503881e+08
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="409600"} 2.05460811e+08
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="1.6384e+06"} 2.0996758e+08
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="6.5536e+06"} 1.478032501e+10
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="2.62144e+07"} 1.4781616372e+10
prometheus_tsdb_compaction_chunk_range_seconds_bucket{le="+Inf"} 1.4781616372e+10
prometheus_tsdb_compaction_chunk_range_seconds_sum 2.6226729879339772e+16
prometheus_tsdb_compaction_chunk_range_seconds_count 1.4781616372e+10
```

与Summary类型的指标相似之处在于Histogram类型的样本同样会反应当前指标的记录的总数(以_count作为后缀)以及其值的总量（以_sum作为后缀）。不同在于Histogram指标直接反应了在不同区间内样本的个数，区间通过标签len进行定义。



同时对于Histogram的指标，我们还可以通过histogram_quantile()函数计算出其值的分位数。不同在于Histogram通过histogram_quantile函数是在服务器端计算的分位数。 而Sumamry的分位数则是直接在客户端计算完成。因此对于分位数的计算而言，Summary在通过PromQL进行查询时有更好的性能表现，而Histogram则会消耗更多的资源。反之对于客户端而言Histogram消耗的资源更少。在选择这两种方式时用户应该按照自己的实际场景进行选择。

## promQL语言

>  PromQL是Prometheus内置的数据查询语言，其提供对时间序列数据丰富的查询，聚合以及逻辑运算能力的支持。并且被广泛应用在Prometheus的日常应用当中，包括对数据查询、可视化、告警处理当中。可以这么说，PromQL是Prometheus所有应用场景的基础，理解和掌握PromQL是Prometheus入门的第一课。



支持数学运算

```shell
+ (加法)
- (减法)
* (乘法)
/ (除法)
% (求余)
^ (幂运算)
```

支持运算符

```
== (相等)
!= (不相等)
> (大于)
< (小于)
>= (大于等于)
<= (小于等于)
```

支持集合运算符

```
and (并且)
or (或者)
unless (排除)
```



使用bool类型 返回0或1

如果http请求总数大于1000 则返回1，否则返回0  如下所示

```shell
http_requests_total > bool 1000
```



在PromQL操作符中优先级由高到低依次为

```
^
*, /, %
+, -
==, !=, <=, <, >=, >
and, unless
or
```



聚合操作

```
sum (求和)
min (最小值)
max (最大值)
avg (平均值)
stddev (标准差)
stdvar (标准差异)
count (计数)
count_values (对value进行计数)
bottomk (后n条时序)
topk (前n条时序)
quantile (分布统计)
```

内置函数

```
increase
rate/irate
predict_linear 预测
```

label动态替换

```
label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)
```

