# GUI设计

原型图：https://modao.cc/app/design/pbkb7r4osucugpli

打开crims_client的URL，首先与127.0.0.1:8000建立socket连接，随后便可发送SQL语句并接收响应来进行通信

一级选单：数据维护；数据查询；数据统计

注：以下称呼一行数据（记录）为一**条**数据

## 1. 数据维护模块

### GUI逻辑

主要是对三张表进行增删改的操作，并且可以导入导出xls文件

+ 当在选单中选择任意一张表的时候，会向后端发送`SELECT * FROM 表名 ORDER BY 主键;`，并将接收到的json对象展示在下方的表格中
+ 当没有在选单中选择任意一张表的时候，所有的5个按钮（==添加、删除、编辑、导入、导出==）都不可点击
+ 每条数据最左端放一个选择框，方便多选，数据表左上方有三个按钮图标：**添加，删除，编辑**
  + 任何时候，「添加」按钮可以点击
    + 此时点击添加，会弹出让用户填写一条数据的文本框（一般会需要填写多个数据项，根据不同的表而不同，这个写死在代码或者配置文件中）
    + 填写完毕若点击「添加」，则向后端发送`INSERT INTO 表名 VALUES(...);`的语句；若点击取消则取消添加。
  + 当选中了多条数据时，「删除」按钮可以点击，「编辑」按钮不可点击
    + 此时点击删除，会弹出确认框「是否确认删除这X条数据？」，用户选择「删除」（红色背景按钮）或者「取消」（灰色背景按钮）
    + 填写完毕若点击「删除」，则向后端发送`DELETE FROM 表名 WHERE 主键=主键值;`的语句，这里不同的表的主键参考api文档
  + 当选中了1条数据时，「编辑」和「删除」按钮可以点击
    + 此时点击编辑，会弹出让用户填写一条数据的文本框，样式类似于上述「添加」按钮弹出的文本框，但是预先上面会填好本条数据，之后用户选择「保存」（蓝色背景按钮）或者「取消」（灰色背景按钮）。
+ 当成功执行了添加，删除，编辑任意一种操作之后，都要再执行一次`SELECT * FROM 表名 ORDER BY 主键;`，来更新当前展示的数据
+ 导出：即将当前表的数据导出为xls文件
+ 导入：弹出文件资源管理器，选择一个本地的xls文件导入，前端解析并生成`INSERT`语句发送到后端（细节TODO）

### 数据表逻辑

#### ①CAR_TYPE车辆类型信息表

| 字段名   | 显示名称     | 对应JS类型 | 备注                                      |
| -------- | ------------ | ---------- | ----------------------------------------- |
| code     | 车辆类型编码 | string     | **主键**，只能是整数型的字符串，从'1'开始 |
| tname    | 车辆类型名称 | string     |                                           |
| quantity | 库存数量     | number     | 只能是非负整数                            |

#### ②CAR_INFO 车辆基本信息表 

| 字段名     | 显示名称     | 对应JS类型 | 备注                            |
| ---------- | ------------ | ---------- | ------------------------------- |
| cid        | 车辆编号     | number     | **主键**，只能是非负整数        |
| plate      | 车牌号       | string     |                                 |
| code       | 车辆类型编码 | string     | 只能是整数型的字符串，从'1'开始 |
| cname      | 车辆名称     | string     |                                 |
| gear       | 排挡方式     | string     |                                 |
| daily_rent | 每日租金     | number     | 浮点数                          |
| rent       | 出租状态     | string     | 只能在'y'和'n'两者之间选择      |

**当添加/删除一个车辆信息时，要将CAR_TYPE表对应类型数量+1/-1**

#### ③RENT_ORDER 租车订单信息表

| 字段名                 | 显示名称     | 对应JS类型  | 备注                                                         |
| ---------------------- | ------------ | ----------- | ------------------------------------------------------------ |
| oid                    | 订单编号     | string      | **主键**，让用户自行填写，形如“2019021505”表示2019年2月15日第5个订单 |
| identity_number        | 身份证号     | string      |                                                              |
| pname                  | 客人姓名     | string      |                                                              |
| phone_number           | 手机号码     | string      | 只能是由0-9组成的字符串                                      |
| cid                    | 租用车辆编号 | number      | 只能是非负整数                                               |
| pickup_time            | 取车时间     | string/Date | 只精确到分钟，SQL语句的时间格式为'%Y-%m-%d/%H:%M'            |
| scheduled_dropoff_time | 预约还车时间 | string/Date | 只精确到分钟，SQL语句的时间格式为'%Y-%m-%d/%H:%M'            |
| deposit                | 押金         | number      | 不允许用户填写，等于应缴费用*5                               |
| actual_dropoff_time    | 实际还车时间 | string/Date | 只精确到分钟，SQL语句的时间格式为'%Y-%m-%d/%H:%M'            |
| scheduled_fee          | 应缴费用     | number      | 浮点数                                                       |
| actual_fee             | 实缴缴用     | number      | 浮点数                                                       |

### SQL语法示例

==TIPS: 添加或删除CAR_INFO时都要到CAR_TYPE中更改数量==

一般情况下，本模块（数据维护）仅需要以下SQL语句：

```sql
--查询整张表
SELECT * FROM CAR_TYPE ORDER BY code;
SELECT * FROM CAR_INFO ORDER BY cid;
SELECT * FROM RENT_ORDER ORDER BY oid;

--删除某条数据, 只需要按照主键删除即可
DELETE FROM CAR_TYPE WHERE code='这条数据的code';
DELETE FROM CAR_INFO WHERE cid='这条数据的cid';
DELETE FROM RENT_ORDER WHERE oid='这条数据的oid';

--插入一条数据, 需要把所有数据处理/拼接好, 括号中的参数顺序就是上文的表格从上到下的顺序
INSERT INTO CAR_TYPE VALUES ('6', '跑车', 100);
INSERT INTO CAR_INFO VALUES (100, '琼B12345', '1', '保时捷', '自动挡', 1000, 'n');
INSERT INTO RENT_ORDER VALUES ('2020061101', '42011011110', '王思聪', '13912345678', 100, '2020-06-11/00:00', '2020-06-11/18:00', 10000, '2020-06-11/19:00', 2000, 2000);

--修改一条数据, 根据主键更新即可😁
UPDATE CAR_TYPE SET quantity=5 WHERE code='1';
UPDATE CAR_INFO SET cname="玛莎拉蒂", gear="氮气" WHERE cid=22;
UPDATE RENT_ORDER SET pname="王健林" WHERE oid='2019062002';
```

## 2. 数据查询模块

数据查询分为普通查询和高级查询，默认情况为普通查询，通过点击按钮可切换到高级查询

### 普通查询

普通类型分为三类查询：车辆分类信息查询（主要对应CAR_TYPE），车辆基本信息查询（主要对应CAR_INFO），租车订单信息查询（主要对应RENT_ORDER）

首先需要选择一种查询功能（三选一），然后根据选择的不同，下方会出现不同的子选项（具体定义见下），点击「查询」按钮后，前端根据用户填写情况来拼接SQL语句发送给后端，随后下方出现查找结果（如果查找结果为空要给出提示，比如弹窗「未查找到结果」等）

#### ①CAR_TYPE 车辆分类信息查询功能

本查询功能固定只展示4列：**车辆类型编码、车辆名称、排挡方式、每日租金**

##### 子选项

| 名称     | 对应字段名 | 查询类型                                                     |
| -------- | ---------- | ------------------------------------------------------------ |
| 车辆类别 | tname      | 在所有的车辆类别的名称中进行选择，可以多选（如果不选则为所有） |
| 出租状态 | rent       | 选择'y'或者'n'，多选和不选效果一样                           |

##### SQL语法示例

```sql
SELECT code, cname, gear, daily_rent
	FROM CAR_INFO
	WHERE rent='y'
	AND code IN (SELECT code FROM CAR_TYPE WHERE tname="经济型" OR tname="商务型");
```

#### ②CAR_INFO 车辆基本信息查询功能

本查询功能固定展示7列：**车辆编号cid，车牌号plate，车辆类型编码code，车辆名称cname，排挡方式gear，每日租金daily_rent，出租状态rent**（即为CAR_INFO的所有列）

模糊查询即是用`LIKE`表达式而非`=`，在字符串前后加上`.*`即可

##### 子选项

| 名称     | 对应字段名 | 查询类型                           |
| -------- | ---------- | ---------------------------------- |
| 车牌号   | plate      | 文本输入，模糊查询                 |
| 车辆名称 | cname      | 文本输入，模糊查询                 |
| 出租状态 | rent       | 选择'y'或者'n'，多选和不选效果一样 |

##### SQL语法示例

```sql
SELECT *
	FROM CAR_INFO
	WHERE plate LIKE '.*鄂A.*'
	AND cname LIKE '.*大众.*'
	AND rent='y';
```

#### ③RENT_ORDER 租车订单信息查询功能

本查询功能固定展示11列：即为RENT_ORDER的所有列

模糊查询即是用`LIKE`表达式而非`=`，在字符串前后加上`.*`即可

==时间字符串的格式为'%Y-%m-%d/%H:%M'==

##### 子选项

| 名称         | 对应字段名      | 查询类型                     |
| ------------ | --------------- | ---------------------------- |
| 身份证号     | identity_number | 文本输入，模糊查询           |
| 手机号码     | phone_number    | 文本输入，模糊查询           |
| 车牌号码     | plate           | 文本输入，模糊查询           |
| 车辆名称     | cname           | 文本输入，模糊查询           |
| 租车时间范围 | pickup_time     | 选定两个日期（精确到天即可） |

##### SQL语法示例

```sql
SELECT *
	FROM RENT_ORDER
	WHERE identity_number LIKE '.*420.*'
	AND phone_number LIKE '.*139.*'
	AND pickup_time >= '2019-02-11' AND pickup_time <= '2019-05-12'
	AND cid in (SELECT cid FROM CAR_INFO WHERE plate LIKE '.*鄂A.*' AND cname LIKE '.*大众.*');
```

### 高级查询

高级查询无需选择表，直接让用户在文本框中输入SQL语句，点击「查询」即直接发送SQL语句去查询。

### 导出查询结果

普通查询可以将当前查到的结果导出为xls文档。（同上）

## 3. 数据统计

总共只要求绘制4种数据统计表/图，因此考虑SQL查询语句直接写定，数据表/条形图的格式分别写定

==图表样式参见《程序设计综合课程设计任务书(二).pdf》==

1. 统计当前每种车辆类型的车辆总数、已出租数、未出租数

```sql
SELECT CAR_TYPE.tname AS "车辆类型名称",
	   CAR_TYPE.quantity AS "车辆总数", 
	   SUM(CAR_INFO.rent='y') AS "已出租数",
	   SUM(CAR_INFO.rent='n') AS "未出租数"
	FROM CAR_TYPE, CAR_INFO
    WHERE CAR_TYPE.code=CAR_INFO.code
    GROUP BY CAR_TYPE.tname;
```

输出表格样式：

![image-20200611195219532](http://shaw.wang:9888/images/2020/06/11/image-20200611195219532.png)

2. 统计每月每种车辆类型的营业额（产生的实缴费用） ，输出当月每种车辆类型的营业额柱状统计图

   TIPS: 需要用户选择年月, 然后拼接为形式如下的SQL语句

```sql
SELECT CAR_TYPE.tname AS "车辆类型", 
	   SUM(RENT_ORDER.actual_fee) AS "营业额"
    FROM CAR_TYPE, CAR_INFO, RENT_ORDER
    WHERE CAR_TYPE.code = CAR_INFO.code
    AND CAR_INFO.cid = RENT_ORDER.cid
    AND RENT_ORDER.pickup_time > '2019-2-1' 
    AND RENT_ORDER.pickup_time < '2019-2-28'
    GROUP BY CAR_TYPE.tname;
```

输出表格样式:

![image-20200611195614525](http://shaw.wang:9888/images/2020/06/11/image-20200611195614525.png)

输出图片样式:

![image-20200611195640704](http://shaw.wang:9888/images/2020/06/11/image-20200611195640704.png)

3. 输入年份，统计该年每辆车的营业额（产生的实缴费用）、租用率

   TIPS: **需要用户输入年份**。第4行需要除以总天数(365)才能获得租用率. 如果是2020年(因为还没过完), 那么365应该改为「今天是今年的第几天」, 才能正确计算租用率（单位：1%）

```sql
SELECT CAR_INFO.plate AS "车牌号", 
	   CAR_INFO.cname AS "车辆名称", 
	   SUM(RENT_ORDER.actual_fee) AS "营业额", 
       SUM(TIMESTAMPDIFF(DAY, RENT_ORDER.pickup_time, RENT_ORDER.actual_dropoff_time))/365*100 AS "租用率"
       FROM CAR_INFO, RENT_ORDER
       WHERE CAR_INFO.cid=RENT_ORDER.cid
       AND RENT_ORDER.pickup_time >= '2019-01-01' AND RENT_ORDER.pickup_time <= '2019-12-31'
       GROUP BY CAR_INFO.plate;
```

输出表格样式:

![image-20200611200349517](http://shaw.wang:9888/images/2020/06/11/image-20200611200349517.png)

4. 列出**当年**来累计出租天数最多的 10 辆车的出租信息，按累计出租天数降序排序后输出。

   TIPS: 第3行的365需要换成「今天是今年的第几天」, 才能得到租用率（单位：1%）

```sql
SELECT CAR_INFO.plate AS "车牌号",
  	   CAR_INFO.cname AS "车辆名称",
  	   SUM(TIMESTAMPDIFF(DAY, RENT_ORDER.pickup_time, RENT_ORDER.actual_dropoff_time)) AS "累计出租天数",
  	   SUM(TIMESTAMPDIFF(DAY, RENT_ORDER.pickup_time, RENT_ORDER.actual_dropoff_time))/365*100 AS "租用率",
         SUM(RENT_ORDER.actual_fee) AS "营业额"
         FROM CAR_INFO, RENT_ORDER
         WHERE CAR_INFO.cid=RENT_ORDER.cid
         AND RENT_ORDER.pickup_time > '2019-1-1' 
         GROUP BY CAR_INFO.plate
         ORDER BY "累计出租天数" DESC
         LIMIT 10;
```

输出表格样式:

![image-20200611201221038](http://shaw.wang:9888/images/2020/06/11/image-20200611201221038.png)