# 用PROFILE API 定位 ES 慢查询

## 简介
有时在发起一个查询时，他会被延迟，或者响应时间很慢，查询缓慢可能会有多种原因；范围包括 shard 问题，或者计算查询中的某些元素。 从 elasticsearch 2.2版本开始提供 Profile API 供用户检查查询执行时间和其他详细信息。在这篇博客中，我们将探讨如何使用profile API查看查询计时。

## Profile API
Profile API 是Elasticsearch 2.2版本的一个实验性特性， Profile API 返回所有分片的详细信息。为了简单起见，我们创建一个只有1个分片的索引。

*Index creation:*
```
curl -XPUT http:localhost:9200/myindex -d '{
"settings":{
    "number_of_replicas": 0,
    "number_of_shards": 1
}
}' 
```

在上面创建的索引中，我们为演示基本的搜索写入若干文档，然后从结果中解释 profile API。让我们索引三个doc:
```
curl -XPOST http://localhost:9200/myindex/mytype/1 -d '{
"brand" : "Cotton Plus"
}'
curl -XPOST http://localhost:9200/myindex/mytype/2 -d '{
"brand" : "Van Huesen"
}'
curl -XPOST http://localhost:9200/myindex/mytype/3 -d '{
"brand" : "Arrow"
}'
```

可以通过在 query 部分上方提供 “profile: true” 来启用Profile API。让我们来看看带profile参数的 query 在单个查询中长什么样子：
```
curl XPOST http://localhost:9200/myindex/mytype/_search -d '{
 "profile": true,
     "query": {
       "match": {
         "brand": "Cotton Plus"
       }
     }
}'
```

Profile API的结果是基于每个分片计算的。由于在我们的例子中只有一个分片，我们在profile API响应的分片数组中只有一个数组元素，如下所示：
```
{
 "shards": [
   {
     "id": "[egPYczsCRTqaeJ8jKhFjtw][myindex][0]",
     "searches": [
       {
         "query": [
           {
             "query_type": "BooleanQuery",
             "lucene": "brand:levi brand:goals",
             "time": "1.293761000ms",
             "breakdown": {
               "score": 0,
               "create_weight": 136078,
               "next_doc": 0,
               "match": 0,
               "build_scorer": 1082113,
               "advance": 0
             },
             "children": [
               {
                 "query_type": "TermQuery",
                 "lucene": "brand:levi",
                 "time": "0.04626300000ms",
                 "breakdown": {
                   "score": 0,
                   "create_weight": 30190,
                   "next_doc": 0,
                   "match": 0,
                   "build_scorer": 16073,
                   "advance": 0
                 }
               },
               {
                 "query_type": "TermQuery",
                 "lucene": "brand:goals",
                 "time": "0.02930700000ms",
                 "breakdown": {
                   "score": 0,
                   "create_weight": 16600,
                   "next_doc": 0,
                   "match": 0,
                   "build_scorer": 12707,
                   "advance": 0
                 }
               }
             ]
           }
         ],
         "rewrite_time": 64032,
         "collector": [
           {
             "name": "TotalHitCountCollector",
             "reason": "search_count",
             "time": "0.002931000000ms"
           }
         ]
       }
     ]
   }
 }
```

## Profile API响应说明
上面的响应显示的是单个分片。每个分片都被分配一个唯一的ID，ID的格式是[nodeID][indexName][shardID]。现在在"shards"数组里还有另外三个元素，它们是：
* query
* rewrrite_tim
* collector

### Query
Query 段由构成Query的元素以及它们的时间信息组成。Profile API结果中Query 部分的基本组成是：
1. query type – 它向我们显示了哪种类型的查询被触发。此处是布尔值。因为多个关键字匹配查询被分成两个布尔查询。
2. lucene – 该字段显示启动查询的lucene方法。这里是 "brand:levi brand:goals"
3. time – lucene 执行此查询所用的时间。单位是毫秒。
4. breakdown – 有关查询的更详细的细节，主要与lucene参数有关。
5. children - 具有多个关键字的查询被拆分成相应术语的布尔查询，每个查询都作为单独的查询来执行。每个子查询的详细信息将填充到Profile API输出的子段中。在上面的章节中，可以看到第一个子元素查询是"levi"，下面给出查询时间和其他breakdown参数等详细信息。同样，对于第二个关键字，有一个名为"goals"的子元素具有与其兄弟相同的信息。从查询中的子段中，我们可以得到关于哪个搜索项在总体搜索中造成最大延迟的信息。

### Rewrite Time
由于多个关键字会分解以创建个别查询，所以在这个过程中肯定会花费一些时间。将查询重写一个或多个组合查询的时间被称为“重写时间”。(以纳秒为单位)。

### Collectors
在Lucene中，收集器是负责收集原始结果，收集和组合结果，执行结果排序等的过程。例如，在上面的执行的查询中，当查询语句中给出size:0时，使用的收集器是"totalHitCountCollector"。这只返回搜索结果的数量（search_count），不返回文档。此外，收集者所用的时间也一起给出了。

## 总结
在这篇博客中，我们看到Profile API非常有用，它让我们清楚地看到查询时间。通过向我们提供有关子查询的详细信息，清楚地知道在那个环节查询慢，这是非常有用的。另外，API返回结果中，关于Lucene的详细信息也让我们深入了解到 elasticsearch 是如何执行查询的。

## 参考
http://cwiki.apachecn.org/display/Elasticsearch/Profile+API