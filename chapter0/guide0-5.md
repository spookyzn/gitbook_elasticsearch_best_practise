# 客户端调用 Elasticsearch 常见问题及解决方法 

## 通过http restful接口访问出现http error code 502 

### 问题描述
客户端访问时返回502，从SLB日志上看该请求是从后端的nginx（集群节点上的nginx）返回的502错误码

1. 节点HTTP网络连接数突然下降
2. 节点nginx 错误日志中有502错误
3. ES服务端出现错误
```
[2017-11-23T12:58:42,327][WARN ][o.e.h.n.Netty4HttpServerTransport] [10.15.80.117] caught exception while handling client http traffic, closing connection [id: 0xd0fe8fda, L:/127.0.0.1:9200 - R:/127.0.0.1:54418]
java.lang.IllegalArgumentException: only one Content-Type header should be provided
        at org.elasticsearch.rest.RestRequest.parseContentType(RestRequest.java:481) ~[elasticsearch-5.3.2.jar:5.3.2]
```

### 问题根源
http请求头部有问题，导致es服务端拒绝服务，返回错误。nginx返回502错误。


### 解决方案
更正http请求包头部，问题解决