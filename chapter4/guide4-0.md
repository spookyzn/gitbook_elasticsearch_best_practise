# 索引速度调优

## 使用bulk请求

批量请求将产生比单文档索引请求更好的性能。为了知道批量请求的最佳大小，您应该在一个节点上运行一个带有单个`shard`的基准。首先尝试一次索引100个文档，然后是200，然后是400，等等，在每次基准测试运行中，将批量请求中的文档数量增加一倍。当索引速度开始趋于平稳时，您就知道您达到了对数据的批量请求的最佳大小。要注意的是，当许多请求并发地发送时，过大的批量请求可能会使集群处于内存压力之下，因此最好避免在每次请求时超过几十兆字节，即使较大的请求看起来性能更好。

## 使用多个线程发送数据至Elasticsearch

发送大量请求的单个线程不太可能最大限度地输出一个弹性搜索集群的索引容量。为了使用集群的所有资源，您应该从多个线程或进程发送数据。除了更好地利用集群的资源之外，这应该有助于降低每个fsync的成本。

请务必注意`TOO_MANY_REQUESTS(429)响应码`(Java客户机的EsRejectedExecutionException)，这是Elasticsearch告诉您它无法跟上当前索引率的方式。当这种情况发生时，您应该在再次尝试之前暂停索引，最好是使用随机指数备份。

与批量请求相似，只有测试才能确定最佳线程数量。这可以通过逐步增加线程数量来测试，直到集群上的I/O或CPU饱和为止。

## 增加refresh interval的值

默认的索引。refresh_interval值为1，这使搜索每秒钟创建一个新的`segment`。增加这个值(比如30s)将允许更大的`segment`刷新并减少未来的合并压力。

## 导入大量数据时，关闭refresh interval & 复制片

如果需要一次加载大量数据，应该通过设置索引来禁用refresh。refresh_interval设置为-1，并设置索引的number_of_replicas为0。这将暂时使您的索引处于危险之中，因为任何`shard`丢失都将导致数据丢失，但是与此同时索引速度将更快，因为文档将只被索引一次。一旦初始加载完成，就可以设置索引的refresh_interval，number_of_replicas回到原始值。

## 关闭操作系统swapping

您应该确保操作系统禁用`swapping`[如何禁用swapping](https://www.elastic.co/guide/en/elasticsearch/reference/master/setup-configuration-memory.html)。

## 给足内存至文件系统缓存

文件系统缓存将用于缓冲I/O操作。您应该确保在文件系统缓存至少使用机器一半的内存。

## 使用自动生成的标识id

当索引具有显式id的文档时，Elasticsearch需要检查具有相同id的文档是否已经存在于相同的`shard`中，这是一个代价高昂的操作，并且随着索引的增长，代价会更高。通过使用自动生成的id，ES可以跳过这个检查，这使得索引更快。

## 使用更快的硬件

如果查询是I/O bound，你必须调查并分配更多的文件系统缓存，或这购买较快的磁盘。SSD硬盘的性能好于机械磁盘，这个大家都懂的。尽可能使用本地存储，远程设备比如，NFS、SMB尽量要避免使用。同时也要注意虚拟存储，比如亚马逊的`elastic block storage`。虚拟存储和es兼容性不错，速度很快同时设置方便，但是和本地磁盘相比性能还有慢一点。如果你把索引放在EBS上，确认设置IOPS为合适值，不然你操作可能会被限制住。

通过在多个ssd上配置RAID 0 索引。记住，它将增加失败的风险，因为任何一个SSD的失败都会破坏索引。然而，这通常是正确的权衡:优化单个`shard`以获得最大的性能，然后跨不同的节点添加副本，以便对任何节点故障都有冗余。您还可以使用[快照和恢复来备份索引](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-snapshots.html)，以便进行进一步冗余保障。

## 调整indexing buffer size

如果节点只执行繁重的索引，请确保[`indices.memory.index_buffer_size`](https://www.elastic.co/guide/en/elasticsearch/reference/master/indexing-buffer.html)足够大，可以在每个shard上提供最多512 MB的索引缓冲区进行繁重索引的任务(除了索引性能通常不会提高)。Elasticsearch采用该设置(java heap usage 的百分比或绝对字节大小)，并将其作为所有活动`shard`的共享缓冲区。非常活跃的`shard`自然会比执行轻量级索引的`shard`更多地使用这个缓冲区。

默认值是10%，通常情况下这已经算很多了。例如，如果您给JVM 10GB的内存，它将给索引缓冲区1GB，这足以hold住两个正在大量索引的`shard`。

## 去掉_field_names 字段

[_field_names](https://www.elastic.co/guide/en/elasticsearch/reference/master/mapping-field-names-field.html) 字段引入了一些索引时开销，因此如果你不使用`exists`查询，禁用它。

## 其他

参考本章中的存储使用调优，也对加速索引速度有用处。

## 参考
https://www.elastic.co/guide/en/elasticsearch/reference/master/tune-for-indexing-speed.html