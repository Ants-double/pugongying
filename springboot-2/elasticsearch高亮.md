# Elasticsearch java高亮显示

- 为什么要写
  1. 因为版本原因很多网上的案例变动较大
  2. 基于springboot 2.2.2 elasticsearch 7.10.1

## 原理

- elastic支持，请求格式如下：

  ```json
  {
      "query": {
          "bool": {
              "should": [
                  {
                      "match": {
                          "addName": "test"
                      }
                  },
                  {
                      "match": {
                          "fileName": "环境"
                      }
                  }
              ],
              "minimum_should_match": 2
          }
      },
      "highlight": {
          "fields": { 
              "fileName":{} 
           } 
       }
  }
  
  ```

  

## 基于API的实现

1. 网上找到的常用版本

   ``` java
   public Object queryKeyWord(HttpServletRequest request,
                               @RequestParam(value = "keyWord")String keyWord,
                               @RequestParam(value = "page")int page,
                               @RequestParam(value = "size")int size) throws IOException {
           // 在索引中进行查询
           SearchRequest searchRequest = new SearchRequest(esDocumentIndex);
           SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
   
           // 搜索名字
           TermQueryBuilder termQuery = QueryBuilders.termQuery("fileName", keyWord);
           sourceBuilder.query(termQuery);
           sourceBuilder.from(page);
           sourceBuilder.size(size);
          // sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
   
           // 添加高亮
           HighlightBuilder highlightBuilder = new HighlightBuilder();
           HighlightBuilder.Field highlightName = new HighlightBuilder.Field("fileName"); //把 fileName 域设为高亮
           highlightBuilder.field(highlightName);
           highlightBuilder.preTags("<span style='color:red'>");
           highlightBuilder.postTags("</span>");
           sourceBuilder.highlighter(highlightBuilder);
   
           // 发起请求
           searchRequest.source(sourceBuilder);
           SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
   
           // 保存高亮
           ArrayList<Map<String,Object>> list = new ArrayList<>();
           SearchHits hits = searchResponse.getHits();
           for (SearchHit hit : hits) {
               // 获取高亮域
               Map<String, HighlightField> highlightFields = hit.getHighlightFields();
               HighlightField highlight = highlightFields.get("fileName");
               // 获取原本的 SourceMap
               Map<String, Object> sourceAsMap = hit.getSourceAsMap();
               // 这里一定要先判断域是否为空，因为可能查询不到结果
               if (highlight != null) {
                   Text[] fragments = highlight.fragments();
                   StringBuilder new_name = new StringBuilder();
                   for (Text text : fragments) {
                       new_name.append(text);
                   }
                   // 覆盖原本的 SourceMap
                   sourceAsMap.put("fileName", new_name);
               }
               list.add(sourceAsMap);
           }
           return list;
       }
   ```

   

2. 方法二

   实现DocumentRepository

   ``` java
   public interface DocumentRepository extends ElasticsearchRepository<DocumentPo,Long> {
   }
   
   ```

   构建查询

   ``` java 
    // 构建查询条件
           NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
           // 添加基本分词查询
           queryBuilder.withQuery(QueryBuilders.matchQuery("fileName", keyWord));
           // 添加高亮
           HighlightBuilder highlightBuilder = new HighlightBuilder();
           HighlightBuilder.Field highlightName = new HighlightBuilder.Field("fileName"); //把 name 域设为高亮
           highlightBuilder.field(highlightName);
           highlightBuilder.preTags("<span style='color:red'>");
           highlightBuilder.postTags("</span>");
           queryBuilder.withHighlightBuilder(highlightBuilder);
           queryBuilder.withPageable(PageRequest.of(page, size));
   
           // 搜索，获取结果
          Page<DocumentPo> items = documentRepository.search(queryBuilder.build());
   ```

   

   发现高亮没有生效，跟踪源码发现在箭头区域给过滤掉了

   ![返回结果时](https://images.cnblogs.com/cnblogs_com/ants_double/1989032/o_210723052117eshightligh.png)

   解决办法

   自己重写这个方法，查看结构自己定义一个mapper转换，返回就行。

   但是查看search方法没有接收参数

   ```java 
   @NoRepositoryBean
   public interface ElasticsearchRepository<T, ID> extends ElasticsearchCrudRepository<T, ID> {
   
   	<S extends T> S index(S entity);
   
   	/**
   	 * This method is intended to be used when many single inserts must be made that cannot be aggregated to be inserted
   	 * with {@link #saveAll(Iterable)}. This might lead to a temporary inconsistent state until {@link #refresh()} is
   	 * called.
   	 */
   	<S extends T> S indexWithoutRefresh(S entity);
   
   	Iterable<T> search(QueryBuilder query);
   
   	Page<T> search(QueryBuilder query, Pageable pageable);
   
   	Page<T> search(SearchQuery searchQuery);
   
   	Page<T> searchSimilar(T entity, String[] fields, Pageable pageable);
   
   	void refresh();
   
   	Class<T> getEntityClass();
   }
   
   ```

   继续向下找，发现search调用的是queryForPage

   ```java
   @Override
   	public Page<T> search(SearchQuery query) {
   
   		return elasticsearchOperations.queryForPage(query, getEntityClass());
   	}
   ```

   继续找发现在下一步参数中多了一个resultsMapper，而出问题的方法也在这个参数对象中，于是我们只需要重写这个参数并调用这个方法就可以了。

   ``` java
   	@Override
   	public <T> AggregatedPage<T> queryForPage(SearchQuery query, Class<T> clazz) {
   		return queryForPage(query, clazz, resultsMapper);
   	}
   
   ```

   重写参数

   ``` java
   public class HighlightResultMapper implements SearchResultMapper {
       @Override
       public <T> AggregatedPage<T> mapResults(SearchResponse searchResponse, Class<T> clazz, Pageable pageable) {
           long totalHits = searchResponse.getHits().getTotalHits();
           List<T> list = new ArrayList<>();
           SearchHits hits = searchResponse.getHits();
           if (hits.getHits().length> 0) {
               for (SearchHit searchHit : hits) {
                   Map<String, HighlightField> highlightFields = searchHit.getHighlightFields();
                   T item = JSON.parseObject(searchHit.getSourceAsString(), clazz);
                   Field[] fields = clazz.getDeclaredFields();
                   for (Field field : fields) {
                       field.setAccessible(true);
                       if (highlightFields.containsKey(field.getName())) {
                           try {
                               field.set(item, highlightFields.get(field.getName()).fragments()[0].toString());
                           } catch (IllegalAccessException e) {
                               e.printStackTrace();
                           }
                       }
                   }
                   list.add(item);
               }
           }
           return new AggregatedPageImpl<>(list, pageable, totalHits);
       }
   
       @Override
       public <T> T mapSearchHit(SearchHit searchHit, Class<T> type) {
           return null;
       }
   
   }
   
   ```

   更改调用过程

   ``` java
   // 构建查询条件
           NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
           // 添加基本分词查询
           queryBuilder.withQuery(QueryBuilders.matchQuery("fileName", keyWord));
           // 添加高亮
           HighlightBuilder highlightBuilder = new HighlightBuilder();
           HighlightBuilder.Field highlightName = new HighlightBuilder.Field("fileName"); //把 name 域设为高亮
           highlightBuilder.field(highlightName);
           highlightBuilder.preTags("<span style='color:red'>");
           highlightBuilder.postTags("</span>");
           queryBuilder.withHighlightBuilder(highlightBuilder);
           queryBuilder.withPageable(PageRequest.of(page, size));
   
           // 搜索，获取结果
         // Page<DocumentPo> items = documentRepository.search(queryBuilder.build());
   
           SearchQuery searchQuery = null;
           searchQuery=  queryBuilder.build();
           Page<DocumentPo> items = new ElasticsearchRestTemplate(restHighLevelClient).queryForPage(searchQuery,DocumentPo.class,new HighlightResultMapper());
   ```

   

---

完！