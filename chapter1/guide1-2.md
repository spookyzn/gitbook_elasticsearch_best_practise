# 最大化elasticsearch索引性能

## 关于参数refrsh interval 

### 关闭refreshd interval  
refresh interval 可以通过参数`index.refresh_interval`来设置，该参数在集群配置中使用，也可以在每个`index`的设置中使用。如果两者都使用，则`index`上的配置会覆盖集群设置。默认值是1，所以新索引的文档最多在1秒后可以被搜索到。

因为`refresh`很昂贵，提高索引吞吐量的一种方法是增加`refresh_interval`。较少的`refresh`意味着较少的负载，更多的资源可以用于`indexing`线程。因此，根据搜索需求，可以考虑将刷新间隔设置为高于1秒的值。甚至可以暂时完全关闭索引刷新(通过将间隔设置为-1)，例如，在批量导入数据时，可以考虑把`refresh`关闭

关闭refresh，设置`refresh-intreval`为-1
```
curl -XPUT 'localhost:9200/test/_settings' -d '{
    "index" : {
        "refresh_interval" : "-1"
    }
}'
```

### 关闭复制片

如果正在进行大量导入，可以禁用副本。当文档被复制时，整个文档被发送到副本节点，索引过程被重复。这意味着每个副本将执行分析、索引和潜在的合并过程。相反，如果您索引的副本为0，然后在导入数据完成时启用副本，则恢复过程实质上是字节对字节的网络传输。这比复制索引过程要有效得多。

```
curl -XPUT 'localhost:9200/my_index/_settings' -d ' {
    "index" : {
        "number_of_replicas" : 0
    }
}'
```

### 恢复索引参数

打开复制片

```
curl -XPUT 'localhost:9200/my_index/_settings' -d ' {
    "index" : {
        "number_of_replicas" : 0
    }
}'
```

打开refresh interval

```
curl -XPUT 'localhost:9200/my_index/_settings' -d '{
    "index" : {
        "refresh_interval" : "1s"
    } 
}'
```

如果该索引导入数据后不再写入新数据，可以对该只读索引执行`force merge`，这样可以提高查询速度。

```
curl -XPOST 'localhost:9200/my_index/_forcemerge?max_num_segments=5'
```


导入完成后，可以手动refresh
```
curl -XPOST 'localhost:9200/my_index/_refresh'
```