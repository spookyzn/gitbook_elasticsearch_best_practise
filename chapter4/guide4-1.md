# 查询调优

## 给予文件系统缓存足够内存

Elasticsearch 为了更快的查询，很多依赖于文件系统缓存。通常需要确认文件系统缓存至少需要系统一半的内存，这样elasticsearch才能把热数据存放在物理内存中。

## 使用较快的硬件

如果查询是I/O bound，你必须调查并分配更多的文件系统缓存，或这购买较快的磁盘。SSD硬盘的性能好于机械磁盘，这个大家都懂的。尽可能使用本地存储，远程设备比如，NFS、SMB尽量要避免使用。同时也要注意虚拟存储，比如亚马逊的`elastic block storage`。虚拟存储和es兼容性不错，速度很快同时设置方便，但是和本地磁盘相比性能还有慢一点。如果你把索引放在EBS上，确认设置IOPS为合适值，不然你操作可能会被限制住。

如果查询是CPU bound，你必须调查并购买更多更快的CPU。

## 文档建模合理

文档必须被建模（合理），这样查询时间才可能快。

通常，`joins` 必须避免。`nested` 会导致查询慢，同时`parent-child` 关系会导致查询慢于正常上百倍。如果一个同样的问题可以不使用joins非规范的文档来解答，那令人难以置信的查询速度是可以预期的。

## 搜索使用尽可能少的字段

`query_string` 和 `multi_match`中包含越是多的字段，查询速度越是慢。一个通常来改善查询速度的技巧，就是把多个字段的值在索引时，复制值到一个字段中，然后在做查询时使用该字段。这些操作在索引的mapping中加入关键字`copy-to`来实现。下面是一个例子，调优一个带有电影信息的索引的查询， 把需要经常搜索使用的电影name & plot字段，使用`copy-to`关键字把值都存放到 name_and_plot 字段。

```
PUT movies
{
  "mappings": {
    "_doc": {
      "properties": {
        "name_and_plot": {
          "type": "text"
        },
        "name": {
          "type": "text",
          "copy_to": "name_and_plot"
        },
        "plot": {
          "type": "text",
          "copy_to": "name_and_plot"
        }
      }
    }
  }
}
```

## 预索引数据

你应该利用查询中的模式来优化数据的索引方式。例如，如果您的所有文档都有一个price字段，并且大多数查询在一个固定的范围列表上运行范围聚合，您可以通过将范围预先索引到索引中并使用`term`聚合来加快聚合速度。

文档大概格式
```
PUT index/_doc/1
{
  "designation": "spoon",
  "price": 13
}
```

查询示例
```
GET index/_search
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 10 },
          { "from": 10, "to": 100 },
          { "from": 100 }
        ]
      }
    }
  }
}
```

然后，可以在索引时使用price_range字段对文档进行加工，并将字段映射为`keyword`:
```
PUT index
{
  "mappings": {
    "_doc": {
      "properties": {
        "price_range": {
          "type": "keyword"
        }
      }
    }
  }
}

PUT index/_doc/1
{
  "designation": "spoon",
  "price": 13,
  "price_range": "10-100"
}
```

然后搜索请求可以聚合这个新字段，而不是在price字段上运行范围聚合。
```
GET index/_search
{
  "aggs": {
    "price_ranges": {
      "terms": {
        "field": "price_range"
      }
    }
  }
}
```

## 使用正确的字段类型

有些数据是数值型的事实并不意味着它应该总是映射为数值字段。通常，存储标识符(如ISBN或标识来自另一个数据库的记录的任何数字)的字段，可能会受益于将其映射为`keyword`，而不是`integer`或`long`。

## 避免使用 scripts

一般来说，应该避免脚本。如果它们是绝对需要的，您应该更喜欢`painless`和`expressions`。

## 使用取整的日期来查询

在日期字段上使用`now`的查询通常不能缓存，因为被匹配的范围一直在变化。但是，从用户体验的角度来看，切换到一个完整的日期通常是可以接受的，并且可以更好地利用查询缓存。

示例查询
```
PUT index/_doc/1
{
  "my_date": "2016-05-11T16:30:55.328Z"
}

GET index/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "my_date": {
            "gte": "now-1h",
            "lte": "now"
          }
        }
      }
    }
  }
}
```

原始查询可以被替换成
```
GET index/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "my_date": {
            "gte": "now-1h/m",
            "lte": "now/m"
          }
        }
      }
    }
  }
}
```

如果当前时间是16:31:29，那么范围查询将匹配my_date字段的值在15:31:00和16:31:59之间的所有值。如果几个用户同时运行一个包含此范围的查询，查询缓存可以帮助加快速度。用于舍入的时间间隔越长，查询缓存就越有帮助，但要注意，过于激进的舍入也可能损害用户体验。

为了能够利用查询缓存，可能很容易将范围分割为一个可缓存部分和较小的不可缓存部分，如下所示
```
GET index/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "bool": {
          "should": [
            {
              "range": {
                "my_date": {
                  "gte": "now-1h",
                  "lte": "now-1h/m"
                }
              }
            },
            {
              "range": {
                "my_date": {
                  "gt": "now-1h/m",
                  "lt": "now/m"
                }
              }
            },
            {
              "range": {
                "my_date": {
                  "gte": "now/m",
                  "lte": "now"
                }
              }
            }
          ]
        }
      }
    }
  }
}
```
然而，这种做法可能会使查询在某些情况下运行得更慢，因为`bool`查询引入的开销可能会抵消更好地利用查询缓存所节省的开销

## force-merge 只读的索引

只读索引将从合并到单个`segment`中获益([merged down to a single segment](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-forcemerge.html))。基于时间的索引通常是这样的:只有当前时间框架的索引获得新文档，而旧索引是只读的。

> 不要强制合并那些仍在被写入的索引，让它们合并到后台合并进程中。

## 预热global ordinals

`global ordinals` 是一个数据结构，用于在`keyword`字段上运行`terms`聚合。它们被延迟地加载到内存中，因为ES不知道哪些字段将用于术语聚合，哪些字段不会。您可以通过配置映射(如下所述)在刷新时告诉Elasticsearch急切地加载`global ordinals`:
```
PUT index
{
  "mappings": {
    "_doc": {
      "properties": {
        "foo": {
          "type": "keyword",
          "eager_global_ordinals": true
        }
      }
    }
  }
}
```

## 预热文件系统缓存

如果重新启动运行Elasticsearch的机器，文件系统缓存将为空，因此操作系统将索引的热点区域加载到内存中需要一段时间，以便搜索操作快速。根据使用`index.store.preload`设置中的文件扩展名，可以显式地告诉操作系统应该将哪些文件加载到内存中。

> 如果文件系统缓存不够大，无法保存所有数据，那么在太多索引或太多文件上迅速地将数据加载到文件系统缓存中会使搜索速度变慢。谨慎使用。

## 标识字段设置为`keyword`

当文档中有数字标识符时，很容易将它们映射为数字，这与它们的json类型一致。然而，Elasticsearch对数值型的“range”查询进行优化的方式，同时`keyword`字段在`term`查询中优化。由于标识符从来没有在`range`查询中使用，所以它们应该被映射为一个`keyword`。

## 使用索引排序来加速conjunctions

[Index sorting](https://www.elastic.co/guide/en/elasticsearch/reference/master/index-modules-index-sorting.html) 是有用的，以便使连接更快地以略微慢的索引为代价。在[index sorting documentation](https://www.elastic.co/guide/en/elasticsearch/reference/master/index-modules-index-sorting-conjunctions.html).阅读更多信息。

## 使用perference 来优化缓存使用

有多个缓存可以帮助搜索性能，例如文件系统缓存、请求缓存或查询缓存。然而，所有这些缓存都是在节点级别上维护的，这意味着，如果您在一行中运行相同的请求两次，拥有一个或多个副本，并使用默认的路由算法round robin，那么这两个请求将进入不同的`shard`副本，从而节点级缓存不起作用。

由于搜索应用程序的用户常常一个接一个地运行类似的请求，例如为了分析索引的一个更小的子集，使用标识当前用户或会话的首选项值可以帮助优化缓存的使用。

## 多复制片可能有益于吞吐量，但不是所有场景下

现在假设你有一个2分片索引和两个节点。在一种情况下，副本的数量为0，这意味着每个节点拥有一个`shard`。在第二种情况下，副本的数量为1，这意味着每个节点有两个`shard`。哪种设置在搜索性能上表现最好? 通常，每个节点的`shard`总数较少的设置会执行得更好。原因在于，它为每个`shard`提供了更大的可用文件系统缓存份额，而文件系统缓存可能是Elasticsearch的头号性能因素。与此同时，要注意，没有副本的设置在出现单个节点故障时可能会出现故障，因此在吞吐量和可用性之间需要进行权衡。

那么正确的副本数量是多少? 如果您有一个`num_nodes`节点的集群，总数为`num_primary` primary shards，如果您希望能够同时处理`max_failure` 节点失败，那么您需要的正确副本数量是 `max(max_failure, ceil(num_nodes / num_primar_node) - 1)`。

## 参考
https://www.elastic.co/guide/en/elasticsearch/reference/master/tune-for-search-speed.html