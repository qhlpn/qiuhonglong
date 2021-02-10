#### Bucket Aggregations 桶聚合

> 类似于 SQL 的分组（group by），一些列满足特定条件的文档的集合

**Aggregations 语法结构** 

```
{
    "aggs": {      # query 同级关键词
        "aggsName1": {   # 自定义聚合名字
            "terms": {   # 聚合类型 Type
                "field": "color"   # colunm field
            },
            "aggs": {   # 子聚合查询（上级聚合后每个分组依次子聚合查询）
            }
        },
        "aggsName2": {  # 同级集合查询
        }
    }
}
```

**Bucket Aggs 聚合类型**

1. **Terms 术语聚合**

   > 场景：按字段将文档聚合分类，类似于 group by column

   ```
   {
       "aggs": {
           "author": {
               "terms": {
                   "field": "author"
               }
           }
       }
   }
   ```

2. **Rare Terms 稀有术语聚合**

   > 场景：与 Terms 术语聚合类似，同时按 doc_count 升序排列（Terms 是降序）

   ```
   {
       "aggs": {
           "author": {
               "rare_terms": {
                   "field": "author",
                   "max_doc_count": 10  # 返回桶 doc_count lte 
               }
           }
       }
   }
   ```

3. **Histogram 直方图聚合**

   > 场景：根据数值或数据范围类型的字段，按固定间隔 interval 将文档分类

   ```
   {
       "aggs": {
           "price": {
               "histogram": {
                   "field": "price",
                   "interval": 2000  # [0,2000) [2000,4000) ...
               }
           }
       }
   }
   ```

4. **Date Histogram 日期直方图聚合**

   > 场景：根据日期或日期范围类型的字段，按固定间隔 interval 将文档分类

   ``` 
   {
       "aggs": {
           "price": {
               "date_histogram": {
                   "field": "createAt",
                   "interval": "1d",  # 按每天
                   "format": "yyyy-MM-dd"
               }
           }
       }
   }
   ```

5. **Auto-interval Date Histogram 自动间隔日期直方图聚合**

   > 场景：类似于 Date Histogram，不过是根据 固定存储桶数 来自动划分 interval 聚合

   ```
   {
       "aggs": {
           "createTime": {
               "auto_date_histogram": {
                   "field": "createAt",
                   "format": "yyyy-MM-dd",
                   "buckets": 3   # 返回的桶数，默认10
               }
           }
       }
   }
   ```

6. **Range 范围聚合**  / **Date Range 日期范围聚合**（针对 Date 类型字段）/ **IP Range IP范围聚合**

   > 场景：用于自定义范围进行聚合（每个范围代表一个桶）

   ```
   {
       "aggs": {
           "price_ranges": {
               "range": {               #  "date_range": {
                   "field": "price",    #  	"field":"date_field", "format":"yyyy-MM-dd"
                   "ranges": [
                       {
                           "to": 100
                       },
                       {
                           "from": 100, #  [from, to)
                           "to": 200
                       },
                       {
                           "from": 200
                       }
                   ]
               }
           }
       }
   }
   ```

7. **Composite 复合聚合**

   > 场景1： 翻页查询

   ```
   // ...
   ```

   > 场景2：多个字段所有的可能组合（笛卡尔乘积）

   ```
   {
       "aggs": {
           "my_buckets": {
               "composite": {
                   "sources": [
                       {
                           "question_A": {
                               "terms": {
                                   "field": "question_A"
                               }
                           }
                       },
                       {
                           "question_B": {
                               "terms": {
                                   "field": "question_B"
                               }
                           }
                       }
                   ]
               }
           }
       }
   }
   ```

8. **Filter 过滤器聚合**

   > 场景：在 query 结果集基础上，先筛选文档再聚合

   ```
   {
       "aggs" : {
           "t_shirts" : {
               "filter" : { "term": { "type": "t-shirt" } },
               "aggs" : {
                   "avg_price" : { "avg" : { "field" : "price" } }
               }
           }
       }
   }
   ```

9. **Filters 过滤器集合聚合**

   > 场景：统计不同的字段，类似于 SQL 多个 Count

10. **Adjacency Matrix 邻接矩阵聚合**

    > 场景：类似 Filters，不同的是统计 交叉滤波矩阵 中的 非空单元

    |      | A    | B    |
    | ---- | ---- | ---- |
    | A    | A    | AB   |
    | B    |      | B    |

11. **Missing 缺少聚合**

    > 场景：类似于 SQL 中 count() where field is null

12. **Geo 地理位置聚合 ...**





#### Metrics Aggregations 指标聚合

> 类似于 SQL 的 count / sum / max 等数学运算方法，对文档字段进行统计分析

1. **Stats 聚合 「含 Avg / Max / Min / Sum 聚合」**  

2. **Value Count value 计数聚合**

   > 场景：统计 Field 不为 null 的文档数

3. **Weighted Avg 加权平均聚合**

4. **Cardinality 基数聚合**

   > 场景：统计字段有多少不同的值，类似于 SQL 中的 select count(distinct column) from

5. **String Stats 字符串统计聚合**

   > 场景：keyword类型字段5个指标（count / min_length / max_length / avg_length / entropy）

6. **Geo 地理位置聚合 ...**





#### Pipeline Aggregations 管道聚合

> 对其它聚合结果进行二次聚合，**增加输出有用信息**；工作于**其他聚合**生成输出结果而不是文档集

> **bucket_path 参数：** 定义要进行二次计算的链接对象

管道聚合根据输出结果位置分为：

1. Parent  结果内嵌到现有的聚合分析结果
2. Sibling  结果和现有分析结果同级 

**Parent**

- **Bucket Script 桶脚本聚合**

  > 场景：执行脚本（script字段），对父级聚合得到的每个分组依次进行计算
  
- **Bucket Selector 桶选择器聚合**

  > 场景：执行脚本（script字段），对父级聚合得到的每个分组依次进行过滤（是否留下分组）

- **Bucket Sort 桶排序聚合**

  > 场景：对父级聚合得到的桶进行排序（sort字段）

- **Cumulative Cardinality 累积基数聚合**

  > 场景：用于计算父直方图聚合中 字段值 累积总和
  >
  > ​			**derivative 字段** ：计算新增的个数（以前一天为基准）

- **Cumulative Sum 累积总和聚合**

  > 场景：用于计算父直方图聚合中 **指定指标** 的累积总和

- **Derivative 导数聚合 ...**

**Sibling**

- **Avg Bucket 平均存储桶聚合**

  > 场景：聚合得到分组，计算所有分组的平均值

- **Max / Min / Sum / Stats ... 桶聚合**  