# 最大化elasticsearch索引性能

## Flushing of Tansaction log

translog帮助防止节点失败时的数据丢失。它的设计目的是帮助`shard`恢复操作，否者数据可能会从内存flush到磁盘时发生意外而丢失。日志每5秒被提交到磁盘上，或者在每个成功的索引、删除、更新或批量请求时提交。

为了防止数据丢失，每个`shard`都有一个事务日志或与之关联的写入日志。任何索引或删除操作在内部Lucene索引处理后被写入到translog中。在崩溃的情况下，当`shard`恢复时，可以从事务日志中重新重放最近的事务。

ES的flush是执行Lucene提交并产生新的translog的过程。它是在后台自动完成的，以确保事务日志不会太大，这将使在恢复期间重放其操作花费大量时间。它也通过API来操作。

与刷新索引`shard`相比，真正昂贵的操作是刷新其事务日志(涉及Lucene提交)。通过延迟刷新或完全禁用它们，可以提高索引吞吐量。但是这种做法有利有弊，延迟的flush当然会花费更长的时间。

以下参数是用于控制flush的频率
- `index.translog.flush_threshold_size` - translog按size大小flush，默认为521M
- `index.translog.flush_threshold_ops` - 多少任务来执行操作，默认为`ulimited`
- `index.translog.flush_threshold_period` - 在触发刷新之前，不管事务日志大小，需要等待多长时间。默认为30m
- `index.translog.sync_interval` - 多久检测是否需要flush。默认为5s
- `index.translog.durability` - sync方式。默认为fsync，可选为async。2者区别，fsync每次操作都会commit到translog。async按设置的sync_interval时间定期commit到translog。 fsync更耗费资源，但可靠性好，async模式下，如果sync_interval间隔时间内发生问题，中间没有commit的操作会全部丢失

### 建议

我们可以增加index.translog.flush_threshold_size从默认的512M到更大的值，比如1gb。这允许在发生刷新之前在translog中积累更大的`segment`。通过让更大的`segment`构建，可以减少刷新的频率，而更大的`segement`合并的频率也更低。所有这些都减少了磁盘I/O开销，提高了索引吞吐量。当然，将需要相应的heap空闲内存来提供额外缓冲空间，调整时此设置时请记住这一点。

### 最佳实践

如没有其他特殊需求。可以使用一下参数来配置

```
index.translog.flush_threshold_size: "1gb"
index.translog.sync_interval: "60s"
index.translog.durability: "async"
```