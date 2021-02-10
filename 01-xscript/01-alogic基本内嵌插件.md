------

此文档记录最近开发中比较常用的 ```alogic``` 内嵌插件，方便检索。

更多新的内嵌插件和实现逻辑可参考 ```alogic-common``` ``` com.alogic.xscript.plugins``` 源码。 

------

> **加载脚本** 

``` xml
<include src="java:///path#class"/>
```

> **导入插件**

``` xml
<using xmlTag="ns" module="Class.name"/>
```

> **打印日志**

```  xml
<log msg="~~~~"/>
```

> **局部代码块**

``` xml
<scope> </scope>  -- 局部变量域
<segment> </segment>  -- 全局变量域
```

> **变量操作**

``` xml
<!-- 检查当前作用域是否存在指定变量 -->
<check id="  "/>
<checkAndSet id="..." dft="  "/>

<!-- 设置常量 -->
<constants id="  " value="  "/>

<!-- 设置变量 -->
<set id="..." value="..."/>

<!-- 环境变量 -->
<setting id="..."  value="...">  -- 从 Settings 提取变量
<setting-set id="..." value="..." />  -- 更新 Settings 变量值

<!-- 文档与变量间转换 -->
<setAsJson id=""/>   -- 将当前文档作用域输出为变量
<getAsJson tag="" content="" extend="是否扩展为子节点">  -- 将变量输出到文档
    
<!-- 变量作用域桥接 -->
<bridge>
    <scope>
    	<bridge-set id="bid" value=" "/>
    </scope>
    <log msg="get bid = ${bid}"/>
</bridge>
```

> **文档操作**

``` xml
<!-- 文档输出 -->
<obj tag=" ">
	<get id=" " value=" "/> 
</obj>

<!-- 解析JSON字符串为文档 -->
<template tag=" " content="{name:Tome,age:18}" extend=" ">
</template>

<!-- 切换文档当前节点 -->
<location jsonPath="$.data.hello"> -- 路径格式： $.对象/数组[*]
</location>

<!-- 遍历文档数组中的元素 -->
<repeat path="$.data.list[*]">   
    -- 若要输出文档，先 set 后 get
</repeat>

<!-- 文档桥接 -->
<obj-bridge-add bridge=""/>  -- 添加
<obj-bridge bridge=""> -- 定位/取出
</obj-bridge>  
<obj-bridge-remove bridge="">  -- 删除
```

> **Together 入参操作**

``` xml
1 - 服务的输入参数(URL或FORM传递)转化成脚本变量
2 - 服务的输入参数(Json文档)转化为脚本工作文档
```

> **逻辑操作**

``` xml
<!-- 分支逻辑 -->
<if-n-equal left="" right=""> </if-n-equal> 
<if-equal left="" right=""> </if-equal>

<if-true value="${var}"> </if-true>  -- null 亦 true
<if-false value=" "> </if-false>     -- null 亦 false

<switch value=" ">
    <log case="enumA" msg=" "/>
    <set case="enumB" id=" " value=" "/>
</switch>
        
<!-- 循环逻辑 -->
<while expr="  "> </while>  -- expr 使用 &lt; &gt; &amp;
<foreach in=" " id=" " delimeter=" "> </foreach>  -- 遍历字符串列表，而非数组
```

> **字符串操作**

``` xml
<trim id=" " value=" "/>  -- 去除空格
<substr id=" " value=" " start=" " length=" "/>  -- idx 包括 0~start

<strlist id=" " delimeter=" ">  -- 字符串列表
    <strlist-add value="${ }"/>
</strlist>
<foreach in=" " id=" " delimeter=" "> </foreach>  -- 遍历字符串列表，而非数组

<lowercase id=" " value=" "/> -- 小写
<uppercase id=" " value=" "/> -- 大写

<match id=" " value=" " pattern="regx"/>  -- 是否匹配
<regex-match value="" pattern="regx"> </regex-match> -- 正则提取
```

> **数组操作**

``` xml
<!-- 输出文档数组 -->
<array tag="user" id="aid"> 
    <foreach>
        <array-item id="aid">
        </array-item>    
    </foreach>
</array> 

<!-- 遍历文档数组中的元素 -->
<repeat path="$.data.list[*]">   
    -- 若要输出文档，先 set 后 get
</repeat>
```

> **集合操作**

``` xml
<!-- 输出集合文档 -->
<array-set tag=" " cid=" " output="作为文档或变量">
	<array-set-add pid=" " member="val"/>
    <array-set-del pid=" " member="val"/>
    <array-set-exist pid=" " member="val"/>
    <array-set-list pid=" " id=" "> </array-set-list>
</array-set>

<!-- 集合文档转为变量 -->
<array-set-save pid=" " id=" " delimeter=" "/>
```

> **运算操作**

``` xml
<!-- 四则运算 -->
<!-- formula 数据类型 string double long boolean date -->
<formula id="num" expr="to_long(num) + 2"/>  -- num=14  long   
<formula id="num" expr="num + 2"/>  -- num=122  str   

<!-- formula 嵌入 ${}: ${# expr #}-->
<set id="big" value="${#choice(to_long(left) > to_long(right), to_long(left), to_long(right))#}"/> -- 三元表达式
<log msg="time = ${#filter_tt(now,'hh:mm')#}"/>  -- 日期格式转换

<!-- 内置函数 -->
时间：to_date | to_char | filter_tt
数据：to_double | to_long | to_boolean
字符串：substr | instr | strlen
判断：choice | not_nvl | nvl | match
```

> **异常操作**

``` xml
<log msg="executed"/>
<throw id="key" msg="value"/>         -- 上方或下发抛出均可 
<log msg="no_executed"/>
<except id="key">					  -- 捕获当下代码块所有异常 可用 segment 隔离代码块
    <log msg="executed"/>
</except>
<log msg="no_executed"/>
<finally>							  -- 固定执行的代码
    <log msg="always_executed."/>
</finally> 
```

> **函数操作**

``` xml
<func-call id="func-declare" callback="func-call">  -- 1. ctx.setObject(callbackId, callback) 
    -- id 指向 函数声明的代码块 
    -- callback 指向 函数调用的代码块
    <log msg="~~~~~ Step 2"/>   			 		
</func-call>   

<func-declare id="func-declare">
	<log msg="~~~~~ Step 1"/>
    <func-callback callback="func-call"/>  			-- 2. ctx.getObject(callbackId)
    <log msg="~~~~~ Step 3"/>						-- func-declare 和 func-call 变量互通
</func-declare>		  					
```

> **异步操作**

```xml
<evt type="on.bizhit" async="true">
	<evt-property id=" " value=" "/> 
</evt>

<process id="on.bizhit">  -- declare in on.bizhit.xml
	<script> 
    </script>
</process>
```

> **RPC 操作**

``` xml
<remote-request clientId="default" method="get/post">
				-- clientId defined alogic.remote.xml
    
    <remote-call-direct path="www.baidu.com">
        
        <remote-set-header id=" " value=" "/>
        <remote-set-content-type id=" " value=" "/>
        
        <remote-by-form/text/json>
        	-- request body
        </remote-by-form/text/json>
        
    	<remote-as-json>
        	-- data put in current doc, not ctx
            -- doc will be remove after this tag
            -- so, do set before get
        </remote-as-json>
        
        <remote-get-header/code/reason id=" "/>
        
    </remote-call-direct>
    
</remote-request>
```

