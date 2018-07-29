# 存储使用调优

## 关闭那些你不使用的功能

Elasticsearch默认对于大多数字段索引文档值，这样这些字段就可以被很方便地查询和聚合。比如你有一个数字型字段`foo`，你需要该字段进行histograms，但你不会对该字段进行过滤，你可以放心的关闭这个字段的indexing
```
PUT index
{
  "mappings": {
    "type": {
      "properties": {
        "foo": {
          "type": "integer",
          "index": false
        }
      }
    }
  }
}
```

`text` 字段在索引中存放着normalization factor，以便于对于文档打分。如果你只需要对于`text`进行匹配操作，根本不关心打分，你可以在索引的mapping中关闭norms参数
```
PUT index
{
  "mappings": {
    "type": {
      "properties": {
        "foo": {
          "type": "text",
          "norms": false
        }
      }
    }
  }
}
```

`text` 字段同时默认也存放频率和位置信息。频率信息是用来对于文档打分使用，位置信息是用来分词。如果你不想使用分词查询功能，你可以设置elasticsearch，索引时不保存位置信息。
```
PUT index
{
  "mappings": {
    "type": {
      "properties": {
        "foo": {
          "type": "text",
          "index_options": "freqs"
        }
      }
    }
  }
}
```

另外你如果也不需要打分，你可以配置elasticsearch只是索引文档。你仍然可以针对这个字段进行搜索，但是尝试进行分词会发生错误，同时打分功能也只会假设所有的terms在每个文档中都出现一次。
```
PUT index
{
  "mappings": {
    "type": {
      "properties": {
        "foo": {
          "type": "text",
          "norms": false,
          "index_options": "freqs"
        }
      }
    }
  }
}
```

## 不要使用默认的字段动态mappings

默认的字段动态mapping功能会将字符类型字段索引成`text`和`keyword`两种类型。如果你想要的只是其中一种类型，这会浪费存储空间。常规来说，`id`字段只需要被索引成`keyword`这一种类型，同时`body`字段也只需要被索引成`text`这种类型。

默认动态mapping功能可以通过直接对于索引设置mapping，或者通过配置template来指定字段是使用`text`还是使用`keyword`来取消。

举例来说，以下是一个template，该template中设置字符串字段为`keyword`
```
PUT index
{
  "mappings": {
    "type": {
      "dynamic_templates": [
        {
          "strings": {
            "match_mapping_type": "string",
            "mapping": {
              "type": "keyword"
            }
          }
        }
    }
  }
}
```

## 注意你的shard size

更大的shard容量对于存储数据是很有效的。为了增加shard size，你可以减少索引主片的数量。创建索引时可以设置更少的主片数量，后者使用shrink API来修改现有索引的主片数量。

同时也需要知道这样做也有缺点，比如一旦发生问题，索引需要长时间的recovery。

## 关闭 _source

字段_source保存了文档的原始JSON。如果你不想访问，你可以关闭。然而如果需要update 和reindex这些需要访问_source的功能，这些也不会work了。

## 使用best_compression

_source字段和字段很容易可以占用了不可忽视的磁盘空间。通过设置索引参数`codec`为best_compression， 这些都可以被强制压缩。

## Force Merge

Elasticsearch中的索引都是以一到多个shard方式存放。每个shard都是一个lucene索引并且由一到多个segments组成。尽量大的segment对于存储数据非常有效。

API _forcemerge 可以用来减少每个shard的segment数量。在很多case中，可以通过设置`max_num_segments=1` 用来确保每个shard只有一个segment。

## 缩小索引

Shrink API 可以让你减少索引的shard数量。和之前提到的force merge API一起使用，这些手段可以非常有效的减少索引的shard数量和segment数量。

## 使用合理的数字类型

针对数值型数据来选择的字段类型对于磁盘空间有很大的影响。尤为重要，数值型需要使用其对应的类型（integer, byte, short, long)。一个float类型的值也可以使用`scaled_float`类型，如果针对实际使用案例是合适的，比如使用float来替换double，或者half_float替换float都可以省下磁盘空间。


