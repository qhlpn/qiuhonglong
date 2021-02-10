#### ES 相关性

> 相关性描述的是⽂档和检索词的 **匹配程度 _score** 

对于**信息检索工具**，衡量其性能有3大指标：

1. **准确率 Precision**：尽可能返回**较少的无关文档**
2. **覆盖率 Recall**：尽可能返回**较多的相关文档**
3. **排序 Ranking**：是否能按相关性排序

其中，排序则与相关性的判断和算分相关



#### TF-IDF 和 BM25

1. **词频 TF（Term Frequency）**

   > 检索词在 **一个文档中** 出现的频度，越高权重越高
   > tf(t in d) = √ frequency ：检索词在文档中出现次数的平方根
   
2. **逆向文档频率 IDF（Inverse Document Frequency）**

   > 检索词在 **所有索引中** 出现的频率，越低权重越高（区分度大，可快速缩小索引范围）
   > idf(t) = 1 + ln ( numDocs / (docFreq + 1) ) ：索引中文档数量除以所有包含该词的文档数，然后求其对数。

3. **字段长度准则 FLN (Field Length Norm)**

   > 检索词所在的文档字段越短，权重越高
   > norm(d) = 1 / √ numTerms ：检索词所在的文档词数平方根的倒数

4. **TF 和 IDF 直观：**
   
   > 索引词在一个文档中出现的次数越多，同时在其它文档中出现的次数越少，该索引词就 越能代表该文档。
   
5. **Lucene TF-IDF 评分公式**

   ```
   _score(q,d)  =                           # 得分
               queryNorm(q)                 # 正则化因子
             · coord(q,d)                   # 协调因子
             · ∑ (                           
                   tf(t in d)
                 · idf(t)²
                 · t.getBoost()
                 · norm(t, d)
               ) (t in d)
   ```

6. **BM25 公式** 

   > 对于 TF-IDF 算法，**TF(t) 部分的值越大，整个公式返回的值就会越大。**
   > 针对这一点，BM25对公式进行改进 ，**随着TF(t) 的逐步加大，算法返回值趋于一个数值。**

   > **_score (doc)  =  ∑ bm25(ti in doc)**
   
   ```
bm25(t) = idf * tfNorm
   
   idf, computed as 
   	log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5))
   
   tfNorm, computed as 
   	(freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength))
   ```

   

#### 相关性控制

> 改变和控制相关性计算的方式，以满足实际场景需求

1. **boost 参数** 【常用】

   > ​    boost > 1，相关性提高
   > ​    0 < boost < 1，相对度降低
   > ​    boost < 0，贡献负分

   ```
   # 比如：博客检索时，标题权重应该比内容权重大
   GET /blogs_index/_search
   {
       "query": {
           "bool": {
               "must": [
                   {
                       "match": {
                           "title": {
                               "query": "es",
                               "boost": 2    #  提高权重
                           }
                       }
                   },
                   {
                       "match": {
                           "content": "es"
                       }
                   }
               ]
           }
       },
       "explain": true    #  展示相关性分数计算过程
   }
   ```

2. **组合查询**

   - **constant_score**

     ```
     # 嵌套 filter 查询，为任意一个匹配的文档指定一个常量评分 boost， 默认值 1 ，忽略 TF-IDF 信息
     GET /blogs_index/_search
     {
         "query": {
             "constant_score" : {
                 "filter" : {
                     "term" : { "title": "es"}
                 },
                 "boost" : 1.2   # 常量评分
             }
         }
     }
     ```

   - **function_score**【灵活，适用场景广】

     ```
     # 定义一个或多个函数，为查询返回的文档计算一个新分数  TODO
     GET /blogs_index/_search
     {
         "query": {
             "function_score": {
                 "query": {
                     "match_all": {}
                 },
                 "boost": "5",
                 "functions": [
                     {
                         "filter": {
                             "match": {
                                 "title": "es"
                             }
                         },
                         "random_score": {},
                         "weight": 23
                     },
                     {
                         "filter": {
                             "match": {
                                 "title": "相关性"
                             }
                         },
                         "weight": 42
                     }
                 ],
                 "max_boost": 42,
                 "score_mode": "max",
                 "boost_mode": "multiply",
                 "min_score": 10
             }
         }
     }
     ```

   - **dis_max**

     ```
     # 使用某个最佳匹配查询子句的分数
     # 可以通过参数 tie_breaker（默认值 0） 控制其他查询子句的分数对 _score 的影响
     # _score = max(BM25) + ∑ other(BM25) * tie_breaker
     GET /blogs_index/_search
     {
         "query": {
             "dis_max": {
                 "tie_breaker": 0.5,
                 "boost": 1.2,
                 "queries": [
                     {
                         "term": {
                             "content": "es"
                         }
                     },
                     {
                         "match": {
                             "content": "相关性"
                         }
                     }
                 ]
             }
         }
     }
     ```

   - **boosting**【常用】

     ```
     # 有效降级 AND与查询 匹配的结果
     # 比如，检索 title 含 elasticsearch es 的文章，同时如果 content 含有 "编程"，则相关性降低
     {
         "query": {
             "boosting": {
                 "positive": {
                     "bool": {
                         "should": [
                             {
                                 "term": {
                                     "title": "es"
                                 }
                             },
                             {
                                 "term": {
                                     "title": "elasticsearch"
                                 }
                             }
                         ]
                     }
                 },
                 "negative": {
                     "term": {
                         "content": "编程"
                     }
                 },
                 "negative_boost": 0.2  # 调整参数：升权( >1 ), 降权( 0<x<1 )
             }
         }
     }
     ```

3. **rescore**

   ```
   # 先 query，再在结果集基础上重新打分 rescore，
   # window_size 是每一分片进行重新评分的顶部文档数量
   GET /blogs_index/_search
   {
       "query": {
           "bool": {
               "should": [
                   {
                       "match": {
                           "content": {
                               "query": "Elasticsearch相关性",
                               "minimum_should_match": "30%"
                           }
                       }
                   },
                   {
                       "match": {
                           "title": {
                               "query": "Elasticsearch"
                           }
                       }
                   }
               ]
           }
       },
       "rescore": {
           "window_size": 3,
           "query": {
               "rescore_query": {
                   "match_phrase": {
                       "content": {
                           "query": "Elasticsearch相关性",
                           "slop": 50
                       }
                   }
               }
           }
       }
}
   ```
   
   