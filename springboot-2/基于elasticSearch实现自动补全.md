# 基于elasticSearch实现自动补全

- 为什么要用es来实现？

  因为能共用一个搜索服务，并且稳定，能利用已有的分词器。

- 有多少种实现方法？本文用的是哪一种？

  [https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html)

  本文用的`completion` suggester 来实现的。

- 已经有那么多文章了，为何还要写？

  1. 很多文章没有及时更新
  2. 记录自己的理解

## 整体流程

1. 添加索引，并单独写一个字节点用来实现自动补全，并设置类型为completion

```  json
PUT search_index
 {
        "mappings": {
            "properties": {
                "fileName": {
                    "type": "text",
                    "analyzer": "ik_max_word",
                    "fields": {
                        "suggest": {
                            "type": "completion",
                            "analyzer": "ik_max_word"
                        }
                    }
                },
                "path": {
                    "type": "keyword",
                    "index": false
                }
            }
        }
    }

```

2. 插入数据

``` json
POST /search_index/_doc
{
    "fileName":"四川成都市龙泉驿生态环境局龙泉驿区3D气溶胶雷达监控采购项目的中标公告"
}
POST /search_index/_doc
{
    "fileName":"气溶胶雷达租赁服务报价清单"
}
POST /search_index/_doc
{
    "fileName":"四川省遂宁市安居生态环境局臭氧在线监测预警系统采购项目竞争性磋商终止（废标）公告"
}
```



3. 实现搜索补全

``` json
POST /search_index/_search
{
  "suggest": {
    "my_suggest_document": {
      "prefix": "四",
      "completion": {
        "field": "fileName.suggest"
      }
    }
  }
}
```



## JAVA基于RestHighLevelClient客户端的实现

1. 构建配置客户端连接

   ```java
   @Configuration
   @EnableElasticsearchRepositories
   public class ElasticsearchConfiguration extends AbstractElasticsearchConfiguration {
   
   
       @Override
       @Bean
       public RestHighLevelClient elasticsearchClient() {
   
           final ClientConfiguration clientConfiguration = ClientConfiguration.builder()
                   .connectedTo("192.168.15.207:9200")
   
                   //.withConnectTimeout(Duration.ofSeconds(5))
                   //.withSocketTimeout(Duration.ofSeconds(3))
                   //.useSsl()
                   //.withDefaultHeaders(defaultHeaders)
                   //.withBasicAuth(username, password)
                   // ... other options
                   .build();
           return RestClients.create(clientConfiguration).rest();
       }
   
   //    @Bean
   //    public ElasticsearchRestTemplate restTemplate() throws Exception {
   //        return new ElasticsearchRestTemplate(elasticsearchClient());
   //    }
   
   }
   
   
   ```

   

2. 配置Java bean

   ``` java
   @Data
   @AllArgsConstructor
   @NoArgsConstructor
   @Document(indexName = "search_index", type = "_doc", shards = 5, replicas = 1)
   public class DocumentPo implements Serializable {
       private static final long serialVersionUID = 3433260756083989671L;
       @Id
       public String id;
       @Field(type = FieldType.Keyword)  
      // @Field(type = FieldType.Text, analyzer = "ik_max_word")
       public String fileName;   
       @Field(index = false, type = FieldType.Keyword)
       public String compassPath;
   
   }
   
   ```

3. 添加数据

   ```java
   public interface DocumentRepository extends ElasticsearchRepository<DocumentPo,Long> {
   }
   
   //添加
   @Autowired
   DocumentRepository documentRepository;
   
   documentRepository.save(documentPo);
   ```

   

4. 实现搜索

   ```java
    @Autowired
       private RestHighLevelClient restHighLevelClient;
   
   @RequestMapping(value = "/by_contact",method = RequestMethod.GET)
       public Object getSearchSuggest(HttpServletRequest request, @RequestParam(value = "keyWord")String keyWord)  {
           //指定在哪个字段搜索
           String suggestField = "fileName.suggest";
           SearchRequest searchRequest = new SearchRequest("search_index");
           searchRequest.types("_doc");
           SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
           BoolQueryBuilder qb = QueryBuilders.boolQuery();
           SuggestionBuilder termSuggestionBuilder =SuggestBuilders.completionSuggestion(suggestField).prefix(keyWord).skipDuplicates(true).size(10);
           SuggestBuilder suggestBuilder = new SuggestBuilder();
           suggestBuilder.addSuggestion("my_suggest_document", termSuggestionBuilder );
           searchSourceBuilder.suggest(suggestBuilder);
           searchRequest.source(searchSourceBuilder);
           SearchResponse response = null;
           try {
               response = restHighLevelClient.search(searchRequest,RequestOptions.DEFAULT);
           } catch (Exception e) {
               System.out.println(e.getMessage());
           }
           Suggest suggest = response.getSuggest();
           List<String> keywords = null;
           if (suggest != null) {
               keywords = new ArrayList<>();
               List<? extends Suggest.Suggestion.Entry<? extends Suggest.Suggestion.Entry.Option>> entries =
                       suggest.getSuggestion("my_suggest_document").getEntries();
               for (Suggest.Suggestion.Entry<? extends Suggest.Suggestion.Entry.Option> entry: entries) {
                   for (Suggest.Suggestion.Entry.Option option: entry.getOptions()) {
                       String keyword = option.getText().string();
                       if (!StringUtils.isEmpty(keyword)) {
                           if (keywords.contains(keyword)) {
                               continue;
                           }
                           keywords.add(keyword);
                           if (keywords.size() >= 9) {
                               break;
                           }
                       }
                   }
               }
           }
           return keywords;
       }
   
   ```

   

## 多个条件的搜索

``` java
 public SearchResponse autosuggestSearch() throws IOException {
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        BoolQueryBuilder qb = QueryBuilders.boolQuery();

        PrefixQueryBuilder namePQBuilder = QueryBuilders.prefixQuery("address", "usa");
        PrefixQueryBuilder addressPQBuilder = QueryBuilders.prefixQuery("address", "usa");
        qb.should(namePQBuilder);
        qb.should(addressPQBuilder); //Similarly add more fields prefix queries.
        sourceBuilder.query(qb);

        SearchRequest searchRequest = new SearchRequest("employee").source(sourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
        System.out.println("Search JSON query \n" + searchRequest.source().toString()); //Generated ES search JSON.
        return searchResponse;
    }
```









### 参考网址

https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html

https://stackoverflow.com/questions/60397369/elasticsearch-7-6-3-java-highlevel-rest-client-auto-suggest-across-multiple-fi

https://stackoverflow.com/questions/54399859/completion-suggester-in-elasticsearch-6-5-4-with-java-rest-client-api?answertab=votes#tab-top

https://stackoverflow.com/questions/48657904/how-to-write-rest-high-level-client-query-for-prefix-suggestion/50707641

https://blog.csdn.net/jonkee/article/details/115421810

