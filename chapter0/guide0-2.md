# 客户端调用 Elasticsearch 常见问题及解决方法 

## 调用elasticsearch restful 接口出现401错误

### 问题描述

服务端nginx日志中有大量401错误，并同时也有成功的200请求
```
[05/Jul/2017:16:03:11 +0800] 10.15.139.120 "POST /_bulk" timeout=1m  8080 group.bnb 10.15.227.5 10.15.189.164 HTTP/1.1 "Apache-HttpClient/4.5.2 (Java/1.8.0_60)" - - group-bnb-es-oy.ops.ctripcorp.com 200
 985 0.032 0.032 127.0.0.1:9200 200
[05/Jul/2017:16:03:11 +0800] 10.15.139.120 "POST /_bulk" timeout=1m  8080 group.bnb 10.15.227.5 10.15.189.164 HTTP/1.1 "Apache-HttpClient/4.5.2 (Java/1.8.0_60)" - - group-bnb-es-oy.ops.ctripcorp.com 200 1122 0.186 0.186 127.0.0.1:9200 200
[05/Jul/2017:16:03:11 +0800] 10.15.139.120 "POST /_bulk" timeout=1m  8080 - 10.15.227.6 10.15.189.164 HTTP/1.1 "Apache-HttpClient/4.5.2 (Java/1.8.0_60)" - - group-bnb-es-oy.ops.ctripcorp.com 401 194 0.001 - - -
[05/Jul/2017:16:03:12 +0800] 10.15.139.120 "POST /_bulk" timeout=1m  8080 - 10.15.227.6 10.15.189.164 HTTP/1.1 "Apache-HttpClient/4.5.2 (Java/1.8.0_60)" - - group-bnb-es-oy.ops.ctripcorp.com 401 194 0.001 - - -
```

java错误
```
java.net.SocketException: Connection reset
at java.net.SocketOutputStream.socketWrite(SocketOutputStream.java:113)
at java.net.SocketOutputStream.write(SocketOutputStream.java:153)
at org.apache.http.impl.io.SessionOutputBufferImpl.streamWrite(SessionOutputBufferImpl.java:126)
at org.apache.http.impl.io.SessionOutputBufferImpl.write(SessionOutputBufferImpl.java:162)
at org.apache.http.impl.io.ContentLengthOutputStream.write(ContentLengthOutputStream.java:115)
at org.apache.http.impl.io.ContentLengthOutputStream.write(ContentLengthOutputStream.java:122)
at org.apache.http.entity.StringEntity.writeTo(StringEntity.java:169)
at org.apache.http.impl.DefaultBHttpClientConnection.sendRequestEntity(DefaultBHttpClientConnection.java:158)
at org.apache.http.impl.conn.CPoolProxy.sendRequestEntity(CPoolProxy.java:162)
at org.apache.http.protocol.HttpRequestExecutor.doSendRequest(HttpRequestExecutor.java:237)
at org.apache.http.protocol.HttpRequestExecutor.execute(HttpRequestExecutor.java:122)
at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:271)
at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:184)
at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:88)
at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:110)
at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:184)
at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:82)
at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:107)
at io.searchbox.client.http.JestHttpClient.executeRequest(JestHttpClient.java:132)
at io.searchbox.client.http.JestHttpClient.execute(JestHttpClient.java:67)
at io.searchbox.client.http.JestHttpClient.execute(JestHttpClient.java:60)
at com.ctrip.hotel.bnb.elasticSearch.client.ESClientImpl.processBulkRequests(ESClientImpl.java:222)
at com.ctrip.hotel.bnb.service.controller.job.ESOptionController.SyncESOptionController.pushtoES(SyncESOptionController.java:431)
at com.ctrip.hotel.bnb.service.controller.job.ESOptionController.SyncESOptionController.access$600(SyncESOptionController.java:33)
at com.ctrip.hotel.bnb.service.controller.job.ESOptionController.SyncESOptionController$6.exec(SyncESOptionController.java:305)
at com.ctrip.hotel.bnb.common.RetyExecCommon.run(RetyExecCommon.java:14)
at com.ctrip.hotel.bnb.service.controller.job.ESOptionController.SyncESOptionController.MultipushtoES(SyncESOptionController.java:315)
at com.ctrip.hotel.bnb.service.controller.job.ESOptionController.SyncESOptionController.access$300(SyncESOptionController.java:33)
at com.ctrip.hotel.bnb.service.controller.job.ESOptionController.SyncESOptionController$3.run(SyncESOptionController.java:161)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
at java.lang.Thread.run(Thread.java:745)
```

### 问题根源
http preemptiveAuth 

### 解决方案
高版本的支持setPreemptiveAuth方法
低版本的使用http client的setCredentials，自己实现preemptiveAuth

### 相关链接
https://github.com/searchbox-io/Jest/issues/207

