# Hive

> b站学习视频：<https://www.bilibili.com/video/BV1EZ4y1G7iL>
>
> Hive官网地址：<https://hive.apache.org>

# 一、Hive的基本概念

## 1.1 什么是Hive

**1）hive简介**

**Hive**：由 Facebook 开源用于解决海量结构化日志的数据统计工具。

**Hive** 是基于 **Hadoop** 的一个<font color=red>数据仓库工具</font>，可以将<font color=red>结构化的数据文件映射成一张表</font>，并提供<font color=red>类SQL</font>查询功能。

**2）Hive本质**：将 HQL 转化成 MapReduce 程序

## 1.2 Hive的优缺点

## 1.3 Hive架构原理

## 1.4 Hive和数据库比较

# 二、Hive安装

# 三、Hive数据类型

# 四、DDL数据定义

## 4.1 创建数据库





# 五、DML定义

# 六、查询

> Hive查询官方文档：<https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Select>

查询语句语法

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
  FROM table_reference
  [WHERE where_condition]
  [GROUP BY col_list]
  [ORDER BY col_list]
  [CLUSTER BY col_list
    | [DISTRIBUTE BY col_list] [SORT BY col_list]
  ]
 [LIMIT number]
```

## 6.1 基本查询（Select...From）

### 6.1.1 全表和特定列查询

**0）数据准备**

（0）原始数据

dept：

```sql
10	ACCOUNTING	1700
20	RESEARCH	1800
30	SALES	1900
40	OPERATIONS	1700
```

emp：

```sql
7369	SMITH	CLERK	7902	1980-12-17	800.00		20
7499	ALLEN	SALESMAN	7698	1981-2-20	1600.00	300.00	30
7521	WARD	SALESMAN	7698	1981-2-22	1250.00	500.00	30
7566	JONES	MANAGER	7839	1981-4-2	2975.00		20
7654	MARTIN	SALESMAN	7698	1981-9-28	1250.00	1400.00	30
7698	BLAKE	MANAGER	7839	1981-5-1	2850.00		30
7782	CLARK	MANAGER	7839	1981-6-9	2450.00		10
7788	SCOTT	ANALYST	7566	1987-4-19	3000.00		20
7839	KING	PRESIDENT		1981-11-17	5000.00		10
7844	TURNER	SALESMAN	7698	1981-9-8	1500.00	0.00	30
7876	ADAMS	CLERK	7788	1987-5-23	1100.00		20
7900	JAMES	CLERK	7698	1981-12-3	950.00		30
7902	FORD	ANALYST	7566	1981-12-3	3000.00		20
7934	MILLER	CLERK	7782	1982-1-23	1300.00		10
```

（1）创建部门表

```sql
create table if not exists dept(
    deptno int,
    dname string,
    loc int
)
row format delimited fields terminated by '\t';
```

（2）创建员工表

```sql
create table if not exists emp(
    empno int,
    ename string,
    job string,
    mgr int,
    hiredate string, 
    sal double, 
    comm double,
    deptno int
)
row format delimited fields terminated by '\t';
```

（3）导入数据

```sql
load data local inpath '/opt/module/datas/dept.txt' into table dept;
load data local inpath '/opt/module/datas/emp.txt' into table emp;
```

**1）全表查询**

```sql
hive (default)> select * from emp;
hive (default)> select empno,ename,job,mgr,hiredate,sal,comm,deptno from emp ;
```

**2）选择特定列查询**

```sql
hive (default)> select empno, ename from emp;
```

注意：

（1）SQL 语言<font color=red>大小写不敏感</font>。 

（2）SQL 可以写在一行或者多行

（3）关键字不能被缩写也不能分行

（4）各子句一般要分行写。

（5）使用缩进提高语句的可读性。

### 6.1.2 列别名

**1）重命名一个列**

**2）便于计算**

**3）紧跟列名，也可以在列名和别名之间加入关键字 'AS'**

**4）案例实操**

查询名称和部门

```sql
hive (default)> select ename AS name, deptno dn from emp;
```

### 6.1.3 算术运算符

| 运算符 | 描述              |
| ------ | ----------------- |
| A+B    | A 和 B 相加       |
| A-B    | A 减去 B          |
| A*B    | A 和 B 相乘       |
| A/B    | A 除以 B          |
| A%B    | A 对 B 取余       |
| A&B    | A 和 B 按位取与   |
| A\|B   | A 和 B 按位取或   |
| A^B    | A 和 B 按位取异或 |
| ~A     | A 按位取反        |

### 6.1.4 常用函数

| 函数名  | 描述     |
| ------- | -------- |
| count() | 求总行数 |
| max()   | 求最大值 |
| min()   | 求最小值 |
| sum()   | 求总和   |
| avg()   | 求平均值 |

### 6.1.5 Limit语句

典型的查询会返回多行数据。LIMIT子句用于限制返回的行数。

### 6.1.6 Where语句

**1）使用 WHERE 子句，将不满足条件的行过滤掉**

**2）WHERE 子句紧随 FROM 子句**

<font color=red>注意</font>：where子句中不能使用字段别名。

### 6.1.7 比较运算符（Between/In/Is Null）

| 操作符                  | 支持的数据类型 | 描述                                                         |
| ----------------------- | -------------- | ------------------------------------------------------------ |
| A=B                     | 基本数据类型   | 如果 A 等于 B 则返回 TRUE ，反之返回 FALSE                   |
| A<=>B                   | 基本数据类型   | 如果 A 和 B 都为 NULL ，则返回 TRUE ，如果一边为 NULL ，返回 False |
| A<>B,A!=B               | 基本数据类型   | A 或者 B 为 NULL 则返回 NULL ；如果 A 不等于 B ，则返回 TRUE ，反之返回 FALSE |
| A<B                     | 基本数据类型   | A 或者 B 为 NULL ，则返回 NULL ；如果 A 小于 B ，则返回 TRUE ，反之返回 FALSE |
| A<=B                    | 基本数据类型   | A 或者 B 为 NULL ，则返回 NULL ；如果 A 小于等于 B ，则返回 TRUE ，反之返回 FALSE |
| A>B                     | 基本数据类型   | A 或者 B 为 NULL ，则返回 NULL ；如果 A 大于 B ，则返回 TRUE ，反之返回 FALSE |
| A>=B                    | 基本数据类型   | A 或者 B 为 NULL ，则返回 NULL ；如果 A 大于等于 B ，则返回 TRUE，反之返回 FALSE |
| A [NOT] BETWEEN B AND C | 基本数据类型   | 如果 A，B 或者 C 任一为 NULL ，则结果为 NULL 。如果 A 的值大于等于 B 而且小于或等于 C ，则结果为 TRUE ，反之为 FALSE 。如果使用 NOT 关键字则可达到相反的效果。 |
| A IS NULL               | 所有数据类型   | 如果 A 等于 NULL ，则返回 TRUE ，反之返回 FALSE              |
| A IS NOT NULL           | 所有数据类型   | 如果 A 不等于 NULL ，则返回 TRUE ，反之返回 FALSE            |
| IN(数值1,数值2)         | 所有数据类型   | 使用 IN 运算显示列表中的值                                   |
| A [NOT] LIKE B          | STRING 类型    | B 是一个 SQL 下的简单正则表达式，也叫通配符模式，如果 A 与其匹配的话，则返回 TRUE ；反之返回 FALSE 。B 的表达式说明如下：'x%' 表示A必须以字母 'x' 开头，'%x' 表示A必须以字母 'x' 结尾，而 '%x%' 表示 A 包含有字母 'x' ，可以位于开头，结尾或者字符串中间。如果使用 NOT 关键字则可达到相反的效果。 |
| A RLIKE B, AREGEXP B    | STRING 类型    | B 是基于 java 的正则表达式，如果 A 与其匹配，则返回 TRUE ；反之返回 FALSE 。匹配使用的是 JDK 中的正则表达式接口实现的，因为正则也依据其中的规则。例如，正则表达式必须和整个字符串 A 相匹配，而不是只需与其字符串匹配。 |

### 6.1.8 Like和RLike

**1）使用 LIKE 运算选择类似的值**

**2）选择条件可以包含字符或数字**

% 代表零个或多个字符(任意个字符)。

_ 代表一个字符。

**3）RLIKE 子句**

RLIKE 子句是 Hive 中这个功能的一个扩展，其可以通过 <font color=red>Java的正则表达式</font> 这个更强大的语言来指定匹配条件。

### 6.1.9 逻辑运算符（And/Or/Not）

| 逻辑运算符 | 含义   |
| ---------- | ------ |
| AND        | 逻辑与 |
| OR         | 逻辑或 |
| NOT        | 逻辑非 |

## 6.2 分组

### 6.2.1 Group By语句

GROUP BY 语句通常会和聚合函数一起使用，按照一个或者多个列队结果进行分组，然后对每个组执行聚合操作。

### 6.2.2 Having语句

**having 与 where 不同点**

（1）where 后面不能写分组函数，而 having 后面可以使用分组函数。

（2）having 只用于 group by 分组统计语句。

## 6.3 Join语句

### 6.3.1 等值Join

Hive 支持通常的 SQL JOIN 语句。

### 6.3.2 表的别名

（1）使用别名可以简化查询。

（2）使用表名前缀可以提高执行效率。

![7种join](https://www.runoob.com/wp-content/uploads/2019/01/sql-join.png)

### 6.3.3 内连接

内连接：只有进行连接的两个表中都存在与连接条件相匹配的数据才会被保留下来。

### 6.3.4 左外连接

左外连接：JOIN操作符左边表中符合WHERE子句的所有记录将会被返回。

### 6.3.5 右外连接

右外连接：JOIN操作符右边表中符合WHERE子句的所有记录将会被返回。

### 6.3.6 满外连接

满外连接：将会返回所有表中符合WHERE语句条件的所有记录。如果任一表的指定字段没有符合条件的值的话，那么就使用NULL值替代。

### 6.3.7 多表连接

注意：连接 n 个表，至少需要 n-1 个连接条件。例如：连接三个表，至少需要两个连接条件。

大多数情况下，Hive 会对每对 JOIN 连接对象启动一个 MapReduce 任务。本例中会首先启动一个 MapReduce job 对表 e 和表 d 进行连接操作，然后会再启动一个 MapReduce job 将第一个 MapReduce job 的输出和表l;进行连接操作。

注意：为什么不是表 d 和表 l 先进行连接操作呢？这是因为 Hive 总是按照从左到右的顺序执行的。

优化：当对3个或者更多表进行 join 连接时，如果每个 on 子句都使用相同的连接键的话，那么只会产生一个 MapReduce job。

### 6.3.8 笛卡尔积

**笛卡尔积会在下面条件下产生**

（1）省略连接条件

（2）连接条件无效

（3）所有表中的所有行互相连接

## 6.4 排序

### 6.4.1 全局排序（order by）

Order By：全局排序，只有一个Reducer

**1）使用 ORDER BY 子句排序**

```sql
ASC（ascend）: 升序（默认）
DESC（descend）: 降序
```

**2）ORDER BY 子句在 SELECT 语句的结尾**

**3）按照别名排序**

**4）多个列排序，关键字后用逗号分隔**

### 6.4.2 每个Reduce内部排序（sort by）

Sort By：对于大规模的数据集 order by 的效率非常低。在很多情况下，并不需要全局排序，此时可以使用 **sort by**。

Sort by 为每个 reducer 产生一个排序文件。每个 Reducer 内部进行排序，对全局结果集来说不是排序。

```sql
设置reduce个数
hive (default)> set mapreduce.job.reduces=3;

查看设置reduce个数
hive (default)> set mapreduce.job.reduces;
```

### 6.4.3 分区（distribute by）

Distribute By： 在有些情况下，我们需要控制某个特定行应该到哪个 reducer ，通常是为了进行后续的聚集操作。**distribute by** 子句可以做这件事。**distribute by **类似 MR 中 partition（自定义分区），进行分区，结合 sort by 使用。 

对于 distribute by 进行测试，一定要分配多 reduce 进行处理，否则无法看到 distribute by 的效果。

<font color=red>注意</font>：

* distribute by 的分区规则是根据分区字段的 hash 码与 reduce 的个数进行模除后，余数相同的分到一个区。
* Hive 要求 DISTRIBUTE BY 语句要写在 SORT BY 语句之前。

### 6.4.4 cluster by

当 distribute by 和 sort by 字段相同时，可以使用 cluster by 方式。

cluster by 除了具有 distribute by 的功能外还兼具 sort by 的功能。但是排序只能是升序排序，不能指定排序规则为 ASC 或者 DESC。

# 七、分区表和分桶表

## 7.1 分区表

分区表实际上就是对应一个 HDFS 文件系统上的独立的文件夹，该文件夹下是该分区所有的数据文件。<font color=red>Hive中的分区就是分目录</font>，把一个大的数据集根据业务需要分割成小的数据集。在查询时通过 WHERE 子句中的表达式选择查询所需要的指定的分区，这样的查询效率会提高很多。

### 7.1.1 分区表基本操作

**1）引入分区表（需要根据日期对日志进行管理, 通过部门信息模拟）**

```sql
dept_20200401.log
dept_20200402.log
dept_20200403.log
……
```

**2）创建分区表语法**

```sql
hive (default)> create table dept_partition(
deptno int, dname string, loc string
)
partitioned by (day string)
row format delimited fields terminated by '\t';
```

<font color=red>注意</font>：分区字段不能是表中已经存在的数据，可以将分区字段看作表的伪列。

**3）加载数据到分区表中**

**（1）数据准备**

dept_20200401.log

```sql
10	ACCOUNTING	1700
20	RESEARCH	1800
```

dept_20200402.log

```sql
30	SALES	1900
40	OPERATIONS	1700
```

dept_20200403.log

```sql
50	TEST	2000
60	DEV	1900
```

**（2）加载数据**

```sql
hive (default)> load data local inpath '/opt/module/hive/datas/dept_20200401.log' into table dept_partition partition(day='20200401');
hive (default)> load data local inpath '/opt/module/hive/datas/dept_20200402.log' into table dept_partition partition(day='20200402');
hive (default)> load data local inpath '/opt/module/hive/datas/dept_20200403.log' into table dept_partition partition(day='20200403');
```

<font color=red>注意</font>：分区表加载数据时，必须指定分区。

**4）查询分区表中数据**

单分区查询

```sql
hive (default)> select * from dept_partition where day='20200401';
```

多分区联合查询

```sql
hive (default)> select * from dept_partition where day='20200401'
              union
              select * from dept_partition where day='20200402'
              union
              select * from dept_partition where day='20200403';
              
hive (default)> select * from dept_partition where day='20200401' or
                day='20200402' or day='20200403' ;	
```

**5）增加分区**

创建单个分区


```sql
hive (default)> alter table dept_partition add partition(day='20200404') ;
```

同时创建多个分区

```sql
hive (default)> alter table dept_partition add partition(day='20200405') partition(day='20200406');
```

**6）删除分区**

删除单个分区

```sql
hive (default)> alter table dept_partition drop partition (day='20200406');
```

同时删除多个分区

```sql
hive (default)> alter table dept_partition drop partition (day='20200404'), partition(day='20200405');
```

**7）查看分区表有多少分区**

```sql
hive> show partitions dept_partition;
```

**8）查看分区表结构**

```sql
hive> desc formatted dept_partition;

# Partition Information          
# col_name              data_type               comment             
month                   string    
```

### 7.1.2 二级分区

**1）创建二级分区表**

```sql
hive (default)> create table dept_partition2(
               deptno int, dname string, loc string
               )
               partitioned by (day string, hour string)
               row format delimited fields terminated by '\t';
```

**2）正常的加载数据**

（1）加载数据到二级分区表中

```sql
hive (default)> load data local inpath '/opt/module`/hive/datas/dept_20200401.log' into table
dept_partition2 partition(day='20200401', hour='12');
```

（2）查询分区数据

```sql
hive (default)> select * from dept_partition2 where day='20200401' and hour='12';
```

**3）把数据直接上传到分区目录上，让分区表和数据产生关联的三种方式**

（1）方式一：上传数据后修复

上传数据

```sql
hive (default)> dfs -mkdir -p /user/hive/warehouse/mydb.db/dept_partition2/day=20200401/hour=13;
hive (default)> dfs -put /opt/module/datas/dept_20200401.log  /user/hive/warehouse/mydb.db/dept_partition2/day=20200401/hour=13;
```

查询数据（查询不到刚上传的数据）

```sql
hive (default)> select * from dept_partition2 where day='20200401' and hour='13';
```

执行修复命令

```sql
hive> msck repair table dept_partition2;
```

再次查询数据

```sql
hive (default)> select * from dept_partition2 where day='20200401' and hour='13';
```

（2）方式二：上传数据后添加分区

上传数据

```sql
hive (default)> dfs -mkdir -p /user/hive/warehouse/mydb.db/dept_partition2/day=20200401/hour=14;
hive (default)> dfs -put /opt/module/hive/datas/dept_20200401.log  /user/hive/warehouse/mydb.db/dept_partition2/day=20200401/hour=14;
```

执行添加分区

```sql
hive (default)> alter table dept_partition2 add partition(day='20200401',hour='14');
```

查询数据

```sql
hive (default)> select * from dept_partition2 where day='20200401' and hour='14';
```

（3）方式三：创建文件夹后load数据到分区

创建目录

```sql
hive (default)> dfs -mkdir -p /user/hive/warehouse/mydb.db/dept_partition2/day=20200401/hour=15;
```

上传数据

```sql
hive (default)> load data local inpath '/opt/module/hive/datas/dept_20200401.log' into table dept_partition2 partition(day='20200401',hour='15');
```

查询数据

```sql
hive (default)> select * from dept_partition2 where day='20200401' and hour='15';
```

### 7.1.3 动态分区

## 7.2 分桶表

## 7.3 抽样查询

# 八、函数

## 8.1 系统内置函数

1）查看系统自带的函数

```sql
hive> show functions;
```

2）显示自带的函数的用法

```sql
hive> desc function upper;
```

3）详细显示自带的函数的用法

```sql
hive> desc function extended upper;
```

## 8.2 常用内置函数

### 8.2.1 空字段赋值

```sql
NVL：给值为NULL的数据赋值，它的格式是NVL(value，default_value)。
它的功能是如果value为NULL，则NVL函数返回default_value的值。否则返回value的值。如果两个参数都为NULL，则返回NULL。
```

### 8.2.2 CASE WHEN THEN ELSE END

### 8.2.3 行转列

### 8.2.4 列转行

### 8.2.5 窗口函数（开窗函数）

### 8.2.6 RANK

### 8.2.7 其他常用函数

## 8.3 自定义函数

## 8.4 自定义UDF函数

## 8.5 自定义UDTF函数

# 九、压缩和存储

# 十、企业级调优

# 十一、Hive实战
