---
comments: true
date: 2011-02-28 15:30:40
layout: post
title: MySQL同时删除多表数据
wordpress_id: 140
categories: [web_dev]
tags:  [mysql, sql]

---

假设有如下三个表：

[sql]
CREATE TABLE `list` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`name` varchar(255) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8

CREATE TABLE `detail` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`list_id` int(11) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

CREATE TABLE `workflow` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`list_id` int(11) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8

[/sql]


detail和workflow表均通过list_id和list表的id关联
如果我们需要删除list表中id为1及其关联的detail和workflow表记录时，除了通过添加外键关联ON DELETE约束外，可否在一条语句中完成?即可否使用类似于如下的语句：
[sql]
delete from list,detail,workflow where list.id=detail.list_id and workflow.list_id=list.id;
[/sql]


> 答案是不行的，但是MySQL是可以支持多表删除的，具体用法如下(摘自MySQL[官方文档](http://dev.mysql.com/doc/refman/5.0/en/delete.html))

对于第一种多表删除语法，只有在FROM语句之前列出的表中的匹配记录会被删除;第二种语法中,只有在FROM语句中列出的表(USING语句之前)中匹配的记录会被删除.效果是你可以同时从多个表中删除记录，而另外一些的表可仅起辅助查询作用.
For the first multiple-table syntax, only matching rows from the tables listed before the FROM clause are deleted. For the second multiple-table syntax, only matching rows from the tables listed in the FROM clause (before the USING clause) are deleted. The effect is that you can delete rows from many tables at the same time and have additional tables that are used only for searching:

DELETE t1, t2 FROM t1 INNER JOIN t2 INNER JOIN t3
WHERE t1.id=t2.id AND t2.id=t3.id;

Or:

DELETE FROM t1, t2 USING t1 INNER JOIN t2 INNER JOIN t3
WHERE t1.id=t2.id AND t2.id=t3.id;

这些语句使用了三个表来查找需要删除的记录，但是仅仅从t1和t2表中删除符合条件的记录
These statements use all three tables when searching for rows to delete, but delete matching rows only from tables t1 and t2.

前面的例子使用的是INNER JOIN，而多表删除时也可以使用其余类型的JOIN，如LEFT JOIN等等。例如，如果需要删除在t1中存在而在t2中不存在的数据，可以使用如下的删除语句:
The preceding examples use INNER JOIN, but multiple-table DELETE statements can use other types of join permitted in SELECT statements, such as LEFT JOIN. For example, to delete rows that exist in t1 that have no match in t2, use a LEFT JOIN:

DELETE t1 FROM t1 LEFT JOIN t2 ON t1.id=t2.id WHERE t2.id IS NULL;

如果多表删除涉及了存在外键约束的InnoDB表,MySQL优化器处理多表的顺序可能会和其父子关系不一致,这样的话,该语句会执行失败并进行回滚。这种情况下，可以通过添加ON DELETE外键约束来完成多表删除。
If you use a multiple-table DELETE statement involving InnoDB tables for which there are foreign key constraints, the MySQL optimizer might process tables in an order that differs from that of their parent/child relationship. In this case, the statement fails and rolls back. Instead, you should delete from a single table and rely on the ON DELETE capabilities that InnoDB provides to cause the other tables to be modified accordingly.


对于开头的三个表删除需求，可以这么进行删除:

[sql]
delete list,detail,workflow from list,detail,workflow where list.id=detail.list_id and workflow.list_id=list.id and list.id=1;

delete list,detail,workflow from list inner join detail inner join workflow where list.id=detail.list_id and workflow.list_id=list.id and list.id=1;

delete from list,detail,workflow using list inner join workflow inner join detail where list.id=detail.list_id and workflow.list_id=list.id and list.id=1;

[/sql]
