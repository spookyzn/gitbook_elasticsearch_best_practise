# 字符型和数值型区分

## 使用正确的字段类型

有些数据是数值型的事实并不意味着它应该总是映射为数值字段。通常，存储标识符(如ISBN或标识来自另一个数据库的记录的任何数字)的字段，可能会受益于将其映射为keyword，而不是integer或long。反之亦然，如果在查询中对一个keyword类型字段，进行`range`操作，所消耗的时间也会很长。

## 原因

由于在5.x以后term query的重大变化，对于`keyword`的优化。导致对于一个数值型类型字段做`term`查询，查询速度可能会有几十倍的差异，具体原理请看参考链接。

## 参考
https://elasticsearch.cn/article/446