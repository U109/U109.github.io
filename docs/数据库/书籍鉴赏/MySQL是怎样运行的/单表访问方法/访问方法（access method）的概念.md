对于我们这些MySQL的使用者来说，MySQL其实就是一个软件，平时用的最多的就是查询功能。DBA时不时丢过来一些慢查询语句让优化，我们如果连查询是怎么执行的都不清楚还优化个毛线，
所以是时候掌握真正的技术了。我们在第一章的时候就曾说过，`MySQL Server`有一个称为`查询优化器`的模块，一条查询语句进行语法解析之后就会被交给查询优化器来进行优化，优化的结果就是生成一个所谓的`执行计划`，
这个执行计划表明了应该使用哪些索引进行查询，表之间的连接顺序是啥样的，最后会按照执行计划中的步骤调用存储引擎提供的方法来真正的执行查询，并将查询结果返回给用户。不过查询优化这个主题有点儿大，
在学会跑之前还得先学会走，所以本章先来瞅瞅MySQL怎么执行单表查询（就是`FROM`子句后边只有一个表，最简单的那种查询～）。不过需要强调的一点是，在学习本章前务必看过前边关于记录结构、
数据页结构以及索引的部分，如果你不能保证这些东西已经完全掌握，那么本章不适合你。

为了故事的顺利发展，我们先得有个表：

```sql
CREATE TABLE single_table (
id INT NOT NULL AUTO_INCREMENT,
key1 VARCHAR(100),
key2 INT,
key3 VARCHAR(100),
key_part1 VARCHAR(100),
key_part2 VARCHAR(100),
key_part3 VARCHAR(100),
common_field VARCHAR(100),
PRIMARY KEY (id),
KEY idx_key1 (key1),
UNIQUE KEY idx_key2 (key2),
KEY idx_key3 (key3),
KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;
```

我们为这个`single_table`表建立了1个聚簇索引和4个二级索引，分别是：

* 为`id`列建立的聚簇索引。

* 为`key1`列建立的`idx_key1`二级索引。

* 为`key2`列建立的`idx_key2`二级索引，而且该索引是唯一二级索引。

* 为`key3`列建立的`idx_key3`二级索引。

* 为`key_part1`、`key_part2`、`key_part3`列建立的`idx_key_part`二级索引，这也是一个联合索引。

然后我们需要为这个表插入10000行记录，除`id`列外其余的列都插入随机值就好了，具体的插入语句我就不写了，自己写个程序插入吧（`id`列是自增主键列，不需要我们手动插入）。

# 访问方法（access method）的概念

想必各位都用过高德地图来查找到某个地方的路线吧（此处没有为高德地图打广告的意思，他们没给我钱，大家用百度地图也可以啊），如果我们搜西安钟楼到大雁塔之间的路线的话，地图软件会给出n种路线供我们选择，
如果我们实在闲的没事儿干并且足够有钱的话，还可以用南辕北辙的方式绕地球一圈到达目的地。也就是说，不论采用哪一种方式，我们最终的目标就是到达大雁塔这个地方。回到MySQL中来，
我们平时所写的那些查询语句本质上只是一种声明式的语法，只是告诉MySQL我们要获取的数据符合哪些规则，至于MySQL背地里是怎么把查询结果搞出来的那是MySQL自己的事儿。对于单个表的查询来说，
设计MySQL的大叔把查询的执行方式大致分为下边两种：

* 使用`全表扫描`进行查询

这种执行方式很好理解，就是把表的每一行记录都扫一遍嘛，把符合搜索条件的记录加入到结果集就完了。不管是啥查询都可以使用这种方式执行，当然，这种也是最笨的执行方式。

* 使用`索引`进行查询

因为直接使用全表扫描的方式执行查询要遍历好多记录，所以代价可能太大了。如果查询语句中的搜索条件可以使用到某个索引，那直接使用索引来执行查询可能会加快查询执行的时间。使用索引来执行查询的方式五花八门，
又可以细分为许多种类：

&emsp; 1、针对主键或唯一二级索引的等值查询

&emsp; 2、针对普通二级索引的等值查询

&emsp; 3、针对索引列的范围查询

&emsp; 4、直接扫描整个索引

设计MySQL的大叔把MySQL执行查询语句的方式称之为`访问方法`或者`访问类型`。同一个查询语句可能可以使用多种不同的访问方法来执行，虽然最后的查询结果都是一样的，但是执行的时间可能差老鼻子远了，
就像是从钟楼到大雁塔，你可以坐火箭去，也可以坐飞机去，当然也可以坐乌龟去。下边细细道来各种访问方法的具体内容。
