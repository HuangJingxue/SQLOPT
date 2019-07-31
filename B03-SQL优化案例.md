# SQL优化经典案例(三)

## 概要

该条慢SQL为我司生产环境的一条，阿里云RDS for MySQL实例规格为4C8G的实例中。为了优化过程不对生产造成影响，本次优化本地虚拟机当中完成。

### 自建MySQL详情

```shell
vmware虚拟机：
2C2G

MySQL版本：
mysql Ver 14.14 Distrib 5.7.17, for linux-glibc2.5 (x86_64) using EditLine wrapper
```

## 完整SQL语句

```sql
SELECT t.code AS t_code, t.account_id AS t_aid, t.type AS t_type, t.tmp_passwd AS t_tmp_pass, t.allow_ip AS t_allow_ip
	, t.safe_mode AS t_safe_mode, t.project_resource_id AS t_prid, nw.id AS nw_id, nw.safe_mode_approver AS nw_approver, nw.safe_mode AS nw_safe_mode
	, apk.pubkey AS apk_pub_key, COUNT(tc.id) AS tc_cnt, r.autologin AS r_autologin, r.csos_check AS r_csos_check, r.ip_address AS r_address
	, r.block_pattern AS r_bp, r.extension AS r_ext, r.name AS r_name, r.id AS r_id, r.resource_uid AS r_uid
	, r.resource_database AS r_db, r.resource_user AS r_user, a.pwd AS a_pwd, a.create_time AS a_create_time, sma.id AS sma_id
FROM tunnel t
	INNER JOIN resource r ON r.id = t.resource_id
	INNER JOIN account a ON a.id = t.account_id
	INNER JOIN network nw ON nw.id = r.network_id
	LEFT JOIN account_pubkey apk ON apk.account_id = t.account_id
		AND BINARY apk.pubkey = ''
	LEFT JOIN tunnel_connection tc ON BINARY tc.tunnel_code = t.code
		AND tc.online = 1
	LEFT JOIN safe_mode_application sma ON sma.`applicant_id` = a.id
		AND sma.`project_resource_id` = t.`project_resource_id`
		AND sma.app_status = 1
		AND sma.expire_time >= (
			SELECT UNIX_TIMESTAMP()
			)
WHERE BINARY t.code = 'WguEJs6C'
AND t.status != 'disable'
AND t.`expire_time` >= (
	SELECT UNIX_TIMESTAMP()
	)
GROUP BY t.code
LIMIT 1

--返回行数：[1]，耗时：1955 ms.
--测试环境，10.72 sec

--返回的结果
+------------------+-----------------+------------------+----------------------+----------------------+-----------------------+------------------+-----------------+-----------------------+------------------------+-----------------------+------------------+-----------------------+------------------------+---------------------+----------------+-------------------------------------------------------------------------+------------------+----------------+----------------------------------+----------------+------------------+------------------------------------------------------------------+-------------------------+------------------+
| t_code           | t_aid           | t_type           | t_tmp_pass           | t_allow_ip           | t_safe_mode           | t_prid           | nw_id           | nw_approver           | nw_safe_mode           | apk_pub_key           | tc_cnt           | r_autologin           | r_csos_check           | r_address           | r_bp           | r_ext                                                                   | r_name           | r_id           | r_uid                            | r_db           | r_user           | a_pwd                                                            | a_create_time           | sma_id           |
+------------------+-----------------+------------------+----------------------+----------------------+-----------------------+------------------+-----------------+-----------------------+------------------------+-----------------------+------------------+-----------------------+------------------------+---------------------+----------------+-------------------------------------------------------------------------+------------------+----------------+----------------------------------+----------------+------------------+------------------------------------------------------------------+-------------------------+------------------+
| WguEJs6C         |            1998 |                  |                      |                      |                     1 |            10777 |            1059 |                     0 |                      0 |                       | 0                |                     0 | close                  | 172.16.2.162        |                | {"ssh_port": 40022, "username": ["zyadmin", "tstadmin"], "os": "linux"} | prod-cms         |           8109 | 622b703b48dc4427b43f7976a8d00026 |                |                  | afbfdc982f878f81678f996185ce0af82f874753cb5179f943960023657878e7 |              1515064116 |                  |
+------------------+-----------------+------------------+----------------------+----------------------+-----------------------+------------------+-----------------+-----------------------+------------------------+-----------------------+------------------+-----------------------+------------------------+---------------------+----------------+-------------------------------------------------------------------------+------------------+----------------+----------------------------------+----------------+------------------+------------------------------------------------------------------+-------------------------+------------------+

--执行计划
+--------------+-----------------------+-----------------+----------------+-------------------------------------------------------------+---------------------+-------------------+----------------------------+----------------+----------------------------------------------------+
| id           | select_type           | table           | type           | possible_keys                                               | key                 | key_len           | ref                        | rows           | Extra                                              |
+--------------+-----------------------+-----------------+----------------+-------------------------------------------------------------+---------------------+-------------------+----------------------------+----------------+----------------------------------------------------+
| 1            | SIMPLE                | t               | ALL            | PRIMARY,account_id,project_resource_id,agent_id,resource_id |                     |                   |                            | 339014         | Using where; Using temporary; Using filesort       |
| 1            | SIMPLE                | a               | eq_ref         | PRIMARY                                                     | PRIMARY             | 4                 | csos.t.account_id          | 1              |                                                    |
| 1            | SIMPLE                | apk             | ref            | account_id                                                  | account_id          | 5                 | csos.t.account_id          | 1              | Using where                                        |
| 1            | SIMPLE                | r               | eq_ref         | PRIMARY,network_id                                          | PRIMARY             | 4                 | csos.t.resource_id         | 1              | Using where                                        |
| 1            | SIMPLE                | nw              | eq_ref         | PRIMARY                                                     | PRIMARY             | 4                 | csos.r.network_id          | 1              |                                                    |
| 1            | SIMPLE                | tc              | ALL            |                                                             |                     |                   |                            | 2871267        | Using where; Using join buffer (Block Nested Loop) |
| 1            | SIMPLE                | sma             | ref            | applicant_id,project_resource_id                            | project_resource_id | 5                 | csos.t.project_resource_id | 1              | Using where                                        |
+--------------+-----------------------+-----------------+----------------+-------------------------------------------------------------+---------------------+-------------------+----------------------------+----------------+----------------------------------------------------+
返回行数：[7]，耗时：7 ms.
```

*注：从执行计划看出`t`表全面扫描行数较大同时用到临时表以及文件排序，`tc`表扫描行数较大，预测以上两点是造成该语句执行慢的原因*

## 获取表结构及表记录数

```sql
--表结构
CREATE TABLE `tunnel` (
  `account_id` int(11) DEFAULT NULL,
  `code` varchar(20) NOT NULL,
  `connect_ip` varchar(20) DEFAULT NULL,
  `project_resource_id` int(11) DEFAULT NULL,
  `resource_id` int(11) DEFAULT NULL,
  `port` int(11) DEFAULT NULL,
  `os` varchar(20) DEFAULT NULL,
  `type` varchar(255) NOT NULL DEFAULT '',
  `tmp_passwd` varchar(20) DEFAULT NULL,
  `status` varchar(255) NOT NULL,
  `allow_ip` varchar(1024) DEFAULT '""',
  `online` int(11) DEFAULT NULL,
  `peer_ip` varchar(255) DEFAULT '""',
  `wetty_ip` varchar(255) DEFAULT NULL,
  `agent_id` int(11) DEFAULT NULL,
  `create_time` int(11) DEFAULT NULL,
  `expire_time` int(11) DEFAULT NULL,
  `update_time` int(11) DEFAULT NULL,
  `safe_mode` tinyint(1) NOT NULL DEFAULT '1',
  PRIMARY KEY (`code`),
  KEY `account_id` (`account_id`),
  KEY `project_resource_id` (`project_resource_id`),
  KEY `agent_id` (`agent_id`),
  KEY `resource_id` (`resource_id`),
  CONSTRAINT `tunnel_ibfk_1` FOREIGN KEY (`account_id`) REFERENCES `account` (`id`) ON DELETE SET NULL,
  CONSTRAINT `tunnel_ibfk_2` FOREIGN KEY (`project_resource_id`) REFERENCES `project_resource` (`id`) ON DELETE SET NULL,
  CONSTRAINT `tunnel_ibfk_3` FOREIGN KEY (`agent_id`) REFERENCES `agent` (`id`) ON DELETE SET NULL,
  CONSTRAINT `tunnel_ibfk_4` FOREIGN KEY (`resource_id`) REFERENCES `resource` (`id`) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8；

--表结构
CREATE TABLE `tunnel_connection` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `account_id` int(11) DEFAULT NULL,
  `company_id` int(11) DEFAULT NULL,
  `tunnel_code` varchar(20) DEFAULT '',
  `way` varchar(20) DEFAULT NULL,
  `project_resource_id` int(11) DEFAULT NULL,
  `connection_id` varchar(64) DEFAULT NULL,
  `online` int(11) DEFAULT NULL,
  `safe_mode` tinyint(1) NOT NULL DEFAULT '1',
  `peer_ip` varchar(50) DEFAULT NULL,
  `wetty_ip` varchar(50) DEFAULT NULL,
  `create_time` int(11) DEFAULT NULL,
  `update_time` int(11) DEFAULT NULL,
  `close_time` int(11) DEFAULT NULL,
  `network_id` int(11) DEFAULT NULL,
  `guacamole` int(11) DEFAULT NULL,
  `extension` text,
  PRIMARY KEY (`id`),
  KEY `account_id` (`account_id`),
  KEY `company_id` (`company_id`),
  KEY `project_resource_id` (`project_resource_id`),
  KEY `network_id` (`network_id`),
  KEY `ix_tunnel_connection_connection_id` (`connection_id`),
  KEY `tunnel_code` (`tunnel_code`) USING BTREE,
  CONSTRAINT `tunnel_connection_ibfk_1` FOREIGN KEY (`account_id`) REFERENCES `account` (`id`) ON DELETE SET NULL,
  CONSTRAINT `tunnel_connection_ibfk_2` FOREIGN KEY (`company_id`) REFERENCES `company` (`id`),
  CONSTRAINT `tunnel_connection_ibfk_4` FOREIGN KEY (`project_resource_id`) REFERENCES `project_resource` (`id`) ON DELETE SET NULL,
  CONSTRAINT `tunnel_connection_ibfk_5` FOREIGN KEY (`network_id`) REFERENCES `network` (`id`) ON DELETE SET NULL
) ENGINE=InnoDB AUTO_INCREMENT=3020934 DEFAULT CHARSET=utf8	

--表记录数
mysql>select count(*) from tunnel;
+--------------------+
| count(*)           |
+--------------------+
| 354724             |
+--------------------+
返回行数：[1]，耗时：136 ms.

mysql>select count(*) from resource
+--------------------+
| count(*)           |
+--------------------+
| 28252              |
+--------------------+
返回行数：[1]，耗时：14 ms.

mysql>select count(*) from network;
+--------------------+
| count(*)           |
+--------------------+
| 1276               |
+--------------------+
返回行数：[1]，耗时：7 ms.
mysql>select count(*) from account_pubkey;
+--------------------+
| count(*)           |
+--------------------+
| 228                |
+--------------------+
返回行数：[1]，耗时：6 ms.
mysql>select count(*) from tunnel_connection;
+--------------------+
| count(*)           |
+--------------------+
| 3018991            |
+--------------------+

mysql>select count(*) from safe_mode_application;
+--------------------+
| count(*)           |
+--------------------+
| 282                |
+--------------------+
返回行数：[1]，耗时：6 ms.
```
## 扫描行数较多的表添加where条件运行
```sql
mysql>select count(*) from tunnel WHERE BINARY code = 'WguEJs6C' --重要内容
AND status != 'disable'
AND `expire_time` >= (
	SELECT UNIX_TIMESTAMP()
	);
+--------------------+
| count(*)           |
+--------------------+
| 1                  |
+--------------------+
返回行数：[1]，耗时：140 ms.

mysql>select count(*) from tunnel_connection where online = 1; --重要内容
+--------------------+
| count(*)           |
+--------------------+
| 61829              |
+--------------------+
返回行数：[1]，耗时：645 ms.
```

## 拆分SQL(一[t])

```sql
--拆分第一条
mysql>SELECT t.code AS t_code, t.account_id AS t_aid, t.type AS t_type, t.tmp_passwd AS t_tmp_pass, t.allow_ip AS t_allow_ip
	, t.safe_mode AS t_safe_mode, t.project_resource_id AS t_prid
FROM tunnel t
WHERE BINARY t.code = 'WguEJs6C'
AND t.status != 'disable'
AND t.`expire_time` >= (
	SELECT UNIX_TIMESTAMP()
	)
GROUP BY t.code
LIMIT 1
+------------------+-----------------+------------------+----------------------+----------------------+-----------------------+------------------+
| t_code           | t_aid           | t_type           | t_tmp_pass           | t_allow_ip           | t_safe_mode           | t_prid           |
+------------------+-----------------+------------------+----------------------+----------------------+-----------------------+------------------+
| WguEJs6C         |            1998 |                  |                      |                      |                     1 |            10777 |
+------------------+-----------------+------------------+----------------------+----------------------+-----------------------+------------------+
返回行数：[1]，耗时：187 ms.

mysql>explain SELECT t.code AS t_code, t.account_id AS t_aid, t.type AS t_type, t.tmp_passwd AS t_tmp_pass, t.allow_ip AS t_allow_ip
	, t.safe_mode AS t_safe_mode, t.project_resource_id AS t_prid
FROM tunnel t
WHERE BINARY t.code = 'WguEJs6C'
AND t.status != 'disable'
AND t.`expire_time` >= (
	SELECT UNIX_TIMESTAMP()
	)
GROUP BY t.code
LIMIT 1;
+--------------+-----------------------+-----------------+----------------+-------------------------------------------------------------+---------------+-------------------+---------------+----------------+-----------------+
| id           | select_type           | table           | type           | possible_keys                                               | key           | key_len           | ref           | rows           | Extra           |
+--------------+-----------------------+-----------------+----------------+-------------------------------------------------------------+---------------+-------------------+---------------+----------------+-----------------+
| 1            | SIMPLE                | t               | index          | PRIMARY,account_id,project_resource_id,agent_id,resource_id | PRIMARY       | 62                |               | 1              | Using where     |
+--------------+-----------------------+-----------------+----------------+-------------------------------------------------------------+---------------+-------------------+---------------+----------------+-----------------+
返回行数：[1]，耗时：7 ms.
```

## 改写SQL

需要缩小结果集来优化此SQL

```sql
--本次改写更慢：
SELECT t.t_code, t.t_aid, t.t_type, t.t_tmp_pass, t.t_allow_ip
	, t.t_safe_mode, t.t_prid, nw.id AS nw_id, nw.safe_mode_approver AS nw_approver, nw.safe_mode AS nw_safe_mode
	, apk.pubkey AS apk_pub_key, COUNT(tc.id) AS tc_cnt, r.autologin AS r_autologin, r.csos_check AS r_csos_check, r.ip_address AS r_address
	, r.block_pattern AS r_bp, r.extension AS r_ext, r.name AS r_name, r.id AS r_id, r.resource_uid AS r_uid
	, r.resource_database AS r_db, r.resource_user AS r_user, a.pwd AS a_pwd, a.create_time AS a_create_time, sma.id AS sma_id
from (SELECT resource_id,code AS t_code, account_id AS t_aid, type AS t_type, tmp_passwd AS t_tmp_pass, allow_ip AS t_allow_ip
	, safe_mode AS t_safe_mode, project_resource_id AS t_prid
FROM tunnel WHERE BINARY code = 'WguEJs6C'
AND status != 'disable'
AND `expire_time` >= (
	SELECT UNIX_TIMESTAMP()
	)GROUP BY code )t
	INNER JOIN resource r ON r.id = t.resource_id
	INNER JOIN account a ON a.id = t.t_aid
	INNER JOIN network nw ON nw.id = r.network_id
	LEFT JOIN account_pubkey apk ON apk.account_id = t.t_aid
		AND BINARY apk.pubkey = ''
	LEFT JOIN tunnel_connection tc ON BINARY tc.tunnel_code = t.t_code
		AND tc.online = 1
	LEFT JOIN safe_mode_application sma ON sma.`applicant_id` = a.id
		AND sma.`project_resource_id` = t.t_prid
		AND sma.app_status = 1
		AND sma.expire_time >= (
			SELECT UNIX_TIMESTAMP()
			)
LIMIT 1

--测试环境执行计划
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
   partitions: NULL
         type: system
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: PRIMARY
        table: r
   partitions: NULL
         type: const
possible_keys: PRIMARY,network_id
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 3. row ***************************
           id: 1
  select_type: PRIMARY
        table: a
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 4. row ***************************
           id: 1
  select_type: PRIMARY
        table: nw
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 5. row ***************************
           id: 1
  select_type: PRIMARY
        table: apk
   partitions: NULL
         type: ref
possible_keys: account_id
          key: account_id
      key_len: 5
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where
*************************** 6. row ***************************
           id: 1
  select_type: PRIMARY
        table: tc
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 2737997
     filtered: 100.00
        Extra: Using where; Using join buffer (Block Nested Loop)
*************************** 7. row ***************************
           id: 1
  select_type: PRIMARY
        table: sma
   partitions: NULL
         type: ref
possible_keys: applicant_id,project_resource_id
          key: applicant_id
      key_len: 5
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where
*************************** 8. row ***************************
           id: 2
  select_type: DERIVED
        table: tunnel
   partitions: NULL
         type: index
possible_keys: PRIMARY,account_id,project_resource_id,agent_id,resource_id
          key: PRIMARY
      key_len: 62
          ref: NULL
         rows: 1
     filtered: 30.00
        Extra: Using where
8 rows in set, 3 warnings (0.78 sec)
```

*注：以上改写后主要由于tc表扫描行数最多导致的运行慢*

## 添加索引优化

```sql
--添加的索引			
ALTER TABLE `tunnel_connection` ADD KEY `tunnel_code`(`online`,`tunnel_code`);	

--改写后执行1.01Sec
root@MySQL-01 11:15:  [csos]> SELECT t.t_code, t.t_aid, t.t_type, t.t_tmp_pass, t.t_allow_ip
    -> , t.t_safe_mode, t.t_prid, nw.id AS nw_id, nw.safe_mode_approver AS nw_approver, nw.safe_mode AS nw_safe_mode
    -> , apk.pubkey AS apk_pub_key, COUNT(tc.id) AS tc_cnt, r.autologin AS r_autologin, r.csos_check AS r_csos_check, r.ip_address AS r_address
    -> , r.block_pattern AS r_bp, r.extension AS r_ext, r.name AS r_name, r.id AS r_id, r.resource_uid AS r_uid
    -> , r.resource_database AS r_db, r.resource_user AS r_user, a.pwd AS a_pwd, a.create_time AS a_create_time, sma.id AS sma_id
    -> from (SELECT resource_id,code AS t_code, account_id AS t_aid, type AS t_type, tmp_passwd AS t_tmp_pass, allow_ip AS t_allow_ip
    -> , safe_mode AS t_safe_mode, project_resource_id AS t_prid
    -> FROM tunnel WHERE BINARY code = 'WguEJs6C'
    -> AND status != 'disable'
    -> AND `expire_time` >= (
    -> SELECT UNIX_TIMESTAMP()
    -> )GROUP BY code LIMIT 1 )t
    -> INNER JOIN resource r ON r.id = t.resource_id
    -> INNER JOIN account a ON a.id = t.t_aid
    -> INNER JOIN network nw ON nw.id = r.network_id
    -> LEFT JOIN account_pubkey apk ON apk.account_id = t.t_aid
    -> AND BINARY apk.pubkey = ''
    -> LEFT JOIN (select tunnel_code,id from tunnel_connection where online = 1) tc ON BINARY tc.tunnel_code = t.t_code
    -> LEFT JOIN safe_mode_application sma ON sma.`applicant_id` = a.id
    -> AND sma.`project_resource_id` = t.t_prid
    -> AND sma.app_status = 1
    -> AND sma.expire_time >= (
    -> SELECT UNIX_TIMESTAMP());
+----------+-------+--------+------------+------------+-------------+--------+-------+-------------+--------------+-------------+--------+-------------+--------------+--------------+------+-------------------------------------------------------------------------+----------+------+----------------------------------+------+--------+------------------------------------------------------------------+---------------+--------+
| t_code   | t_aid | t_type | t_tmp_pass | t_allow_ip | t_safe_mode | t_prid | nw_id | nw_approver | nw_safe_mode | apk_pub_key | tc_cnt | r_autologin | r_csos_check | r_address    | r_bp | r_ext                                                                   | r_name   | r_id | r_uid                            | r_db | r_user | a_pwd                                                            | a_create_time | sma_id |
+----------+-------+--------+------------+------------+-------------+--------+-------+-------------+--------------+-------------+--------+-------------+--------------+--------------+------+-------------------------------------------------------------------------+----------+------+----------------------------------+------+--------+------------------------------------------------------------------+---------------+--------+
| WguEJs6C |  1998 |        |            |            |           1 |  10777 |  1059 |           0 |            0 | NULL        |      0 |           0 | close        | 172.16.2.162 | NULL | {"ssh_port": 40022, "username": ["zyadmin", "tstadmin"], "os": "linux"} | prod-cms | 8109 | 622b703b48dc4427b43f7976a8d00026 | NULL | NULL   | afbfdc982f878f81678f996185ce0af82f874753cb5179f943960023657878e7 |    1515064116 |   NULL |
+----------+-------+--------+------------+------------+-------------+--------+-------+-------------+--------------+-------------+--------+-------------+--------------+--------------+------+-------------------------------------------------------------------------+----------+------+----------------------------------+------+--------+------------------------------------------------------------------+---------------+--------+
1 row in set (1.01 sec)

--执行计划
explain SELECT t.t_code, t.t_aid, t.t_type, t.t_tmp_pass, t.t_allow_ip
	, t.t_safe_mode, t.t_prid, nw.id AS nw_id, nw.safe_mode_approver AS nw_approver, nw.safe_mode AS nw_safe_mode
	, apk.pubkey AS apk_pub_key, COUNT(tc.id) AS tc_cnt, r.autologin AS r_autologin, r.csos_check AS r_csos_check, r.ip_address AS r_address
	, r.block_pattern AS r_bp, r.extension AS r_ext, r.name AS r_name, r.id AS r_id, r.resource_uid AS r_uid
	, r.resource_database AS r_db, r.resource_user AS r_user, a.pwd AS a_pwd, a.create_time AS a_create_time, sma.id AS sma_id
from (SELECT resource_id,code AS t_code, account_id AS t_aid, type AS t_type, tmp_passwd AS t_tmp_pass, allow_ip AS t_allow_ip
	, safe_mode AS t_safe_mode, project_resource_id AS t_prid
FROM tunnel WHERE BINARY code = 'WguEJs6C'
AND status != 'disable'
AND `expire_time` >= (
	SELECT UNIX_TIMESTAMP()
	)GROUP BY code LIMIT 1 )t
	INNER JOIN resource r ON r.id = t.resource_id
	INNER JOIN account a ON a.id = t.t_aid
	INNER JOIN network nw ON nw.id = r.network_id
	LEFT JOIN account_pubkey apk ON apk.account_id = t.t_aid
		AND BINARY apk.pubkey = ''
	LEFT JOIN (select tunnel_code,id from tunnel_connection where online = 1) tc ON BINARY tc.tunnel_code = t.t_code
	LEFT JOIN safe_mode_application sma ON sma.`applicant_id` = a.id
		AND sma.`project_resource_id` = t.t_prid
		AND sma.app_status = 1
		AND sma.expire_time >= (
			SELECT UNIX_TIMESTAMP());
+----+-------------+-------------------+------------+--------+----------------------------------------------------------------------+--------------+---------+-------+---------+----------+--------------------------+
| id | select_type | table             | partitions | type   | possible_keys                                                        | key          | key_len | ref   | rows    | filtered | Extra                    |
+----+-------------+-------------------+------------+--------+----------------------------------------------------------------------+--------------+---------+-------+---------+----------+--------------------------+
|  1 | PRIMARY     | <derived2>        | NULL       | system | NULL                                                                 | NULL         | NULL    | NULL  |       1 |   100.00 | NULL                     |
|  1 | PRIMARY     | r                 | NULL       | const  | PRIMARY,network_id                                                   | PRIMARY      | 4       | const |       1 |   100.00 | NULL                     |
|  1 | PRIMARY     | a                 | NULL       | const  | PRIMARY                                                              | PRIMARY      | 4       | const |       1 |   100.00 | NULL                     |
|  1 | PRIMARY     | nw                | NULL       | const  | PRIMARY                                                              | PRIMARY      | 4       | const |       1 |   100.00 | NULL                     |
|  1 | PRIMARY     | apk               | NULL       | ref    | account_id                                                           | account_id   | 5       | const |       1 |   100.00 | Using where              |
|  1 | PRIMARY     | tunnel_connection | NULL       | ref    | tunnel_code                                                          | tunnel_code  | 5       | const | 2737997 |   100.00 | Using where; Using index |
|  1 | PRIMARY     | sma               | NULL       | ref    | applicant_id,project_resource_id,index1                              | applicant_id | 5       | const |       1 |   100.00 | Using where              |
|  2 | DERIVED     | tunnel            | NULL       | index  | PRIMARY,account_id,project_resource_id,agent_id,resource_id,ix_test1 | PRIMARY      | 62      | NULL  |       1 |    30.00 | Using where              |
+----+-------------+-------------------+------------+--------+----------------------------------------------------------------------+--------------+---------+-------+---------+----------+--------------------------+
8 rows in set, 3 warnings (0.95 sec)
```

## 测试环境优化后结果对比

| 序号 | 优化前    | 优化后   | 备注                            |
| ---- | --------- | -------- | ------------------------------- |
| 1    | 10.72 sec | 1.01 sec | SQL改写（缩小结果集）+ 创建索引 |

## 总结

- 该条SQL执行缓慢是由于查询表tunnel_connection表缺少合理索引导致的。将原只有tunnel_code列索引，修改为online，tunnel_code的复合索引
- 并通过改写缩小t表结果集