# **老板电器SQL优化 v.2019.12.10**

**通过本次优化沉淀：SQL优化需要先与客户沟通是应用在什么业务场景**

· 如果是分析型业务场景，就是要返回那么多的数据，建议采用分析型的数据库类型(ploardb，ADS,dataworks等)

· 如果是OLTP场景，建议做分页来提高效率

![](D:\智能团队相关资料\08_学习内容\03_github仓库\SQLOPT\pic\wps1.jpg)

![img](pic\wps1.jpg)

这个1是查找当前会员购买总金额 2是查找当前会员对应的介绍人 

 

**源SQL**  展开源码 

SELECT m.*,ml.ml_name,m2.m_name as m_recommend_name,

(select sum(o_pay) from fx_orders where o_pay_status=1 and m_id=m.m_id) as totalpay 

FROM fx_members as m 

LEFT JOIN fx_members as m2 on m.m_recommended=m2.m_id 

LEFT JOIN fx_members_level as ml on m.ml_id=ml.ml_id 

WHERE ( (m.m_create_time BETWEEN '2018-12-03 00:00:00' AND '2019-01-04 00:00:00' ) )

 

**SQL优化过程**  展开源码 

总行数

mysql>SELECT count(*) FROM fx_members;

+--------------------+

| count(*)           |

+--------------------+

| 565511             |

+--------------------+

返回行数：[1]，耗时：101 ms.

 

原SQL

SELECT m.*,ml.ml_name,m2.m_name as m_recommend_name,

(select sum(o_pay) from fx_orders where o_pay_status=1 and m_id=m.m_id) as totalpay 

FROM fx_members as m 

LEFT JOIN fx_members as m2 on m.m_recommended=m2.m_id 

LEFT JOIN fx_members_level as ml on m.ml_id=ml.ml_id 

WHERE ( (m.m_create_time BETWEEN '2018-12-03 00:00:00' AND '2019-01-04 00:00:00' ) )

 

24.490s 68129条

 

 

缩小结果集

select m.*,ml.ml_name,m2.m_name as m_recommend_name,(select sum(o_pay) from fx_orders where o_pay_status=1 and m_id=m.m_id) as totalpay 

(select * from fx_members where m.m_create_time BETWEEN '2018-12-03 00:00:00' AND '2019-01-04 00:00:00') m

left join fx_members as m2 on m.m_recommended=m2.m_id

left join  fx_members_level as ml on m.ml_id=ml.ml_id 

 

24.490s 68129条

 

 

时间未有改变，查看执行计划，不走索引

 

 

select * from fx_members where m.m_create_time BETWEEN '2018-12-03 00:00:00' AND '2019-01-04 00:00:00'

2.741s 68129条 --不走索引

 

 

select * from fx_members where m.m_create_time > '2018-12-03 00:00:00'

不走索引

 

 

验证为什么create_time 有索引但是不走

mysql> explain select * from fx_members where m_create_time > '2018-12-03 00:00:00';

+----+-------------+------------+------+---------------+------+---------+------+------+-------------+

| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |

+----+-------------+------------+------+---------------+------+---------+------+------+-------------+

|  1 | SIMPLE      | fx_members | ALL  | m_create_time | NULL | NULL    | NULL |    7 | Using where |

+----+-------------+------------+------+---------------+------+---------+------+------+-------------+

1 row in set (0.00 sec)

 

 

mysql> explain select m_id from fx_members where m_create_time > '2018-12-03 00:00:00';

+----+-------------+------------+-------+---------------+---------------+---------+------+------+--------------------------+

| id | select_type | table      | type  | possible_keys | key           | key_len | ref  | rows | Extra                    |

+----+-------------+------------+-------+---------------+---------------+---------+------+------+--------------------------+

|  1 | SIMPLE      | fx_members | index | m_create_time | m_create_time | 4       | NULL |    7 | Using where; Using index |

+----+-------------+------------+-------+---------------+---------------+---------+------+------+--------------------------+

1 row in set (0.00 sec)

 

是由于返回的字段列较多，优化器判断走索引不如直接全表扫描

 

只获取需要字段

select m.m_id,m.m_name,m.open_source,m.m_verify,m.is_fenxiao,m.total_point,m.m_create_time,m.m_last_login_time,m.m_from_url,m.m_from,ml.ml_name,m2.m_name as m_recommend_name,(select sum(o_pay) from fx_orders where o_pay_status=1 and m_id=m.m_id) as totalpay from 

(select m_id,m_name,open_source,m_verify,is_fenxiao,total_point,m_create_time,m_last_login_time,m_from_url,m_from,m_recommended,ml_id from fx_members where  m_create_time between '2018-12-03 00:00:00' AND '2019-01-04 00:00:00' ) m

left join fx_members as m2 on m.m_recommended=m2.m_id

left join  fx_members_level as ml on m.ml_id=ml.ml_id

 

 

6.988s

 

不强制

select m.m_id,m.m_name,m.open_source,m.m_verify,m.is_fenxiao,m.total_point,m.m_create_time,m.m_last_login_time,m.m_from_url,m.m_from,ml.ml_name,m2.m_name as m_recommend_name,(select sum(o_pay) from fx_orders where o_pay_status=1 and m_id=m.m_id) as totalpay from 

(select m_id,m_name,open_source,m_verify,is_fenxiao,total_point,m_create_time,m_last_login_time,m_from_url,m_from,m_recommended,ml_id from fx_members where  m_create_time between '2018-12-03 00:00:00' AND '2019-01-04 00:00:00' ) m

left join fx_members as m2 on m.m_recommended=m2.m_id

left join  fx_members_level as ml on m.ml_id=ml.ml_id

 

6.044s

 

 

强制

select m.m_id,m.m_name,m.open_source,m.m_verify,m.is_fenxiao,m.total_point,m.m_create_time,m.m_last_login_time,m.m_from_url,m.m_from,ml.ml_name,m2.m_name as m_recommend_name,(select sum(o_pay) from fx_orders where o_pay_status=1 and m_id=m.m_id) as totalpay from 

(select m_id,m_name,open_source,m_verify,is_fenxiao,total_point,m_create_time,m_last_login_time,m_from_url,m_from,m_recommended,ml_id from fx_members force index (m_create_time) where m_create_time BETWEEN '2018-12-03 00:00:00' AND '2019-01-04 00:00:00') m

left join fx_members as m2 on m.m_recommended=m2.m_id

left join  fx_members_level as ml on m.ml_id=ml.ml_id

 

5.892s

 

 

强制，并区间范围采用>= <=

select m.m_id,m.m_name,m.open_source,m.m_verify,m.is_fenxiao,m.total_point,m.m_create_time,m.m_last_login_time,m.m_from_url,m.m_from,ml.ml_name,m2.m_name as m_recommend_name,(select sum(o_pay) from fx_orders where o_pay_status=1 and m_id=m.m_id) as totalpay from 

(select m_id,m_name,open_source,m_verify,is_fenxiao,total_point,m_create_time,m_last_login_time,m_from_url,m_from,m_recommended,ml_id from fx_members force index (m_create_time) where  m_create_time >= '2018-12-03 00:00:00' and m_create_time <= '2019-01-04 00:00:00' ) m

left join fx_members as m2 on m.m_recommended=m2.m_id

left join  fx_members_level as ml on m.ml_id=ml.ml_id

5.986s

 

 

 

SELECT

​	m_id

FROM

​	fx_members

WHERE

​	m_create_time >= '2018-12-03 00:00:00'

AND m_create_time <= '2019-01-04 00:00:00'

0.473s

 

SELECT

​	m_id,

​	m_name,

​	open_source,

​	m_verify,

​	is_fenxiao,

​	total_point,

​	m_create_time,

​	m_last_login_time,

​	m_from_url,

​	m_from,

​	m_recommended,

​	ml_id

FROM

​	fx_members a

WHERE

​	a.m_id IN (

​		SELECT

​			m_id

​		FROM

​			fx_members

​		WHERE

​			m_create_time >= '2018-12-03 00:00:00'

​		AND m_create_time <= '2019-01-04 00:00:00'

​	)

6.021S 

 

 

SELECT

​	a.m_id,

​	a.m_name,

​	a.open_source,

​	a.m_verify,

​	a.is_fenxiao,

​	a.total_point,

​	a.m_create_time,

​	a.m_last_login_time,

​	a.m_from_url,

​	a.m_from,

​	a.m_recommended,

​	a.ml_id

FROM

​	fx_members a

join (

​		SELECT

​			m_id

​		FROM

​			fx_members

​		WHERE

​			m_create_time >= '2018-12-03 00:00:00'

​		AND m_create_time <= '2019-01-04 00:00:00'

​	) as m

on a.m_id = m.m_id

 

 

SELECT

​	m_id,

​	m_name,

​	open_source,

​	m_verify,

​	is_fenxiao,

​	total_point,

​	m_create_time,

​	m_last_login_time,

​	m_from_url,

​	m_from,

​	m_recommended,

​	ml_id

FROM

​	fx_members

WHERE

​	m_create_time >= '2018-12-03 00:00:00'

AND m_create_time <= '2019-01-04 00:00:00';

6.814s 

 

 

业务优化：

表进行分区，控制数据量

返回到业务端的数据进行分页展示 --- 业务上不允许，是做导出数据用。建议放到数仓

 

SELECT

​	a.m_id,

​	a.m_name,

​	a.open_source,

​	a.m_verify,

​	a.is_fenxiao,

​	a.total_point,

​	a.m_create_time,

​	a.m_last_login_time,

​	a.m_from_url,

​	a.m_from,

​	a.m_recommended,

​	a.ml_id

FROM

​	fx_members a

join (

​		SELECT

​			m_id

​		FROM

​			fx_members

​		WHERE

​			m_create_time >= '2018-12-03 00:00:00'

​		AND m_create_time <= '2019-01-04 00:00:00'

​	) as m

on a.m_id = m.m_id

limit 100,10;

 

0.186s

 

 

**表的ddl**  展开源码 

扩展：

DDL

CREATE TABLE `fx_members` (

  `m_id` mediumint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '会员ID',

  `m_name` varchar(50) CHARACTER SET utf8 COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '会员名',

  `m_password` varchar(32) NOT NULL DEFAULT '' COMMENT '会员密码',

  `m_real_name` varchar(25) NOT NULL DEFAULT '' COMMENT '姓名',

  `m_sex` tinyint(1) NOT NULL DEFAULT '2' COMMENT '性别，0为女，1为男，2为保密',

  `cr_id` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '联系地址表里面的最终一级的ID',

  `m_address_detail` varchar(255) NOT NULL DEFAULT '' COMMENT '联系地址详细',

  `m_birthday` date NOT NULL DEFAULT '0000-00-00' COMMENT '生日',

  `m_zipcode` varchar(10) NOT NULL DEFAULT '' COMMENT '邮编',

  `m_mobile` varchar(200) NOT NULL DEFAULT '' COMMENT '手机',

  `m_telphone` varchar(200) NOT NULL DEFAULT '' COMMENT '固定电话',

  `m_status` tinyint(1) NOT NULL DEFAULT '0' COMMENT '会员状态（0为停用1 为启用）',

  `m_email` varchar(255) NOT NULL DEFAULT '' COMMENT 'email',

  `ml_id` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '会员等级id',

  `mo_id` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '在线客服id',

  `m_wangwang` varchar(255) NOT NULL DEFAULT '' COMMENT '旺旺',

  `m_qq` varchar(20) NOT NULL DEFAULT '' COMMENT 'QQ',

  `m_website_url` varchar(255) NOT NULL DEFAULT '' COMMENT '网站地址',

  `m_verify` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否已经审核，0为未审核，1为审核中，2为审核通过，3为审核未通过,4待审核',

  `m_balance` decimal(10,3) NOT NULL DEFAULT '0.000' COMMENT '账户余额',

  `open_name` varchar(50) NOT NULL DEFAULT '' COMMENT '第三方用户名',

  `open_token` varchar(255) NOT NULL DEFAULT '' COMMENT '第三方登录token',

  `open_source` varchar(200) NOT NULL DEFAULT '' COMMENT '第三方来源(QQ,新浪,支付宝)',

  `open_id` varchar(100) NOT NULL DEFAULT '' COMMENT '第三方登录唯一标示ID',

  `m_all_cost` decimal(10,3) NOT NULL DEFAULT '0.000' COMMENT '账户消费总金额',

  `total_point` int(10) NOT NULL DEFAULT '0' COMMENT '当前积分',

  `freeze_point` int(10) NOT NULL DEFAULT '0' COMMENT '当前冻结积分',

  `m_point_desc` varchar(255) NOT NULL,

  `m_create_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '记录创建时间',

  `m_last_login_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '记录最后登入时间',

  `m_update_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '记录最后更新时间',

  `thd_guid` varchar(100) NOT NULL DEFAULT '' COMMENT '第三方用户唯一标识(包含erp)',

  `m_recommended` int(50) NOT NULL DEFAULT '0' COMMENT '推荐人',

  `m_security_deposit` decimal(10,3) NOT NULL DEFAULT '0.000' COMMENT '会员的保证金就是押金',

  `m_alipay_name` varchar(50) NOT NULL DEFAULT '' COMMENT '支付宝账户',

  `m_balance_name` varchar(20) NOT NULL DEFAULT '' COMMENT '支付宝账户或银行账户',

  `m_order_status` tinyint(1) NOT NULL DEFAULT '0' COMMENT '代理下单审核( 0-否,1-是 )',

  `is_proxy` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否申请代理商（0为否，1 为是）',

  `login_type` tinyint(1) NOT NULL DEFAULT '0' COMMENT '登录方式：1第三方授权登录 0传统登录',

  `m_subcompany_id` int(11) DEFAULT NULL COMMENT '子公司ID',

  `m_bonus` decimal(10,3) NOT NULL COMMENT '红包金额',

  `m_type` tinyint(1) NOT NULL DEFAULT '0' COMMENT '会员类型：1批发商,2供货商,0普通会员',

  `m_cards` decimal(10,3) NOT NULL COMMENT '会员储蓄卡余额',

  `m_jlb` decimal(10,3) NOT NULL COMMENT '会员的金币余额',

  `m_card_no` varchar(36) NOT NULL COMMENT '会员卡卡号',

  `m_ali_card_no` varchar(36) NOT NULL COMMENT '阿里会员卡卡号',

  `m_id_card` varchar(50) NOT NULL DEFAULT '' COMMENT '身份证号',

  `m_head_img` varchar(255) NOT NULL DEFAULT '' COMMENT '会员头像',

  `union_data` varchar(500) NOT NULL DEFAULT '',

  `m_saltId` varchar(12) NOT NULL DEFAULT '' COMMENT 'ç›',

  `qq_nickname` varchar(100) DEFAULT NULL,

  `qq_openid` varchar(255) DEFAULT NULL,

  `qq_headurl` varchar(255) DEFAULT NULL,

  `wx_nickname` varchar(100) DEFAULT NULL,

  `wx_openid` varchar(255) DEFAULT NULL COMMENT '微信公众号平台openid',

  `wx_headurl` varchar(255) DEFAULT NULL,

  `wx_unionid` varchar(255) DEFAULT NULL COMMENT '微信平台wx_unionid',

  `nickname` varchar(255) DEFAULT NULL,

  `wx_openid2` varchar(255) DEFAULT NULL COMMENT '微信开放平台openid',

  `is_fenxiao` tinyint(1) DEFAULT '0' COMMENT '0未申请1审核完成2待审核',

  `m_from` varchar(200) DEFAULT '',

  `dis_balance` decimal(10,2) NOT NULL DEFAULT '0.00' COMMENT '分销账户余额',

  `dis_all_cost` decimal(10,2) NOT NULL DEFAULT '0.00' COMMENT '分销账户累计收益',

  `m_distribution_id` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '上线id, 0表示总部',

  `m_account_info` varchar(512) NOT NULL DEFAULT '' COMMENT '提现账号信息',

  `dis_create_time` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '加入分销时间',

  `cps` varchar(200) CHARACTER SET utf8 COLLATE utf8_persian_ci DEFAULT NULL COMMENT '提供给cps查看订单用的标识',

  `m_ip` varchar(100) DEFAULT NULL COMMENT 'ip地址',

  `is_fenxiao_time` timestamp NULL DEFAULT '0000-00-00 00:00:00',

  `is_fenxiao_rank` tinyint(10) DEFAULT '0' COMMENT '分销等级',

  `m_note` text COMMENT '备注',

  `is_fenxiao_rank_update` timestamp NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,

  `m_from_url` varchar(255) DEFAULT NULL COMMENT '来源url',

  `m_first_visit_time` timestamp NULL DEFAULT NULL COMMENT '首次访问时间',

  `m_first_visit_browser` text COMMENT '首次访问时间',

  `myjob` varchar(255) DEFAULT NULL COMMENT '工作',

  `applyresion` text COMMENT '合伙人申请原因',

  `fxcode` int(20) DEFAULT '0' COMMENT '合伙人ID',

  `m_abc_openid` varchar(255) DEFAULT NULL,

  PRIMARY KEY (`m_id`),

  UNIQUE KEY `m_name` (`m_name`),

  KEY `a_id` (`cr_id`),

  KEY `m_status` (`m_status`),

  KEY `m_create_time` (`m_create_time`),

  KEY `m_update_time` (`m_update_time`),

  KEY `m_email` (`m_email`),

  KEY `ml_id` (`ml_id`),

  KEY `m_verify` (`m_verify`),

  KEY `thd_guid` (`thd_guid`),

  KEY `m_mobile` (`m_mobile`),

  KEY `m_password` (`m_password`),

  KEY `index_m_recommended` (`m_recommended`),

  KEY `wx_unionid` (`wx_unionid`) USING BTREE,

  KEY `m_abc_openid` (`m_abc_openid`)

) ENGINE=InnoDB AUTO_INCREMENT=610546 DEFAULT CHARSET=utf8 COMMENT='会员基本表';

**profiles**  展开源码 

set profiling=1;

SELECT

​	a.m_id,

​	a.m_name,

​	a.open_source,

​	a.m_verify,

​	a.is_fenxiao,

​	a.total_point,

​	a.m_create_time,

​	a.m_last_login_time,

​	a.m_from_url,

​	a.m_from,

​	a.m_recommended,

​	a.ml_id

FROM

​	fx_members a

join (

​		SELECT

​			m_id

​		FROM

​			fx_members

​		WHERE

​			m_create_time >= '2018-12-03 00:00:00'

​		AND m_create_time <= '2019-01-04 00:00:00'

​	) as m

on a.m_id = m.m_id;

 

show PROFILES;

 

show PROFILE for query 24;

 

show PROFILE ALL for query 62;

 

show profile context switches for query 88;



```sql
Status	Duration	CPU_user	CPU_system	Context_voluntary	Context_involuntary	Block_ops_in	Block_ops_out	Messages_sent	Messages_received	Page_faults_major	Page_faults_minor	Swaps	Source_function	Source_file	Source_line
starting	0.000139	0	0	0	0	0	0	0	0	0	0	0			
checking permissions	1.20E-05	0	0	0	0	0	0	0	0	0	0	0	check_access	sql_authorization.cc	853
checking permissions	1.20E-05	0	0	0	0	0	0	0	0	0	0	0	check_access	sql_authorization.cc	853
Opening tables	2.50E-05	0	0	0	0	0	0	0	0	0	0	0	open_tables	sql_base.cc	5650
init	7.10E-05	0	0	0	0	0	0	0	0	0	0	0	handle_query	sql_select.cc	121
System lock	2.10E-05	0	0	0	0	0	0	0	0	0	0	0	mysql_lock_tables	lock.cc	323
optimizing	2.30E-05	0	0	0	0	0	0	0	0	0	0	0	optimize	sql_optimizer.cc	151
statistics	0.000126	0	0	0	0	0	0	0	0	0	0	0	optimize	sql_optimizer.cc	367
preparing	2.70E-05	0	0	0	0	0	0	0	0	0	0	0	optimize	sql_optimizer.cc	475
executing	1.00E-05	0	0	0	0	0	0	0	0	0	0	0	exec	sql_executor.cc	119
Sending data	5.518589	0.600908	0.027995	1157	57	0	552	0	0	0	0	0	exec	sql_executor.cc	195
end	3.60E-05	0	0	0	0	0	0	0	0	0	0	0	handle_query	sql_select.cc	199
query end	2.00E-05	0	0	0	0	0	0	0	0	0	0	0	mysql_execute_command	sql_parse.cc	5174
closing tables	2.00E-05	0	0	0	0	0	0	0	0	0	0	0	mysql_execute_command	sql_parse.cc	5226
freeing items	6.70E-05	0	0	0	0	0	16	0	0	0	0	0	mysql_parse	sql_parse.cc	5802
logging slow query	1.50E-05	0	0	0	0	0	0	0	0	0	0	0	log_slow_do	log.cc	1716
Opening tables	0.000117	0.001	0	0	0	0	0	0	0	0	0	0	open_ltable	sql_base.cc	6323
System lock	0.000104	0	0	0	0	0	16	0	0	0	0	0	mysql_lock_tables	lock.cc	323
cleaning up	1.20E-05	0	0	0	0	0	0	0	0	0	0	0	dispatch_command	sql_parse.cc	2010
															
上图中横向栏意义															
+----------------------+----------+----------+------------+															
Status: "query end", 状态															
Duration: "1.751142", 持续时间															
CPU_user: "0.008999", cpu用户															
CPU_system: "0.003999", cpu系统															
Context_voluntary: "98", 上下文主动切换															
Context_involuntary: "0", 上下文被动切换															
Block_ops_in: "8", 阻塞的输入操作															
Block_ops_out: "32", 阻塞的输出操作															
Messages_sent: "0", 消息发出															
Messages_received: "0", 消息接受															
Page_faults_major: "0", 主分页错误															
Page_faults_minor: "0", 次分页错误															
Swaps: "0", 交换次数															
Source_function: "mysql_execute_command", 源功能															
Source_file: "sql_parse.cc", 源文件															
Source_line: "4465" 源代码行															
+----------------------+----------+----------+------------+															
															
上图中纵向栏意义															
+----------------------+----------+----------+------------+															
starting：开始															
checking permissions：检查权限															
Opening tables：打开表															
init ： 初始化															
System lock ：系统锁															
optimizing ： 优化															
statistics ： 统计															
preparing ：准备															
executing ：执行															
Sending data ：发送数据															
Sorting result ：排序															
end ：结束															
query end ：查询 结束															
closing tables ： 关闭表 ／去除TMP 表															
freeing items ： 释放物品															
cleaning up ：清理															
+----------------------+----------+----------+------------+															
														

```

