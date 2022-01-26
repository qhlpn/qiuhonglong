#### Lambda

| 函数接口       | 抽象方法        | 功能                   | 参数   | 返回类型 |
| :------------- | :-------------- | :--------------------- | :----- | :------- |
| Predicate      | test(T t)       | 判断真假               | T      | boolean  |
| Consumer       | accept(T t)     | 消费消息               | T      | void     |
| Function       | R apply(T t)    | 将T映射为R（转换功能） | T      | R        |
| Supplier       | T get()         | 生产消息               | None   | T        |
| UnaryOperator  | T apply(T t)    | 一元操作               | T      | T        |
| BinaryOperator | apply(T t, U u) | 二元操作               | (T，T) | (T)      |



| Stream流函数              | 功能                                                  | 底层       | 类型         |
| :------------------------ | :---------------------------------------------------- | :--------- | :----------- |
| s.collect                 | 流转换成集合，Collectors.toList() / toSet() / toMap() |            | Terminal     |
| s.filter                  | 过滤筛选                                              | Predicate  | Intermediate |
| s.map                     | 映射转换                                              | Function   | Intermediate |
| s.flatMap                 | 将多个Stream合并成一个Stream                          | Function   | Intermediate |
| s.max / min               | 流中最大和值最小值                                    | Comparator | Terminal     |
| s.count                   | 流元素个数                                            |            | Terminal     |
| s.reduce                  | 累计操作                                              |            | Terminal     |
| Collectors.partitioningBy | 将流拆分成两个集合                                    | Predicate  |              |
| Collectors.groupingBy     | 将流元素进行集合分组                                  | Function   |              |