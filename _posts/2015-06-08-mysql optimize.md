---
layout: post
title: "MySQL优化之-工欲善其事，必先利其器"
description:
headline:
modified: 22015-06-08 20:39:21 +0800
category: mysql
tags: [mysql]
imagefeature: 
mathjax:
chart:
featured: true
---
----------
一开始从开发转数据库，觉得写SQL很简单？后来认识到too young too simple, sometimes naive. 正所谓：“查询容易，优化难，难于上青天。。。”，不过依然可以借助一些辅助工具逐渐入门，记录一些优化方式及心得。

----------
1. 优化不能漫无目的，海中捞针，自己累不说，工作也没有成效，应结合业务找出性能较差的SQL，一步一步的解决；
2. I/O访问永远是数据库性能开销的突出地方，应尽量避免多次I/O访问，也是优化最有效果的地方。（磁盘读取数据靠机械运动，每次读取数据花费的时间包括寻道时间、旋转延迟、传输时间，寻道是磁臂移动到指定磁道所需要的时间，旋转延迟指磁盘转速，传输时间指从磁盘读出或写入磁盘的时间）；
2. 结合MySQL的查询分析器（explain、profile）慢日志分析工具，反复查看执行计划，新增、修改索引，不断优化SQL；

----------
######以下是自己写的一些SQL工具：
-- 查看某表、某列的区分度情况
```sql
SELECT COUNT(DISTINCT column_name) AS '不重复记录数',
	COUNT(*) AS '总记录数',
	COUNT(DISTINCT column_name) / COUNT(*) AS '区分度',
	COUNT(*) / COUNT(DISTINCT column_name) AS '检索次数'
FROM table_name
```

 -- (查询系统表，利用列转行）计算QPS、TPS
```sql
SELECT SUM(Questions) / SUM(Uptime) AS 'QPS',
	(SUM(Com_commit) + SUM(Com_rollback)) / SUM(Uptime) AS 'TPS'
FROM (
	SELECT
		CASE VARIABLE_NAME WHEN 'Questions' THEN VARIABLE_VALUE ELSE 0 END AS 'Questions',
		CASE VARIABLE_NAME WHEN 'Uptime' 	THEN VARIABLE_VALUE ELSE 0 END AS 'Uptime',
		CASE VARIABLE_NAME WHEN 'Com_commit' THEN VARIABLE_VALUE ELSE 0 END AS 'Com_commit',
		CASE VARIABLE_NAME WHEN 'Com_rollback' THEN VARIABLE_VALUE ELSE 0 END AS 'Com_rollback'
	FROM
		information_schema.GLOBAL_STATUS
	WHERE
		VARIABLE_NAME IN ('Questions', 'Uptime', 'Com_commit', 'Com_rollback')
) AS tmp
```

-- 禁止MySQL缓存hack（通过在查询结果集加入时间，避免查询时使用缓存，影响性能分析）
```sql
SELECT t.*, now() FROM rental t 
```

-- 禁止MySQL缓存，根据数据库版本，有些时候不灵光
```sql
BEGIN;
	RESET QUERY CACHE;
	SELECT SQL_NO_CACHE t.* FROM rental t ;
END;
```

-- 强制使用指定的某个索引
```sql
SELECT t.* FROM rental t FORCE INDEX (index_update_time) 
	WHERE type = 2 AND update_time > 0 ORDER BY update_time DESC;
```