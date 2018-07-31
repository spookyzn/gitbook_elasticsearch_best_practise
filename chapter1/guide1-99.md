# 最大化elasticsearch索引性能

## Segments的合并

`segment` 合并在所使用的开销是非常昂贵的，并且会消耗大量的磁盘I/O。合并被安排在后台操作，因为它们可能需要很长时间来完成，尤其是大的`segment`合并。这通常是没有问题的，因为大`segment`的合并是比较罕见的。

但有时合并会落后于数据写入的速度。如果发生这种情况，那么Elasticsearch将自动将索引请求限制到单个线程。这可以防止`segment`爆掉的问题。

ES默认值是保守的:你不希望搜索性能受到后台`segement`合并的影响。但有时(特别是在SSD或日志场景中)，阈值限制太低。

默认值是20MB/s，这是机械磁盘的设置。如果您有ssd，可以考虑将其增加到100-200 MB/s

```
curl -XPUT 'localhost:9200/_cluster/settings' -d '{
    "persistent" : {
        "indices.store.throttle.max_bytes_per_sec" : "100mb"
    }
}'
```

如果你正在进行批量导入，并且根本不关心搜索，那么可以完全禁用merge throttling。这将使索引尽可能快的写入磁盘:

```
curl -XPUT 'localhost:9200/_cluster/settings' -d '{
    "transient" : {
        "indices.store.throttle.type" : "none" 
    }
}'
```

将阈值类型设置为none，将完全禁用合并阈值。导入完成后，将其设置为merge以重新启用合并机制。

```
curl -XPUT 'localhost:9200/_cluster/settings' -d '{
    "transient" : {
        "indices.store.throttle.type" : "merge" 
    }
}'
```

