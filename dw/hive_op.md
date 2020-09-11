# mapreduce过程资源优化


 要了解mapreduce的资源优化，首先应该要熟悉整个mapreduce的过程，大致流程可以分**为map，sort，spill，shuffle，reduce**等过程，分阶段的来分析一下，都有哪些可以进行调优。

 ![map reduce 流程图](https://oss.dataown.cn/data/2020/9/a7d1f7bf628b9aa6.png)


<a name="6a89956c"></a>
##  map阶段调优


 这里的map阶段仅指map函数过程以及之前，并不涉及环形缓冲区，因为那属于shuffle阶段。在map阶段有FileInputFormat这么一个过程，切片，一个切片对应一个map task，每个切片大小的计算公式：


```sql
num_Map_tasks = max[${Mapred.min.split.size}，min(${dfs.block.size}，${Mapred.max.split.size})] 
Mapred.min.split.size指的是数据的最小分割单元大小。 
Mapred.max.split.size指的是数据的最大分割单元大小。 
dfs.block.size指的是HDFS设置的数据块大小。
```





 一般来说
 
  dfs.block.size
 
 这个值是一个已经指定好的值，而且这个参数Hive是识别不到的。所以实际上只有
 
  Mapred.min.split.size
 
 和
 
  Mapred.max.split.size
 
 这两个参数（本节内容后面就以min和max指代这两个参数）来决定Map数量。在Hive中min的默认值是1B，max的默认值是256MB。



 
```sql
Hive> set Mapred.min.split.size; Mapred.min.split.size=1 
Hive> set Mapred.max.split.size; Mapred.max.split.size=256000000
```

 


 所以如果不做修改的话，就是1个Map task处理128MB数据(其实就是
 
  dfs.block.size
 
 )，我们就以调整max为主。通过调整max可以起到调整Map数的作用，减小max可以增加Map数，但是增大max不一定能影响到Map数。需要提醒的是，直接调整
 
  Mapred.Map.tasks
 
 这个参数是没有效果的。



 调整大小的时机根据查询的不同而不同，总的来讲可以通过观察Map task的完成时间来确定是否需要增加Map资源。如果Map task的完成时间都是接近1分钟，甚至几分钟了，那么往往增加Map数量，使得每个Map task处理的数据量减少，能够让Map task更快完成；而如果Map task的运行时间已经很少了，比如10-20秒，这个时候增加Map不太可能让Map task更快完成，反而可能因为Map需要的初始化时间反而让Job总体速度变慢，这个时候反而需要考虑是否可以把Map的数量减少，这样可以节省更多资源给其他Job。


<a name="8ef0805b"></a>
##  reduce阶段调优


 Reduce阶段优化的主要工作也是选择合适的Reduce task数量，跟上面的Map优化类似。与Map优化不同的是，Reduce优化时，可以直接设置
 
  Mapred.Reduce.tasks
 
 参数从而直接指定Reduce的个数。当然直接指定Reduce个数虽然比较方便，但是不利于自动扩展。Reduce数的设置虽然相较Map更灵活，但是也可以像Map一样设定一个自动生成规则，这样运行定时Job的时候就不用担心原来设置的固定Reduce数会由于数据量的变化而不合适。



 Hive估算Reduce数量的时候，使用的是下面的公式：



 
```sql
 num_Reduce_tasks = min[${Hive.exec.Reducers.max}， (${input.size} / ${ Hive.exec.Reducers.bytes.per.Reducer})]
```

 


 也就是说，根据输入的数据量大小来决定Reduce的个数，默认
 
  Hive.exec.Reducers.bytes.per.Reducer
 
 为1G，而且Reduce个数不能超过一个上限参数值，这个参数是
 
  hive.exec.reducers.max
 
 ，新版本默认是1009。所以我们可以调整
 
  Hive.exec.Reducers.bytes.per.Reducer
 
 来设置Reduce个数。



 
  ![](https://oss.dataown.cn/docs/2020/9/0d729f704cc17b1c.png)
 



 设置Reduce数同样也是根据运行时间作为参考调整，并且可以根据特定的业务需求、工作负载类型总结出经验，所以不再赘述


<a name="b0f4b6d8"></a>
##  shuffle阶段调优


 这里可以调节的参数比较多，下面一一介绍一下：


<a name="63d7410b"></a>
###  map buffer（环形缓冲区）


 当map task开始运算，并产生中间数据时，其产生的中间结果并非直接就简单的写入磁盘。这中间的过程比较复杂，并且利用到了内存buffer来进行已经产生的部分结果的缓存，并在内存buffer中进行一些预排序来优化整个map的性能。这个buffer（一般称之为环形缓冲区）默认是100MB大小，但是这个大小是可以根据job提交时的参数设定来调整的.
 <br />
 该参数即为：
 
  mapreduce.task.io.sort.mb
 
 <br />
 当map的产生数据非常大时，并且把mapreduce.task.io.sort.mb调大，那么map在整个计算过程中spill的次数就势必会降低，map task对磁盘的操作就会变少.


<a name="0b137c07"></a>
###  map spill size（溢出百分占比）


 map在运行过程中，不停的向该buffer中写入已有的计算结果，但是该buffer并不一定能将全部的map输出缓存下来，当map输出超出一定阈值（比如80M），那么map就必须将该buffer中的数据写入到磁盘中去，这个过程在mapreduce中叫做spill。
 <br />
 map并不是要等到将该buffer全部写满时才进行spill，因为如果全部写满了再去写spill，势必会造成map的计算部分等待buffer释放空间的情况。所以，map其实是当buffer被写满到一定程度（比如80%）时，就开始进行spill。
 <br />
 这个阈值也是由一个job的配置参数来控制，即
 
  mapreduce.map.sort.spill.percent
 
 ，默认为0.80或80%
 <br />
 这个参数同样也是影响spill频繁程度，进而影响map task运行周期对磁盘的读写频率的。但非特殊情况下，通常不需要人为的调整。调整
 
  mapreduce.task.io.sort.mb
 
 对用户来说更加方便。


<a name="6c706519"></a>
###  map spill file merge（溢出文件合并数量）


 当 map task 的计算部分全部完成后，如果map有输出，就会生成一个或者多个spill文件，这些文件就是map的输出结果。map在正常退出之前(cleanup)，需要将这些spill合并（merge）成一个，所以map在结束之前还有一个merge的过程。merge的过程中，有一个参数可以调整这个过程的行为，该参数为：
 
  mapreduce.task.io.sort.factor
 
 。该参数默认为10。它表示当merge spill文件时，最多能有多少并行的stream向merge文件中写入。比如如果map产生的数据非常的大，产生的spill文件大于10，而
 
  mapreduce.task.io.sort.factor
 
 使用的是默认的10，那么当map计算完成做merge时，就没有办法一次将所有的spill文件merge成一个，而是会分多次，每次最多10个stream。这也就是说，当map的中间结果非常大，调大
 
  mapreduce.task.io.sort.factor
 
 ，有利于减少merge次数，进而减少map对磁盘的读写频率，有可能达到优化作业的目的。


<a name="a21f9329"></a>
###  map combiner（map端的reduce）


 当job指定了combiner的时候，我们都知道map结束后会在map端根据combiner定义的函数将map结果进行合并。运行combiner函数的时机有可能会是merge完成之前，或者之后，这个时机可以由一个参数控制，即
 
  mapreduce.map.combine.minspills
 
 （default 3）。
 <br />
 当job中设定了combiner，并且spill数大于等于3的时候，那么combiner函数就会在merge产生结果文件之前运行。通过这样的方式，就可以在spill非常多需要merge，并且很多数据需要做conbine的时候，减少写入到磁盘文件的数据数量，同样是为了减少对磁盘的读写频率，有可能达到优化作业的目的。


<a name="b1ffe3d9"></a>
###  map output compress（map输出压缩）


 减少中间结果读写进出磁盘的方法不止这些，还有就是压缩。也就是说map的中间，无论是spill的时候，还是最后merge产生的结果文件，都是可以压缩的。压缩的好处在于，通过压缩减少写入读出磁盘的数据量。对中间结果非常大，磁盘速度成为map执行瓶颈的job，尤其有用。控制map中间结果是否使用压缩的参数为：
 
  mapreduce.map.output.compress
 
 (true/false)，
 
  mapred.map.output.compression.codec
 
 可以指定压缩算法，那么map在写中间结果时，就会将数据压缩后再写入磁盘，读结果时也会采用先解压后读取数据。这样做的后果就是：写入磁盘的中间结果数据量会变少，但是cpu会消耗一些用来压缩和解压。所以这种方式通常适合job中间结果非常大，瓶颈不在cpu，而是在磁盘的读写的情况。说的直白一些就是用cpu换IO。根据观察，通常大部分的作业cpu都不是瓶颈，除非运算逻辑异常复杂。所以对中间结果采用压缩通常来说是有收益的。


<a name="f99779ae"></a>
###  reduce shuffle parallelcopies（reduce并行拉取任务数）


 Reduce task在做shuffle时，实际上就是从不同的已经完成的map上去下载属于自己这个reduce的部分数据。由于map通常有许多个，所以对一个reduce来说，下载也可以是并行的从多个map下载这个并行度是可以调整的，调整参数为：
 
  mapreduce.reduce.shuffle.parallelcopies
 
 （default 5）。默认情况下，每个只会有5个并行的下载线程在从map下数据，如果一个时间段内job完成的map有100个或者更多，那么reduce也最多只能同时下载 5个map的数据，所以这个参数比较适合map很多并且完成的比较快的job的情况下调大，有利于reduce更快的获取属于自己部分的数据。


<a name="523bffd7"></a>
###  reduce merge（reduce端的溢写文件合并数）


 Reduce将map结果下载到本地时，同样也是需要进行merge的，所以
 
  mapreduce.task.io.sort.factor
 
 的配置选项同样会影响reduce进行merge时的行为，该参数的详细介绍上文已经提到，当发现reduce在shuffle阶段iowait非常的高的时候，就有可能通过调大这个参数来加大一次merge时的并发吞吐，优化reduce效率。


<a name="ae3d1b9a"></a>
###  reduce spill（reduce端的溢写）


 Reduce在shuffle阶段对下载来的map数据，并不是立刻就写入磁盘的，而是会先缓存在内存中，然后当使用内存达到一定量的时候才刷入磁盘。这个内存大小的控制就不像map一样可以通过
 
  mapreduce.task.io.sort.mb
 
 来设定了，而是通过另外一个参数来设置：
 
  mapreduce.reduce.shuffle.input.buffer.percent
 
 （default 0.7)。
 <br />
 这个参数其实是一个百分比，意思是说，shuffile在reduce内存中的数据最多使用内存量为：0.7 × maxHeap of reduce task。也就是说，如果该reduce task的最大heap使用量（通常通过
 
  mapreduce.reduce.java.opts
 
 来设置，比如设置为-Xmx1024m）的一定比例用来缓存数据。
 <br />
 默认情况下，reduce会使用其heapsize的70%来在内存中缓存数据。如果reduce的heap由于业务原因调整的比较大，相应的缓存大小也会变大，这也是为什么reduce用来做缓存的参数是一个百分比，而不是一个固定的值了。假设
 
  mapreduce.reduce.shuffle.input.buffer.percent
  
   为 0.7，reduce task的max heapsize为1G，那么用来做下载数据缓存的内存就为大概700MB左右，这700M的内存，跟map端一样，也不是要等到全部写满才会往磁盘刷的，而是当这700M中被使用到了一定的限度（通常是一个百分比），就会开始往磁盘刷。这个限度阈值也是可以通过job参数来设定的，设定参数为：
  
  mapreduce.reduce.shuffle.merge.percent
 
 （default 0.66）。如果下载速度很快，很容易就把内存缓存撑大，那么调整一下这个参数有可能会对reduce的性能有所帮助。
 <br />
 当reduce task真正进入reduce函数的计算阶段的时候，有一个参数也是可以调整reduce的计算行为。也就是：
 
  mapreduce.reduce.input.buffer.percent
 
 （default 0.0）。由于reduce计算时肯定也是需要消耗内存的，而在读取reduce需要的数据时，同样是需要内存作为buffer，这个参数是控制，需要多少的内存百分比来作为reduce读已经sort好的数据的buffer百分比。默认情况下为0，也就是说，默认情况下，reduce是全部从磁盘开始读处理数据。 如果这个参数大于0，那么就会有一定量的数据被缓存在内存并输送给reduce，当reduce计算逻辑消耗内存很小时，可以分一部分内存用来缓存数据， 反正reduce的内存闲着也是闲着。


<a name="747742eb"></a>
###  map和reduce的内存资源调整总结


 以下是几个能够调节map和reduce内存大小的参数：



 
```sql
set mapreduce.map.memory.mb=10240; 
set mapreduce.reduce.memory.mb=10240; 
set mapred.map.child.java.opts=-server -Xmx9000m -Djava.net.preferIPv4Stack=true; 
set mapred.reduce.child.java.opts=-server -Xmx9000m -Djava.net.preferIPv4Stack=true;
```

 


 主要分为两类，一类是map/reduce总体的内存，一类是JVM对应的内存，那么分别控制哪一块呢？熟悉mapreduce2工作机制的前提下，都知道，mapTask和reduceTask都是运行在JVM里面的程序，所以调整JVM的内存，可以提供map和reduce程序使用，比如map和reduce的缓冲区，以及代码运行时需要的内存等等。map和reduce的总体内存是指各个节点想resourcemanager申请的container的内存，包含JVM所需内存的，所以可以发现10240比9000要大，额外部分需要为java code等非JVM的内存使用预留些空间，一般java.opts的大小为整个container内存的0.75倍。


<a name="8583b04e"></a>
#  Job整体优化

<a name="206cc533"></a>
##  Job执行模式


 Hadoop的Map Reduce Job可以有3种模式执行，即本地模式，伪分布式，还有真正的分布式。本地模式和伪分布式都是在最初学习Hadoop的时候往往被说成是做单机开发的时候用到。但是实际上对于处理数据量非常小的Job，直接启动分布式Job会消耗大量资源，而真正执行计算的时间反而非常少。这个时候就应该使用本地模式执行mr Job，这样执行的时候不会启动分布式Job，执行速度就会快很多。比如一般来说启动分布式Job，无论多小的数据量，执行时间一般不会少于20s，而使用本地mr模式，10秒左右就能出结果。
 <br />
 设置执行模式的主要参数有三个，一个是
 
  Hive.exec.mode.local.auto
 
 ，把他设为true就能够自动开启local mr模式。但是这还不足以启动local mr，输入的文件数量和数据量大小必须要控制，这两个参数分别为
 
  Hive.exec.mode.local.auto.tasks.max
 
 和
 
  Hive.exec.mode.local.auto.inputbytes.max
 
 ，默认值分别为4和128MB，即默认情况下，Map处理的文件数不超过4个并且总大小小于128MB就启用local mr模式。


<a name="eb973bc5"></a>
##  JVM重用


 正常情况下，MapReduce启动的JVM在完成一个task之后就退出了，但是如果任务花费时间很短，又要多次启动JVM的情况下（比如对很大数据量进行计数操作），JVM的启动时间就会变成一个比较大的overhead。在这种情况下，可以使用jvm重用的参数：



 
```sql
 set Mapred.Job.reuse.jvm.num.tasks = 5;
```

 


 他的作用是让一个jvm运行多次任务之后再退出。这样一来也能节约不少JVM启动时间。


<a name="600c1dbd"></a>
##  严格模式


 所谓严格模式，就是强制不允许用户执行3种有风险的HiveSQL语句，一旦执行会直接失败。这3种语句是：


- 
  查询分区表时不限定分区列的语句；
 
- 
  两表join产生了笛卡尔积的语句；
 
- 
  用order by来排序但没有指定limit的语句。
  <br />
  要开启严格模式，需要将参数
  
   hive.mapred.mode
  
  设为strict。
 

<a name="5d44361a"></a>
##  Job间并行


 首先，在Hive生成的多个Job中，在有些情况下Job之间是可以并行的，典型的就是子查询。当需要执行多个子查询union all或者join操作的时候，Job间并行就可以使用了。比如下面的代码就是一个可以并行的场景示意：



 
```sql
select * 
from ( select count(*) from logs
where log_date = 20130801 
and item_id = 1 
union all 
select count(*) 
from logs 
where log_date = 20130802 
and item_id = 2 
union all 
select count(*) 
from logs
 where log_date = 20130803 
and item_id = 3 )t
```

 


 设置Job间并行的参数是
 
  Hive.exec.parallel
 
 ，将其设为true即可。默认的并行度为8，也就是最多允许sql中8个Job并行。如果想要更高的并行度，可以通过
 
  Hive.exec.parallel.thread.number
 
 参数进行设置，但要避免设置过大而占用过多资源。


<a name="578d592c"></a>
##  减少Job数


 另外在实际开发过程中也发现，一些实现思路会导致生成多余的Job而显得不够高效。比如这个需求：查询某网站日志中访问过页面a和页面b的用户数量。低效的思路是面向明细的，先取出看过页面a的用户，再取出看过页面b的用户，然后取交集，代码如下：



 
```sql
select count(*) 
from 
(
	select distinct user_id 
	from logs where page_name = 'a'
) a join 
(
	select distinct user_id from logs where blog_owner = 'b'
) b on a.user_id = b.user_id;
```

 


 这样一来，就要产生2个求子查询的Job，一个用于关联的Job，还有一个计数的Job，一共有4个Job。但是我们直接用面向统计的方法去计算的话（也就是用group by替代join），则会更加符合M/R的模式，而且生成了一个完全不带子查询的sql，只需要用一个Job就能跑完：



 
```sql
select count(*) 
from logs 
group by user_id having 
(
count(case when page_name = ‘a’ then 1 end) > 0 
and count(case when page_name = ‘b’ then 1 end) > 0
)
```

 

# sql整体优化
<a name="29da88a8"></a>
##  列裁剪和分区裁剪


 最基本的操作。所谓列裁剪就是在查询时只读取需要的列，分区裁剪就是只读取需要的分区。以我们的日历记录表为例：



 
```sql
select uid
	   ,event_type
	   ,record_data 
from calendar_record_log 
where pt_date >= 20190201 
and pt_date <= 20190224 
and status = 0;
```

 
当列很多或者数据量很大时，如果select _或者不指定分区，全列扫描和全表扫描效率都很低。Hive中与列裁剪优化相关的配置项是hive.optimize.cp，与分区裁剪优化相关的则是hive.optimize.pruner_*，默认都是true。在HiveSQL解析阶段对应的则是ColumnPruner逻辑优化器。
<a name="b05af00f"></a>
##  谓词下推


 在关系型数据库如MySQL中，也有谓词下推（Predicate Pushdown，PPD）的概念。它就是将SQL语句中的where谓词逻辑都尽可能提前执行，减少下游处理的数据量。 例如以下HiveSQL语句：



 
```sql
select a.uid
	   ,a.event_type
	   ,b.topic_id
	   ,b.title 
from calendar_record_log a left outer join 
(
	select uid
		   ,topic_id
		   ,title from 
	forum_topic 
	where pt_date = 20190224 
	and length(content) >= 100 
) b on a.uid = b.uid where a.pt_date = 20190224 and status = 0;
```

 
对calendar_record_log 做过滤的where语句写在子查询内部，而不是外部。Hive中有谓词下推优化的配置项hive.optimize.ppd，默认值true，与它对应的逻辑优化器是PredicatePushDown。该优化器就是将OperatorTree中的FilterOperator向上提，见下图。
  
   ![](https://oss.dataown.cn/docs/2020/9/fb90b9512d1f5ad5.png)
  
 




<a name="ba2991ba"></a>
##  sort by代替order by


 HiveSQL中的order by与其他SQL方言中的功能一样，就是将结果按某字段全局排序，这会导致所有map端数据都进入一个reducer中，在数据量大时可能会长时间计算不完，所以在严格模式下要求limit。如果使用sort by，那么还是会视情况启动多个reducer进行排序，并且保证每个reducer内局部有序。为了控制map端数据分配到reducer的key，往往还要配合distribute by一同使用。如果不加distribute by的话，map端数据就会随机分配到reducer。
 <br />
 举个例子，假如要以UID为key，以上传时间倒序、记录类型倒序输出记录数据：



 
```sql
select uid
	  ,upload_time
	  ,event_type
	  ,record_data 
from calendar_record_log 
where pt_date >= 20190201 
and pt_date <= 20190224 
distribute by uid sort by upload_time desc,event_type desc;
```

 
## group by代替distinct

 当要统计某一列的去重数时，如果数据量很大，count(distinct)就会非常慢，原因与order by类似，count(distinct)逻辑只会有很少的reducer来处理。这时可以用group by来改写：



 
```sql
select count(1) 
from 
(
	select uid 
	from calendar_record_log 
	where pt_date >= 20190101 group by uid 
) t;
```

 


 但是这样写会启动两个MR job（单纯distinct只会启动一个），所以要确保数据量大到启动job的overhead远小于计算耗时，才考虑这种方法。当数据集很小或者key的倾斜比较明显时，group by还可能会比distinct慢。


<a name="1025c539"></a>
#  join和group by基础优化

<a name="3ccdd885"></a>
##  group by配置调整

<a name="e084feae"></a>
###  map端预聚合


 group by时，如果先起一个combiner在map端做部分预聚合，可以有效减少shuffle数据量。预聚合的配置项是hive.map.aggr，默认值true，对应的优化器为GroupByOptimizer，简单方便。
 <br />
 通过
 
  hive.groupby.mapaggr.checkinterval
 
 参数也可以设置map端预聚合的行数阈值，超过该值就会分拆job，默认值100000。


<a name="61199399"></a>
###  倾斜均衡配置项


 group by时如果某些key对应的数据量过大，就会发生数据倾斜。Hive自带了一个均衡数据倾斜的配置项
 
  hive.groupby.skewindata
 
 ，默认值false。其实现方法是在group by时启动两个MR job。第一个job会将map端数据随机输入reducer，每个reducer做部分聚合，相同的key就会分布在不同的reducer中。第二个job再将前面预处理过的数据按key聚合并输出结果，这样就起到了均衡的效果。但是，配置项毕竟是死的，单纯靠它有时不能根本上解决问题，详细的数据倾斜解决见下文介绍。


<a name="0e5f7eb6"></a>
##  join基础优化

<a name="a69cf558"></a>
###  多表join时key相同


 这种情况会将多个join合并为一个MR job来处理，例如：



 
```sql
select a.event_type
	  ,a.event_code
	  ,a.event_desc
	  ,b.upload_time 
from calendar_event_code a inner join 
(
	select event_type
		   ,upload_time 
    from calendar_record_log
    where pt_date = 20190225 
) b on a.event_type = b.event_type inner join 
( 
	select event_type
	  	   ,upload_time 
	from calendar_record_log_2 
	where pt_date = 20190225 
) c on a.event_type = c.event_type;
```

 
如果上面两个join的条件不相同，比如改成a.event_code = c.event_code，就会拆成两个MR job计算。

 负责这个的是相关性优化器CorrelationOptimizer，它的功能除此之外还非常多。


<a name="6c73725c"></a>
###  利用map join特性


 map join特别适合大小表join的情况。Hive会将calendar_event_code 和calendar_record_log在map端直接完成join过程，消灭了reduce，效率很高。



 
```sql
select a.event_type
	  ,b.upload_time 
from calendar_event_code a inner join 
( 
	select event_type
		  ,upload_time 
	from calendar_record_log 
	where pt_date = 20190225 
) b on a.event_type < b.event_type;
```

 


 map join的配置项是
 
  hive.auto.convert.join
 
 ，默认值true，对应逻辑优化器是MapJoinProcessor。还有一些参数用来控制map join的行为，比如
 
  hive.mapjoin.smalltable.filesize
 
 ，当build table大小小于该值就会启用map join，默认值25000000（25MB）。还有
 
  hive.mapjoin.cache.numrows
 
 ，表示缓存build table的多少行数据到内存，默认值25000。


<a name="61d34a92"></a>
###  分桶表map join


 map join对分桶表还有特别的优化。由于分桶表是基于一列进行hash存储的，因此非常适合抽样（按桶或按块抽样）。它对应的配置项是
 
  hive.optimize.bucketmapjoin
 
 ，优化器是BucketMapJoinOptimizer。但我们的业务中用分桶表较少，所以就不班门弄斧了，只是提一句。


<a name="059842e8"></a>
###  倾斜均衡配置项


 这个配置与上面group by的倾斜均衡配置项异曲同工，通过
 
  hive.optimize.skewjoin
 
 来配置，默认false。如果开启了，在join过程中Hive会将计数超过阈值
 
  hive.skewjoin.key
 
 （默认100000）的倾斜key对应的行临时写进文件中，然后再启动另一个job做map join生成结果。通过
 
  hive.skewjoin.mapjoin.map.tasks
 
 参数还可以控制第二个job的mapper数量，默认10000。


<a name="e82d70c2"></a>
#  数据倾斜解决方案


 其实之前在第四节介绍了一部分hive里面自带的一些解决数据倾斜的方法（group by和join的倾斜均衡配置项），但是那是治标不治本的，真正根治数据倾斜还是需要在sql层面或者说在数据层面杜绝。


<a name="3cbb4557"></a>
## 空值或无意义值


 这种情况很常见，比如当事实表是日志类数据时，往往会有一些项没有记录到，我们视情况会将它置为null，或者空字符串、-1等。如果缺失的项很多，在做join时这些空值就会非常集中，拖累进度。因此，若不需要空值数据，就提前写where语句过滤掉。需要保留的话，将空值key用随机方式打散，例如将用户ID为null的记录随机改为负值：



 
```sql
select a.uid
	  ,a.event_type
	  ,b.nickname
	  ,b.age from 
( 
	select (case when uid is null then cast(rand()-10240 as int) else uid end) as uid
		   ,event_type 
	from calendar_record_log 
	where pt_date >= 20190201 
) a left outer join 
( 
	select uid
	   	   ,nickname
	       ,age 
	from user_info 
	where status = 4 
) b on a.uid = b.uid;
```

 
## 单独处理倾斜key

 这其实是上面处理空值方法的拓展，不过倾斜的key变成了有意义的。一般来讲倾斜的key都很少，我们可以将它们抽样出来，对应的行单独存入临时表中。
 * 着重介绍下这里的做法，如果是group by的话，只要将group by对应的key后面加上一个随机数，比如0~9，这样key值就变得离散了，自然也就不会存在数据倾斜的问题，不过记得group by完以后，要将随机数去除。
 * 还有一种是join的方式，join会设计到两张表，可以选取一张较小的表，每行记录都加上0～9这10个数字，这样这个表就被放大十倍了，另外一个表，只要随机加上0～9就可以了，在进行join，就不会存在数据倾斜的问题了，join完以后同样需要将key后面的随机数进行去除。这种方式会使得一张表被放大十倍，所以该方式要先测试一下运行时间，不见得一定就能比正常跑来得快。

