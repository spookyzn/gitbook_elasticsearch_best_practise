# 客户端调用 Elasticsearch 常见问题及解决方法 

## SLB reload 导致connection reset  

### 问题描述
程序中不时有connection reset错误发生。

### 问题根源
SLB的reload会在有创建访问入口、服务器集群调整等变更时触发的重新加载SLB配置信息的操作。reload操作对大部分请求都是无害的，但由于在reload当时客户端和服务端对同一个TCP连接的状态认知不一致（即服务端已经关闭但客户端尚未知晓），可能会导致Connection reset。

由于客户端感知服务端连接关闭需要一定时间，故这种问题无法完全消灭，只能通过重试的方式来减轻对业务的影响。ES的RestClient底层是使用Apache HttpAsyncClient发送的请求。但HttpAsyncClient 4目前并未内置自动重试出错请求的功能，所以只能有业务调用方自行编写代码重试。具体的业务逻辑可以参考下方问题解决方案中Apache HttpClient的RetryHandler的代码（需要注意HttpAsyncClient和HttpClient遇到SLB reload时抛出的异常是不同的）。

### 相关参考
SLB Connection Reset和NoHttpResponseException问题排查报告：http://conf.ctripcorp.com/pages/viewpage.action?pageId=143858900
HttpAsyncClient的"Connection reset by peer"问题：http://conf.ctripcorp.com/pages/viewpage.action?pageId=148245394
ConnectionReset和NoHttpResponseException问题解决方案：http://conf.ctripcorp.com/pages/viewpage.action?pageId=143876499