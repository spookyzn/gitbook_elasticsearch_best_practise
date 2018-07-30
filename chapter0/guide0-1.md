# 客户端使用最佳实践

使用 ES 有两种连接方式 : transport、rest，transport方式是通过tcp连接访问ES(只支持java)，rest方式是通过http api访问ES(没有语言限制)。
ES官方提供 Java client 连接ES集群，下面提供shade后的包解决和依赖冲突问题

## Transport

### 5.x&6.x 版本ES:

1 加入依赖
```
<dependency>
    <groupId>com.ctrip.es.client</groupId>
    <artifactId>transport</artifactId>
    <version>${elasticsearch.version}</version>
</dependency>
```

| groupId | artifactId | version |
|----------|:---------|:--------:|
|com.ctrip.es.client | transport | 5.0.1 | 
|com.ctrip.es.client | transport | 5.0.2 | 
|com.ctrip.es.client | transport | 5.1.1 | 
|com.ctrip.es.client | transport | 5.1.2 | 
|com.ctrip.es.client | transport | 5.2.2 | 
|com.ctrip.es.client | transport | 5.3.0 | 
|com.ctrip.es.client | transport | 5.3.2 | 
|com.ctrip.es.client | transport | 5.6.3.ctrip-2 | 
|com.ctrip.es.client | transport | 6.3.0 | 

2 创建transport client对象

```
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.InetSocketTransportAddress;
import org.elasticsearch.transport.client.PreBuiltTransportClient;
  
Settings settings = Settings.builder()
        .put("cluster.name", "myClusterName").build();
TransportClient client = new PreBuiltTransportClient(settings)
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("host1"), 9300))
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("host2"), 9400));
```

## Rest
使用 rest http 方式，官方提供了基础了rest client 和 high level rest client，其中rest client 支持版本为5.0及以后版本，high level rest client支持版本为 5.6及以后版本，也可以使用其他http client库自己实现调用ES rest API

1 加入依赖  注意：transport包 中包含 rest, `已引入transport不能够再引入 rest`，由于是shade的包会发生类冲突

```
<dependency>
    <groupId>com.ctrip.es.client</groupId>
    <artifactId>rest</artifactId>
    <version>${elasticsearch.version}</version>
</dependency>
```

| groupId | artifactId | version |
|----------|:---------|:--------:|
|com.ctrip.es.client | rest | 5.0.1 | 
|com.ctrip.es.client | rest | 5.0.2 | 
|com.ctrip.es.client | rest | 5.1.1 | 
|com.ctrip.es.client | rest | 5.1.2 | 
|com.ctrip.es.client | rest | 5.2.2 | 
|com.ctrip.es.client | rest | 5.3.0 | 
|com.ctrip.es.client | rest | 5.3.2 | 
|org.elasticsearch.client |	elasticsearch-rest-high-level-client | 5.6.3.ctrip-SNAPSHOT |
|org.elasticsearch.client |	elasticsearch-rest-high-level-client | 6.0.1.ctrip |
|com.ctrip.ops.es.client |	elasticsearch-rest-high-level-client | 6.3.0 |

2 创建rest client对象，和使用

```
import com.ctrip.es.apache.http.HttpEntity;
import com.ctrip.es.apache.http.HttpHost;
import com.ctrip.es.apache.http.entity.ContentType;
import com.ctrip.es.apache.http.nio.entity.NStringEntity;
import com.ctrip.es.apache.http.util.EntityUtils;
import org.elasticsearch.client.Response;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClient.FailureListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
 
import java.io.IOException;
import java.util.Collections;
  
RestClient restClient = RestClient.builder(
        new HttpHost("host", 9200, "http")).build();
Response response = restClient.performRequest("GET", "/",
        Collections.singletonMap("pretty", "true"));
System.out.println(EntityUtils.toString(response.getEntity()));
 
//index a document
HttpEntity entity = new NStringEntity(
        "{\n" +
        "    \"message\" : \"trying out Elasticsearch\"\n" +
        "}", ContentType.APPLICATION_JSON);
 
response = restClient.performRequest(
        "PUT",
        "/twitter/tweet/1",
        Collections.<String, String>emptyMap(),
        entity);
 
System.out.println("status = " + response.getStatusLine().getStatusCode());
System.out.println(EntityUtils.toString(response.getEntity()));
 
 
FailureListener failureListener = new FailureListener();
System.out.println("shaded = " + failureListener.getClass().getProtectionDomain().getCodeSource());
```

3 创建high level client对象

```
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.index.query.RangeQueryBuilder;
import org.elasticsearch.rest.RestStatus;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
 
import com.ctrip.es.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
  
logger.info("main start");
RestHighLevelClient client = new RestHighLevelClient(
        RestClient.builder(
                new HttpHost("host", 9200, "http")).build());
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
boolQuery.should(QueryBuilders.termQuery("name", "hatsune"));
boolQuery.must(QueryBuilders.termQuery("name", "jun"));
RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("@timestamp");
rangeQuery.from(1512553489779L);
rangeQuery.to(1513158289779L);
boolQuery.must(rangeQuery);
sourceBuilder.query(boolQuery);
 
String s = sourceBuilder.toString();
System.out.println("sourceBuilder = " + s);
SearchRequest searchRequest = new SearchRequest();
searchRequest.source(sourceBuilder);
SearchResponse searchResponse = client.search(searchRequest);
RestStatus status = searchResponse.status();
System.out.println("status = " + status.getStatus());
```

## 注意

> ES 集群可以通过 transport 和 http rest 两种方式来访问，ES官方建议使用rest方式，如需使用官方的Java client可以用上述的包，如果上面所列的官方client包的shade包没有所需的版本，联系ES Team (ops_es@Ctrip.com)，会根据需求添加