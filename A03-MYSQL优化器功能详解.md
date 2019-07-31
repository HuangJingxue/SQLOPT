 # A03-MYSQL优化器功能详解

**MySQL 8.0新增特性**

```wiki
use_invisible_indexes
	是否使用不可见索引，MySQL 8.0新增可以创建invisible索引，这一开关控制优化器是否使用invisible索引，on表示考虑使用。
```

**MySQL 5.7新增**

```wiki
derived_merge
	派生表合并，类似Oracle的视图合并，当派生SQL中存在以下操作是无法展开UNION 、GROUP 、DISTINCT、LIMIT及聚合操作
duplicateweedout
	是否使用使用临时表对semi-join产生的结果集去重
condition_fanout_filter	
	cost模型在jion 代价计算时考虑condition,是否还要还要考虑condition上的filter，如果是on表示考虑
```

**MySQL 5.6 新增**

```wiki
mrr和mrr_cost_based
       针对多列索引，也叫组合索引来做基本扫描，然后对匹配的记录按照主键排序，这样按照有序的主键顺序从磁盘上扫描需要的全部记录。根本功能是把对磁盘的随机扫描转化为顺序扫描。
batched_key_access
	对于多表join语句，当MySQL使用索引访问第二个join表的时候，使用一个join buffer来收集第一个操作对象生成的相关列值。BKA构建好key后，批量传给引擎层做索引查找。key是通过MRR接口提交给引擎的. 这样，MRR使得查询更有效率。
block_nested_loop
	将外层循环的行/结果集存入join buffer, 内层循环的每一行与整个buffer中的记录做比较，从而减少内层循环的次数.
index_condition_pushdown
	当ICP打开时，用于二级索引的range、 ref、 eq_ref或ref_or_null扫描，如果部分where条件能使用索引的字段，MySQL server会把这部分下推到引擎层，可以利用index过滤的where条件在存储引擎层进行数据过滤。
use_index_extensions
	索引扩展使用，主要用于INNODB的第二索引，也就是普通的索引,把索引中包含的主键值利用到。比如主键为(a,b),索引为(c). 如果用到了索引c，那么把索引变成(c,a,b) 这样，就可以用到新的组合索引了。不过这种场合用的也比较少，一般是根据组合主键中的第一个字段和普通索引一起来做检索的时候。
semijoin
	是否启用semijoin，MySQL主要支持以下五种半连接策略
		DuplicateWeedout: 使用临时表对semi-join产生的结果集去重。
		FirstMatch: 只选用内部表的第1条与外表匹配的记录。
		LooseScan: 把inner-table数据基于索引进行分组，取每组第一条数据进行匹配。
		Materializelookup: 将inner-table去重固化成临时表，遍历outer-table，然后在固化表上去寻找匹配。
		MaterializeScan: 将inner-table去重固化成临时表，遍历固化表，然后在outer-table上寻找匹配。
firstmatch
	只选用内表的第一条与外表匹配的记录。
loosescan
	把内表的数据基于索引分组，取每组第一条数据即可。
materialization和subquery_materialization_cost_based
	把内表去重然后生成有对应索引的临时表（有点类似其他数据中的物化视图），然后通过外表的对应键值遍历这张临时表。
```

**MySQL 5.5及以下版本新增**

```wiki
engine_condition_pushdown 
	只用于NDB引擎，开启后时按照WHERE条件过滤后的数据发送到SQL节点来处理，不开启所有数据节点的数据都发送到SQL节点来处理。
index_merge 
	index_merge_intersection
		如果有两个单独的索引都可用，但是其中任何一个都不是最优化的，那么优化器选择合并两个索引并且在他俩的结果集中做一个交集，然后根据这个交集对磁盘数据进行匹配。
	index_merge_union
		用于OR，把所有相关索引连接起来，找到记录对应的ROWID，然后根据ROWID获取磁盘上的数据。
	index_merge_sort_union
		用于OR，把所有相关索引连接起来，找到记录对应的ROWID，并且好顺序，然后根据ROWID获取磁盘上的数据。

```


参考文档(除官档外)：	
http://blog.sina.com.cn/s/blog_4673e60301011qvx.html  mysql中semi-join的优化策略介绍
http://ourmysql.com/archives/1225   MySQL 5.6里坑人的index_condition_pushdown  