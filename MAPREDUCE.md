# MapReduce

## 概述
Hadoop MapReduce是一款以可靠、容错的方式为并行处理在大型聚簇（千级）上的TB级的数据集的应用而生的软件架构。

一个MapReduce任务通常需要将输入的数据集分割为多个互相独立且由map任务并行处理的部分。框架会将输出的数据排序并作为reduce任务的输入数据。输入和输出的数据都存储在文件系统。框架负责调度、监控任务以及重新执行失败的任务。

MapReduce框架和Hadoop分布是文件系统在同一组节点上执行，即计算和存储在同一个节点。这样使得框架可以更有效的调度已存在数据的节点上的任务，从而在整体上提高聚簇的带宽。

MapReduce框架由一个ResourceManager，每节点一个的NodeManeger，每应用一个的MRAppMaster组成。

应用通过接口和抽象类列出输入/输出数据的位置并支持map和reduce功能，这和其他工作字段一并构成了工作配置（job configuration）。之后Hadoop工作客户端向ResourceManager提交工作和配置信息，ResourceManager负责为工作客户端分发软件/配置到各个worker，调度和监控任务，提供状态和诊断信息。

## 输入和输出
MapReduce只操作```<key, value>```键值对，即MapReduce将输入数据看作一系列的键值对并输出一系列的键值对。键和值都需要被框架序列化，因此需要实现Writable接口。此外，键需要实现```WritableComparable```接口以便于框架对其进行排序。

一个MapReduce工作的输入/输出类型如下：

(input) <k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3> (output)

## 接口
### Mapper
Mapper将输入的键值对映射为中间态的键值对。被转换的中间状态记录不必和输入记录保持同一种数据类型。输入的键值对可以被映射为0到多个输出记录。

Hadoop MapReduce框架会为每个InputFormat生成的InputSplit生成一个map任务。

Mapper的实现会通过Job.setMapperClass(Class)传递给工作。之后框架会为该任务对每个InputSplit的键值对调用map(WritableComparable, Writable, Context)。应用之后可以重载cleanup(Context)以进行任何需要的清理。输出的键值对会被context.write(WritableComparable, Writable)收集。

所有和中间值相关的输出键随后会被框架进行分类，并传递给Reducer决定最终输出。用户可以由Job.setGroupingComparatorClass(Class)设定Comparator对分类进行控制。

Mapper的输出会被分类，随后会被分发到各个Reducer。分区总数和工作的reduce任务数相等。用户可以通过实现Partitioner接口控制特定的键或记录被分配到特定的Reducer。

用户可以通过Job.setCombinerClass(Class)指定combiner进而对中间输出进行本地聚合，从而减少Mapper传输到Reducer的数据量。

排序的中间输出往往以(key-len, key, value-len, value)的格式存储。应用可以通过Configuration决定是否/如何压缩中间输出以及使用CompressionCodec。

### Mapper的数量
映射的次数往往由输入数据（输入文件的数据块数量）决定。映射的并行等级应为每节点10-100次映射，对cpu计算要求低的映射可以被设置到300次。最佳情况下，任务需要至少一分钟的执行时间。

### Reducer
Reducer减少既有相同键的中间数据以得到更小的数据集。

减少（reduce）工作的数量由Job.setNumReduceTasks(int)决定。Reduce的实现类将Job通过Job.setReducerClass(Class)传递。之后框架为每个分组的输入中的<key, (list of values)>调用reduce(WritableComparable, Iterable<Writable>, Context)。应用之后可以重载cleanup(Context)以进行任何需要的清理。

Reducer有3个主要阶段：shuffle，sort，reduce。具体描述如下：
 - Shuffle：Reducer的输入是排序的Mapper输出。在这个阶段框架通过HTTP找到Mapper输出中的所有相关分区。
 - Sort：框架根据Reducer输入的键将输入分组。Shuffle和Sort同时开始，当获取到映射操作的输出后两者会被合并。
 - Secondary Sort：若输出的键需要和输入的键有不同的数据类型，可以通过Job.setSortComparatorClass(Class)指定Comparator。
 - Reduce：reduce(WritableComparable, Iterable<Writable>, Context)在这个阶段会为每个被分组的<key, (list of values)>输入调用。减少任务的输出由Context.write(WritableComparable, Writable)写入文件系统。应用可已使用Counter报告统计状态。Reducer的输出是未经排序的。
 
### 减少（reduce）的数量
reduce的数量应为节点数量*
每节点最大容器数量的0.95到1.75倍。
 - 当倍数为0.95时reduce能在map结束后马上执行并开始传输map的输出。
 - 当倍数为1.75时最快的节点在完成第一轮reduce后会执行第二轮有这更好负载均衡的reduce。

reduce数量的增加会增加框架的开销，但也增加了负载平衡并降低了故障成本。

检测到的reduce数会略小于上述数量，差额的部分用于做有风险的或失败的reduce任务的预备。

### 没有reducer的情况
将reduce任务的数量设置为0 是符合规范的。在这种情况下map任务的输出会被直接写入文件系统，输出地址由FileOutputFormat.setOutputPath(Job, Path)设置。框架在map输出写入文件系统前不会对其进行排序。

### Patitioner
Patitioner控制了map的中间输出的分区。键的子集通常通过哈希函数用于派生分区（默认Patitioner是HashPatitioner）。分区数和reduce任务的数量相等，因此Patitioner控制了任务的中间输出的键被发送到reduce任务的过程。

### Counter
Counter用于MapReduce应用的统计汇报。Mapper和Reducer的实现类可以使用Counter汇报统计数据。

## 参考文档
 * [https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html](https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html)