@[toc]
## MySQL 调优系列
[MySQL 调优之性能监控](https://blog.csdn.net/AlphaBr/article/details/105752496)

## 执行计划
在企业的应用场景中，为了知道优化 SQL 语句的执行，需要查看 SQL 语句的具体执行过程，以加快 SQL 语句的执行效率。
可以使用 explain+SQL 语句来模拟优化器执行 SQL 查询语句，从而知道 mysql 是如何处理 sql 语句的，**查询有没有走索引**。
**执行计划中包含的信息**
|    Column     |                    Meaning                     |
| :-----------: | :--------------------------------------------: |
|      id       |            The `SELECT` identifier             |
|  select_type  |               The `SELECT` type                |
|     table     |          The table for the output row          |
|  partitions   |            The matching partitions             |
|     type      |                 The join type                  |
| possible_keys |         The possible indexes to choose         |
|      key      |           The index actually chosen            |
|    key_len    |          The length of the chosen key          |
|      ref      |       The columns compared to the index        |
|     rows      |        Estimate of rows to be examined         |
|   filtered    | Percentage of rows filtered by table condition |
|     extra     |             Additional information             |
### id
select 查询的序列号，包含一组数字，表示查询中执行 select 子句或者操作表的顺序。

id 号分为三种情况：
1. 如果 id 相同，那么执行顺序从上到下
	```sql
	explain select * from emp e join dept d on e.deptno = d.deptno join salgrade sg on e.sal between sg.losal and sg.hisal;
	```
	在连接查询的执行计划中，每个表都会对应一条记录，这些记录的 id 列的值是相同的；出现在前面的表表示驱动表，出现在后面的表表示被驱动表。
2. 如果 id 不同，如果是子查询，id 的序号会递增，id 值越大优先级越高，越先被执行。

	```sql
	explain select * from emp e where e.deptno in (select d.deptno from dept d where d.dname = 'SALES');
	```
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200425223609117.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FscGhhQnI=,size_16,color_FFFFFF,t_70)
	查询优化器可能对涉及子查询的查询语句进行重写，从而转换为连接查询（半连接）。
	![在这里插入图片描述](https://img-blog.csdnimg.cn/f7ec8ac488f74f46ac270ab11c94cae7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA56eA5by6,size_20,color_FFFFFF,t_70,g_se,x_16)
	虽然查询语句中包含一个子查询，但是执行计划中 s1 和 s2 表对应的记录的 id 值全部是 1. 这就表明查询优化器将子查询转换为了连接查询。

3. id 相同和不同的，同时存在：相同的可以认为是一组，从上往下顺序执行，在所有组中，id 值越大，优先级越高，越先执行。
4. id 为 null 的情况。比如下面这个查询：
	![在这里插入图片描述](https://img-blog.csdnimg.cn/2f92cf795b7c43cd81efd72daf45dbb0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA56eA5by6,size_20,color_FFFFFF,t_70,g_se,x_16)
UNION 子句会把多个查询的结果集合并起来并对结果集中的记录进行去重。MySQL 使用的是内部临时表，UNION 子句为了把 id 为 1 的查询和 id 为 2 的查询的结果集合并起来并去重，在内部创建了一个名为<union1,2>的临时表。íd 为 NULL 表明这个临时表是为了合并两个查询的结果集而创建。
与 UNlON 比起来，UNlON ALL 就不需要对最终的结果集进行去重。它只是单纯地把多个查询结果集中的记录合并成一个并返回给用户，所以也就不需要使用临时表。所以在包含 UNlON ALL 子句的查询的执行计划中，就没有那个íd 为 NULL 的记录，如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/3e483dfea01448349acd31bc5de6d913.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA56eA5by6,size_20,color_FFFFFF,t_70,g_se,x_16)

### select_type
主要用来分辨查询的类型，是普通查询还是联合查询还是子查询

| `select_type` Value  |                           Meaning                            |
| :------------------: | :----------------------------------------------------------: |
|        SIMPLE        |        Simple SELECT (not using UNION or subqueries)         |
|       PRIMARY        |                       Outermost SELECT                       |
|        UNION         |         Second or later SELECT statement in a UNION          |
|   DEPENDENT UNION    | Second or later SELECT statement in a UNION, dependent on outer query |
|     UNION RESULT     |                      Result of a UNION.                      |
|       SUBQUERY       |                   First SELECT in subquery                   |
|  DEPENDENT SUBQUERY  |      First SELECT in subquery, dependent on outer query      |
|       DERIVED        |                        Derived table                         |
| UNCACHEABLE SUBQUERY | A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query |
|  UNCACHEABLE UNION   | The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY) |
- simple: 简单的查询，不包含子查询和 union
	```sql
	explain select * from emp;
	```
- primary: 查询中若包含任何复杂的子查询，最外层查询则被标记为 Primary
	```sql
	explain select staname,ename supname from (select ename staname,mgr from emp) t join emp on t.mgr=emp.empno ;
	```
- union: 若第二个 select 出现在 union 之后，则被标记为 union
	```sql
	explain select * from emp where deptno = 10 union select * from emp where sal >2000;
	```
- dependent union: 跟 union 类似，此处的 depentent 表示 union 或 union all 联合而成的结果会受外部表影响
	```sql
	explain select * from emp e where e.empno  in ( select empno from emp where deptno = 10 union select empno from emp where sal >2000)
	```
- union result: 从 union 表获取结果的 select

	```sql
	explain select * from emp where deptno = 10 union select * from emp where sal >2000;
	```

- subquery: 在 select 或者 where 列表中包含子查询

	```sql
	explain select * from emp where sal > (select avg(sal) from emp) ;
	```

- dependent subquery:subquery 的子查询要受到外部表查询的影响

	```sql
	explain select * from emp e where e.deptno in (select distinct deptno from dept);
	```

- DERIVED: from 子句中出现的子查询，也叫做派生类，

	```sql
	explain select staname,ename supname from (select ename staname,mgr from emp) t join emp on t.mgr=emp.empno;
	```

- UNCACHEABLE SUBQUERY：表示使用子查询的结果不能被缓存

	```sql
	explain select * from emp where empno = (select empno from emp where deptno=@@sort_buffer_size);
	```
- uncacheable union: 表示 union 的查询结果不能被缓存

### table

对应行正在访问哪一个表，表名或者别名，可能是临时表或者 union 合并结果集
1. 如果是具体的表名，则表名从实际的物理表中获取数据，当然也可以是表的别名
2. 表名是 derivedN 的形式，表示使用了 id 为 N 的查询产生的衍生表
3. 当有 union result 的时候，表名是 union n1,n2 等的形式，n1,n2 表示参与 union 的 id
### type
type 显示的是访问类型，访问类型表示我是以何种方式去访问我们的数据，最容易想的是全表扫描，直接暴力的遍历一张表去寻找需要的数据，效率非常低下，访问的类型有很多，效率从最好到最坏依次是：
system > const > eq_ref > ***ref*** > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > ***range*** > index > ALL
**一般情况下，得保证查询至少达到 range 级别，最好能达到 ref**
#### all
全表扫描，一般情况下出现这样的 sql 语句而且数据量比较大的话那么就需要进行优化。

```sql
	explain select * from emp;
```
#### index
全索引扫描这个比 all 的效率要好，主要有两种情况，一种是当前的查询时**覆盖索引**，即我们需要的数据在索引中就可以索取，或者是使用了索引进行**排序**，这样就避免数据的重排序。
```sql
	explain  select empno from emp;
```
#### range
使用索引执行查询时，对应的扫描区间为若干个单点扫描区间或者范围扫描区间的访问方法称为 range （仅包含一个单点扫描区间的访问方法不能称为 range 访问方法，扫描区间为 (-∞，+∞) 的访问方法也不能称为 range 访问方法）。在指定范围内进行查询，这样避免了 index 的全索引扫描，适用的操作符： =, <>, >, >=, <, <=, IS NULL, BETWEEN, LIKE, IN() 。
```sql
	explain select * from emp where empno between 7000 and 7500;
```

#### index_subquery
利用索引来关联子查询，不再扫描全表。

```sql
	explain select * from emp where emp.job in (select job from t_job);
```

#### unique_subquery
该连接类型类似与 index_subquery，使用的是唯一索引。

```sql
	 explain select * from emp e where e.deptno in (select distinct deptno from dept);
```

 
#### index_merge
在查询过程中需要多个索引组合使用

#### ref_or_null
不仅想找出某个二级索引列的值等于某个常数的记录，而且还想把该列中值为 NULL 的记录也找出来，比如下面这个查询：
```sql
	explain select * from emp e where  e.mgr is null or e.mgr=7369;
```
当使用二级索引而不是全表扫描的方式执行该查询时，对应的扫描区间就是 [NULL，NULL]
以及 ['abc'， 'abc'] ，此时执行这种类型的查询所使用的访问方法就称为 ref_or_null。可以看到 ref_or_null 访问方法只是比 ref 访问方法多扫描了一些值为 NULL 的二级索引记录（值为 NULL  的记录会被放在索引的最左边）。
#### ref
搜索条件为二级索引列与常数进行等值比较，形成的扫描区间为单点扫描区间， 采用二级索引来执行查询的访问方法称为 ref。
```sql
	 SELECT * FROH xxx_table WHERE key1 = 'abc';
```
对于普通的二级索引来说，通过索引列进行等值比较后可能会匹配到多条连续的二级索引记录，而不是像主键或者唯一二级索引那样最多只能匹配一条记录。所以这种 ref 访问方法比 const 差了那么一点。
- 在二级索引列允许存储 NULL 值时，无论是普通的二级索引，还是唯一二级索引，它
们的索引列并不限制 NULL 值的数量，所以在执行包含 "key IS NULL" 形式的搜索条件的查询时，最多只能使用 ref 访问方法， 而不能使用 const 访问方法。
- 对于索引列中包含多个列的二级索引来说，只要最左边连续的列是与常数进行等值比较，就可以采用 ref 访问方法。如下所示：
```sql
		SELECT * FROM single_table WHERE key_part1 = 'AAA';
		SELECT * FROM single_table WHERE key_part1 = 'AAA' and key_part2 = 'BBB';
		SELECT * FROM single_table WHERE key_part1 = 'AAA' and key_part2 = 'BBB' and key_part3 = 'CCC';
```
如果索引列中最左边连续的列不全部是等值比较的话，它的访问方法就不能称为 ref 了。
```sql
		SELECT * FROM single_table WHERE key_part1 = 'AAA' and key_part2 > 'BBB';
```
#### eq_ref 
用于联表查询的状况，按联表的主键或惟一键联合查询。
```sql
	explain select * from emp,emp2 where emp.empno = emp2.empno;
```
#### const
通过**主键或者唯一二级索引列来定位一条记录**的访问方法定义为 const（意思是常数级别的， 代价是可以忽略不计的） 。不过这种 const 访问方法只能在主键列或者唯一二级索引列与一个常数进行等值比较时才有效。如果主键或者唯一二级索引的索引列由多个列构成，则只有在索引列中的每一个列都与常数进行等值比较时，这个 const 访问方法才有效（这是因为只有在该索引的每一个列都采用等值比较时， 才可以保证**最多只有一条记录符合搜索条件**)。
对于唯一二级索引列来说， 在查询列为 NULL 值时， 情况比较特殊。比如下面这样：
```sql
	SELECT * FROM single table WHERE key2 IS NULL；
```
因为唯一二级索引列并不限制 NULL 值的数量， 所以上述语句可能访问到多条记录。 
#### system
表只有一行记录（等于系统表），这是 const 类型的特例，平时不会出现。
 ### possible_keys
显示可能应用在这张表中的索引，一个或多个，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询实际使用

```sql
explain select * from emp,dept where emp.deptno = dept.deptno and emp.deptno = 10;
```

### key
实际使用的索引，如果为 null，则没有使用索引，查询中若使用了覆盖索引，则该索引和查询的 select 字段重叠。

```sql
explain select * from emp,dept where emp.deptno = dept.deptno and emp.deptno = 10;
```

### key_len
表示索引中使用的字节数，可以通过 key_len 计算查询中使用的索引长度，在不损失精度的情况下长度越短越好。

```sql
explain select * from emp,dept where emp.deptno = dept.deptno and emp.deptno = 10;
```

### ref
显示索引的哪一列被使用了，如果可能的话，是一个常数

```sql
explain select * from emp,dept where emp.deptno = dept.deptno and emp.deptno = 10;
```

### rows

根据表的统计信息及索引使用情况，大致估算出找出所需记录需要读取的行数，此参数很重要，直接反应的 sql 找了多少数据，在完成目的的情况下越少越好

```sql
explain select * from emp;
```

### extra

包含额外的信息。

- using filesort: 说明 mysql 无法利用索引进行排序，只能利用排序算法进行排序，会消耗额外的位置

	```sql
	explain select * from emp order by sal;
	```

- using temporary: 建立临时表来保存中间结果，查询完成之后把临时表删除

	```sql
	explain select ename,count(*) from emp where deptno = 10 group by ename;
	```

- using index: 这个表示当前的查询时覆盖索引的，直接从索引中读取数据，而不用访问数据表。如果同时出现 using where 表名索引被用来执行索引键值的查找，如果没有，表面索引被用来读取数据，而不是真的查找

	```sql
	explain select deptno,count(*) from emp group by deptno limit 10;
	```

- using where: 使用 where 进行条件过滤

	```sql
	explain select * from t_user where id = 1;
	```

- using join buffer: 使用连接缓存

- impossible where：where 语句的结果总是 false

	```sql
	explain select * from emp where empno = 7469;
	```

## 参考资料
- MySQL 官网
	- [EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- 《MySQL 是怎样运行的》
 
