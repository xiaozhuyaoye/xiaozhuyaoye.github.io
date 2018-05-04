---
layout: post
title: Python学习笔记——基础知识
date:  2018-01-10
description:  
img:  /python_study_note/python_logo.jpg
tags: [Python]
author:  Taylor Toby
---
- \# -*- coding: utf-8 -*- 指定字符集
- <b>list</b> 
	- 列表，有序，用[]表示
	- 按照索引进行访问，可以正序可以倒序，注意不能越界
	- 添加元素：
		- 使用append()方法，将元素添加到list的结尾
		- 使用insert()方法，将元素添加到指定位置   
		  &emsp;<i>insert有两个参数，第一个参数是索引号，第二个参数是待添加的新元素。插入元素属于前插，所以通过倒叙索引进行插入数据的时候不能把数据添加到list的最后</i>
	- 删除元素：
		- 通过pop()方法删除元素，返回被删除的元素  
	      &emsp;<i>不带参数时删除最后一个元素，带参数时，参数为要删除的元素的索引</i>
	- 替换元素：
		- 通过索引重新赋值
- <b>tuple</b>
	- 元组，有序，用()表示
	- 按照索引进行访问，可以正序可以倒序，注意不能越界
	- 一旦创建不能修改
	- 注意：一个元素的元祖有歧义，所以会在元素后面加一个'，'用来与数学运算符中的小括号进行区分
- <b>dict</b>
	- 字典，无序，用{}表示，key不能重复
	- 类似Java中的map，写法像javascript中的对象
	- 也是集合，可以使用len()函数
	- 访问dict:
		- 通过d[key]访问，缺点是如果key不存在会报错
		- 通过d.get(key)访问，key不存在的时候会返回None
		- 可以通过key in d来判断dict中是否包含key
	- 遍历dict:
		- for key in d: 获取key
		- for (key,value) in d.items(): 获取键值对tuple
- <b>set</b>
	- 无序，不重复
	- 调用set()方法传入一个list获得
	- 使用add()方法添加元素
	- 使用remove()方法删除元素，如果元素不存在会报错