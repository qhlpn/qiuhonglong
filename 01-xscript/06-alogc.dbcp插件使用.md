------

此文档记录 ```alogic.dbcp``` 基本插件使用方法

------

> **db**	建立数据库连接

``` xml
<db dbcpId="default" autoCommit="true" dbconn="parent">  
    <!-- - dbcpId  配置的数据源ID	
    	 - autoCommit	是否自动提交
    	 - dbconn	给数据源起变量名    -->
</db>
```



> **db-new**	增加数据

``` xml
<db-new sql="insert into user (name, age) values ('Tom', 18)" transform="false" dbconn="parent" debug="true"/> 
<!-- - transform true可解析sql语句中alogic变量${}，下面的插件同理
	 - dbconn 根据变量名获取数据源，下面的插件同理
	 - debug 是否开启控制台输出，下面的插件同理                   -->
```



> **db-delete**	删除数据

``` xml
<db-delete sql="delete from user where id = 1"/>
```



> **db-update**	更新数据

``` xml
<db-update sql="update user set name = 'Jack' where id = 1"/>
```



> **db-select**	查询数据（单条记录，输出变量）

``` xml
<db-select id="size" sql="select name, age from user where id = 1"/>
<!-- id 查询结果行数 -->
```



> **db-scan**	查询数据（多条记录，输出变量）

``` xml
<db-scan id="size" sql="select name, age from user" autoClear="false">
	<!-- 遍历每行记录
        - id 查询结果行数
        - autoClear 每轮循环最后是否需要清除字段变量 -->
</db-scan>
```



> **db-query**	查询数据（单条记录，输出文档）

``` xml
<db-query tag="obj" extend="false" sql="select name, age from user where id = 1"/> 
<!-- - tag 文档对象节点名
	 - extend 是否扩展为父文档的子节点 -->
```



> **db-list**	查询数据（多条记录，输出文档）

``` xml
<db-list tag="list" sql="select name, age from user"/>
<!-- tag 文档对象节点名 -->
```



> **db-keyvalues**	查询数据（输出变量为 select 前两列字段竖表形式 List<Pair<column1, column2>>）

``` xml
<db-keyvalues id="size" sql="select name, age, ... from user"/>
<!-- - output List<Pair<name, age>> 
 	 - id 查询结果个数               -->
```



> **db-n-exist**	查询数据，不存在时进行操作

``` xml
<db-n-exist id="size" sql="select name, age from user where id < 10">
	<!-- 查询结果不存在时才进入此代码块 -->
</db-n-exist>
	<!-- 查询结果存在时，id 指查到的行数
```



> **db-commit**	提交事务
>
> **db-rollback**	回滚事务

``` xml
<db dbcpId="default" autoCommit="false">  <!-- 关闭事务自动提交 -->
    <segment>	<!-- db 代码块需要两对 segment 才能捕获异常并回滚 -->
        <db-delete sql="delete from user where id = 1"/>
        <db-delete sql="delete from user where id = 2"/>	
        <db-commit/>   <!-- 正常，手动提交事务 -->
        <except>
            <segment>
                <db-rollback/>  <!-- 异常，手动回滚事务 -->
            </segment>
        </except>
    </segment>
</db>
```

