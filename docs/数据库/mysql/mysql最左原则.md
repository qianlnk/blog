# 最左原则

最左匹配原则就是指在联合索引中，如果你的 SQL 语句中用到了联合索引中的最左边的索引，那么这条 SQL 语句就可以利用这个联合索引去进行匹配。例如某表现有索引(a,b,c)，现在你有如下语句：

```sql
select * from t where a=1 and b=1 and c =1;     #这样可以利用到定义的索引（a,b,c）,用上a,b,c

select * from t where a=1 and b=1;     #这样可以利用到定义的索引（a,b,c）,用上a,b

select * from t where b=1 and a=1;     #这样可以利用到定义的索引（a,b,c）,用上a,b（mysql有查询优化器）

select * from t where a=1;     #这样也可以利用到定义的索引（a,b,c）,用上a

select * from t where b=1 and c=1;     #这样不可以利用到定义的索引（a,b,c）

select * from t where a=1 and c=1;     #这样可以利用到定义的索引（a,b,c），但只用上a索引，b,c索引用不到
```

## 为什么要使用联合索引

- 1、减少开销。

    建一个联合索引(col1,col2,col3)，实际相当于建了(col1),(col1,col2),(col1,col2,col3)三个索引。每多一个索引，都会增加写操作的开销和磁盘空间的开销。对于大量数据的表，使用联合索引会大大的减少开销！

- 2、覆盖索引。

    对联合索引(col1,col2,col3)，如果有如下的sql: select col1,col2,col3 from test where col1=1 and col2=2。那么MySQL可以直接通过遍历索引取得数据，而无需回表，这减少了很多的随机io操作。减少io操作，特别的随机io其实是dba主要的优化策略。所以，在真正的实际应用中，覆盖索引是主要的提升性能的优化手段之一。

- 3、效率高。
    
    索引列越多，通过索引筛选出的数据越少。有1000W条数据的表，有如下sql:select from table where col1=1 and col2=2 and col3=3,假设假设每个条件可以筛选出10%的数据，如果只有单值索引，那么通过该索引能筛选出1000W10%=100w条数据，然后再回表从100w条数据中找到符合col2=2 and col3= 3的数据，然后再排序，再分页；如果是联合索引，通过索引筛选出1000w10% 10% *10%=1w，效率提升可想而知！

 

## 使用索引优化查询问题

- 1、创建单列索引还是多列索引？

    如果查询语句中的where、order by、group 涉及多个字段，一般需要创建多列索引，比如：

    ```sql
    select * from user where nick_name = 'ligoudan' and job = 'dog';
    ```

- 2、多列索引的顺序如何选择？

    一般情况下，把选择性高德字段放在前面，比如查询sql：
    
    ```sql
    select * from user where age = '20' and name = 'zh' order by nick_name;
    ```
    
    这时候如果建索引的话，首字段应该是age，因为age定位到的数据更少，选择性更高。
    
    但是务必注意一点，满足了某个查询场景就可能导致另外一个查询场景更慢。

- 3、避免使用范围查询

    很多情况下，范围查询都可能导致无法使用索引。

- 4、尽量避免查询不需要的数据

    ```sql
    explain select * from user where job like '%ligoudan%';
    explain select job from user where job like '%ligoudan%';
    ```

    同样的查询，不同的返回值，第二个就可以使用覆盖索引，第一个只能全表遍历了。

- 5、查询的数据类型要正确

    ```sql
    explain select * from user where create_date >= now();
    explain select * from user where create_date >= '2020-05-01 00:00:00';
    ```

    第一条语句就可以使用create_date的索引，第二个就不可以。