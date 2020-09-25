# 数据接入

数据接入是整个大数据的源头，汇聚了整个异构数据到大数据的仓库中，也是整个数据计算的起点。
数据接入有以下要求和特点
* 流控，避免对源头业务库造成影响
* 并发，要能快速的获取同步数据
* 异常处理，对异常数据能够抛出错误，及时发现

这里要讲的是阿里开源的DataX这个工具。
DataX 是阿里开源的一个异构数据源离线同步工具

## 架构

采用Framework + plugin架构构建
将数据源读取和写入抽象成为Reader/Writer插件

![dataX模型图](https://oss.dataown.cn/data/2020/9/c2723627442b1203.jpg)

* Reader：数据采集模块，负责采集数据源的数据，将数据发送给Framework。

* Writer： 数据写入模块，负责不断向Framework取数据，并将数据写入到目的端。

* Framework：用于连接reader和writer，作为两者的数据传输通道，并处理**缓冲，流控，并发，数据转换**等核心技术问题。


## 流程

* DataX完成单个数据同步的作业，我们称之为Job，DataX接受到一个Job之后，将启动一个进程来完成整个作业同步过程。DataX Job模块是单个作业的中枢管理节点，承担了数据清理、子任务切分(将单一作业计算转化为多个子Task)、TaskGroup管理等功能。

* DataXJob启动后，会根据不同的源端切分策略，将Job切分成多个小的Task(子任务)，以便于并发执行。Task便是DataX作业的最小单元，每一个Task都会负责一部分数据的同步工作。

* 切分多个Task之后，DataX Job会调用Scheduler模块，根据配置的并发数据量，将拆分成的Task重新组合，组装成TaskGroup(任务组)。每一个TaskGroup负责以一定的并发运行完毕分配好的所有Task，默认单个任务组的并发数量为5。

* 每一个Task都由TaskGroup负责启动，Task启动后，会固定启动Reader—>Channel—>Writer的线程来完成任务同步工作。

* DataX作业运行起来之后， Job监控并等待多个TaskGroup模块任务完成，等待所有TaskGroup任务完成后Job成功退出。否则，异常退出，进程退出值非0。

## 缺点
1. 没有自增主键，对于数据量大的表，抽取速度慢
2. 需要依赖调度系统实现分布式
3. 不支持自动创建表和分区，写入的hdfs路径必须存在
4. 生成配置文件比较繁琐（每张表需要生成一张配置文件，可以使用代码生成）

## 优点
1. 除比较大的表之外，速度明显比sqoop快
2. Datax的速度可以配置（配置 channel 的数量）
3. 对于脏数据的处理

    * 在大量数据的传输过程中，必定会由于各种原因导致很多数据传输报错(比如类型转换错误)，这种数据DataX认为就是脏数据。DataX目前可以实现脏数据精确过滤、识别、采集、展示，提供多种的脏数据处理模式。

    * Job支持用户对于脏数据的自定义监控和告警，包括对脏数据最大记录数阈值（record值）或者脏数据占比阈值（percentage值），当Job传输过程出现的脏数据大于用户指定的数量/百分比，DataX Job报错退出。
    * 图中的配置的意思是当脏数据大于10条，或者脏数据比例达到0.05%，任务就会报错
    
4. 健壮的容错机制 （重试次数设置）
5. 丰富的数据转换功能
6. DataX作为一个服务于大数据的ETL工具，除了提供数据快照搬迁功能之外，还提供了丰富数据转换的功能，让数据在传输过程中可以轻松完成数据脱敏，补全，过滤等数据转换功能，另外还提供了自动groovy函数，让用户自定义转换函数。

## 利用dataX和调度接口构建OneClick应用

> 例如将 mysql 增量同步到 hive 表中，主要是实现以下标准流程。

1. hive stage层，ods层表结构根据mysql表结构确定（同时对字段类型做转换，对关键字处理）
2. 数据的增量获取（通过对mysql binlog 订阅消费）
3. stage 层缓冲的数据跟 ods 历史全量数据合并
### 第一点实现
通过 `mysql information_schema.COLUMNS，information_schema.TABLES` 获取对应表信息

```sql
select TABLE_SCHEMA ,TABLE_NAME ,COLUMN_NAME ,DATA_TYPE ,COLUMN_TYPE ,COLUMN_COMMENT 
from information_schema.COLUMNS 
where TABLE_SCHEMA = '{0}' and DATA_TYPE not in ('blob') 
```

### 第二点实现
通过 canal 模拟 mysql 从库，获取 binlog 数据，otter 将数据sink到rocket mq中

![](https://oss.dataown.cn/images/2020/08/24/d5308625a21bd7c1fb11dd5716d1d47a.jpg)

并新增部分关键字段

```
"execute_time", -- 事件落到 mq 中的时间
"event_type", -- 事件类型
"binlog_name", -- binlog 名字
"binlog_position" -- binlog 文件中偏移量
```

### 第三点实现
> 增量表与历史全量实现 full join 合并生成新的全量表（先 row_number 排序再 full outer join）

### 流程图

![](https://oss.dataown.cn/data/2020/9/53eab810f4843b3f.png)

