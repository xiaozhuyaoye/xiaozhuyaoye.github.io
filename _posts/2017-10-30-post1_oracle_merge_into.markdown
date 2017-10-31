---
layout: post
title: Oralce中merge into的用法
date:  2017-10-30
img:  /post1/oracle_logo.jpg
tags: [数据库,Oracle]
author:  Taylor Toby
---

# 使用场景

&emsp;在开发一些中后台项目时，常做的一个功能就是上传附件批量处理数据的功能，要求是上传的数据，按照某个字段（比如ID，日期）确定数据是否存在，存在更新，不存在新增。一种思路是将上传附件中的数据与数据库中的进行比对，放到list1中，剩余的放到list2中，再分别进行批量update和insert。显然这种做法效率并不高，这个时候merge into很好的解决了这个问题。

# 用法介绍

## 1.两张表数据merge into的操作

### 1.1.表和数据的准备

&emsp;这里我创建了两张节假日表holiday和holiday_temp，包括ID、日期、名称、状态、是否为节假日、创建时间和修改时间，其中记录通过日期字段进行判重，日期相同则判定数据相同。两张表的结构完全一致。

- table的结构
<img src="../assets/img/post1/table_structure.png"/>

- holiday表中的数据（红框标出来的数据与holiday_temp表中重复）
<img src="../assets/img/post1/holiday_init_data.png"/>

- holiday_temp表中的数据（红框标出来的数据与holiday表中重复）
<img src="../assets/img/post1/holiday_temp_init_data.png"/>

### 1.2.merge into的语法规则

```
MERGE into table [t_alias] 
USING [schema .] { table | view | subquery } [t_alias] 
ON ( condition ) 
WHEN MATCHED THEN merge_update_clause 
WHEN NOT MATCHED THEN merge_insert_clause;
```

### 1.3.编写sql将holiday_temp的数据合并到holiday中

```
merge into holiday h         --合并到holiday表中
using holiday_temp t         --使用holiday_temp表
on (h.holidaydate = t.holidaydate)  --指定匹配条件(注意加括号)
when matched then          --匹配的时候更新
  update
     set h.holidayname = t.holidayname,
         h.status      = t.status,
         h.isholiday   = t.isholiday
when not matched then      --不匹配的时候新增
  insert
    (holidaydate, holidayname, status, isholiday)
  values
    (t.holidaydate, t.holidayname, t.status, t.isholiday);

commit;    --提交
```

### 1.4.对比SQL执行前后的数据

&emsp;sql执行后的holiday表中的数据，可以看出刚刚重合的日期2017/10/14的holidayname,status和isholiday字段的值由holiday_temp中holidaydate相同的记录替换，且主键ID没有变更，createtime没有变化，但是modifytime变成当前时间。日期在2017/10/16以后的日期以新数据的形式插入到表中。
<img src="../assets/img/post1/holiday_after.png"/>

&emsp;在sql执行之后，holiday_temp中的值没有发生改变。
<img src="../assets/img/post1/holiday_temp_after.png">

## 2.将excel文件中的数据合并到表中

&emsp;上面介绍的是将数据库中一张表的数据合并到另外一张表的情况，但是实际应用中，尤其是一些中后台项目，经常用到的是上传xlsx、csv后缀的附件，将其中的数据合并到数据表中。这种情况也是可以使用merge into来解决的。

&emsp;那么这种情况下的sql要怎么写呢？

### 2.1.之前的准备

&emsp;假设还是上面的表结构，在此基础上进行操作。并且附件上传，分析附件，读取内容等过程在此不作介绍，这些过程之后我们的到了一个list,将excel中每一行转成了一个实体，放在list中。

### 2.2.代码实现

```
ArrayList<Holiday> list = *****  //通过一些方法，我们将excel中的每行对应一个实体，得到他们的集合

String sql = "merge into holiday h
		using (select ? holidaydate, ? holidayname, ? status, ? isholisay from dual)t //使用占位符
		  on (h.holidaydate = t.holidaydate)
		  when matched then
			update set h.holidayname = t.holidayname,h.status = t.status,h.isholiday = t.isholiday
		  when not matched then      --不匹配的时候新增
		  	insert (holidaydate, holidayname, status, isholiday)
		  	values (t.holidaydate, t.holidayname, t.status, t.isholiday);

int results = this.getJdbcTemplate().batchUpdate(sql,new BatchPreparedStatementSetter() {

		@Override
		public void setValues(PreparedStatement ps,int i) throws SQLException {
			Holiday holiday = list.get(i);
			ps.setDate(1,new java.sql.Date(holiday.getHolidatDate()));
			ps.setString(2,holiday.getHolidayDateName());
			ps.setLong(3,Holiday.getIsHoliday());
		}

		@Override
		public void getBatchSize(){
			return list.size();
		}
	})
```

通过以上的代码就能将读取到的数据批量的合并到表中