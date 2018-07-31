# 如何设置慢查询日志

## 使用入口

只要是通过ops es组维护的集群，都在espaas平台中集群首页。登录集群首页。如图点开界面
![PNG](/images/image2018-7-25 10_56_40.png)

## 操作简介

![PNG](/images/image2018-4-11 10_45_1.png)

1. 选择需要配置慢查询阈值的索引
2. 选择相应的索引后，右侧的slow log settings列表会显示该索引的慢查询阈值配置。点击按钮进行修改或则删除。
3. 当需要修改相关配置时，先在右侧列表中选中需要修改配置，然后在左侧表单中修改，配置确认后，点击ADD按钮确认修改。
4. 当需要添加配置时，直接在表单中输入配置，点击ADD按钮确认添加。
5. 选择索引所有阈值配置完成后，点击Set Slow Log Settings按钮确认，把配置列表中的阈值配置写入集群。

## 慢日志查看

阈值配置完成后，一旦有超过阈值的慢查询和慢索引，都会记录到日志。日志可以通过下拉菜单中的View Slow Log来查看。如何调优查询请看这篇 [用PROFILE API 定位 ES 慢查询](chapter3/guide3-0.md)
![PNG](/images/image2018-4-11 16_47_38.png)

## 其他

Type 类型
- index  - 索引数据操作(数据写入)
- query - 查询数据操作
- fetch - fetch操作

## 参考
https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-slowlog.html