---
title: HBase shell 基本操作
top: false
cover: false
toc: true
mathjax: false
date: 2020-05-09 14:46:32
password:
summary:
categories: 大数据
img:
keywords: hbase 大数据
tags: 
	- hbsse 
	- 大数据
---

* 创建表：
	* `create 'student','info'`

* 插入数据到表：
	* `put 'student','1001','info:sex','male’`
	* `put 'student','1001','info:age','18'`

* 扫描查看表数据
	* `scan 'student'`
	* `scan 'student',{STARTROW => '1001', STOPROW => '1001'}`
	* `scan 'student',{STARTROW => '1001'}`

* 更新指定字段的数据
	* `put 'student','1001','info:name','Nick'`
	* `put 'student','1001','info:age','100'`

* 查看“指定行”或“指定列族:列”的数据
	* `get 'student','1001'`
	* `get 'student','1001','info:name'`

* 删除数据
	* 删除某 rowkey 的全部数据：`deleteall 'student','1001'`
	* 删除某 rowkey 的某一列数据：`delete 'student','1001','info:sex'`

* 清空表数据
	* `truncate 'student'`

* 删除表
	* `disable 'student'`
	* `drop 'student'`