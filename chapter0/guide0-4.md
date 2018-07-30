# 客户端调用 Elasticsearch 常见问题及解决方法 

## 使用restful http协议访问的客户端出现connection closed 问题 

### 问题描述

java 客户端错误
```
org.apache.http.ConnectionClosedException: Connection closed
       at org.apache.http.nio.protocol.HttpAsyncRequestExecutor.endOfInput(HttpAsyncRequestExecutor.java:341)
       at org.apache.http.impl.nio.DefaultNHttpClientConnection.consumeInput(DefaultNHttpClientConnection.java:263)
       at org.apache.http.impl.nio.client.InternalIODispatch.onInputReady(InternalIODispatch.java:81)
       at org.apache.http.impl.nio.client.InternalIODispatch.onInputReady(InternalIODispatch.java:39)
       at org.apache.http.impl.nio.reactor.AbstractIODispatch.inputReady(AbstractIODispatch.java:116)
       at org.apache.http.impl.nio.reactor.BaseIOReactor.readable(BaseIOReactor.java:164)
       at org.apache.http.impl.nio.reactor.AbstractIOReactor.processEvent(AbstractIOReactor.java:339)
       at org.apache.http.impl.nio.reactor.AbstractIOReactor.processEvents(AbstractIOReactor.java:317)
       at org.apache.http.impl.nio.reactor.AbstractIOReactor.execute(AbstractIOReactor.java:278)
       at org.apache.http.impl.nio.reactor.BaseIOReactor.execute(BaseIOReactor.java:106)
       at org.apache.http.impl.nio.reactor.AbstractMultiworkerIOReactor$Worker.run(AbstractMultiworkerIOReactor.java:590)
       at java.lang.Thread.run(Thread.java:744)
```

### 问题根源
应用发起查询时使用了已经被关闭的connection

### 解决方案
1. 网络故障，或者丢包。 也就是上次大面积网络故障的时候，遇到的一次问题。
2. 为了提高http性能，SLB开启了keepalive来复用链接。  如果client端的keepalive设置和服务端协调有问题，可能导致某些链接idle一段时间后，SLB主动关闭了空闲链接，在到达client端之前，client端同时发出了http request。单因为slb已经关闭了链接，导致请求失败。  这种情况下需要保证client开启了keep alive，并且timeout时间要小于slb的设置，保证空闲链接的关闭由client主动发起。（只适用于接入SLB的应用)
3. 某些库的bug，比如之前公司内部有.net用户遇到同类问题，调查后发现是.net framework处理keep alive机制上有问题，需要打补丁。

### 相关链接
https://stackoverflow.com/questions/10570672/get-nohttpresponseexception-for-load-testing/10680629#10680629

### 连接重试相关示例代码
```
JestClientFactory factory = new JestClientFactory(){
                                     @Override
                                     protected HttpClientBuilder configureHttpClient(HttpClientBuilder builder) {
                                               builder = super.configureHttpClient(builder);
                                               builder.setRetryHandler(new DefaultHttpRequestRetryHandler(3,true));
                                    return builder;
                                     }
                            };
```
