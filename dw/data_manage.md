# 数据管理

## 元数据管理

> 数据治理的基础是要有一个好的元数据管理。这里的元数据管理包括技术元数据和业务元数据。
元数据的质量直接影响到数据管理的准确性，如何把元数据建设好
将起到至关重要的作用。元数据建设的目标是打通数据接入到加 ，再
到数据消费整个链路，规范元数据体系与模型，提供统 的元数据服
出口，保障元数据产出的稳定性和质量。

1. 技术元数据
    * hive 表信息，包括库表列。表的责任人，分区信息，文件大小，生命周期，注释等
    * 任务运行信息，类似 sqark 任务的调度信息，包括作业类型，执行参数，实例名称，输入输出，执行时间等。
    * 数据平台中的任务同步日志，输入输出表和字段任务依赖，依赖的类型，不同任务类型的调度信息
    * 数据运维和任务监控信息，监控日志，运维报警，数据质量，数据故障
2. 业务元数据（业务同学都能理解的数据）
    * 包含 ondata 体系定义的业务过程指标，维度及属性等规范化的定义。用于更好的管理和使用数据
    * 数据应用元数据，包括报表，自助取数，数据产品等运行的元数据

## 元数据架构体系

![](https://oss.dataown.cn/data/2020/9/fc45f42e02a775bd.jpeg)