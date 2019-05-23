# SQL优化经典案例（一）

## 前生今世

一天某客户负责人说目前查询商城的搜索接口中主要的一条sql非常慢，在生产上的执行时间需要134s，完全相同的数据在开发环境执行只需要3.5s。

上周五将生产环境的product库删除，通过.sql文件（原生产实例中导出的product库）的备份重新导入数据库当中，并在product库中添加了3万商品。

本周开始查询较慢。

## 慢SQL信息

### 自建MySQL详情

```shell
服务器信息：

IP：192.168.20.1

MySQL版本：

[root@data-20-1 ~]# mysql -V

mysql Ver 14.14 Distrib 5.7.17, for linux-glibc2.5 (x86_64) using EditLine wrapper
```

### SQL详情

> 使用MySQL Workbench将慢SQL格式化，便于后续的拆分。

```sql
SELECT 
    su.id,
    su.merchant_product_serial_number,
    su.merchant_id,
    su.standard_product_unit_id,
    su.name,
    su.sale_price,
    su.demo_sale_price,
    su.competing_sale_price,
    su.market_price,
    su.status,
    su.sold_base,
    su.store_id,
    su.stock_warning,
    su.sale_way,
    su.buy_type,
    su.front_serial_number,
    spu.commodity_template_id,
    (SELECT 
            p.url
        FROM
            picture p,
            merchant_picture mp,
            standard_unit_picture sup
        WHERE
            mp.picture_id = p.id
                AND sup.merchant_picture_id = mp.id
                AND sup.type = 1
                AND sup.standard_unit_id = su.id) picture_url,
    sales_volume
FROM
    standard_unit su,
    standard_product_unit spu,
    (SELECT 
        su.id,
            (SELECT 
                    SUM(sales_volume)
                FROM
                    merchant_prod_sales_record
                WHERE
                    standard_unit_id = su.id) sales_volume
    FROM
        standard_unit su
    LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
    LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
    LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
    LEFT JOIN merchant m ON su.merchant_id = m.id
    WHERE
        m.is_stop = 0 AND su.is_vendible = 0
            AND su.is_visible = 0
            AND suc.client_id = 2
            AND sucy.company_type = 0
            AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
            OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
            OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
            OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
            AND su.platform_id = 7
    GROUP BY su.id
    ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '')
    LIMIT 0 , 10) sus
WHERE
    sus.id = su.id
        AND su.standard_product_unit_id = spu.id
```

### SQL主框架

> SQL简化是为了提炼出复杂SQL语句的框架，便于查看SQL语句的结构。将子查询拿出，使用别名替换。

```sql
SQL主框架
SELECT 
    su.id,
    su.merchant_product_serial_number,
    su.merchant_id,
    su.standard_product_unit_id,
    su.name,
    su.sale_price,
    su.demo_sale_price,
    su.competing_sale_price,
    su.market_price,
    su.status,
    su.sold_base,
    su.store_id,
    su.stock_warning,
    su.sale_way,
    su.buy_type,
    su.front_serial_number,
    spu.commodity_template_id,
	picture_url,
    sales_volume
FROM
    standard_unit su,
    standard_product_unit spu,
	sus
WHERE
    sus.id = su.id
    AND su.standard_product_unit_id = spu.id;
```


### 子查询详情

#### 子查询(一[picture_url])

```sql
SELECT 
	p.url
FROM
	picture p,
	merchant_picture mp,
	standard_unit_picture sup
WHERE
	mp.picture_id = p.id
		AND sup.merchant_picture_id = mp.id
		AND sup.type = 1
		AND sup.standard_unit_id = su.id;
```

#### 子查询(二[sus])

```sql
SELECT 
	su.id,
		(SELECT 
				SUM(sales_volume)
			FROM
				merchant_prod_sales_record
			WHERE
				standard_unit_id = su.id) sales_volume
FROM
	standard_unit su
LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
LEFT JOIN merchant m ON su.merchant_id = m.id
WHERE
	m.is_stop = 0 AND su.is_vendible = 0
		AND su.is_visible = 0
		AND suc.client_id = 2
		AND sucy.company_type = 0
		AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
		AND su.platform_id = 7
GROUP BY su.id
ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '')
LIMIT 0 , 10;
```

### 问题定位

#### 执行子查询一

```sql
SELECT 
	p.url
FROM
	standard_unit su,
	picture p,
	merchant_picture mp,
	standard_unit_picture sup
WHERE
	mp.picture_id = p.id
	AND sup.merchant_picture_id = mp.id
	AND sup.type = 1
	AND sup.standard_unit_id = su.id;
	
返回结果

......
| https://image.51fugj.com/20180805094135egeo155002                                                           |
| https://image.51fugj.com/20181203174537egeo172221                                                           |
| https://image.51fugj.com/20190318143052egeo140152                                                           |
| https://image.51fugj.com/20190318142120egeo140152                                                           |
| https://image.51fugj.com/20190520103800egeo133212                                                           |
| https://image.51fugj.com/20190520104159egeo140152                                                           |
| http://img13.360buyimg.com/n1/s375x375_jfs/t1/26949/5/4474/251411/5c331144E0d399ecc/89272589d46b2287.jpg    |
| http://img13.360buyimg.com/n1/s375x375_jfs/t25354/48/157553194/95232/d843fd42/5b67c666N0f910156.jpg         |
+-------------------------------------------------------------------------------------------------------------+
33675 rows in set (0.28 sec)
```

#### 执行子查询二

```sql
SELECT 
	su.id,
		(SELECT 
				SUM(sales_volume)
			FROM
				merchant_prod_sales_record
			WHERE
				standard_unit_id = su.id) sales_volume
FROM
	standard_unit su
LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
LEFT JOIN merchant m ON su.merchant_id = m.id
WHERE
	m.is_stop = 0 AND su.is_vendible = 0
		AND su.is_visible = 0
		AND suc.client_id = 2
		AND sucy.company_type = 0
		AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
		AND su.platform_id = 7
GROUP BY su.id
ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '')
LIMIT 0 , 10;

返回结果

+-------+--------------+
| id    | sales_volume |
+-------+--------------+
|   379 |         NULL |
|   378 |         NULL |
| 19943 |         NULL |
| 25601 |         NULL |
| 25576 |         NULL |
| 15924 |         NULL |
| 32206 |         NULL |
| 32203 |         NULL |
| 32212 |         NULL |
| 32196 |         NULL |
+-------+--------------+
10 rows in set (2min 33sec)


```

> **可以看到第二个子查询的执行时间为2min 33s，基本等于整个SQL语句的执行时间，所以问题可以定位到第二个子查询。**



### 定位问题点

#### 第一步：将各个表单独执行

```sql
--跟su相关的：

SELECT 
	su.id
FROM
	standard_unit su
WHERE
	su.is_vendible = 0
	AND su.is_visible = 0
	AND su.platform_id = 7
GROUP BY su.id
LIMIT 0 , 10;

mysql> SELECT 
    -> su.id
    -> FROM
    -> standard_unit su
    -> WHERE
    -> su.is_vendible = 0
    -> AND su.is_visible = 0
    -> AND su.platform_id = 7
    -> GROUP BY su.id
    -> LIMIT 0 , 10;
+-----+
| id  |
+-----+
| 316 |
| 317 |
| 318 |
| 319 |
| 320 |
| 321 |
| 322 |
| 323 |
| 338 |
| 340 |
+-----+
10 rows in set (0.00 sec)

--跟sales_volume相关：

mysql> SELECT 
    -> SUM(sales_volume)
    -> FROM
    -> merchant_prod_sales_record a,standard_unit b
    -> WHERE
    -> a.standard_unit_id = b.id;
+-------------------+
| SUM(sales_volume) |
+-------------------+
|              1614 |
+-------------------+
1 row in set (0.01 sec)

--跟suc相关：

mysql> SELECT 
    -> su.id
    -> FROM
    -> standard_unit su
    -> LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
    -> WHERE
    -> suc.client_id = 2
    -> GROUP BY su.id
    -> LIMIT 0 , 10;
+-----+
| id  |
+-----+
| 316 |
| 317 |
| 318 |
| 319 |
| 320 |
| 321 |
| 322 |
| 323 |
| 324 |
| 325 |
+-----+
10 rows in set (0.13 sec)

--跟susr相关：

mysql> SELECT 
    -> su.id
    -> FROM
    -> standard_unit su
    -> LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
    -> WHERE
    -> (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
    -> GROUP BY su.id
    -> ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '') 
    -> LIMIT 0 , 10;
+-------+
| id    |
+-------+
|   426 |
|   379 |
|   378 |
| 19943 |
|   771 |
| 25601 |
| 25576 |
| 15924 |
| 32206 |
| 32203 |
+-------+
10 rows in set (0.11 sec)

--跟sucy相关：

mysql> SELECT 
    -> su.id
    -> FROM
    -> standard_unit su
    -> LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
    -> WHERE
    -> sucy.company_type = 0
    -> GROUP BY su.id
    -> LIMIT 0 , 10;
+-----+
| id  |
+-----+
| 316 |
| 317 |
| 319 |
| 320 |
| 321 |
| 322 |
| 323 |
| 324 |
| 325 |
| 326 |
+-----+
10 rows in set (0.13 sec)

跟m相关：

mysql> SELECT  
    -> su.id
    -> FROM
    -> standard_unit  su
    -> LEFT  JOIN  merchant  m  ON  su.merchant_id  =  m.id
    -> WHERE
    -> m.is_stop  =  0  AND  su.is_vendible  =  0
    -> GROUP  BY  su.id
    -> LIMIT  0  ,  10;
+-----+
| id  |
+-----+
| 316 |
| 317 |
| 318 |
| 319 |
| 320 |
| 321 |
| 322 |
| 323 |
| 324 |
| 325 |
+-----+
10 rows in set (0.09 sec)

```

> *发现各个表单独执行时，执行都很快。*

#### 第二步：将表关联逐步加入

```sql
(1)su、susr
SELECT 
	su.id,
		(SELECT 
				SUM(sales_volume)
			FROM
				merchant_prod_sales_record
			WHERE
				standard_unit_id = su.id) sales_volume
FROM
	standard_unit su
LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
WHERE
		susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%')
GROUP BY su.id
ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '') 
LIMIT 0 , 10;

+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+-------------------------------+-------+----------+----------------------------------------------+
| id | select_type        | table                      | partitions | type   | possible_keys | key     | key_len | ref                           | rows  | filtered | Extra                                        |
+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+-------------------------------+-------+----------+----------------------------------------------+
|  1 | PRIMARY            | susr                       | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                          | 31392 |    37.57 | Using where; Using temporary; Using filesort |
|  1 | PRIMARY            | su                         | NULL       | eq_ref | PRIMARY       | PRIMARY | 8       | product.susr.standard_unit_id |     1 |   100.00 | Using index                                  |
|  2 | DEPENDENT SUBQUERY | merchant_prod_sales_record | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                          |   430 |    10.00 | Using where                                  |
+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+-------------------------------+-------+----------+----------------------------------------------+

(2)su、susr、m
SELECT 
	su.id,
		(SELECT 
				SUM(sales_volume)
			FROM
				merchant_prod_sales_record
			WHERE
				standard_unit_id = su.id) sales_volume
FROM
	standard_unit su
LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
LEFT JOIN merchant m ON su.merchant_id = m.id
WHERE
	m.is_stop = 0 AND su.is_vendible = 0
		AND su.is_visible = 0
		AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
		AND su.platform_id = 7
GROUP BY su.id
ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '') 
LIMIT 0 , 10;

+-------+--------------+
| id    | sales_volume |
+-------+--------------+
|   426 |         NULL |
|   379 |         NULL |
|   378 |         NULL |
| 19943 |         NULL |
|   771 |         NULL |
| 25601 |         NULL |
| 25576 |         NULL |
| 15924 |         NULL |
| 32206 |         NULL |
| 32203 |         NULL |
+-------+--------------+
10 rows in set (1.42 sec)

+----+--------------------+----------------------------+------------+------+---------------+------+---------+------+-------+----------+----------------------------------------------------+
| id | select_type        | table                      | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                                              |
+----+--------------------+----------------------------+------------+------+---------------+------+---------+------+-------+----------+----------------------------------------------------+
|  1 | PRIMARY            | m                          | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |     8 |    12.50 | Using where; Using temporary; Using filesort       |
|  1 | PRIMARY            | su                         | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL | 33017 |     0.01 | Using where; Using join buffer (Block Nested Loop) |
|  1 | PRIMARY            | susr                       | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 31392 |     3.76 | Using where; Using join buffer (Block Nested Loop) |
|  2 | DEPENDENT SUBQUERY | merchant_prod_sales_record | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   430 |    10.00 | Using where                                        |
+----+--------------------+----------------------------+------------+------+---------------+------+---------+------+-------+----------+----------------------------------------------------+

(3)su、susr、m、suc
SELECT 
	su.id,
		(SELECT 
				SUM(sales_volume)
			FROM
				merchant_prod_sales_record
			WHERE
				standard_unit_id = su.id) sales_volume
FROM
	standard_unit su
LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
LEFT JOIN merchant m ON su.merchant_id = m.id
WHERE
	m.is_stop = 0 AND su.is_vendible = 0
		AND su.is_visible = 0
		AND suc.client_id = 2
		AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
		AND su.platform_id = 7
GROUP BY su.id
ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '') 
LIMIT 0 , 10;

+-------+--------------+
| id    | sales_volume |
+-------+--------------+
|   426 |         NULL |
|   379 |         NULL |
|   378 |         NULL |
| 19943 |         NULL |
|   771 |         NULL |
| 25601 |         NULL |
| 25576 |         NULL |
| 15924 |         NULL |
| 32206 |         NULL |
| 32203 |         NULL |
+-------+--------------+
10 rows in set (1.86 sec)

+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+------------------------------+-------+----------+----------------------------------------------------+
| id | select_type        | table                      | partitions | type   | possible_keys | key     | key_len | ref                          | rows  | filtered | Extra                                              |
+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+------------------------------+-------+----------+----------------------------------------------------+
|  1 | PRIMARY            | m                          | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL                         |     8 |    12.50 | Using where; Using temporary; Using filesort       |
|  1 | PRIMARY            | suc                        | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                         | 67309 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
|  1 | PRIMARY            | su                         | NULL       | eq_ref | PRIMARY       | PRIMARY | 8       | product.suc.standard_unit_id |     1 |     5.00 | Using where                                        |
|  1 | PRIMARY            | susr                       | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                         | 31392 |     3.76 | Using where; Using join buffer (Block Nested Loop) |
|  2 | DEPENDENT SUBQUERY | merchant_prod_sales_record | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                         |   430 |    10.00 | Using where                                        |
+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+------------------------------+-------+----------+----------------------------------------------------+

(4)su、susr、m、suc、sucy
SELECT 
	su.id,
		(SELECT 
				SUM(sales_volume)
			FROM
				merchant_prod_sales_record
			WHERE
				standard_unit_id = su.id) sales_volume
FROM
	standard_unit su
LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
LEFT JOIN merchant m ON su.merchant_id = m.id
WHERE
	m.is_stop = 0 AND su.is_vendible = 0
		AND su.is_visible = 0
		AND suc.client_id = 2
		AND sucy.company_type = 0
		AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
		AND su.platform_id = 7
GROUP BY su.id
ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '') 
LIMIT 0 , 10;

+-------+--------------+
| id    | sales_volume |
+-------+--------------+
|   379 |         NULL |
|   378 |         NULL |
| 19943 |         NULL |
| 25601 |         NULL |
| 25576 |         NULL |
| 15924 |         NULL |
| 32206 |         NULL |
| 32203 |         NULL |
| 32212 |         NULL |
| 32196 |         NULL |
+-------+--------------+
10 rows in set (2min 33sec)

+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+------------------------------+-------+----------+----------------------------------------------------+
| id | select_type        | table                      | partitions | type   | possible_keys | key     | key_len | ref                          | rows  | filtered | Extra                                              |
+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+------------------------------+-------+----------+----------------------------------------------------+
|  1 | PRIMARY            | m                          | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL                         |     8 |    12.50 | Using where; Using temporary; Using filesort       |
|  1 | PRIMARY            | suc                        | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                         | 67309 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
|  1 | PRIMARY            | su                         | NULL       | eq_ref | PRIMARY       | PRIMARY | 8       | product.suc.standard_unit_id |     1 |     5.00 | Using where                                        |
|  1 | PRIMARY            | sucy                       | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                         | 97020 |     1.00 | Using where; Using join buffer (Block Nested Loop) |
|  1 | PRIMARY            | susr                       | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                         | 31392 |     3.76 | Using where; Using join buffer (Block Nested Loop) |
|  2 | DEPENDENT SUBQUERY | merchant_prod_sales_record | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                         |   430 |    10.00 | Using where                                        |
+----+--------------------+----------------------------+------------+--------+---------------+---------+---------+------------------------------+-------+----------+----------------------------------------------------+
```

> *可以发现，su、susr、m、suc这4个表进行连接执行时，速度是1.86s，当加入sucy表（standard_unit_company），执行时间为2min 33s。问题就出在standard_unit_company这个表。*

## SQL优化

#### 分析表关联逐步加入导致慢的表

```sql
SELECT 
	su.id,
		(SELECT 
				SUM(sales_volume)
			FROM
				merchant_prod_sales_record
			WHERE
				standard_unit_id = su.id) sales_volume
FROM
	standard_unit su
LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
LEFT JOIN merchant m ON su.merchant_id = m.id
WHERE
	m.is_stop = 0 AND su.is_vendible = 0
		AND su.is_visible = 0
		AND suc.client_id = 2
		AND sucy.company_type = 0
		AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
		OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
		AND su.platform_id = 7
GROUP BY su.id
ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '') 
LIMIT 0 , 10;

```

> join与where条件中与standard_unit_company相关的列为：standard_unit_id 和 company_type
> 查看表standard_unit_company的结构，在standard_unit_id 和 company_type列上均没有索引。

```sql
mysql> show create table standard_unit_company\G;
*************************** 1. row ***************************
       Table: standard_unit_company
Create Table: CREATE TABLE `standard_unit_company` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `standard_unit_id` bigint(20) DEFAULT NULL COMMENT 'suid',
  `company_id` bigint(20) DEFAULT NULL COMMENT '公司id',
  `create_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间:创建记录时数据库会自动set值',
  `company_type` int(3) DEFAULT NULL COMMENT '公司类型 0:正式公司(默认值) 1:测试公司 2:竞品公司',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=104211 DEFAULT CHARSET=utf8 COMMENT='su公司关系表'
1 row in set (0.00 sec)

mysql> select count(distinct(standard_unit_id)) from standard_unit_company;
+-----------------------------------+
| count(distinct(standard_unit_id)) |
+-----------------------------------+
|                             33675 |
+-----------------------------------+
1 row in set (0.11 sec)

mysql> select count(distinct(company_type)) from standard_unit_company;
+-------------------------------+
| count(distinct(company_type)) |
+-------------------------------+
|                             3 |
+-------------------------------+
1 row in set (0.05 sec)

```

*在列standard_unit_id上建立索引。*

#### 添加索引

```sql
mysql> alter table standard_unit_company add index standard_unit_id_index(standard_unit_id);

--查看表standard_unit_company结构

mysql> show create table standard_unit_company\G;
*************************** 1. row ***************************
       Table: standard_unit_company
Create Table: CREATE TABLE `standard_unit_company` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `standard_unit_id` bigint(20) DEFAULT NULL COMMENT 'suid',
  `company_id` bigint(20) DEFAULT NULL COMMENT '公司id',
  `create_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间:创建记录时数据库会自动set值',
  `company_type` int(3) DEFAULT NULL COMMENT '公司类型 0:正式公司(默认值) 1:测试公司 2:竞品公司',
  PRIMARY KEY (`id`),
  KEY `standard_unit_id_index` (`standard_unit_id`)
) ENGINE=InnoDB AUTO_INCREMENT=104211 DEFAULT CHARSET=utf8 COMMENT='su公司关系表'
1 row in set (0.00 sec)

--查看该子查询的执行计划

mysql> explain SELECT 
    -> su.id,
    -> (SELECT 
    -> SUM(sales_volume)
    -> FROM
    -> merchant_prod_sales_record
    -> WHERE
    -> standard_unit_id = su.id) sales_volume
    -> FROM
    -> standard_unit su
    -> LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
    -> LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
    -> LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
    -> LEFT JOIN merchant m ON su.merchant_id = m.id
    -> WHERE
    -> m.is_stop = 0 AND su.is_vendible = 0
    -> AND su.is_visible = 0
    -> AND suc.client_id = 2
    -> AND sucy.company_type = 0
    -> AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
    -> AND su.platform_id = 7
    -> GROUP BY su.id
    -> ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '') 
    -> LIMIT 0 , 10;

+----+--------------------+----------------------------+------------+--------+------------------------+------------------------+---------+------------------------------+-------+----------+----------------------------------------------------+
| id | select_type        | table                      | partitions | type   | possible_keys          | key                    | key_len | ref                          | rows  | filtered | Extra                                              |
+----+--------------------+----------------------------+------------+--------+------------------------+------------------------+---------+------------------------------+-------+----------+----------------------------------------------------+
|  1 | PRIMARY            | m                          | NULL       | ALL    | PRIMARY                | NULL                   | NULL    | NULL                         |     8 |    12.50 | Using where; Using temporary; Using filesort       |
|  1 | PRIMARY            | suc                        | NULL       | ALL    | NULL                   | NULL                   | NULL    | NULL                         | 67309 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
|  1 | PRIMARY            | su                         | NULL       | eq_ref | PRIMARY                | PRIMARY                | 8       | product.suc.standard_unit_id |     1 |     5.00 | Using where                                        |
|  1 | PRIMARY            | sucy                       | NULL       | ref    | standard_unit_id_index | standard_unit_id_index | 9       | product.suc.standard_unit_id |     2 |    10.00 | Using where                                        |
|  1 | PRIMARY            | susr                       | NULL       | ALL    | NULL                   | NULL                   | NULL    | NULL                         | 31392 |     3.76 | Using where; Using join buffer (Block Nested Loop) |
|  2 | DEPENDENT SUBQUERY | merchant_prod_sales_record | NULL       | ALL    | NULL                   | NULL                   | NULL    | NULL                         |   430 |    10.00 | Using where                                        |
+----+--------------------+----------------------------+------------+--------+------------------------+------------------------+---------+------------------------------+-------+----------+----------------------------------------------------+
6 rows in set, 2 warnings (0.00 sec)
```

> 可以看到，对表sucy的查询已经走了standard_unit_id_index索引。

#### 添加索引后执行子查询

```sql


mysql> SELECT 
    -> su.id,
    -> (SELECT 
    -> SUM(sales_volume)
    -> FROM
    -> merchant_prod_sales_record
    -> WHERE
    -> standard_unit_id = su.id) sales_volume
    -> FROM
    -> standard_unit su
    -> LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
    -> LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
    -> LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
    -> LEFT JOIN merchant m ON su.merchant_id = m.id
    -> WHERE
    -> m.is_stop = 0 AND su.is_vendible = 0
    -> AND su.is_visible = 0
    -> AND suc.client_id = 2
    -> AND sucy.company_type = 0
    -> AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
    -> OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
    -> AND su.platform_id = 7
    -> GROUP BY su.id
    -> ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '') 
    -> LIMIT 0 , 10;
+-------+--------------+
| id    | sales_volume |
+-------+--------------+
|   379 |         NULL |
|   378 |         NULL |
| 19943 |         NULL |
| 25601 |         NULL |
| 25576 |         NULL |
| 15924 |         NULL |
| 32206 |         NULL |
| 32203 |         NULL |
| 32212 |         NULL |
| 32196 |         NULL |
+-------+--------------+
10 rows in set (2.55 sec)
```

> *执行时间已经减少至2.55s.*

### 优化后验证

```sql
执行整个SQL语句，进行验证。

mysql> SELECT 
    ->     su.id,
    ->     su.merchant_product_serial_number,
    ->     su.merchant_id,
    ->     su.standard_product_unit_id,
    ->     su.name,
    ->     su.sale_price,
    ->     su.demo_sale_price,
    ->     su.competing_sale_price,
    ->     su.market_price,
    ->     su.status,
    ->     su.sold_base,
    ->     su.store_id,
    ->     su.stock_warning,
    ->     su.sale_way,
    ->     su.buy_type,
    ->     su.front_serial_number,
    ->     spu.commodity_template_id,
    ->     (SELECT 
    ->             p.url
    ->         FROM
    ->             picture p,
    ->             merchant_picture mp,
    ->             standard_unit_picture sup
    ->         WHERE
    ->             mp.picture_id = p.id
    ->                 AND sup.merchant_picture_id = mp.id
    ->                 AND sup.type = 1
    ->                 AND sup.standard_unit_id = su.id) picture_url,
    ->     sales_volume
    -> FROM
    ->     standard_unit su,
    ->     standard_product_unit spu,
    ->     (SELECT 
    ->         su.id,
    ->             (SELECT 
    ->                     SUM(sales_volume)
    ->                 FROM
    ->                     merchant_prod_sales_record
    ->                 WHERE
    ->                     standard_unit_id = su.id) sales_volume
    ->     FROM
    ->         standard_unit su
    ->     LEFT JOIN standard_unit_client suc ON su.id = suc.standard_unit_id
    ->     LEFT JOIN standard_unit_company sucy ON su.id = sucy.standard_unit_id
    ->     LEFT JOIN su_serach_rule susr ON su.id = susr.standard_unit_id
    ->     LEFT JOIN merchant m ON su.merchant_id = m.id
    ->     WHERE
    ->         m.is_stop = 0 AND su.is_vendible = 0
    ->             AND su.is_visible = 0
    ->             AND suc.client_id = 2
    ->             AND sucy.company_type = 0
    ->             AND (susr.standard_unit_name LIKE CONCAT('%', '日本', '%')
    ->             OR susr.standard_unit_keyword LIKE CONCAT('%', '日本', '%')
    ->             OR susr.standard_unit_tag LIKE CONCAT('%', '日本', '%')
    ->             OR susr.standard_unit_category LIKE CONCAT('%', '日本', '%'))
    ->             AND su.platform_id = 7
    ->     GROUP BY su.id
    ->     ORDER BY REPLACE(susr.standard_unit_keyword, '日本', '') , REPLACE(susr.standard_unit_name, '日本', '') , REPLACE(susr.standard_unit_tag, '日本', '') , REPLACE(susr.standard_unit_category, '日本', '')
    ->     LIMIT 0 , 10) sus
    -> WHERE
    ->     sus.id = su.id
    ->         AND su.standard_product_unit_id = spu.id;
+-------+--------------------------------+-------------+--------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+------------+-----------------+----------------------+--------------+--------+-----------+----------+---------------+----------+----------+---------------------+-----------------------+--------------------------------------------------------------------------------------------------------+--------------+
| id    | merchant_product_serial_number | merchant_id | standard_product_unit_id | name                                                                                                                                    | sale_price | demo_sale_price | competing_sale_price | market_price | status | sold_base | store_id | stock_warning | sale_way | buy_type | front_serial_number | commodity_template_id | picture_url                                                                                            | sales_volume |
+-------+--------------------------------+-------------+--------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+------------+-----------------+----------------------+--------------+--------+-----------+----------+---------------+----------+----------+---------------------+-----------------------+--------------------------------------------------------------------------------------------------------+--------------+
|   379 | 04060300000835001              |           1 |                      835 | BALMUDA K01H 巴慕达 日本人气蒸汽烤箱 体验刚出炉的美味复热                                                                               |    2469.05 |         2469.05 |                 0.00 |      2599.00 |      3 |         0 |        1 |             0 |        1 |        1 |              999999 |                     2 | https://image.51fugj.com/20180629155451egeo115846                                                      |         NULL |
|   378 | 04070200000834001              |           1 |                      834 | BALMUDA/巴慕达电风扇落地日本原装进口台式家用静音果岭EGF1680                                                                             |    3134.05 |         3134.05 |                 0.00 |      3299.00 |      3 |         0 |        1 |             0 |        1 |        1 |              999999 |                     2 | https://image.51fugj.com/20180629155541egeo115811                                                      |         NULL |
| 19943 | 01020349001                    |           6 |                    20349 | INTERIGHT 太阳镜女款男士经典蛤蟆偏光镜情侣款复古墨镜潮流驾驶眼镜(C4浅日本金框灰色)                                                      |      90.00 |           90.00 |                90.00 |       168.00 |      3 |         0 |        1 |             0 |        1 |        1 |               99999 |                     2 | http://img13.360buyimg.com/n1/s375x375_jfs/t17692/60/2392888567/368055/9d57a72d/5af25050N7a031998.jpg  |         NULL |
| 25601 | 08026007001                    |           6 |                    26007 | ORBIS奥蜜思动感抗阳防晒露SPF50 40ml（高倍防晒乳霜 防水防汗 清爽保湿 薄透不黏腻）（日本原装进口）                                        |     153.26 |          153.26 |               153.26 |       195.00 |      3 |         0 |        1 |             0 |        1 |        1 |               99999 |                     2 | http://img13.360buyimg.com/n1/s375x375_jfs/t5434/276/2431856474/68825/96794edb/591aaf96Nc36f60d1.jpg   |         NULL |
| 25576 | 07025982001                    |           6 |                    25982 | ORBIS奥蜜思水感澄净卸妆露150ml（无油保湿卸妆水 温和清洁 眼唇可用）（日本原装进口））                                                    |     175.23 |          175.23 |               175.23 |       234.00 |      3 |         0 |        1 |             0 |        1 |        1 |               99999 |                     2 | http://img13.360buyimg.com/n1/s375x375_jfs/t18838/268/2582185248/110928/580e2745/5afe9d2eN1574b8a9.jpg |         NULL |
| 15924 | 01016330001                    |           6 |                    16330 | Petio 日本品牌逗猫玩具 彩色加长型逗猫棒 彩虹 1个装                                                                                      |      20.16 |           20.16 |                20.16 |        28.00 |      3 |         0 |        1 |             0 |        1 |        1 |               99999 |                     2 | http://img13.360buyimg.com/n1/s375x375_jfs/t28027/212/1321508882/56434/c5df31e7/5cdd2d60Nc2e09715.jpg  |         NULL |
| 32206 | 08032612001                    |           6 |                    32612 | ST 艾饰庭消臭力 檀木香卫生间厕所空气清新剂 液体芳香剂除臭去异味 400ml（日本原装进口）                                                   |      32.69 |           32.69 |                32.69 |        41.90 |      3 |         0 |        1 |             0 |        1 |        1 |               99999 |                     2 | http://img13.360buyimg.com/n1/s375x375_jfs/t23905/209/731149727/265952/2779715a/5b3f0ba3N59734abb.jpg  |         NULL |
| 32203 | 08032609001                    |           6 |                    32609 | ST 艾饰庭消臭力水果香（宠物味用） 空气清新剂液体芳香剂除臭去异味400ml（日本原装进口）                                                   |      31.13 |           31.13 |                31.13 |        39.90 |      3 |         0 |        1 |             0 |        1 |        1 |               99999 |                     2 | http://img13.360buyimg.com/n1/s375x375_jfs/t22201/33/1905511632/234386/4220392a/5b3e9a10N7b28b226.jpg  |         NULL |
| 32212 | 08032618001                    |           6 |                    32618 | ST 艾饰庭消臭力皂香卫生间用厕所空气清新剂液体香氛剂除臭去异味 400ml(日本原装进口)                                                       |      32.69 |           32.69 |                32.69 |        41.90 |      3 |         0 |        1 |             0 |        1 |        1 |               99999 |                     2 | http://img13.360buyimg.com/n1/s375x375_jfs/t24214/23/742300683/231221/45d47110/5b3f10aeN4313b7d7.jpg   |         NULL |
| 32196 | 08032602001                    |           6 |                    32602 | ST 艾饰庭消臭力葡萄柚柠檬味卫生间厕所空气清新剂液体 芳香剂除臭去异味 400ml（日本原装进口）                                              |      32.69 |           32.69 |                32.69 |        41.90 |      3 |         0 |        1 |             0 |        1 |        1 |               99999 |                     2 | http://img13.360buyimg.com/n1/s375x375_jfs/t23374/103/718750962/299010/14d7915e/5b3f0b54N6157d92d.jpg  |         NULL |
+-------+--------------------------------+-------------+--------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+------------+-----------------+----------------------+--------------+--------+-----------+----------+---------------+----------+----------+---------------------+-----------------------+--------------------------------------------------------------------------------------------------------+--------------+
10 rows in set (2.62 sec)

```

> *执行时间减少至2.62s.*

## SQL优化结果
| 序号 | 优化之前 | 优化之后 |
| ---- | -------- | -------- |
| 1    | 2min 33s | 2.62s    |

## 总结

- 该条SQL执行缓慢是由于查询表standard_unit_company的standard_unit_id列缺少索引导致的。

- 在standard_unit_id列上添加索引之后，查询时间由2min 33s减少至2.62s.