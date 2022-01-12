--odps sql
--********************************************************************--
--author:算法工程师-韦广繁
--create time:2021-11-26 16:38:42
--********************************************************************--
---表操作
---1)创建非分区表
CREATE TABLE test_01 (key STRING );
---2)创建一张分区表
CREATE TABLE if not exists  test_02(
shop_name STRING ,
customer_id STRING ,
total_price DOUBLE

)
PARTITIONED BY (sale_date STRING ,region STRING );
---3)创建一个新表,将test_02的数据复制到test_04中，并设置生命周期。
CREATE TABLE test_04 LIFECYCLE 10 as SELECT * FROM test_02 WHERE sale_date = '2013' and region = 'china';

---4)查看到表的结构及生命周期等详细信息。
DESC EXTENDED test_04;
---5）创建一个新表，在select子句中使用常量作为列的值。
---5.1）指定列的名字。
CREATE TABLE test_001
AS
SELECT  shop_name,customer_id,total_price,'2013' AS sale_date,'China' AS region FROM test_02;

---5.2）不指定列的名字。
CREATE TABLE test_002
AS
SELECT shop_name,customer_id,total_price,'2013','China' FROM test_02;
---6)创建一个新表,与test_02具有相同的表结构，并设置生命周期。
CREATE TABLE test_003
LIKE test_02 LIFECYCLE 10;

---7)创建使用新数据类型的表
set odps.sql.type.system.odps2 = true;
CREATE TABLE test_004(
c1 TINYINT
,c2 SMALLINT
,c3 INT
,c4 BIGINT
,c5 FLOAT
,c6 DOUBLE
,c7 DECIMAL
,c8 BINARY
,c9 TIMESTAMP
,c10 ARRAY<MAP <BIGINT ,BIGINT >>
,c11 MAP<STRING,ARRAY<BIGINT>>
,c12 STRUCT <s1:STRING ,s2:BIGINT>
,c13 VARCHAR(20)
)
LIFECYCLE 1;

---修改表的所有人
---1)修改表的所有人，即表Owner。
ALTER TABLE test_001 CHANGEOWNER TO  'RAM$admin@jiamiantech.com:guangfan.wei';

---修改表的注释
---1)修改表的注释内容
ALTER TABLE test_001 SET COMMENT 'new comments for table test_001';

---修改表的修改时间
---1)MaxCompute SQL提供touch操作用来修改表的LastDataModifiedTime，可将表的LastDataModifiedTime修改为当前时间。此操作会改变表的LastDataModifiedTime的值，MaxCompute会认为表的数据有变动，生命周期的计算会重新开始。


ALTER TABLE test_001  TOUCH;

---重命名表
-- 重命名表的名称。仅修改表的名字，不改动表中的数据。

ALTER TABLE test_001 RENAME TO  test_001;

---清空非分区表里的数据
-- 将指定的非分区表中的数据清空。

CREATE TABLE test_005(id TINYINT ,name STRING ) LIFECYCLE 1;
INSERT INTO test_005 VALUES (1Y,'wang'),(2Y,'li');
TRUNCATE TABLE test_005;
SELECT * FROM test_005;

CREATE TABLE test_006 LIFECYCLE 10 AS SELECT * FROM test_02  WHERE sale_date is not NULL  and region is not NULL ;
SELECT * FROM test_006;
TRUNCATE TABLE test_006;
---清空分区数据
-- 清空分区表中指定分区的数据。
CREATE TABLE test_007 (id INT ,class STRING ) PARTITIONED BY (cteatedon STRING ,region STRING );
ALTER TABLE test_007 ADD PARTITION (cteatedon='2020',region='korea');
INSERT INTO TABLE test_007 PARTITION (cteatedon='2020',region='korea') VALUES (1,'first'),(2,'second'),(3,'third');
set odps.sql.ALLOW.fullscan=true;
SELECT * FROM test_007;
ALTER TABLE test_007 ADD PARTITION (cteatedon='2021',region='china');
INSERT INTO TABLE test_007 PARTITION (cteatedon='2021',region='china') VALUES (4,'forth'),(5,'fifth'),(6,'sixth');

TRUNCATE TABLE test_007 PARTITION (cteatedon='2020',region='korea');

-- 删除表
-- 删除非分区表或分区表。
DROP TABLE IF EXISTS test_005 ;

DROP TABLE IF EXISTS test_006;

-- 查看表或视图信息
-- 查看MaxCompute内部表、视图、外部表、聚簇表或Transactional表的信息
DESC  test_01;
DESC EXTENDED test_01;
DESC test_007;
DESC EXTENDED test_007;

-- 查看分区信息
-- 查看某个分区表具体的分区的信息。
DESC test_007 PARTITION (cteatedon='2021',region='china');


-- 查看建表语句
-- 生成创建表的SQL DDL语句，方便您通过SQL重建Schema。
SHOW CREATE TABLE test_01;
SHOW CREATE TABLE test_007;

-- 列出项目空间下的表和视图
-- 列出项目空间下所有的表和视图，或符合某规则的表和视图。

SHOW TABLES ;
SHOW TABLES like 'test_*';

-- 列出所有分区
-- 列出一张表中的所有分区，表不存在或为非分区表时，返回报错。
SHOW PARTITIONS test_007;

-- 分区和列操作
-- 添加分区
-- 为已存在的分区表新增分区。

ALTER TABLE test_007 ADD IF NOT EXISTS PARTITION (cteatedon='2022',region='russia');

SHOW PARTITIONS test_007;

ALTER TABLE test_007 ADD IF NOT EXISTS PARTITION (cteatedon='2022',region='china') PARTITION (cteatedon='2022',region='japan');

-- 删除分区
-- 为已存在的分区表删除分区。

ALTER TABLE test_007 DROP IF EXISTS PARTITION (cteatedon='2020',region='korea');
ALTER TABLE test_007 DROP IF EXISTS PARTITION (cteatedon='2021',region='china') ,PARTITION (cteatedon='2022',region='japan') ;

ALTER TABLE test_007 ADD IF NOT EXISTS PARTITION (cteatedon='202201',region='china')
PARTITION (cteatedon='202202',region='china')
PARTITION (cteatedon='202203',region='china')
PARTITION (cteatedon='202204',region='china')
PARTITION (cteatedon='202205',region='china')
PARTITION (cteatedon='202206',region='china')
PARTITION (cteatedon='202207',region='china')
PARTITION (cteatedon='202208',region='china')
PARTITION (cteatedon='202209',region='china')
PARTITION (cteatedon='2022010',region='china');

ALTER TABLE test_007 DROP IF EXISTS PARTITION (cteatedon < 202204 );

ALTER TABLE test_007 DROP IF EXISTS PARTITION (cteatedon >= 2022010);

ALTER TABLE test_007 DROP IF EXISTS PARTITION (cteatedon BETWEEN 202204 AND 202206);

ALTER TABLE test_007 DROP IF EXISTS PARTITION (cteatedon <202208 ),PARTITION (region = 'china');

-- 修改分区的更新时间
-- MaxCompute SQL提供touch操作，用于修改分区表中分区的LastDataModifiedTime。此操作会将LastDataModifiedTime修改为当前时间。此时，MaxCompute会认为数据有变动，重新计算生命周期。
DESC EXTENDED test_007;

ALTER TABLE test_007 TOUCH PARTITION (cteatedon='2022',region='china');


-- 修改分区值
-- MaxCompute SQL支持通过rename操作更改分区表的分区值。
ALTER TABLE test_007 PARTITION (cteatedon='2022',region='china') RENAME TO PARTITION (cteatedon='2023',region='korea');

-- 合并分区
-- MaxCompute SQL提供merge partition对分区表的分区进行合并，即将同一个分区表下的多个分区合并成一个分区，同时删除被合并的分区维度的信息，把数据移动到指定分区。
ALTER TABLE test_007 MERGE PARTITION (cteatedon='2022') OVERWRITE PARTITION (cteatedon= '2023',region='china');
show PARTITIONS test_007;

ALTER TABLE test_007 ADD PARTITION (cteatedon='2021',region='russia');
ALTER TABLE test_007 MERGE PARTITION (cteatedon='2021',region='russia'),PARTITION (cteatedon='2023',region='korea') OVERWRITE PARTITION (cteatedon='2023',region='china');

-- 添加列或注释
-- 为已存在的非分区表或分区表添加列或注释。

ALTER TABLE test_007 ADD COLUMNS (name STRING COMMENT '名字'，age INT COMMENT '年龄')；
DESC EXTENDED test_007;

-- 修改列名
-- 为已存在的非分区表或分区表修改列名称。

ALTER TABLE test_007 CHANGE COLUMN age RENAME TO true_age;

DESC EXTENDED test_007;


-- 修改列注释
-- 为已存在的非分区表或分区表修改列注释。

ALTER TABLE test_007 CHANGE COLUMN true_age COMMENT '真实年龄'；

-- 修改列名及注释
-- 修改非分区表或分区表的列名或注释。

ALTER TABLE test_007 CHANGE COLUMN true_age age INT  COMMENT '年龄';

-- 修改表的列非空属性
-- 修改表的非分区列的非空属性。即如果表的非分区列值禁止为NULL，您可以通过本命令修改分区列值允许为NULL。
-- 您可以通过desc extended table_name;命令查看Nullable属性值，判断列的非空属性。如果Nullable为true，表示允许为NULL；如果Nullable为false，表示禁止为NULL。
-- 使用限制
-- 修改分区列值允许为NULL后，不可回退，不支持再修改分区列值禁止为NULL，请谨慎操作。
CREATE TABLE IF NOT EXISTS  test_008(id INT COMMENT '序号',name STRING NOT NULL  COMMENT '姓名',number INT NOT NULL  COMMENT '身份证号码') PARTITIONED BY (createdon STRING NOT NULL  COMMENT '日期');

DESC EXTENDED test_008;

ALTER TABLE test_008 CHANGE COLUMN name NULL;

-- 生命周期操作
-- 生命周期
-- 您可以在创建表时，通过lifecycle关键字指定生命周期。
-- 在MaxCompute中，每当表的数据被修改后，表的LastDataModifiedTime将会被更新。MaxCompute会根据每张表的LastDataModifiedTime以及生命周期的设置来判断是否要回收此表：
-- 如果表是非分区表，自最后一次数据被修改开始计算，经过days天后数据仍未被改动，则此表无需您干预，MaxCompute会自动回收，类似drop table操作。
-- 如果表是分区表，则根据各分区的LastDataModifiedTime判断该分区是否该被回收。分区表的最后一个分区被回收后，该表不会被删除。

CREATE TABLE test_009(level STRING ) LIFECYCLE 10;
DESC EXTENDED test_009;
CREATE TABLE test_010 LIFECYCLE 10 as SELECT * FROM  test_007 WHERE cteatedon IS NOT NULL ;
DESC EXTENDED test_010;

CREATE TABLE test_011  LIKE test_007 LIFECYCLE 11;

DESC EXTENDED test_011 ;
-- 修改表的生命周期
-- 修改已存在的分区表或非分区表的生命周期。
ALTER TABLE test_009 SET  LIFECYCLE 2;

ALTER TABLE test_010 SET LIFECYCLE  3;

ALTER TABLE test_011 SET LIFECYCLE 2;

-- 禁止或恢复生命周期
-- 禁止或恢复指定表或分区的生命周期。

ALTER TABLE test_011 ADD PARTITION (cteatedon='2022',region='china');
ALTER TABLE test_011 ADD PARTITION (cteatedon='2022',region='japan');

SHOW PARTITIONS test_011;

ALTER TABLE test_011 DISABLE LIFECYCLE ;

ALTER TABLE test_011 ENABLE LIFECYCLE ;

ALTER TABLE test_011 PARTITION (cteatedon='2022',region='japan') DISABLE LIFECYCLE ;

ALTER TABLE test_011 PARTITION (cteatedon='2022',region='japan') ENABLE  LIFECYCLE ;

