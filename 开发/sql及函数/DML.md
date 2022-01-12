--odps sql
--********************************************************************--
--author:算法工程师-韦广繁
--create time:2021-11-26 18:20:15
--********************************************************************--
---插入或覆写数据（INSERT INTO | INSERT OVERWRITE）
-- 在使用MaxCompute SQL处理数据时，insert into或insert overwrite操作可以将select查询的结果保存至目标表中。二者的区别是：
-- insert into：直接向表或静态分区中插入数据。您可以在insert语句中直接指定分区值，将数据插入指定的分区。如果您需要插入少量测试数据，可以配合VALUES使用。
-- insert overwrite：先清空表中的原有数据，再向表或静态分区中插入数据。
--非分区表test_01
CREATE TABLE test_01 (key STRING );

INSERT INTO TABLE test_01 VALUES ('hello');

SELECT * FROM test_01;

--分区表test_02
CREATE TABLE if not exists  test_02(
shop_name STRING ,
customer_id STRING ,
total_price DOUBLE

)
PARTITIONED BY (sale_date STRING ,region STRING );

DESC EXTENDED test_02;

-- 执行insert into命令向分区表test_02中追加数据。
alter table test_02 add partition(sale_date = '2013',region = 'china');

INSERT into table test_02 partition (sale_date = '2013',region = 'china') VALUES ('s1','c1',100.1),('s2','c2',100.2),('s3','c3',100.3);

SELECT * FROM test_02 WHERE sale_date = '2013' AND  region = 'china';

-- 开启全表扫描，仅此Session有效。执行select语句查看表test_02中的数据。
set odps.sql.allow.fullscan = true;
SELECT * FROM test_02;

-- 执行insert overwrite命令更新表test_03中的数据。
CREATE TABLE test_03 LIKE test_02 ;

alter TABLE test_03 add partition(sale_date = '2013',region = 'china');

SHOW PARTITIONS test_03;

show create table test_03;

show TABLES ;

-- 从源表test_02中取出数据插入目标表test_03。注意不需要声明目标表字段，也不支持重排目标表字段顺序。
-- 对于静态分区目标表，分区字段赋值已经在partition()部分声明，不需要在select_statement中包含，只要按照目标表普通列顺序查出对应字段，按顺序映射到目标表即可。
-- 源表与目标表的对应关系依赖于select子句中列的顺序，而不是表与表之间列名的对应关系。
-- 向某个分区插入数据时，分区列不允许出现在select子句中。
-- partition的值只能是常量，不可以为表达式。
-- 动态分区表则需要在select中包含分区字段，详情请参见插入或覆写动态分区数据（DYNAMIC PARTITION）。
INSERT OVERWRITE TABLE test_03 PARTITION (sale_date = '2013',region = 'china')
SELECT shop_name,
customer_id,
total_price
FROM test_02
WHERE sale_date = '2013' and region = 'china'
order BY  customer_id,total_price ;

set odps.sql.allow.fullscan=true;
SELECT * FROM test_03;

-- 插入或覆写动态分区数据（DYNAMIC PARTITION）
-- 将源表中的数据插入到目标表中。在运行SQL语句之前，您无法得知会产生哪些分区。只有在语句运行结束后，才能通过region字段产生的值确定产生的分区。
create table test_012 (revenue double) partitioned by (region string);

insert overwrite table test_012 partition(region)
select total_price as revenue, region from test_02;

SHOW PARTITIONS test_012;

SET odps.ODPS.SQL.ALLOW.FULLSCAN=T TRUE ;
SELECT * FROM test_012;

-- 将源表中的数据插入到目标表中。多级分区，指定一级分区sale_date。
-- 如果目标表只有一级动态分区，则select子句的最后一个字段值即为目标表的动态分区值。
-- 动态分区中，select_statement字段和目标表动态分区的对应关系是由字段顺序决定，并不是由列名称决定的。
-- 向动态分区插入数据时，动态分区列必须在select列表中，否则会执行失败。
-- 向动态分区插入数据时，不能仅指定低级子分区，而动态插入高级分区，否则会执行失败。
create table test_013 like test_02;

DESC EXTENDED test_013;

insert overwrite table test_013 partition (sale_date='2013', region)
select shop_name,customer_id,total_price,region from test_02;

SET odps.ODPS.SQL.ALLOW.FULLSCAN=T TRUE ;
SELECT * FROM test_013;

INSERT OVERWRITE TABLE test_013 PARTITION (sale_date,region)
SELECT shop_name,customer_id,total_price,sale_date,region FROM test_02;

-- MaxCompute在向动态分区插入数据时，如果分区列的类型与对应select中列的类型不严格一致，会隐式转换，
create table test_014 (c int, d string) partitioned by (e int);
alter table test_014 add if not exists partition (e=201312);
insert into test_014 partition (e=201312) values (1,100.1),(2,100.2),(3,100.3);

SELECT * FROM test_014;

create table test_015(a int, b double) partitioned by (p string);
insert into test_015 partition (p) select c, d, current_timestamp() from test_014;

SELECT * FROM test_015;

-- 多路输出（MULTI INSERT）
-- MaxCompute SQL支持您在一条SQL语句中通过insert into或insert overwrite操作将数据插入不同的目标表或者分区中，实现多路输出。
-- 在使用MaxCompute SQL处理数据时，multi insert操作可以将数据插入不同的目标表或分区中，实现多路输出。
-- 同一目标分区不允许出现多次，否则会返回报错

CREATE TABLE test_016 LIKE test_02;

SET odps.ODPS.SQL.ALLOW.FULLSCAN=T TRUE ;
FROM test_02
INSERT OVERWRITE TABLE test_016 PARTITION (sale_date='2013',region='china')
SELECT shop_name, customer_id, total_price
INSERT OVERWRITE TABLE test_016 PARTITION (sale_date='2014',region='china')
SELECT shop_name,customer_id,total_price;

SELECT * FROM test_016;

-- VALUES
-- 如果需要向表中插入少量数据，您可以通过insert … values或values table操作向数据量小的表中插入数据。
-- 通过insert … values或values table操作向表中插入数据时，不支持通过insert overwrite操作指定插入列，只能通过insert into操作指定插入列。
-- 通过insert … values操作向特定分区内插入数据。
create table if not exists test_017 (key string,value bigint) partitioned by (p string);
DESC EXTENDED test_017;
alter table test_017 add if not exists partition (p='abc');
SHOW PARTITIONS test_017;
insert into table test_017 partition (p='abc') values ('a',1),('b',2),('c',3);
SET odps.ODPS.SQL.ALLOW.FULLSCAN=T TRUE ;
select * from test_017 where p='abc';

-- 通过insert … values操作向非特定分区内插入数据。
create table if not exists test_018 (key string,value bigint) partitioned by (p string);
DESC EXTENDED test_018;
insert into table test_018 partition (p)(key,p) values ('d','20170101'),('e','20170101'),('f','20170101');
SET odps.ODPS.SQL.ALLOW.FULLSCAN=T TRUE ;
SELECT * FROM test_018;

-- 使用复杂数据类型构造常量，通过insert操作导入数据。
create table if not exists test_019 (key string,value array<int>) partitioned by (p string);
DESC EXTENDED test_019;
alter table test_019 add if not exists partition (p='abc');

insert into table test_019 partition (p='abc') select 'a', array(1, 2, 3);
select * from test_019 where p='abc';

-- 通过insert … values操作写入DATETIME或TIMESTAMP数据类型，需要在values中指定类型名称。
create table if not exists test_020 (key string, value timestamp) partitioned by (p string);
DESC EXTENDED test_020;
alter table test_020 add if not exists partition (p='abc');
SHOW PARTITIONS test_020;
insert into table test_020 partition (p='abc') values (datetime'2017-11-11 00:00:00',timestamp'2017-11-11 00:00:00.123456789');
select * from test_020 where p='abc';

-- 通过values table操作插入数据。
-- values (…), (…) t(a, b)相当于定义了一个名为t，列为a和b，数据类型分别为STRING和BIGINT的表。列的类型需要从values列表中推导。
create table if not exists test_021 (key string,value bigint) partitioned by (p string);
insert into table test_021 partition (p)
select concat(a,b), length(a)+length(b),'20170102' from values ('d',4),('e',5),('f',6) t(a,b);

select * from test_021 where p='20170102';

-- 取代select * from与union all组合的方式，构造常量表。
select 1 c union all select 2 c;
select * from values (1), (2) t(c);

-- 通过values table的特殊形式插入数据，不带from子句。
create table if not exists test_022 (key string,value bigint) partitioned by (p string);
DESC EXTENDED test_022;
insert into table test_022 partition (p) select abs(-1), length('abc'), getdate();
select * from test_022;

-- 使用非常量表达式
select * from values ('a'),(to_date('20190101', 'yyyyMMdd')),(getdate()) t(d);
