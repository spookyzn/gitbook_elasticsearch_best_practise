# 索引更新及索引提交 

## 增加refresh interval的值

默认的索引。refresh_interval值为1，这使搜索每秒钟创建一个新的`segment`。增加这个值(比如30s)将允许更大的`segment`刷新并减少未来的合并压力。

## 导入大量数据时，关闭refresh interval & 复制片

如果需要一次加载大量数据，应该通过设置索引来禁用refresh。refresh_interval设置为-1，并设置索引的number_of_replicas为0。这将暂时使您的索引处于危险之中，因为任何`shard`丢失都将导致数据丢失，但是与此同时索引速度将更快，因为文档将只被索引一次。一旦初始加载完成，就可以设置索引的refresh_interval，number_of_replicas回到原始值。

## translog 设置