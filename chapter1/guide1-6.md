# 使用自动生成的标识id
当索引具有显式id的文档时，Elasticsearch需要检查具有相同id的文档是否已经存在于相同的shard中，这是一个代价高昂的操作，并且随着索引的增长，代价会更高。通过使用自动生成的id，ES可以跳过这个检查，这使得索引更快。

如果你使用自己的ID，尝试选择一个对于lucene友好的ID。比如`zero-padded sequential ids`, `UUID-1` 还有`nanotime`；这些id具有一致的、连续的模式，可以很好地压缩。与此相反，`UUID4`这种完全随机并且压缩率极低的ID，会拖慢Lucene。


## 参考
https://www.sohamkamani.com/blog/2016/10/05/uuid1-vs-uuid4/

