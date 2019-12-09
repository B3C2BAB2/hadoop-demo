# HDFS
HDFS是一款为在商用硬件上运行而设计的分布式文件系统。

## 特点
 - 硬件故障：HDFS实例往往包含上千个或以上的服务器，每个组件都有可能发生故障进而失去作用。因此故障监控和快速、自动恢复是HDFS的核心架构目标。
 - 数据流访问：HDFS在设计上重视数据获取的高吞吐而不是低延迟。因此HDFS上运行的应用需要使用流的方式访问它们的数据集。
 - 大数据集：HDFS中的文件往往有GB、TB级的大小。因此，HDFS应提供高的数据带宽并在单个聚簇中包含大量节点，同时在单个实例中支持千万级的文件数量。
 - 简单一致性模型：HDFS的文件需要一个“write-once-read-many”的访问模型。文件一旦被创建、编写、关闭，就不应做扩展或删节外的变更操作。
 - 变更计算的位置的成本低于变更数据的位置（Moving Computation is Cheaper than Moving Data）：当一个应用作请求时，数据在被存储的位置被计算要优于传输数据后由请求方计算。HDFS为应用提供了让数据在存储位置进行计算的接口。
 - 在不同硬件和软件平台上的可移植性：HDFS易于在不同平台间移植。

### NameNode和DataNode
![hdfs-architecture](image/hdfsarchitecture.png)

HDFS拥有master/slave架构。一个HDFS据测包含一个NameNode（一个管理文件系统的命名空间并控制客户端对文件的访问权限的master）和若干个DataNode（管理它们运行的节
点上的存储的slave，通常只有一个）。HDFS暴露文件系统的命名空间并允许将用户数据存储到文件中，文件或被分为若干个块并存储到一组DataNode中。NameNode执行文件系统的打开、关闭、重命名
文件或文件夹等操作。NameNode也决定了文件块和DataNode之间的映射关系。DataNode负责服务文件服务器客户端的读写请求。DataNode也根据NameNode的指令对文件块进行创建、删除、复制等操作
。

## 文件系统命名空间
HDFS支持传统的层级文件组织。操作者可以在目录中创建目录、存储文件。HDFS的命名空间层级结构和大多数现有的文件系统类似，操作者可以创建、移除、移动、重命名文件。

HDFS支持用户磁盘配额和访问权限控制。HDFS不支持硬连接和软连接，但HDFS架构不排除实现这些功能的可能。

NameNode维持文件系统的命名空间。任何文件系统命名空间和其本身参数的改动都会纪录到NameNode。HDFS上的应用可以从NameNode上获取应被HDFS维持的文件的副本数量（replication factor）。
    
## 数据副本
HDFS被设计用于在同一聚簇的多个服务器上存储大文件。每个文件被存储为一系列的块。为提高容错性文件块会被复制多份。每个文件的块大小和副本数量是分别配制的。除了最后一个块之外所有的块大小一致，对扩展和同步添加可变长度块的支持后用户可以在不填满最后一块的情况下增加新块。

应用可以指定文件副本数量。副本数量可以在文件创建时设置并在之后更改。HDFS的文件只进行一次写操作（不包括扩展和删节）并且在任何时候都只有一个writer。

NameNode决定了所有块副本的相关内容。NameNode周期性的从聚簇中的DataNode接收心跳和块报告（BlockReport）。接受到心跳表明DataNode正在正常运行。块报告包含了所有DataNode上块的列表。

![hdfs-data-nodes](image/hdfsdatanodes.png)

### 副本位置
乐观副本放置（Optimizing replica placement）将HDFS区别于其他分布式文件系统。机架感知副本放置策略（rack-aware replica placement policy）旨在体改数据可靠性、可用性以及带宽使用率。
聚簇上的大型HDFS实例普遍散布于多个机架。不同几家上节点的通信需要通过交换机实现。大多数情况下，同一机架上服务器间的带宽要高于不同机架上的服务器。
副本数量不小于3时，HDFS的放置策略如下：
 - 第一份：若writer在DataNode上，则放置在本地节点。否则放置在和writer同一机架上的任意节点。
 - 第二份：放置在和第一份不同的远端机架上的任意节点。
 - 第三份：防止在和第二份同一机架的不同节点。
 - 超过三份：随机分配的同时保证每个机架的副本数量<(副本数量-1)/(机架数量+2)。

由于NameNode不允许DataNode拥有同一块的多个副本，副本的数量上限不超过DataNode的数量。 

在增加存储类型和存储策略（Storage Types and Storage Policies）的支持后，NameNode将其作为机架感知的补充。NameNode会先根据计价感知进行判断，之后检查可选节点是否拥有文件的存储策略需要的存储。如果可选节点没有需要的存储类型，NameNode会寻找其他节点。如果在一级路经找不到足够的符合条件的节点，NameNode会在二级路经中寻找有备用存储类型的节点。

### 副本选择
为了最小化全局带宽的消费以及读延迟，HDFS处理读请求时会尝试读取距离reader最近的副本。如果在reader所在的机架上存在需要的副本，那么该副本会被优先选择。如果HDFS聚簇有复数数据中心，那么本地数据中心的副本的优先级高于远端数据中心。

### 安全模式
在启动时，NameNode会进入安全模式。NameNode在安全模式下时数据块的副本不会出现。当被NameNode检查的副本数量达到数量下限时数据块可以被认为被安全备份。在NameNode中配制安全备份的数据块比例后，经过30秒，NameNode退出安全模式。之后NameNode会决定副本数量未达到数量限制的数据块。之后NameNode将这些数据块备份到其他的DataNode。

## 元数据持久化
 - 命名空间：由NameNode进行存储。
 - EditLog：命名空间使用EditLog（一种事物日志）持久化记录文件系统元数据的所有变更。EditLog存储于NameNode本地操作系统的文件系统。
 - FsImage：整个文件系统的命名空间（包含数据块和文件的映射以及系统参数）存储于FsImage文件中。FsImage存储于NameNode本地操作系统的文件系统。

NameNode在内存中保存了整个文件系统命名空间的映像以及文件块映射。当NameNode启动或检查点触发配置门槛时，会从磁盘读取FsImage和EditLog，并将所以体育EditLog上的事务应用到内存中表示FsImage的部分，之后将新版本的FsImage重新写入磁盘。由于旧的事务已经在FsImage上实现持久化，之后NameNode可以删节旧的EditLog。这个过程被称为检查点（check point）。检查点旨确保在HDFS能够通过获取系统元数据的快照并应用到FsImage上从而持续获取系统元数据的内容。读取FsImage的效率很高，增量编辑FsImage则不然，故HDFS在EditLog中将编辑持久化。检查点会在预定的时间间隔（dfs.namenode.checkpoint.period）后被周期性触发，也可以在预定的若干个事务（dfs.namenode.checkpoint.txns）后被触发。两个参数都有设置时，第一个参数会成为触发条件。

DataNode将HDFS数据存储在本地文件系统的文件中。DataNode并不能获取HDFS文件的相关信息。DataNode将HDFS的每个数据块存储在不同的本地文件系统的文件中。由于本地文件系统可能不能有效支持单个文件夹中存放巨量文件，DataNode不会在统一文件夹中存放所有文件，而是决定最优的每个文件夹的文件数进而创建子文件夹。当启动时，DataNode会扫描本地文件系统，生成本地文件对应的一系列HDFS数据块，并向NameNode发送块报告。

## 通讯协议
所有HDFS通讯协议都是TCP/IP层面的协议。客户端对NameNode服务器上配置的TCP端口使用ClientProtocol协议建立连接。DataNode对NameNode的通讯使用DataNode协议。ClientProtocal协议和DataNode协议都属于Remote Protocal Call（RPC）。NameNode不发起RPC，只处理来自DataNode和客户端的RPC
请求。

## 鲁棒性
HDFS的主要目标是在即使有故障的情况下也能可靠的存储数据。常见故障有：
 - NameNode故障
 - DataNode故障
 - 网络分区故障

### 数据磁盘故障、心跳和再复制
每个DataNode都周期性的向NameNode发送心跳。网络分区故障可以导致DataNode的一个子集失去NameNode的连接。NameNode可以通过心跳消息的缺失侦测到这种情况，将没有最近心跳的DataNode标记为死亡并不在向其发起IO请求。对于HDFS，任何注册在死亡的DataNode上的数据都不在有效。DataNode的死亡可能导致一些数据块的备份数量低于预定的值。NameNode不断最总需要被备份的数据块并在需要的时候对该数据块进行初始化。再复制的重要性可能因为以下原因而提升：
 - DataNode不可用。
 - 备份损坏。
 - DataNode所在服务器的硬盘故障。
 - 一个文件的备份数量设置被提升。

为了避免因为DataNode状态变化导致备份骤增，判断DataNode死亡的超时时间被设置的尽量长（默认10秒）。用户可以设置一个稍短的时间间隔用于将DataNode状态标记为过期（stale），并避免过期DataNode因性能敏感的相关配置而进行的读写操作。

### 聚簇再平衡
HDFS架构兼容数据再平衡方案。数据再平衡方案可以在一个DataNode的剩余存储空间低于某个特定阀值时将数据移动到另一个DataNode。

### 数据完整性
在出现存储设备故障、网络错误或软件异常时，从DataNode上获取的数据可能损坏。HDFS客户端软件实现了HDFS文件内容的校验码检测。在客户端船舰HDFS文件时，会计算出文件的每个数据块的校验码并分别存储在同一命名空间的不同匿名文件。当客户端查询这些内容时会比较接受的数据块的检验码和存储校验码的文件中的校验码。如果校验失败，客户端可以从其它DataNode获取该数据块。

### 元数据磁盘故障
FsImage和EditLog是HDFS的核心数据结构，这些文件的损坏会导致整个HDFS实例失去作用。因此，NameNode可以通过配置维护多个FsImage和EditLog的拷贝。FsImage和EditLog的任何变更都会使所有FsImage和EditLog同步这些变更。这种同步操作可能导致NameNode可支持的命名空间处理事务的速度有所下降。当NameNode重起时，会使用最新的FsImage和EditLog。

此外可以使用shared storage on NFS或分布式的EditLog (Journal)。

### 快照
快照支持在特定时间点存储数据的拷贝。可以用于回滚损坏的HDFS实例。

## 数据组织
### 数据块
典型的数据块大小为128MB，每个数据块分布在不同DataNode。

### 副本管道
HDFS根据副本目标选择算法列出存放数据块的DataNode，列出的DataNode依次接受数据写入本地，然后将数据传输给下一个DataNode。

## 访问HDFS的方式
### FS Shell
FS Shell适用于需要脚本语言进行数据交互的应用。常用指令：
 - 创建目录 /foodir		        ```bin/hadoop dfs -mkdir /foodir```
 - 删除目录 /foodir		        ```bin/hadoop fs -rm -R /foodir```
 - 查看文件 /foodir/myfile.txt	```bin/hadoop dfs -cat /foodir/myfile.txt```

### DFSAdmin
DSFAdmin是用于管理HDFS聚簇的命令集。常用指令：
 - 让聚簇进入安全模式		    ```bin/hdfs dfsadmin -safemode enter```
 - 声称一系列DataNode		    ```bin/hdfs dfsadmin -report```
 - 启用/停用DataNode		        ```bin/hdfs dfsadmin -refreshNodes```

### 浏览器
可以通过配置使用户在浏览器上阅览HDFS命名空间的文件内容。

## 空间回收
### 文件删除和回滚
若垃圾回收的相关设置启动，通过FS Shell移除的文件不会立即从HDFS移除，而是移动到```/user/<username>/.Trash```。最近被删除的文件会被移动到```/user/<username>/.Trash/Current```，在经过预先配置的时间间隔后，HDFS会将文件移动到```/user/<username>/.Trash/<date>```并删除过期文件。删除文件会使与该文件关联的数据块的空间被释放。 

以下为通过FS Shell删除delete文件夹下的文件test1和test2的示例：
 - 创建测试用文件。
```
$ hadoop fs -mkdir -p delete/test1
$ hadoop fs -mkdir -p delete/test2
$ hadoop fs -ls delete/
Found 2 items
drwxr-xr-x   - hadoop hadoop          0 2015-05-08 12:39 delete/test1
drwxr-xr-x   - hadoop hadoop          0 2015-05-08 12:40 delete/test2
```

 - 移除文件test1。
```
$ hadoop fs -rm -r delete/test1
Moved: hdfs://localhost:8020/user/hadoop/delete/test1 to trash at: hdfs://localhost:8020/user/hadoop/.Trash/Current
```

 - 使用skipTrash参数移除test2。
```
$ hadoop fs -rm -r -skipTrash delete/test2
Deleted delete/test2
```

 - Trash文件夹下只有test1，test2被永久删除。
```
$ hadoop fs -ls .Trash/Current/user/hadoop/delete/
Found 1 items\
drwxr-xr-x   - hadoop hadoop          0 2015-05-08 12:39 .Trash/Current/user/hadoop/delete/test1
```

### 减少副本数量
当配置的文件副本数量减少后，NameNode选择多余的副本进行删除，并在下一次心跳时将信息传输给DataNode。之后DataNode移除相应的数据块并释放空间。

## 参考文档
 * [https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)