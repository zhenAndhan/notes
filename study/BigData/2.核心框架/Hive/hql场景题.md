# Hive（HQL）经典面试题

# 一、连续问题

如下数据为蚂蚁森林中用户领取的减少碳排放量

```sql
id  dt  lowcarbon
1001    2021-12-12  123
1002    2021-12-12  45
1001    2021-12-13  43
1001    2021-12-13  45
1001    2021-12-13  23
1002    2021-12-14  45
1001    2021-12-14  230
1002    2021-12-15  45
1001    2021-12-15  23
… …
```

找出连续 **3** 天及以上减少碳排放量在 100 以上的用户

```sql
 1)按照用户ID及时间字段分组，计算每个用户单日减少的碳排放量
select
    id,
    dt,
    sum(lowcarbon) lowcarbon
from test1
group by id,dt
having lowcarbon>100; t1

等差数列法：两个等差数列如果等差相同，则相同位置的数据相减得到的结果相同
2)按照用户分组，同时按照时间排序，计算每条数据的Rank值
select
    id,
    dt,
    lowcarbon,
    rank() over(partition by id order by dt) rk
from t1; t2

3)将每行数据中的日期减去Rank值
select
    id,
    dt,
    lowcarbon,
    date_sub(dt,rk) flag
from t2;t3

4)按照用户及flag分组，求每个组有多少条数据，并找出大于等于3条的数据
select
    id,
    flag,
    count(*) ct
from t3
group by id,flag
having ct>=3;

5)最终HQL
select
    id,
    flag,
    count(*) ct
from 
(select
    id,
    dt,
    lowcarbon,
    date_sub(dt,rk) flag
from 
(select
    id,
    dt,
    lowcarbon,
    rank() over(partition by id order by dt) rk
from 
(select
    id,
    dt,
    sum(lowcarbon) lowcarbon
from test1
group by id,dt
having lowcarbon>100)t1)t2)t3
group by id,flag
having ct>=3;
```

# 二、分组问题

如下为电商公司用户访问时间数据

```sql
id  ts(秒)
1001    17523641234
1001    17523641256
1002    17523641278
1001    17523641334
1002    17523641434
1001    17523641534
1001    17523641544
1002    17523641634
1001    17523641638
1001    17523641654
```

某个用户连续的访问记录如果时间间隔小于 **60** 秒，则分为同一个组，结果为：

```sql
id  ts(秒)   group
1001    17523641234 1
1001    17523641256 1
1001    17523641334 2
1001    17523641534 3
1001    17523641544 3
1001    17523641638 4
1001    17523641654 4
1002    17523641278 1
1002    17523641434 2
1002    17523641634 3
```

```sql
1)将上一行时间数据下移
lead:领导
lag:延迟
select
    id,
    ts,
    lag(ts,1,0) over(partition by id order by ts) lagts
from 
    test2;t1
1001    17523641234 0
1001    17523641256 17523641234
1001    17523641334 17523641256
1001    17523641534 17523641334
1001    17523641544 17523641534
1001    17523641638 17523641544
1001    17523641654 17523641638
1002    17523641278 0
1002    17523641434 17523641278
1002    17523641634 17523641434

2)将当前行数据减去上一行时间数据
select
    id,
    ts,
    ts-lagts tsdiff
from 
    t1;t2
1001    17523641234 17523641234
1001    17523641256 22
1001    17523641334 78
1001    17523641534 200
1001    17523641544 10
1001    17523641638 94
1001    17523641654 16
1002    17523641278 17523641278
1002    17523641434 156
1002    17523641634 200

3)计算每个用户范围内从第一行到当前行tsdiff大于等于60的总个数(分组号)
select
    id,
    ts,
    sum(if(tsdiff>=60,1,0)) over(partiton by order by ts) groupid
from 
    t2;

4)最终HQL
select
    id,
    ts,
    sum(if(tsdiff>=60,1,0)) over(partiton by order by ts) groupid
from 
    (select
    id,
    ts,
    ts-lagts tsdiff
from 
    select
    id,
    ts,
    lag(ts,1,0) over(partition by id order by ts) lagts
from 
    test2)t1)t2;
```

# 三、间隔连续问题

某游戏公司记录的用户每日登录数据

```sql
id  dt
1001    2021-12-12
1002    2021-12-12
1001    2021-12-13
1001    2021-12-14
1001    2021-12-16
1002    2021-12-16
1001    2021-12-19
1002    2021-12-17
1001    2021-12-20
```

计算每个用户最大的连续登录天数，可以间隔一天。解释：如果一个用户在 1,3,5,6 登 录游戏，则视为连续 **6** 天登录。

```sql
结果
1001    5
1002    2


思路一：等差数列
1001    2021-12-12  1
1001    2021-12-13  2
1001    2021-12-14  3
1001    2021-12-16  4
1001    2021-12-19  5
1001    2021-12-20  6

1001    2021-12-12  1   2021-12-11
1001    2021-12-13  2   2021-12-11
1001    2021-12-14  3   2021-12-11
1001    2021-12-16  4   2021-12-12
1001    2021-12-19  5   2021-12-14
1001    2021-12-20  6   2021-12-14

1001    2021-12-11  3
1001    2021-12-12  1
1001    2021-12-14  1

1001    2021-12-11  3   1   2021-12-10
1001    2021-12-12  1   2   2021-12-10
1001    2021-12-14  1   3   2021-12-11


思路二：分组
1001    2021-12-12
1001    2021-12-13
1001    2021-12-14
1001    2021-12-16
1001    2021-12-19
1001    2021-12-20

1)将上一行时间数据下移
1001    2021-12-12  1970-01-01
1001    2021-12-13  2021-12-12
1001    2021-12-14  2021-12-13
1001    2021-12-16  2021-12-14
1001    2021-12-19  2021-12-16
1001    2021-12-20  2021-12-19
select
	id,
	dt,
	lag(dt,1,'1970-01-01') over(partition by id order by dt) lagdt
from
	test3;t1

2)将当前行时间减去上一行时间数据(datediff(dt1,dt2))
1001    2021-12-12  18973
1001    2021-12-13  1
1001    2021-12-14  1
1001    2021-12-16  2
1001    2021-12-19  3
1001    2021-12-20  1
select
	id,
	dt,
	datediff(dt,lagdt) flag
from
	t1;t2

3)按照用户分组，同时按照时间排序，计算从第一行到当前行大于2的数据的总条数(sum(if(flag>2,1,0)))
1001    2021-12-12  1
1001    2021-12-13  1
1001    2021-12-14  1
1001    2021-12-16  1
1001    2021-12-19  2
1001    2021-12-20  2
select
	id,
	dt,
	sum(if(flag>2,1,0)) over(partition by id order by dt) flag
from
	t2;t3

4)按照用户和flag分组，求最大时间减去最小时间并加上1
select
	id,
	flag,
	datediff(max(dt),min(dt)) days
from
	t3
group by id,flag;t4


5)取连续登录天数的最大值
select
	id,
	max(days)+1
from
	t4
group by id;

6)最终HQL
select
	id,
	max(days)+1
from
	(select
	id,
	flag,
	datediff(max(dt),min(dt)) days
from
	(select
	id,
	dt,
	sum(if(flag>2,1,0)) over(partition by id order by dt) flag
from
	(select
	id,
	dt,
	datediff(dt,lagdt) flag
from
	(select
	id,
	dt,
	lag(dt,1,'1970-01-01') over(partition by id order by dt) lagdt
from
	test3)t1)t2)t3
group by id,flag)t4
group by id;
```

# 四、打折日期交叉问题

如下为平台商品促销数据：字段为品牌，打折开始日期，打折结束日期

```sql
id   stt edt
oppo    2021-06-05  2021-06-09
oppo    2021-06-11  2021-06-21
vivo    2021-06-05  2021-06-15
vivo    2021-06-09  2021-06-21
redmi   2021-06-05  2021-06-21
redmi   2021-06-09  2021-06-15
redmi   2021-06-17  2021-06-26
huawei  2021-06-05  2021-06-26
huawei  2021-06-09  2021-06-15
huawei  2021-06-17  2021-06-21
```

计算每个品牌总的打折销售天数，注意其中的交叉日期，比如 **vivo** 品牌，第一次活动时间为 **2021-06-05** 到 **2021-06-15**，第二次活动时间为 **2021-06-09** 到 **2021-06-21** 其中 **9** 号到 **15** 号为重复天数，只统计一次，即 **vivo** 总打折天数为 **2021-06-05** 到 **2021-06-21** 共计 **17** 天。

```sql
1)将当前行以前的数据中最大的edt放置当前行
select
    id,
    stt,
    edt,
    max(edt) over(partition by id order by stt rows between UNBOUNDED PRECEDING and 1 PRECEDING) maxEdt
from test4;t1
redmi   2021-06-05  2021-06-21  null
redmi   2021-06-09  2021-06-15  2021-06-21
redmi   2021-06-17  2021-06-26  2021-06-21

2)比较开始时间与移动下来的数据，如果开始时间大，则不需要操作，
反之则需要将移动下来的数据加一替换当前行的开始时间
如果是第一行数据，maxEdt为null，则不需要操作
select
    id,
    if(maxEdt is null,stt,if(stt>maxEdt,stt,data_add(maxEdt,1))) stt,
    edt
from t1;t2
redmi   2021-06-05  2021-06-21
redmi   2021-06-22  2021-06-15
redmi   2021-06-22  2021-06-26

3)将每行数据中的结束日期减去开始日期
select
    id,
    datediff(edt,stt) days
from
    t2;t3
redmi   16
redmi   -7
redmi   4

4)按照品牌分组，计算每条数据加一的总和
select
    id,
    sum(if(days>=0,days+1,0)) days
from
    t3
group by id;
redmi   22

5)最终HQL
select
    id,
    sum(if(days>=0,days+1,0)) days
from
    (select
    id,
    datediff(edt,stt) days
from
    (select
    id,
    if(maxEdt is null,stt,if(stt>maxEdt,stt,data_add(maxEdt,1))) stt,
    edt
from (select
    id,
    stt,
    edt,
    max(edt) over(partition by id order by stt rows between UNBOUNDED PRECEDING and 1 PRECEDING) maxEdt
from test4)t1)t2)t3
group by id;
```

# 五、同时在线问题

如下为某直播平台主播开播及关播时间，根据该数据计算出平台最高峰同时在线的主播人数。

```sql
id  stt edt
1001    2021-06-14 12:12:12 2021-06-14 18:12:12
1003    2021-06-14 13:12:12 2021-06-14 16:12:12
1004    2021-06-14 13:15:12 2021-06-14 20:12:12
1002    2021-06-14 15:12:12 2021-06-14 16:12:12
1005    2021-06-14 15:18:12 2021-06-14 20:12:12
1001    2021-06-14 20:12:12 2021-06-14 23:12:12
1006    2021-06-14 21:12:12 2021-06-14 23:15:12
1007    2021-06-14 22:12:12 2021-06-14 23:10:12
… …
```

```sql
流式！

1)对数据分类，在开始数据后添加正1，表示有主播上线，同时在关播数据后添加-1，表示有主播下限
select id,stt dt,1 p from test5
union
select id,edt dt,-1 p from test5;t1

1001    2021-06-14 12:12:12 1
1001    2021-06-14 18:12:12 -1
1001    2021-06-14 20:12:12 1
1001    2021-06-14 23:12:12 -1
1002    2021-06-14 15:12:12 1
1002    2021-06-14 16:12:12 -1
1003    2021-06-14 13:12:12 1
1003    2021-06-14 16:12:12 -1
1004    2021-06-14 13:15:12 1
1004    2021-06-14 20:12:12 -1
1005    2021-06-14 15:18:12 1
1005    2021-06-14 20:12:12 -1
1006    2021-06-14 21:12:12 1
1006    2021-06-14 23:15:12 -1
1007    2021-06-14 22:12:12 1 
1007    2021-06-14 23:10:12 -1

2)按照时间排序，计算累加人数
select
    id,
    dt,
    sum(p) over(order by dt) sum_p
from
    t1;t2

1001    2021-06-14 12:12:12 1
1003    2021-06-14 13:12:12 2
1004    2021-06-14 13:15:12 3
1002    2021-06-14 15:12:12 4
1005    2021-06-14 15:18:12 5
1003    2021-06-14 16:12:12 3
1002    2021-06-14 16:12:12 3
1001    2021-06-14 18:12:12 2
1005    2021-06-14 20:12:12 1
1001    2021-06-14 20:12:12 1
1004    2021-06-14 20:12:12 1
1006    2021-06-14 21:12:12 2
1007    2021-06-14 22:12:12 3
1007    2021-06-14 23:10:12 2
1001    2021-06-14 23:12:12 1
1006    2021-06-14 23:15:12 0

3)找出同时在线人数最大值
select
    max(sum_p)
from
    t2;


4)最终HQL
select
    max(sum_p)
from
    (select
    id,
    dt,
    sum(p) over(order by dt) sum_p
from
    (select id,stt dt,1 p from test5
union
select id,edt dt,-1 p from test5)t1)t2;
```

