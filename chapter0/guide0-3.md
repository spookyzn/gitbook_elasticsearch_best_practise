# 客户端调用 Elasticsearch 常见问题及解决方法 

## .NET 客户端偶尔出现“The underlying connection was closed: A connection that was expected to be kept alive was closed by the server.”错误

### 问题描述
从访问日志来看基本正常，没有任何错误；

应用内部会出现如下报错
```
获取详情数据失败 Invalid NEST response built from a unsuccessful low level call on POST: /orderdetailinfo/corpbasicorderdetailinfo/_search # Audit trail of this API call: - [1] BadResponse: Node: http://biz-bid-search.ops.ctripcorp.com/ Took: 00:00:14.2447268 # OriginalException: System.Net.WebException: 远程服务器返回错误: (502) 错误的网关。 在 System.Net.HttpWebRequest.GetResponse() 在 Elasticsearch.Net.HttpConnection.Request[TReturn](RequestData requestData) # Request: # Response:
```

### 解决方案
- Disable keep-alive. This is not recommended because you are going to exactly what HTTP 1.1 was designed to avoid.
- Decrease the idle timeout in the .NET Framework by changing the value of the  System.Net.ServicePointManager.MaxServicePointIdleTime property. The value should be lower than 15 seconds to force the client to close the connection before the query server does it.
- Extend the idle timeout in the query server. (Please contact FAST Technical Support if you need to implement this option.)

### 相关链接
https://support.microsoft.com/en-us/help/2017977/the-underlying-connection-was-closed-a-connection-that-was-expected-to