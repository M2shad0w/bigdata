## 工具知识点

### 组件

1. kafka
2. redis
3. hbase
4. flink
5. spark
6. hive

### hive

1. 支持动态分区设置
```
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
``` 
2. 让子查询并行跑
```
set hive.exec.parallel=true;
```
3. 将顶层的聚合操作放在Map阶段执行，从而减轻清洗阶段数据传输和Reduce阶段的执行时间，提升总体性能。

    ```
    #缺点：该设置会消耗更多的内存

    set hive.map.aggr=true;
    ```
4. 对MapReduce程序map out数据进行压缩配置
```
set mapreduce.map.output.compress=true;
set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec;
``` 
5. 开启中间压缩(map输出结果(临时的)压缩)
```
set hive.exec.compress.intermediate=true;
```
6. 设置Mapper中的Kvbuffer的大小,默认100M,调大的话,会减少磁盘spill的次数
```
set mapreduce.task.io.sort.mb=1024;
``` 
7. 写数据到磁盘时，进行merge的时候最多能同时merge多少spill的设置,默认10
```
set mapreduce.task.io.sort.factor=100;
```
8. 当Map Task完成的比例达到该值后才会为Reduce Task申请资源，默认是0.05
```
set mapreduce.job.reduce.slowstart.completedmaps=0.7;
```
  
9. 在map执行前合并小文件，减少map数
```
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```
10. 设置合并文件的大小，100000000表示100M

    ```
    set mapred.max.split.size=100000000;
    set mapred.min.split.size.per.node=100000000;
    set mapred.min.split.size.per.rack=100000000;
    
    注意各参数的设置大小，不要冲突，否则会异常，大小顺序如下
    mapred.max.split.size <= mapred.min.split.size.per.node <= mapred.min.split.size.per.rack
    ```  
11. map和reduce的内存大小设置
    ```
    set mapreduce.map.memory.mb=10240;
    set mapreduce.reduce.memory.mb=10240;
    #jvm的内存大小设置
    set mapreduce.map.java.opts=-Xmx8196m;
    set mapreduce.reduce.java.opts=-Xmx8196m;
    ``` 
12. reduce个数设置
```
set hive.exec.reducers.max=400;
``` 
13. reduce端小文件合并设置
```
set hive.merge.mapredfiles=true;
```
14. 如果是join过程出现倾斜 应该设置为true
```
set hive.optimize.skewjoin=true;
```
15. 如果是group by过程出现倾斜 应该设置为true
```
set hive.groupby.skewindata=true;
```
16. 让hive自动识别表的join方式，并变成合适的Map Join.

    ```
    #当设置为true的时候，hive会自动获取两张表的数据，判定哪个是小表，然后放在内存中

    set hive.auto.convert.join=true;
    ```

17. 大表和大表join时，设置为SMB
```hive
(Sort-Merge-Bucket)的Join方式，显著提升性能
set hive.auto.convert.sortmerge.join=true;
set hive.optimize.bucketmapjoin=true;
set hive.optimize.bucketmapjoin.sortedmerge=true;
```