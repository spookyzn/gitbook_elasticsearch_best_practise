# 禁止深度分页

ES本身是支持分页查询的，使用方式跟MySQL非常类似：

- from: Indicates the number of initial results that should be skipped, defaults to 0
- size: Indicates the number of results that should be returned, defaults to 10

```
GET /_search?size=5&from=10
```

但是跟MySQL不同，ES是分布式存储的，查询结果一般都是跨多个分片的(spans multiple shards)，每个shard产生自己的排序结果，最后协调节点(coordinating node)对所有分片返回结果进行汇总排序截断返回给客户端。所以深度分页（分页深度）或者一次请求很多结果（分页大小）都会极大降低ES的查询性能。所以ES就默认限制最多只能访问前1w个文档。这是通过`index.max_result_window`控制的。

深度分页有一个巨大的性能问题。假设某个index有 5 个primary shard。我们要获取第1000页的内容，页面大小是10，那么需要排序的文档数是 10001~10010 (pageSize * pageNumber)。那么每个分片需要本地排序产生前10010条记录，然后协调节点/聚合层(Coordinating node/Aggregator)需要对5个主分片返回的 10010 * 5 = 50050 （docs * Shards）条记录进行排序，最后只返回10条数据给客户端。其他的50040条全部都扔掉了。

也就是说为了返回第 pageNumber 页的数据，一共需要对 `pageSize * pageNumber * shardNumber` 个文档进行排序。最后只返回 pageSize 条数据，性价比非常差。

## 参考
https://www.elastic.co/guide/en/elasticsearch/reference/6.3/search-request-from-size.html