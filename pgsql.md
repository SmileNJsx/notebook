# 数据库递归查询设计

## 场景

+ 多级部门，部门下有人员
+ 多级区域，区域下有设备
+ 小区，栋，楼，层，户，人员
+ boss以及下属，组织图

## 问题分析

+ 自增长主键 uuid snowflake三种方式对性能影响 _暂不考虑_
+ 百万级以上数据 _暂不考虑_
+ 索引对性能影响 _暂不考虑_

## 前提

+ 表设计无任何索引和对性能测试有争议的设计
+ 本文只包含对CTE优化和松散扫描做分析，但不涉及原理

## 设计思路

+ 单表结构递归查询 _单表与多表类似_
+ 多表结构联合递归查询
+ 解决数据库循环查询的问题

## 解决方案

+ 数据库CTE 和 游标 _暂不考虑_
+ EXPLAIN good performance 分析
+ 提高服务器响应到达客户端的时间，大带宽，同机柜 _暂不考虑_


## 源码
+ 小区住户测试表构造
+ 小区住户测试数据构造
+ 解决问题、解决代码

### 表结构

#### 单张表设计
```sql
DROP TABLE IF EXISTS building_household_person_recursive;
CREATE TABLE IF NOT EXISTS building_household_person_recursive (
  "id" bigserial NOT NULL,
  "code" varchar(24) NOT NULL,
  "name" varchar(24) NOT NULL,
  "parent_code" varchar(24),
  "create_time" timestamp,
  "update_time" timestamp
);
```

#### 三张表设计
```sql
-- 小区栋数表 --
DROP TABLE IF EXISTS building_recursive;
CREATE TABLE IF NOT EXISTS building_recursive (
  "id" bigserial NOT NULL,
  "code" varchar(24) NOT NULL,
  "name" varchar(24) NOT NULL,
  "parent_code" varchar(24),
  "create_time" timestamp,
  "update_time" timestamp
);

-- 小区户数表 --
DROP TABLE IF EXISTS household_recursive;
CREATE TABLE IF NOT EXISTS household_recursive (
  "id" bigserial NOT NULL,
  "code" varchar(24) NOT NULL,
  "name" varchar(24) NOT NULL,
  "parent_code" varchar(24) NOT NULL,
  "create_time" timestamp,
  "update_time" timestamp
);

-- 小区住户表 --
DROP TABLE IF EXISTS person_recursive;
CREATE TABLE IF NOT EXISTS person_recursive (
  "id" bigserial NOT NULL,
  "code" varchar(24) NOT NULL,
  "name" varchar(24) NOT NULL,
  "parent_code" varchar(24) NOT NULL,
  "create_time" timestamp,
  "update_time" timestamp
);
```

### 数据构造


#### 单张表数据构造
```sql
CREATE OR REPLACE FUNCTION func_buildata(buildingNo INT, householdNo INT, peronNo INT) RETURNS VOID AS
$$
DECLARE
  var1 INTEGER := 0;
  var2 INTEGER := 0;
  var3 INTEGER := 0;
BEGIN
  WHILE var1 < buildingNo LOOP
    var1 := var1 + 1;
    INSERT INTO building_household_person_recursive ("code", "name", "parent_code", "create_time", "update_time") VALUES ('BUILDING' || var1, var1 || '栋', NULL, NOW(), NOW());
    WHILE var2 < householdNo LOOP
      var2 := var2 + 1;
      INSERT INTO building_household_person_recursive ("code", "name", "parent_code", "create_time", "update_time") VALUES ('HOUSEHOLD' || var2, var1 || '栋' || var2 || '户', 'BUILDING' || var1, NOW(), NOW());
      WHILE var3 < peronNo LOOP
        var3 := var3 + 1;
        INSERT INTO building_household_person_recursive ("code", "name", "parent_code", "create_time", "update_time") VALUES ('PERSON' || var3, var1 || '栋' || var2 || '户' || var3 || '人', 'HOUSEHOLD' || var2, NOW(), NOW());
      END LOOP;
      var3 := 0;
    END LOOP;
    var2 := 0;
  END LOOP;
END;
$$ LANGUAGE PLPGSQL;
```

#### 三张表数据构造
```sql
-- 数据构造 函数 --
CREATE OR REPLACE FUNCTION func_buildata(buildingNo INT, householdNo INT, peronNo INT) RETURNS VOID AS
$$
DECLARE
  var1 INTEGER := 0;
  var2 INTEGER := 0;
  var3 INTEGER := 0;
BEGIN
  WHILE var1 < buildingNo LOOP
    var1 := var1 + 1;
    INSERT INTO building_recursive ("code", "name", "parent_code", "create_time", "update_time") VALUES ('BUILDING' || var1, var1 || '栋', NULL, NOW(), NOW());
    WHILE var2 < householdNo LOOP
      var2 := var2 + 1;
      INSERT INTO household_recursive ("code", "name", "parent_code", "create_time", "update_time") VALUES ('HOUSEHOLD' || var2, var1 || '栋' || var2 || '户', 'BUILDING' || var1, NOW(), NOW());
      WHILE var3 < peronNo LOOP
        var3 := var3 + 1;
        INSERT INTO person_recursive ("code", "name", "parent_code", "create_time", "update_time") VALUES ('PERSON' || var3, var1 || '栋' || var2 || '户' || var3 || '人', 'HOUSEHOLD' || var2, NOW(), NOW());
      END LOOP;
      var3 := 0;
    END LOOP;
    var2 := 0;
  END LOOP;
END;
$$ LANGUAGE PLPGSQL;

-- 构造数据执行 此处是百万级数据 修改入参 构造不同级别数据--
SELECT func_buildata(100, 10000, 3);
```

百万级数据构造耗时 execution: 1m 9s 454ms, fetching: 4ms _此处采用自增主键设计_

### 方案代码

查询n栋下所有用户信息和人员信息，执行语句优化分析

#### 检索栋下所有户

##### 单表解决方案
```sql
WITH RECURSIVE person AS (
  SELECT name, code, parent_code FROM building_household_person_recursive WHERE code = 'BUILDING1' -- no recursive term
  UNION ALL
  SELECT t1.name, t1.code, t1.parent_code FROM building_household_person_recursive t1 INNER JOIN person t2 ON t1.parent_code = t2.code WHERE t1.code LIKE '%HOUSEHOLD%' -- recursive term
)
SELECT * FROM person;
```

##### 多表解决方案

```sql
SELECT * FROM household_recursive WHERE parent_code = 'BUILDING1';
```

#### 检索户下所有人

##### 单表解决方案
```sql
WITH RECURSIVE person AS (
    SELECT name, code, parent_code FROM building_household_person_recursive WHERE code = 'HOUSEHOLD1'
    UNION ALL
    SELECT t1.name, t1.code, t1.parent_code FROM building_household_person_recursive t1 INNER JOIN person t2 ON t1.parent_code = t2.code WHERE t1.code LIKE '%PERSON%'
)
SELECT * FROM person;
```
##### 多表解决方案

```sql
SELECT * FROM person_recursive WHERE parent_code = 'HOUSEHOLD1';
```

#### 检索栋下所有人

##### 单表解决方案
```sql
WITH RECURSIVE person AS (
    SELECT name, code, parent_code FROM building_household_person_recursive WHERE code = 'BUILDING1'
    UNION ALL
    SELECT t1.name, t1.code, t1.parent_code FROM building_household_person_recursive t1 INNER JOIN person t2 ON t1.parent_code = t2.code
)
SELECT * FROM person;
```

##### 多表解决方案
```sql
SELECT * FROM person_recursive WHERE parent_code IN (
    SELECT code FROM household_recursive WHERE parent_code IN (
        SELECT code FROM building_recursive WHERE code = 'BUILDING1'
    )
);
```

#### 语句执行计划

### 单表多表对比

+ 明显的在一些简单场景下多表要更适合，但其实通过改写单表语句，单表性能也不会差太多
+ 随着迭代层次的升级，明显的单表的迭代查询设计要更高效和简单
+ 由于单表不是多表简单的子查询，采用CTE使得迭代的性能表现更优秀

## 局限  （问题可能升级的方向）

+ 单表百万级数据测试，待验证 _（十万级别没有问题）_
+ 分库分表设计，递归查询设计？
+ 利用缓存加速，递归查询的解决方法？
+ 三张表改进成一张表 _（三表合一）_
+ 比优化器更懂优化
