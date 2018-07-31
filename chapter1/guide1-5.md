# 禁用动态mapping，请自定义mapping

## 字符型和数值型区分

### 使用正确的字段类型

有些数据是数值型的事实并不意味着它应该总是映射为数值字段。通常，存储标识符(如ISBN或标识来自另一个数据库的记录的任何数字)的字段，可能会受益于将其映射为keyword，而不是integer或long。反之亦然，如果在查询中对一个keyword类型字段，进行`range`操作，所消耗的时间也会很长。

### 原因

由于在5.x以后term query的重大变化，对于`keyword`的优化。导致对于一个数值型类型字段做`term`查询，查询速度可能会有几十倍的差异，具体原理请看参考链接。

### 参考
https://elasticsearch.cn/article/446


## 自定义mapping分词器
对于分析的字段，使用最简单的分析器来满足字段的要求。或者可以使用`not_analyzed`

默认情况下，字符串类型的字段被认为包含全文。也就是说，值将在被索引之前传入到`analyzer`中，同时对于该字段全文搜索也要在查询前经过`analyzer`。

重要的mapping参数对于字符型字段，一个是`index`，一个是`analyzer`

### index

`index`是控制一个字符串如何被索引。有3个可用值
- analyzed: 首先分析字符串，然后存放至索引。不然存放该字符串的全文
- not_analyze： 索引该字段，使其可以被搜索。存放字符串全文。不用分词器。
- no: 不用索引该字段。该字段不可能搜索

默认字段该值被设置为`analyzed`。如果我们不希望该字段被分词，我们需要把该值设置为`not_analyzed`:

```
{
    "tag": {
        "type":     "string",
        "index":    "not_analyzed"
    }
}
```

其他简单类型(如long、double、date等)也接受索引参数，但唯一相关的值是no和not_analyze，这些值无需被分词。

### analyzer

针对需要分词的字符型字段，使用`analyzer`参数来配置指定的分词器同时在搜索和索引阶段。默认，ES使用`standard`的默认分词器，但是你可以更改为指定的默认分词器。比如, `whitespace`, `simple`, `english`。

```
{
    "tweet": {
        "type":     "string",
        "analyzer": "english"
    }
}
```

## 为了节省空间，不要使用默认的字段动态mappings

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

## 使用自动生成的标识id
当索引具有显式id的文档时，Elasticsearch需要检查具有相同id的文档是否已经存在于相同的shard中，这是一个代价高昂的操作，并且随着索引的增长，代价会更高。通过使用自动生成的id，ES可以跳过这个检查，这使得索引更快。