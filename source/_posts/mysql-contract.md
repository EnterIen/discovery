---
title: 团队 MySQL 开发规范
date: 2022-02-06 20:55:44
tags:
categories: "MySQL CookBook"
toc: true
top: 2
---
## 一、规范背景及说明

系统趋于稳定，着手开发新零售系统，为了避免系统“野蛮生长“，开发和维护都需要一套更好的 SQL 规范进行数据的管理。该规范旨在达到以下目标(**即亟待解决的问题**)：

*  字段风格统一,避免开发不一致性
* 选用合理的字段类型,节省服务器成本
* 优化字段类型和索引,提高读写效率
* 对 SQL 变更进行版本管理,降低运维风险

<!--more-->

![](images/WechatIMG347.jpeg)

## 二、SQL 时间成本
> 先搞清楚问题，然后有的放矢
*   `I/O`成本
    
    我们的表经常使用的`MyISAM`、`InnoDB`存储引擎都是将数据和索引都存储到磁盘上的，当我们想查询表中的记录时，需要先把数据或者索引加载到内存中然后再操作。这个从磁盘到内存这个**加载的过程**损耗的时间称之为`I/O`成本。
    
*   `CPU`成本
    
    读取以及检测记录是否满足对应的搜索条件、对结果集进行**排序、分组**等这些操作损耗的时间称之为`CPU`成本。

> 对于`InnoDB`存储引擎来说，页是磁盘和内存之间交互的基本单位(每页默认16KB)，`InnoDB`规定读取一个页面花费的成本常数默认是`1.0`，读取以及检测一条记录是否符合搜索条件的成本常数默认是`0.2`

## 三、规范流程及优化思路
![](images/截屏2021-09-10上午1.15.48.png)
## 四、规范内容

### 1.表与字段
#### 1.1 命名
1.1.1 结合业务
1.1.2 避免使用**保留关键字**
1.1.3 表和字段名只能使用字母、数字和下划线组成，一律**小写**
#### 1.2 注释
1.2.1 表名和字段都需要注释
#### 1.3 编码
1.3.1 规定使用 `utf8mb4` 字符集和 `utf8mb4` 比较规则
1.3.2 命令：`SET NAMES 字符集名` 可以统一以下系统变量,保证不乱码
***character\_set\_client***、***character\_set\_connection***、***character\_set\_results***
#### 1.4 数据类型
1.4.1 整型
|   数据类型  | 占用字节    |  有符号最小值   |  有符号最大值   |无符号最小值|有符号最大值|  
| --- | --- | --- | --- |---|---|
|  TINYINT   | 1    |  -127   |  128   | 0 | 255  
|  SMALLINT   | 2    | -32768   |  32767   | 0 | **65535**  
|  INT   | 4   |  -2147483648   |  2147483647   | 0 | 4294967295  

1.4.2 时间类型
|   数据类型  | 占用字节    |  格式   |  说明   
| --- | --- | --- | --- |---|---|
|  date   | 3    |  yyyy-MM-dd   |  只存年月日（用户浏览历史）   | 
|  time   | 3    | HH:mm:ss   |  只存时分秒(线下店铺)   | 
|  datetime   | 8   |  yyyy-MM-dd HH:mm:ss   |     | 
|  timestamp   | 4   |  yyyy-MM-dd HH:mm:ss   |     | 
> datetime 与 timestamp 的区别
> 1.日期范围不同
> 2.**时区敏感**
> 3.占用字节不同
> 4.**查询效率不同**

1.4.3 定点型 DECIMAL
涉及金钱的字段一律使用 DECIMAL 数据类型，规定设置为 DECIMAL(20, 4)

1.4.4 字符类型
`varchar(M)`:变长字段；适用于不确定长度的字符；存储引擎有额外维护,查询效率不如 `char`
`char(M)`:定长字段；适用于手机号、状态值；
> Mysql Server 规定一行所有文本最多存 65535 字节,在不同字符集下 M 都有最大值，超多最大值 varchar 会自动转换为 Text

1.4.5 JSON 类型
> JSON 数据类型最大存储量受系统变量 `max_allowed_packet` 限制
> JSON 数据类型不能有非 NULL 的默认值

JSON 对象查询：
```
假设列 category_front  = 
{
	"space": [
		{
			"first": 124,
			"second": 127
		}
	],
	"category": [
		{
			"first": 28,
			"second": 77
		}
                {
			"first": 26,
			"second": 74
		}
	]
}

可使用 $query->whereJsonContains('category_front->category', ['first' => 26])
```

JSON 数组查询：
```
假设列 category_front = [1,2,3]
可使用 $query->whereJsonContains('category_front, 2)
```

JSON 对象模糊查询
```
$query->whereRaw('json_extract(parameters, ' . "'$.name'" . ') like "%{$conditions["name"]}%"')
```
JSON 对象 key exist 查询
```
$query->whereRaw("json_extract(parameters, 'one', '$.name')")
```
#### 1.5 默认值
不建议字段设置允许为 NULL
1.5.1 [参考：字段为NULL会导致的问题](https://juejin.cn/post/6912242046484873224)
1.5.2 存储引擎额外维护成本,如图：
![存储引擎额外维护成本](images/截屏2021-09-08下午11.53.00.png)
#### 1.6 常用字段约定
1.6.1 对于是否为 xx 的字段 1 默认都表示是，0 默认表示为否
| 参考字段  |   参考含义  |  说明   |
| --- | --- | --- | --- |---|
|   is_enable  | 在职｜离职   | 用于人员在职状态 | 
|   is_ open   | 开启｜关闭   | 用于后台配置按钮 | 
|   is_ show| 展示｜隐藏   | 用户小程序开关控制 | 
|   is_ [状态]|    |  | 
1.6.2 常用字段建议
`人物信息`
| 参考字段  |   参考含义  |  说明   |
| --- | --- | --- | --- |---|
|   nickname  | 用户昵称   | | 
|   username  | 用户真实姓名   | | 
|   avatar  | 头像   | 头像避免使用 image 字段 | 
|   phone   | 手机号   |  | 
|   wechat   | 微信号   |  | 
|   gender   | 用户性别   |  | 
|  userid | 用户 ID | 外键关联 | 
|  openid | 微信应用下用户 ID |  | 
|  unionid | 微信开放平台下用户 ID |  | 
|  external_userid| 企微应用下外部联系人 ID |  | 

`时间信息`

| 参考字段  |   参考含义  |  说明   |
| --- | --- | --- | --- |---|
|   created_at  | 创建时间 | | 
|   updated_at  | 更新时间 | | 
|   begined_at | 开始时间 | | 
|   expired_at  | 过期时间 | | 
|   deleted_at  | 删除时间 | | 
|   rejected_at | 驳回时间 | | 
|   finished_at | 完成时间 | | 

`状态类型信息`
> 表示状态(除去是否状态)和类型的字段还是**建议用汉字其次用英文单词最次用数字**。
> 三者查询效率从高到低依次为：数字>英文>汉字
> 三者**理解效率**从高到低依次为：汉字>英文>数字

| 参考字段  |   参考含义  |  说明   |
| --- | --- | --- | --- |---|
|   wait  | 等待/未开始   | | 
|   ongoing  | 进行中/正在处理   | | 
|   finished  | 已完成   | | 
|   rejected  | 已驳回   | | 
|   deleted  | 已删除   | | 

### 2.索引
#### 2.1 B+ 树索引结构
[索引为什么快？如何利用索引结构的特性](索引为什么快？如何利用索引结构的特性)
![](images/截屏2021-09-08下午8.09.34.png)
![](images/截屏2021-09-09上午12.31.48.png)
#### 2.2 创建与使用索引约定
2.2.1 索引命名建议以 `uq_` 或 `idx_` + 字段名称组成
2.2.2  只为用于搜索、排序或分组的列创建索引，参考 [Order By 的执行原理](https://zhuanlan.zhihu.com/p/95169428)

2.2.3  索引列的散列程度尽量高
2.2.4  索引列的类型尽量小
2.2.5 避免创建冗余索引
2.2.6 避免索引失效 参考 [哪些情况会导致索引失效]()

### 3.分析 SQL 语句
#### **3.1 explain**
![](images/1631248279463.jpg)

> **通常优化至少到 range 级别，最好能优化到 ref**

`const` 通过主键列来定位一条记录
![](images/截屏2021-09-10上午1.08.29.png)
`ref` 通过普通的二级索引列来定位一条记录
![](images/截屏2021-09-10上午1.18.57.png)
`range` 通过范围匹配来定位一条记录
![](images/截屏2021-09-10上午1.21.03.png)
#### **3.2 profiling**
![](images/截屏2021-09-10下午12.38.18.png)
### 4.运维
将每次 SQL 变动纳入版本管理，记录变更的 SQL 语句，参考如下：
**4.1 创建**

```
CREATE TABLE user (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(11) NOT NULL COMMENT ‘用户id’
  `username` varchar(45) NOT NULL COMMENT '真实姓名',
  `email` varchar(30) NOT NULL COMMENT ‘用户邮箱’,
  `nickname` varchar(45) NOT NULL COMMENT '昵称',
  `avatar` int(11) NOT NULL COMMENT '头像',
  `birthday` date NOT NULL COMMENT '生日',
  `sex` tinyint(4) DEFAULT '0' COMMENT '性别',
  `short_introduce` varchar(150) DEFAULT NULL COMMENT '一句话介绍自己，最多50个汉字',
  `user_resume` varchar(300) NOT NULL COMMENT '用户提交的简历存放地址',
  `user_register_ip` int NOT NULL COMMENT ‘用户注册时的源ip’,
  `create_time` timestamp NOT NULL COMMENT ‘用户记录创建的时间’,
  `update_time` timestamp NOT NULL COMMENT ‘用户资料修改的时间’,
  `user_review_status` tinyint NOT NULL COMMENT ‘用户资料审核状态，1为通过，2为审核中，3为未通过，4为还未提交审核’,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_user_id` (`user_id`),
  KEY `idx_username`(`username`),
  KEY `idx_create_time`(`create_time`,`user_review_status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='网站用户基本信息';
```
在某列前新增字段，如在`email`之前新增 `phone` 字段
```
ALTER TABLE `user` ADD COLUMN `phone` CHAR(11) NOT NULL DEFAULT '' COMMENT '手机号'  BEFORE `email`
```

在某列前新增字段，如在`email`之前新增 `phone` 字段
```
ALTER TABLE `user` ADD COLUMN `phone` CHAR(11) NOT NULL DEFAULT '' COMMENT '手机号'  BEFORE `email`
```

在某列前新增字段，如在`email`之前新增 `phone` 字段
```
ALTER TABLE `user` ADD COLUMN `phone` CHAR(11) NOT NULL DEFAULT '' COMMENT '手机号'  BEFORE `email`
```

修改一个字段的类型  
```
alter table user MODIFY new1 VARCHAR(10);  
```
修改一个字段的名称，此时一定要重新指定该字段的类型  
```
alter table user CHANGE new1 new4 int;
```

