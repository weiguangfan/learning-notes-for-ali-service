--odps sql
--********************************************************************--
--author:算法工程师-韦广繁
--create time:2021-11-29 11:58:47
--********************************************************************--
-- SELECT语法
-- MaxCompute支持通过select语句查询数据。本文为您介绍select命令格式及如何实现嵌套查询、分组查询、排序等操作。
-- select语句用于从表中选取满足指定条件的数据
-- 当使用select语句时，屏显最多只能显示10000行结果。当select语句作为子句时则无此限制，select子句会将全部结果返回给上层查询。
-- select语句查询分区表时默认禁止全表扫描。
-- 如果您需要对分区表进行全表扫描，可以在全表扫描的SQL语句前加上命令set odps.sql.allow.fullscan=true;
-- 如果整个项目都需要开启全表扫描，项目空间Owner执行如下命令打开开关：setproject odps.sql.allow.fullscan=true;
> set odps.sql.allow.fullscan = true;

> SELECT * FROM test_02;

show CREATE table test_01;

DESC test_01;

SHOW CREATE TABLE test_02;

DESC test_02;

SHOW PARTITIONS test_02;

-- 列表达式（select_expr）
-- select_expr格式为col1_name, col2_name, 列表达式,...，表示待查询的普通列、分区列或正则表达式。
-- 用列名指定要读取的列。
SELECT shop_name FROM test_02 WHERE sale_date = '2013' and region = 'china';
-- 用星号（*）代表查询所有的列。可配合where子句指定过滤条件。
SELECT * FROM test_02 WHERE sale_date = '2013' and region = 'china'and total_price >=100.2;

-- 可以使用正则表达式。
-- 选出test_02表中排除列名以t开头的其它列。
SELECT `(t.*)?+.+` FROM test_02;
SELECT `s.*` FROM test_02;
SELECT `(customer_id)?+.+` FROM test_02;
SELECT `(shop_name)?+.+` FROM test_02;

-- 选出test_02表中排除shop_name和customer_id两列的其它列。
SELECT `(shop_name|customer_id)?+.+` FROM test_02;

-- 在排除多个列时，如果col2是col1的前缀，则需保证col1写在col2的前面（较长的col写在前面）。
SELECT `(sh.*|s.*)?+.+` FROM test_02;

-- 选出test_02表中所有列名以sh开头的列
SELECT `sh.*` FROM test_02;

SELECT * FROM test_02 WHERE shop_name = 's1';

SELECT sale_date,region FROM test_02;

SELECT shop_name FROM test_02;

set odps.sql.allow.fullscan = true;
SELECT sale_date,region FROM test_02;

-- 在选取的列名前可以使用distinct去掉重复字段，只返回去重后的值。使用all会返回字段中所有重复的值。不指定此选项时，默认值为all。

SELECT DISTINCT region FROM test_02;
SELECT ALL region FROM test_02;

-- 去重多列时，distinct的作用域是select的列集合，不是单个列。
SELECT DISTINCT sale_date,region FROM test_02;


-- 嵌套子查询。
SELECT region,sale_date FROM test_02;
SELECT * FROM (SELECT region,sale_date FROM test_02)t where region = 'china';

-- 目标表信息（table_reference）
-- 必填。table_reference表示查询的目标表信息。
-- 直接指定目标表名。
SELECT customer_id FROM test_02;

-- WHERE子句（where_condition）
-- 可选。where子句为过滤条件。如果表是分区表，可以实现列裁剪。
-- 在where子句中，您可以指定分区范围，只扫描表的指定部分，避免全表扫描。
-- 在列表达式（select_expr）中，如果被重命名的列字段（赋予了列别名）使用了函数，则不能在where子句中引用列别名。
SELECT * FROM test_02 WHERE sale_date >= '2008' and sale_date <= '2014';
SELECT * FROM test_02 WHERE sale_date BETWEEN 2008 and 2014;

-- GROUP BY分组查询（col_list）
-- 可选。通常，group by和聚合函数配合使用，根据指定的普通列、分区列或正则表达式进行分组。group by使用规则如下：
-- group by操作优先级高于select操作，因此group by的取值是select输入表的列名或由输入表的列构成的表达式。需要注意的是：
-- group by取值为正则表达式时，必须使用列的完整表达式。
-- select语句中没有使用聚合函数的列必须出现在group by中。
-- 直接使用输入表列名region作为group by的列，即以region值分组。
SELECT region FROM test_02 GROUP BY region;

-- select的所有列中没有使用聚合函数的列，必须出现在group by中，否则返回报错
SELECT region,total_price FROM test_02 GROUP BY region,total_price;

-- 以列表达式分组，必须使用列的完整表达式。
SELECT 2 + total_price as t from test_02 GROUP BY 2 + total_price;

-- 以select列的别名分组
SELECT region as r from test_02 GROUP BY region;
SELECT region as r from test_02 GROUP BY r;

-- 以region值分组，返回每一组的销售额总量。
SELECT SUM(total_price) FROM test_02 GROUP BY region;

-- 以region值分组，返回每一组的region值（组内唯一）及销售额总量。
SELECT region,SUM(total_price) FROM test_02 GROUP BY region;
-- 当SQL语句设置了属性，即set odps.sql.groupby.position.alias=true;，group by中的整型常量会被当做select的列序号处理。
--与下一条SQL语句一起执行。
set odps.sql.groupby.position.alias=true;
--1代表select的列中第一列即region，以region值分组，返回每一组的region值（组内唯一）及销售额总量。
select region, sum(total_price) from test_02 group by 1;

insert into test_02 partition (sale_date='2014', region='shanghai') values ('null','c5',null),('s6','c6',100.4),('s7','c7',100.5);

-- HAVING子句（having_condition）
-- 可选。通常having子句与聚合函数一起使用，实现过滤。
-- 使用having子句配合聚合函数实现过滤。
SELECT region,SUM(total_price) FROM test_02 GROUP BY region HAVING SUM(total_price) <305;


setproject odps.sql.allow.fullscan=true;
setproject odps.sql.allow.fullscan=false;

-- ORDER BY全局排序（order_condition）
-- 可选。order by用于对所有数据按照指定普通列、分区列或指定常量进行全局排序。order by使用规则如下：
-- 默认对数据进行升序，如果降序排序，需要使用desc关键字。
-- order by默认要求带limit数据行数限制，没有limit会返回报错。如您需要解除order by必须带limit的限制，详情请参见LIMIT NUMBER限制输出行数>解除ORDER BY必须带LIMIT的限制。
-- Session级别：设置set odps.sql.validate.orderby.limit=false;关闭order by必须带limit的限制，需要与SQL语句一起提交。
-- Session级别：设置set odps.sql.validate.orderby.limit=false;关闭order by必须带limit的限制，需要与SQL语句一起提交。
-- 如果关闭order by必须带limit的限制，在单个执行节点有大量数据排序的情况下，资源消耗或处理时长等性能表现会受到影响。
set odps.sql.validate.orderby.limit=false;
SETPROJECT  odps.sql.validate.orderby.limit=false;

-- 查询表test_02的信息，并按照total_price升序排列前3条。
-- 在使用order by排序时，NULL会被认为比任何值都小，这个行为与MySQL一致，但是与Oracle不一致。
SELECT * FROM test_02 ORDER by total_price LIMIT 3;

-- 查询表test_02的信息，并按照total_price降序排列前3条。
SELECT * FROM test_02 order by total_price DESC LIMIT 3;
SELECT * FROM test_02 ORDER by total_price DESC;
select * FROM test_02 ORDER BY total_price;

-- order by后面需要加上select列的别名。当select某列时，如果没有指定列的别名，则列名会被作为列的别名。
-- order by加列的别名。
select total_price as t from test_02 order by total_price limit 3;
select total_price as t from test_02 order by t limit 3;

-- 当SQL语句设置了属性，即set odps.sql.orderby.position.alias=true;，order by中的整型常量会被当做select的列序号处理。
--与下一条SQL语句一起执行。
set odps.sql.orderby.position.alias = true;
SELECT * FROM test_02 order by 3 LIMIT 3;
-- offset可以和order by...limit语句配合使用，用于指定跳过的行数，格式为order by...limit m offset n，也可以简写为order by...limit n, m。
-- 其中：limit m控制输出m行数据，offset n表示在开始返回数据之前跳过的行数。offset 0与省略offset子句效果相同。
SELECT * FROM test_02 ORDER BY total_price LIMIT 5,2;

-- DISTRIBUTE BY哈希分片（distribute_condition）
-- 可选。distribute by用于对数据按照某几列的值做Hash分片。
-- distribute by控制Map（读数据）的输出在Reducer中是如何划分的，
-- 如果不希望Reducer的内容存在重叠，或需要对同一分组的数据一起处理，您可以使用distribute by来保证同组数据分发到同一个Reducer中。
-- 必须使用select的输出列别名，当select某列时，如果没有指定列的别名，则列名会被作为列的别名。
--查询表test_02中的列region值并按照region值进行哈希分片。
SELECT region FROM test_02 DISTRIBUTE BY region;
--等价于如下语句。
select region as r from test_02 distribute by region;
SELECT region AS r from test_02 DISTRIBUTE BY r;

-- SORT BY局部排序（sort_condition）
-- 可选。通常，配合distribute by使用。sort by使用规则如下：
-- sort by默认对数据进行升序，如果降序排序，需要使用desc关键字。
-- 如果sort by语句前有distribute by，sort by会对distribute by的结果按照指定的列进行排序。
-- 查询表test_02中的列region和total_price的值并按照region值进行哈希分片，然后按照total_price对哈希分片结果进行局部升序排序。
--排序不起作用
SELECT region,total_price FROM test_02 distribute by region SORT by total_price;
SELECT region,total_price FROM test_02 distribute by region;
SELECT region,total_price FROM test_02 distribute by region SORT by total_price desc;

-- 如果sort by语句前没有distribute by，sort by会对每个Reduce中的数据进行局部排序。
-- 保证每个Reduce的输出数据都是有序的，从而增加存储压缩率，同时读取时如果有过滤，能够减少真正从磁盘读取的数据量，提高后续全局排序的效率
select region,total_price from test_02 sort by total_price desc;

-- SELECT语序
-- 对于按照select语法格式书写的select语句，它的逻辑执行顺序与标准的书写语序并不相同。
-- 基于order by不和distribute by、sort by同时使用，group by也不和distribute by、sort by同时使用的限制，
-- 场景1：from->where->group by->having->select->order by->limit
-- 场景2：from->where->select->distribute by->sort by


-- 该命令的执行逻辑如下：
-- 从test_02表（from test_02）中取出满足where total_price > 100条件的数据。
-- 对于i中得到的结果按照region进行分组（group by region）。
-- 对于ii中得到的结果筛选分组中满足total_price之和大于305的数据（having sum(total_price)>305）。
-- 对于iii中得到的结果select region,max(total_price)。
-- 对于iv中得到的结果按照region进行排序（order by region）。
-- 对于v中得到的结果仅显示前5条数据（limit 5）。
SELECT region,MAX(total_price) FROM test_02 WHERE total_price >100 GROUP BY region HAVING SUM(total_price) > 305 ORDER by region LIMIT 5;
-- 该命令的执行逻辑如下：
-- 从sale_detail表（from sale_detail）中取出满足where total_price > 100.2条件的数据。
-- 对于i中得到的结果select shop_name, total_price, region。
-- 对于ii中得到的结果按照region进行哈希分片（distribute by region）。
-- 对于iii中得到的结果按照total_price进行升序排列（sort by total_price）。

SELECT shop_name,total_price,region FROM test_02 WHERE total_price > 100.2 DISTRIBUTE BY region sort by total_price;

SELECT DISTINCT region FROM test_02;
SELECT shop_name,region FROM test_02;
SELECT DISTINCT shop_name,region FROM test_02;
SELECT customer_id FROM test_02;

-- 子查询（SUBQUERY）
-- 当您需要在某个查询的执行结果基础上进一步执行查询操作时，可以通过子查询操作实现。
-- 子查询指在一个完整的查询语句之中，嵌套若干个不同功能的小查询，从而一起完成复杂查询的一种编写形式。
-- 基础子查询
-- 普通查询操作的对象是目标表，但是查询的对象也可以是另一个select语句，这种查询为子查询。
-- 在from子句中，子查询可以被当作一张表，与其他表或子查询进行join操作。join详情请参见JOIN。
SELECT * FROM (SELECT shop_name FROM test_02) a;

SELECT (SELECT * FROM test_02 WHERE shop_name='s1') FROM test_02;

SELECT * FROM test_02 WHERE shop_name='s1';

SELECT * FROM (SELECT shop_name,region FROM test_02)t WHERE region = 'china';
-- 使用格式1子查询语法，在from子句中，子查询可以被当做一张表，与其他的表或子查询进行join操作。
-- 先新建一张表，再执行join操作。
create table test_05 as SELECT shop_name,customer_id,total_price FROM test_02;
SELECT * FROM test_05;
SELECT a.shop_name,a.customer_id,a.total_price FROM (SELECT * FROM test_05) a JOIN test_02 on a.shop_name = test_02.shop_name;

CREATE TABLE test_06 like test_02;
DESC test_06;
SELECT * FROM test_06;
alter table test_06 add partition (sale_date='2013', region='china');
SHOW PARTITIONS test_06;
insert into test_06 partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s5','c2',100.2);

-- JOIN
-- MaxCompute支持通过join操作连接表并返回符合连接条件和查询条件的数据。
-- 本文为您介绍左连接、右连接、全连接、内连接、自然连接、隐式连接和多路连接的使用方法。
-- MaxCompute支持如下join操作：
-- 左连接（left outer join）
-- 可简写为left join。返回左表中的所有记录，即使右表中没有与之匹配的记录。
-- 右连接（right outer join）
-- 可简写为right join。返回右表中的所有记录，即使左表中没有与之匹配的记录。
-- 全连接（full outer join）
-- 可简写为full join。返回左右表中的所有记录。
-- 内连接（inner join）
-- 关键字inner可以省略。左右表中至少存在一个匹配行时，inner join返回数据行。
-- 自然连接（natural join）
-- 参与join的两张表根据字段名称自动决定连接字段。支持outer natural join，支持使用using子句执行join，输出字段中公共字段只出现一次。
-- 隐式连接
-- 即不指定join关键字执行连接。
-- 多路连接
-- 多路join连接。支持通过括号指定join的优先级，括号内的join优先级较高。
-- 如果SQL语句中包含where过滤条件，且join在where条件之前，先进行join操作，然后对join的结果执行where条件过滤，获取的结果是两个表的交集，而不是全表。


-- 左连接。
--分区表需要开启全表扫描功能，否则join操作会执行失败。
set odps.sql.allow.fullscan=true;
--由于表test_02及test_06中都有shop_name列，因此需要在select子句中使用别名进行区分。
SELECT a.shop_name as ashop_name,b.shop_name as bshop_name FROM test_02 a LEFT JOIN test_06 b on a.shop_name = b.shop_name;

-- 右连接。
--分区表需要开启全表扫描功能，否则join操作会执行失败。
set odps.sql.allow.fullscan=true;
--由于表test_02及test_06中都有shop_name列，因此需要在select子句中使用别名进行区分。
SELECT a.shop_name as ashop_name,b.shop_name as bshop_name FROM test_02 a RIGHT JOIN test_06 b on a.shop_name = b.shop_name;

-- 全连接。
--分区表需要开启全表扫描功能，否则join操作会执行失败。
set odps.sql.allow.fullscan=true;
--由于表test_02及test_06中都有shop_name列，因此需要在select子句中使用别名进行区分。
SELECT a.shop_name as ashop_name,b.shop_name as bshop_name FROM test_02 a full JOIN test_06 b on a.shop_name = b.shop_name;

-- 内连接。
--分区表需要开启全表扫描功能，否则join操作会执行失败。
set odps.sql.allow.fullscan=true;
--由于表test_02及test_06中都有shop_name列，因此需要在select子句中使用别名进行区分。
SELECT a.shop_name as ashop_name,b.shop_name as bshop_name FROM test_02 a inner JOIN test_06 b on a.shop_name = b.shop_name;

-- 自然连接。
--分区表需要开启全表扫描功能，否则join操作会执行失败。
--相当于显示交集。
set odps.sql.allow.fullscan=true;
SELECT * FROM test_02 a natural JOIN test_06 b;
--等效于如下语句。
SELECT a.* FROM test_02 a inner JOIN test_06 b
on a.shop_name = b.shop_name and a.customer_id =b.customer_id and a.total_price = b.total_price and a.sale_date = b.sale_date and a.region = b.region;

-- 隐式连接
--分区表需要开启全表扫描功能，否则join操作会执行失败。
--慎用！相当于将交集列，进行合并了。
set odps.sql.allow.fullscan=true;
--隐式连接。
select * from test_02, test_06 where test_02.shop_name = test_06.shop_name;
--等效于如下语句。
SELECT * FROM test_02 join test_06 on test_02.shop_name = test_06.shop_name;

-- 多路连接，不指定优先级。
--分区表需要开启全表扫描功能，否则join操作会执行失败。
set odps.sql.allow.fullscan=true;
SELECT a.* FROM test_06 a FULL JOIN test_02 b on a.shop_name = b.shop_name full join test_05 c on a.shop_name = c.shop_name;

-- 多路连接，通过括号指定优先级。
SELECT * FROM test_06 a JOIN (test_02 b JOIN test_05 c on b.shop_name = c.shop_name) on a.shop_name = b.shop_name;

-- join与where相结合，查询两表中region为china且shop_name一致的记录数，保留test_02表的全部记录。
SELECT a.shop_name,a.customer_id,a.total_price
FROM (SELECT * FROM test_06 WHERE region = 'china') a
LEFT JOIN (SELECT * FROM test_02 WHERE region = 'china') b
on a.shop_name = b.shop_name;

-- SEMI JOIN（半连接）
-- MaxCompute支持半连接操作，通过右表过滤左表的数据，右表的数据不出现在结果集中。
-- 本文为您介绍半连接中left semi join和left anti join两种语法的使用方法。

-- left semi join
-- 当join条件成立时，返回左表中的数据。如果左表中满足指定条件的某行数据在右表中出现过，则此行保留在结果集中。
-- 在MaxCompute中，与left semi join类似的操作为in subquery，请参见IN SUBQUERY。您可以自行选择其中一种方式。

-- left anti join
-- 当join条件不成立时，返回左表中的数据。如果左表中满足指定条件的某行数据没有在右表中出现过，则此行保留在结果集中。
-- 在MaxCompute中，与left anti join类似的操作为not in subquery，但并不完全相同，请参见NOT IN SUBQUERY。

create table if not exists test_07
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);

alter table test_07 add partition (sale_date='2013', region='china');

insert into test_07 partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s5','c2',100.2),('s2','c2',100.2);
insert into test_06 partition (sale_date='2013', region='china') values ('s3','c3',100.3),('s4','c4',100.4);
insert into test_07 partition (sale_date='2013', region='china') values ('s7','c7',100.7);
-- 查询test_06表中，total_price出现在test_07表中的数据集。
-- 只会返回test_06中的数据，只要test_06的total_price在test_07的total_price中出现过。
SELECT * FROM test_06 a LEFT SEMI JOIN test_07 b on a.total_price = b.total_price;

-- 查询test_06表中，total_price没有出现在test_07表中的数据集。
-- 只会返回test_06中的数据，只要test_06的total_price在test_07的total_price中没有出现过。
SELECT * FROM test_06 a LEFT ANTI JOIN test_07 b on a.total_price = b.total_price;




-- IN SUBQUERY
-- in subquery与left semi join用法类似。
-- 格式1
-- select <select_expr1> from <table_name1> where <select_expr2> in (select <select_expr2> from <table_name2>);
--等效于left semi join如下语句。
-- select <select_expr1> from <table_name1> <alias_name1> left semi join <table_name2> <alias_name2> on <alias_name1>.<select_expr1> = <alias_name2>.<select_expr2>;

SELECT * FROM test_06 WHERE shop_name in (SELECT shop_name FROM test_07);
--等效于left semi join如下语句。
SELECT * FROM test_06 a LEFT SEMI JOIN test_07 b ON a.shop_name = b.shop_name;

-- MaxCompute不仅支持in subquery，还支持Correlated条件。
-- 子查询中的where <table_name2_colname> = <table_name1>.<colname>即是一个Correlated条件。
-- MaxCompute 2.0已支持这种用法，这种过滤条件构成了semi join中on条件的一部分
-- 格式2
-- select <select_expr1> from <table_name1> where <select_expr2> in (select <select_expr2> from <table_name2> where <table_name2_colname> = <table_name1>.<colname>);

-- 使用格式2子查询语法。
SELECT * FROM test_06 WHERE shop_name in (SELECT shop_name FROM test_07 WHERE customer_id = test_06.customer_id);

-- in subquery不作为join条件。
-- 因为where中包含了and，所以无法转换为semi join，会单独启动作业执行子查询。
SELECT * FROM test_06 WHERE shop_name in (SELECT shop_name FROM test_07) and total_price > 100.1;

-- 格式3
-- 在上述能力及限制的基础上，兼容PostgreSQL支持多列的需求，相较于拆分为多个Subquery的实现方式，会减少一次JOIN过程并节省计算资源。支持的多列用法如下：
-- in后的表达式可以为简单的SELECT多列语句。
-- in后的表达式中可以使用聚合函数。更多聚合函数信息，请参见聚合函数。
-- in后的表达式可以为常量。
-- SELECT多列场景。

-- 为方便理解，此处重新构造示例数据。
create table if not exists test_08(a bigint,b bigint,c bigint,d bigint,e bigint);
create table if not exists test_09(a bigint,b bigint,c bigint,d bigint,e bigint);
insert into table test_08 values (1,3,2,1,1),(2,2,1,3,1),(3,1,1,1,1),(2,1,1,0,1),(1,1,1,0,1);
insert into table test_09 values (1,3,5,0,1),(2,2,3,1,1),(3,1,1,0,1),(2,1,1,0,1),(1,1,1,0,1);
--场景一：in后的表达式为简单的SELECT多列语句。
SELECT a,b from test_08 WHERE (c,d) in (SELECT a,b FROM test_09 WHERE e = test_08.e);

-- in后的表达式使用聚合函数。
SELECT a,b from test_08 WHERE (c,d) in (SELECT MAX(a),b FROM test_09 WHERE e = test_08.e GROUP BY b HAVING MAX(a)>0);

-- in后的表达式为常量。
SELECT * FROM test_08 WHERE (SELECT COUNT(*) FROM test_09 WHERE test_08.a = test_09.a) = 3;
SELECT a,b from test_08 WHERE (c,d) in ((1,3),(1,1));


-- NOT IN SUBQUERY
-- not in subquery与left anti join用法类似，但并不完全相同。
-- 如果查询目标表的指定列名中有任意一行为NULL，则not in表达式值为NULL，导致where条件不成立，无数据返回，这点与left anti join不同。
-- 格式1
-- select <select_expr1> from <table_name1> where <select_expr2> not in (select <select_expr2> from <table_name2>);
--等效于left anti join如下语句。
-- select <select_expr1> from <table_name1> <alias_name1> left anti join <table_name2> <alias_name2> on <alias_name1>.<select_expr1> = <alias_name2>.<select_expr2>;
select * from test_06 where shop_name not in (select shop_name from test_07);

-- 格式2
-- MaxCompute不仅支持not in subquery，还支持Correlated条件。
-- 子查询中的where <table_name2_colname> = <table_name1>.<colname>即是一个Correlated条件。
-- MaxCompute 2.0已支持这种用法，这种过滤条件构成了anti join中on条件的一部分。
SELECT a,b from test_08 WHERE (c,d) not in (SELECT a,b FROM test_09 WHERE e = test_08.e);

-- 格式3
-- 在上述能力的基础上，兼容PostgreSQL支持多列的需求，相较于拆分为多个Subquery的实现方式，会减少一次JOIN过程并节省计算资源。支持的多列场景如下：
-- not in后的表达式可以为简单的SELECT多列语句。
-- not in后的表达式中可以使用聚合函数。更多聚合函数信息，请参见聚合函数。
-- not in后的表达式可以为常量。
SELECT a,b from test_08 WHERE (c,d) not in (SELECT a,b FROM test_09 WHERE e = test_08.e);
SELECT a,b from test_08 WHERE (c,d) not in (SELECT MAX(a),b FROM test_09 WHERE e = test_08.e GROUP BY b HAVING MAX(a)>0);
SELECT a,b from test_08 WHERE (c,d) not in ((1,3),(1,1));

-- EXISTS SUBQUERY
-- 使用exists subquery时，当子查询中有至少一行数据时，返回True，否则返回False。
-- MaxCompute只支持含有Correlated条件的where子查询。exists subquery实现的方式是转换为left semi join。
-- select <select_expr> from <table_name1> where exists (select <select_expr> from <table_name2> where <table_name2_colname> = <table_name1>.<colname>);
SELECT * FROM test_07 WHERE EXISTS (SELECT * FROM test_06 WHERE customer_id = test_07.customer_id);
--等效于以下语句。
SELECT * FROM test_07 a LEFT SEMI JOIN test_06 b on a.customer_id = b.customer_id;

-- NOT EXISTS SUBQUERY
-- 使用not exists subquery时，当子查询中无数据时，返回True，否则返回False。
-- MaxCompute只支持含有Correlated条件的where子查询。not exists subquery实现的方式是转换为left anti join。
SELECT * FROM test_07 WHERE NOT EXISTS (SELECT * FROM test_06 WHERE shop_name = test_07.shop_name);
--等效于以下语句。
SELECT * FROM test_07 a LEFT ANTI JOIN test_06 b on a.shop_name = b.shop_name;


-- SCALAR SUBQUERY
-- 当子查询的输出结果为单行单列时，可以做为标量使用，即可以参与标量运算。
-- 所有的满足一行一列输出值的子查询，都可以按照如下命令格式重写。如果查询的结果只有一行，在外面嵌套一层max或min操作，其结果不变。
-- select <select_expr> from <table_name1> where (<select count(*) from <table_name2> where <table_name2_colname> = <table_name1>.<colname>) <标量运算符> <scalar_value>;
--等效于以下语句。
-- select <table_name1>.<select_expr> from <table_name1> left semi join (select <colname>, count(*) from <table_name2> group by <colname> having count(*) <标量运算符> <scalar_value>) <table_name2> on <table_name1>.<colname> = <table_name2>.<colname>;
select * from test_07 where (select count(*) from test_06 where test_06.shop_name = test_07.shop_name) >= 1;
--等效于以下语句。
SELECT * FROM test_07 LEFT SEMI JOIN (SELECT shop_name ,COUNT(*) FROM test_06 GROUP BY shop_name HAVING COUNT(*) >= 1 ) test_06 on test_07.shop_name = test_06.shop_name;

-- SCALAR SUBQUERY还支持多列用法如下：
-- SELECT列为包含多列的SCALAR SUBQUERY表达式，只支持等值表达式。
-- SELECT列可以为BOOLEAN表达式，只支持等值比较。
-- where支持多列比较，只支持等值比较。

-- scalar subquery支持引用外层查询的列，当嵌套多层scalar subquery时，只支持引用直接外层的列。
-- scalar subquery只能在where子句中使用。
--允许的操作。
select * from test_07 where (select count(*) from test_06 where test_06.shop_name = test_07.shop_name) = 1;

--为方便理解，此处重新构造示例数据。
create table if not exists test_11(a bigint,b bigint,c double);
create table if not exists test_12(a bigint,b bigint,c double);
insert into table test_11 values (1,3,4.0),(1,3,3.0);
insert into table test_12 values (1,3,4.0),(1,3,5.0);
SELECT * FROM test_11;
SELECT * FROM test_12;
--场景一：SELECT列为包含多列的SCALAR SUBQUERY表达式，只支持等值表达式。
SELECT (SELECT a,b FROM test_12 WHERE c = test_11.c) as (a,b),a FROM test_11;
--场景二：SELECT列为BOOLEAN表达式，只支持等值比较。
SELECT (a,b) = (SELECT a,b FROM test_11 WHERE c = test_12.c) FROM test_12;
--场景三：where支持多列比较，只支持等值比较。
SELECT * FROM test_12 WHERE c > 3.0 and (a,b) = (SELECT a,b FROM test_11 WHERE c = test_12.c);

SELECT * FROM test_12 WHERE c > 3.0 or (a,b) = (SELECT a,b FROM test_11 WHERE c = test_12.c);

-- MAPJOIN HINT
-- 当您对一个大表和一个或多个小表执行join操作时，可以在select语句中显式指定mapjoin Hint提示以提升查询性能。本文为您介绍如何通过mapjoin hint连接表。
-- 整个JOIN过程包含Map、Shuffle和Reduce三个阶段。通常情况下，join操作在Reduce阶段执行表连接。
-- mapjoin在Map阶段执行表连接，而非等到Reduce阶段才执行表连接，可以缩短大量数据传输时间，提升系统资源利用率，从而起到优化作业的作用。
-- 在对大表和一个或多个小表执行join操作时，mapjoin会将您指定的小表全部加载到执行join操作的程序的内存中，在Map阶段完成表连接从而加快join的执行速度。
-- 此外，MaxCompute SQL不支持在普通join的on条件中使用不等值表达式、or等逻辑复杂的join条件，但是在mapjoin中可以进行上述操作。

-- mapjoin操作的使用限制如下：
-- mapjoin在Map阶段会将指定表的数据全部加载在内存中，因此指定的表仅能为小表，且表被加载到内存后占用的总内存不得超过512 MB。由于MaxCompute是压缩存储，因此小表在被加载到内存后，数据大小会急剧膨胀。此处的512 MB是指加载到内存后的空间大小。
-- mapjoin中join操作的限制如下：
-- left outer join的左表必须是大表。
-- right outer join的右表必须是大表。
-- 不支持full outer join。
-- inner join的左表或右表均可以是大表。
-- mapjoin最多支持指定128张小表，否则报语法错误。

-- 您需要在select语句中使用Hint提示/*+ mapjoin(<table_name>) */才会执行mapjoin。需要注意的是：
-- 引用小表或子查询时，需要引用别名。
-- mapjoin支持小表为子查询。
-- 在mapjoin中，可以使用不等值连接或or连接多个条件。您可以通过不写on语句而通过mapjoin on 1 = 1的形式，实现笛卡尔乘积的计算。
-- 例如select /*+ mapjoin(a) */ a.id from shop a join table_name b on 1=1;，但此操作可能带来数据量膨胀问题。
-- mapjoin中多个小表用英文逗号（,）分隔，例如/*+ mapjoin(a,b,c)*/。

-- 对表test_06和test_07执行join操作，满足test_06中的total_price小于test_07中的total_price或test_06中的total_price与sale_detail中的total_price之和小于500的条件
SELECT /*+ MAPJOIN(a) */ a.shop_name,a.total_price,b.total_price FROM test_06 a JOIN test_07 b ON a.total_price < b.total_price or a.total_price + b.total_price < 500;

SELECT /*+ MAPJOIN(a) */ a.shop_name,a.total_price,b.total_price FROM test_06 a JOIN test_07 b ON a.total_price < b.total_price;

SELECT /*+ MAPJOIN(a) */ a.shop_name,a.total_price,b.total_price FROM test_06 a JOIN test_07 b ON  a.total_price + b.total_price < 500;



create table if not exists test_10
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);
alter table test_10 add partition (sale_date='2013', region='china');
insert into test_10 partition (sale_date='2013', region='china') values ('null','null',null),('s2','c2',100.2),('s3','c3',100.3);

SELECT * FROM test_10;
SELECT * FROM test_10 WHERE shop_name not in (SELECT shop_name FROM test_06);


select * from values (1, 2), (1, 2), (3, 4), (5, 6) t(a, b)
intersect all
select * from values (1, 2), (1, 2), (3, 4), (5, 7) t(a, b);

select * from values (1, 2), (1, 2), (3, 4), (5, 6) t(a, b)
intersect distinct
select * from values (1, 2), (1, 2), (3, 4), (5, 7) t(a, b);

-- SKEWJOIN HINT
-- 当两张表Join存在热点，导致出现长尾问题时，您可以通过取出热点key，将数据分为热点数据和非热点数据两部分处理，最后合并的方式，提高Join效率。
-- SkewJoin Hint可以通过自动或手动方式获取两张表的热点key，分别计算热点数据和非热点数据的Join结果并合并，加快Join的执行速度。
-- 您需要在select语句中使用Hint提示/*+ skewJoin(<table_name>[(<column1_name>[,<column2_name>,...])][((<value11>,<value12>)[,(<value21>,<value22>)...])]*/才会执行MapJoin。
-- table_name为倾斜表名，column_name为倾斜列名，value为倾斜key值。

--方法1：Hint表名（注意Hint的是表的alias）。
select /*+ skewjoin(a) */ * from test_06 a join test_07 b on a.shop_name = b.shop_name and a.customer_id = b.customer_id;

--方法2：Hint表名和认为可能产生倾斜的列，例如表a的c0和c1列存在数据倾斜。
SELECT /*+ skewjoin(a(shop_name,customer_id)) */ * FROM test_06 a join test_07 b on a.shop_name = b.shop_name and a.customer_id = b.customer_id and a.total_price = b.total_price;

--方法3：Hint表名和列，并提供发生倾斜的key值。如果是STRING类型，需要加上引号。例如(a.c0=1 and a.c1="2")和(a.c0=3 and a.c1="4")的值都存在数据倾斜。
SELECT /*+ skewjoin(a(shop_name,customer_id)(("s2","c2"),("s5","c2"))) */ * FROM test_06 a join test_07 b on a.shop_name = b.shop_name and a.customer_id = b.customer_id and a.total_price = b.total_price;

-- 热值Key指出现次数很多的key值。
-- 在不加SkewJoin Hint的情况下，将表T0和表T1进行Join，由于T0和T1的数量都很大，只能进行MergeJoin，因此相同的热值都会Shuffle到一个节点，导致数据倾斜。
-- 加SkewJoin Hint后，优化器会运行一个Aggregate动态获取重复行数前20的热值，并将表T0中属于热值的值（数据A）、T0中不属于热值的值（数据B）拆分；
-- 将表T1中能与T0中热值Join的值（数据C）、表T1中不能与T0中热值Join的值（数据D）进行拆分。
-- 然后将数据A与数据C进行MapJoin（由于数据C量很少，可以进行MapJoin），
-- 将数据B和数据D进行MergeJoin。
-- 最后将MapJoin和MergeJoin的结果Union，生成最后的结果

-- SkewJoin Hint支持的Join类型：
-- Inner Join可以对Join两侧表中的任意一侧进行Hint。
-- Left Join、Semi Join和Anti Join只可以Hint左侧表。
-- Right Join只可以Hint右侧表。
-- Full Join不支持Skew Join Hint。
-- 建议只对一定会出现数据倾斜的Join添加Hint，因为Hint会运行一个Aggregate，存在一定代价。
-- 被Hint的Join的Left Side Join Key的类型需要与Right Side Join Key的类型一致，否则SkewJoin Hint不生效。
-- 您可以通过在子查询中将Join key进行Cast从而保持一致。
-- create table T0(c0 int, c1 int, c2 int, c3 int);
-- create table T1(c0 string, c1 int, c2 int);
--方法1：
-- select /*+ skewjoin(a) */ * from T0 a join T1 b on cast(a.c0 as string) = cast(b.c0 as string) and a.c1 = b.c1;
-- --方法2：
-- select /*+ skewjoin(b) */ * from (select cast(a.c0 as string) as c00 from T0 a) b join T1 c on b.c00 = c.c0;

-- 加SkewJoin Hint后，优化器会运行一个Aggregate获取前20的热值。20是默认值，您可以通过set odps.optimizer.skew.join.topk.num = xx;进行设置。
-- SkewJoin Hint只支持对Join其中一侧进行Hint。
-- 被Hint的Join一定要有left key = right key，不支持笛卡尔积Join。
-- 与其它Hint一起使用的方法如下，但需要注意，被MapJoin Hint的Join不能再添加SkewJoin Hint。
-- select /*+ mapjoin(c), skewjoin(a) */ * from T0 a join T1 b on a.c0 = b.c3 join T2 c on a.c0 = c.c7;
-- 您可以在Logview的Json Summary中搜索是否出现topk_agg字段判断SkewJoin Hint是否生效


-- 动态过滤器（Dynamic Filter）
-- JOIN是分布式系统中常见的操作，同时也是一个耗时、耗资源的操作，因为其涉及到的Shuffle操作尤其在海量数据场景下，会耗费较多的资源和时间。
-- 针对Shuffle操作，MaxCompute可以利用JOIN本身的等值连接属性进行优化。

-- 一个典型的包含JOIN的SQL语句如下：
-- select * from (table1) A join (table2) B on A.a= B.b;
-- 基于JOIN等值连接的特性，MaxCompute可以通过表A的数据生成一个过滤器，在Shuffle或JOIN之前提前过滤表B的数据，甚至可以将过滤器下推到底层存储，在源头过滤数据。
-- 这种在作业运行时动态生成过滤器的功能称为动态过滤器DF（Dynamic Filter）
-- 动态过滤器功能是利用等值JOIN的特性，基于运行时动态生成过滤器，以便在Shuffle或JOIN之前提前过滤数据，实现加速查询运行。
-- 该功能适用于维度表和事实表执行JOIN的场景。

-- 动态范围过滤器或布隆过滤器（Dynamic Range|Bloom Filter）
-- 从上图可知，在原始的执行计划中不存在过滤器，过滤器是由系统根据JOIN的特性自动产生的，它的作用就是判断B表中的元素是否存在于A表生成的集合中，如不存在，则过滤掉。
-- 从空间和时间效率上看，Bloom Filter正是上述功能的有效选择。但在实现中，动态过滤器不仅包含Bloom Filter，还可以包含Range Filter，即利用[min, max]来过滤数据。
-- 动态过滤器是一个典型的生产者和消费者模式

-- DFP（Dynamic Filter Producer）算子：动态过滤器的生产者（Producer），利用小表侧的数据生成Bloom Filter及获取JOIN Key对应的min、max值（Range Filter），然后发送至DFC。
-- DFC（Dynamic Filter Consumer）算子：动态过滤器的消费者（Consumer），利用Bloom Filter和Range Filter过滤大表侧的数据。Range Filter会尽可能将过滤条件下推到底层存储，以便从源头过滤数据。

-- 对于不同类型的JOIN语义，JOIN对象可担任的角色不同：
-- A JOIN B：A或B都能作为生产者、消费者。
-- A LEFT JOIN B：A只能作为生产者，B只能作为消费者。
-- A RIGHT JOIN B：A只能作为消费者，B只能作为生产者。
-- A FULL OUTER JOIN B：无法使用动态过滤器功能。

-- 动态分区裁剪（Dynamic Partition Pruning）
-- 上述Bloom Filter或Range Filter的例子是基于非分区表的优化，即JOIN Key是非分区列。
-- 当JOIN Key为分区列时，动态范围过滤器或布隆过滤器（Dynamic Range|Bloom Filter）仍然可用，但MaxCompute会读取完整个分区的数据后再过滤数据，读取分区数据的过程可以进一步优化。
-- 即在读取数据前，将无用的分区裁剪掉，即动态分区裁剪DPP（Dynamic Partition Pruning）功能。

--A为非分区表，表中a列的值为20200701。
--B为分区表，表中ds列的值包含3个分区20200701、20200702、20200703。
-- select * from (table1) A join (table2) B on A.a= B.ds;

-- 打开动态分区裁剪功能后，优化器会根据表是否是分区表来决定是否采用动态分区裁剪功能。
-- 动态分区裁剪功能生效后，MaxCompute会采集小表侧数据生成Bloom Filter，然后过滤大表侧的分区列表，再把需要读取的分区列表聚合，裁剪掉不需要扫描的分区。
-- 如果一个运行进程所有待读的分区都被裁剪了，则该进程不被调度。
-- 在上述示例中，由于A表中的a列值只有20200701，因而打开动态分区裁剪功能后，B表中的20200702和20200703分区会被裁剪掉，既节省了资源，也降低了作业运行时长。
-- 动态分区裁剪功能的使用方法请参见动态分区裁剪的使用方法。

-- 动态过滤器的使用方法
-- MaxComopute提供了如下打开动态过滤器的方式：
-- 方式一：在Session级别通过开关强制打开动态过滤器，与SQL语句一起提交执行。
-- set odps.optimizer.force.dynamic.filter=true;
-- 该属性也支持在Project级别设置，但推荐您在Session级别设置。
-- 由于Project级别开启后，对所有JOIN作业都会启用动态过滤器，当JOIN无法过滤数据时，处理效率反而更低。
-- 使用该方式时，对所有能打开动态过滤器的JOIN，都插入动态过滤器。

-- 方式二：在Session级别通过开关智能打开动态过滤器。
-- set odps.optimizer.enable.dynamic.filter=true;
-- 使用该方式时，优化器会智能地估计插入动态过滤器是否有足够的资源或时间获益，如果有收益则插入动态过滤器，否则不会插入。
-- 该方式依赖元数据统计，例如ndv，更多元数据统计信息，请参见优化器信息收集。因为元数据统计是优化器的估算结果，可能不准确，因此会存在无法如预期地插入动态过滤器。

-- 方式三：在SQL语句中通过HINT方式打开动态过滤器。
-- HINT格式为/*+dynamicfilter(Producer, Consumer1[, Consumer2,...])*/，允许一个生产者过滤多个消费者。
-- select /*+dynamicfilter(A, B)*/ * from (table1) A join (table2) B on A.a= B.b;

-- 动态分区裁剪的使用方法
-- 在Session级别通过开关打开动态分区裁剪功能，与SQL语句一起提交执行。
-- set odps.optimizer.dynamic.filter.dpp.enable=true;
-- 该属性也支持在Project级别设置，但推荐您在Session级别设置。
-- 由于Project级别开启后，对所有JOIN作业中涉及到的分区表都会启用动态分区裁剪功能，当JOIN无法过滤数据时，处理效率反而更低。

-- 验证方法
-- 如果您已按照动态过滤器的使用方法或动态分区裁剪的使用方法完成配置，运行SQL作业后，查看作业的Logview信息。
-- 如果Logview中出现类似DynamicFilterConsumer1的算子，说明动态过滤器已生效。


-- Lateral View
-- MaxCompute支持通过Lateral View与UDTF（表生成函数）结合，将单行数据拆成多行数据。
-- 本文为您介绍如何使用Lateral View拆分行数据，并执行聚合操作。
-- 直接在select语句中使用UDTF会存在限制，为解决此问题，您可以通过MaxCompute的Lateral View与UDTF结合使用，将一行数据拆成多行数据，并对拆分后的数据进行聚合。
-- 当您定义的UDTF不输出任何一行时，对应的输入行在Lateral View结果中依然保留，且所有UDTF输出列为NULL。
-- lateralView: lateral view [outer] <udtf_name>(<expression>) <table_alias> as <columnAlias> (',' <columnAlias>)
-- fromClause: from <baseTable> (lateralView) [(lateralView) ...]
-- from后可以有多个Lateral View语句，后面的Lateral View语句能够引用它前面的所有表和列名，实现对不同列的行数据进行拆分。

-- test_023有三列数据，第一列是pageid string，第二列是col1 array<int>，第三列是col2 array<string>
CREATE TABLE IF NOT EXISTS  test_023(pageid string ,col1 ARRAY <INT>,col2 ARRAY <STRING >);
INSERT INTO TABLE test_023 VALUES ('front_page',array(1,2,3),array('a','b','c')),('contact_page',array(3,4,5),array('d','e','f'));
SELECT * FROM test_023;

-- 单个Lateral View语句
-- 示例1：拆分col1。
select pageid, col1_new, col2 from test_023 lateral view explode(col1) adTable as col1_new;

-- 拆分col1并执行聚合统计。
select col1_new, count(1) as count from test_023 lateral view explode(col1) adTable as col1_new group by col1_new;

-- 多个Lateral View语句
-- 拆分col1和col2。
select pageid,mycol1, mycol2 from test_023
lateral view explode(col1) myTable1 as mycol1
lateral view explode(col2) myTable2 as mycol2;














