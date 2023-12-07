---
title: 数据库系统复习资料
description: 
published: true
date: 2023-06-24T04:04:04.285Z
tags: 
editor: markdown
dateCreated: 2023-06-18T15:10:38.667Z
---

# DB复习

## 典型题目

闫中敏最后一节课如是说：er图一道、规范化一道、写sql、计算关系代数式、数据库基本概念题、查询处理、查询优化、事务管理一两道

往年题目汇总：

| 年份 | 链接                                                         |
| ---- | ------------------------------------------------------------ |
| 2022 | <https://blog.csdn.net/xx2215058009/article/details/125481599> |
| 2021 | <https://blog.csdn.net/weixin_46841376/article/details/118307456> |
| 2019 | <https://blog.csdn.net/weixin_42925536/article/details/94399575> |
| 2018 | <https://blog.csdn.net/LockeSher/article/details/86750766>   |
| 2017 | <https://blog.csdn.net/Jemary_/article/details/80895170>     |
|      |                                                              |



### 数据库规范化

可能出现的问题有：

- 验证无损分解：两个关系时使用快速法，多个关系使用列表法
- 验证保持依赖：通过一个算法
- 判断多值依赖
- 验证BCNF：只需要验证F
- 验证BCNF子模式：需要验证F+，是一个NP问题
- 验证3NF模式：只需要验证F，但是需要先找候选码
- BCNF分解（无损性）：可以不先求正则覆盖
- 3NF分解（保持依赖性）：需要先求正则覆盖
- 寻找候选码

#### 判断无损分解

有快速法和表格法，两个关系时可以使用快速法。

#### 判断保持依赖

参考此文<https://www.scholat.com/vpost.html?pid=143851>

#### 验证是否符合范式

根据定义，主要是3NF和BCNF：

- 3NF：找到候选码；$\beta - \alpha$的**每一个**属性都在候选码内
- BCNF：函数依赖左侧都是主码

#### 寻找候选码

寻找没有在函数依赖右侧出现过的属性。这些属性构成候选码的真子集；然后找就完事儿了。

#### 计算正则覆盖/最小覆盖

#### 范式分解

主要分解3NF和BCNF，根据口诀分解即可。

- 3NF分解：

  ```shell
  保函依赖分解题，先求最小依赖集。
  依赖两侧未出现，分成子集放一边，剩余依赖变子集。
  若要连接成无损，再添候选做子集。
  ```

  本质上是拆分具有依赖的关系集合。

- BCNF分解：

  ```shell
  先求最小依赖集，候码非码成子集。
  余下左侧全候码，完成BCNF题。
  ```

  

  本质上是不断将不符合BCNF的关系分解。需要注意分解的是$F^+$，否则结果会分解不完。

![范式分解](https://s2.loli.net/2023/06/24/71h24JXqlrmGFPw.png)

### SQL

#### 基本语句的编写

#### 对“全部”的表示

其实就是对关系代数里除法的实现和理解，一般：

- 使用`NOT EXISTS (B EXCEPT A)`
- 使用`NOT IN ( NOT IN )`
- 使用`NOT EXISTS (NOT EXISTS)`

需要注意的是通常需要在进行除法之前先将数据投影出来。

#### 空值

使用`IS NULL`进行判断。

#### 使用聚集函数的一些子查询

例如求平均成绩最高的学生等计算，需要注意`GROUP BY `和`HAVING `等关键字的使用。

#### 索引

在一个关系R(A,B,C)上建立索引(B,C)，我们采用B+树实现相应索引，问在执行下列SQL语句的时候是否用到了这个索引，为什么？

```sql
select A
	from r
		where B = 'b1';
```

或者考察对`LIKE`子句，`%`家在匹配字符串左侧时无法使用索引。

### 关系代数

#### 对关系和集合的理解

请说明一个关系中的元组之间是否存在顺序，为什么？

#### 优化语法树

例如，已知instructor(ID,sname,dept_name,salary). 写出下列SQL语句的语法树及其优化后的语法树。

```sql
select sname
	from instructor a,instructor b
		where a.salary>b.salary and dept_name = 'Comp_Sci';
```

可以参考<https://blog.csdn.net/dyyay521/article/details/111987032>

![语法树优化示例1](https://s2.loli.net/2023/06/24/3sYgdB6INlyTK9O.png)

![](https://s2.loli.net/2023/06/24/QqhrDBVXcxlmNFt.png)

![语法树优化示例2](https://s2.loli.net/2023/06/24/hid5SLw6x4orWvF.png)

### ER图

#### 设计ER图

需要注意的是ER图中的多对多关系书写方式

![设计ER图](https://s2.loli.net/2023/06/24/HOISa8y3KfjUBmr.png)

#### 给出关系模式

### 事务

#### 分析事务

![分析事务问题](https://s2.loli.net/2023/06/24/wdIOUTbhSvcxKPV.png)

这种问题需要分类讨论：

- 立即修改
- 延迟修改

#### 判断事务是否可串行化

绘制优先图

#### 判断事务回滚情况

需要根据延迟修改还是立即修改进行判断，写出日志画表即可。

#### 分析两阶段封锁协议和时间戳排序协议

首先是给定事务和要求的协议，给出调度结果。例如：

![分析时间戳排序协议](https://s2.loli.net/2023/06/24/yAGvztE6ChfWQJe.png)

- 时间戳排序协议：给出`R`和`W`两个记号向后根据时间戳推演
- 两阶段封锁：按规则封锁，检查有无死锁

#### 处理死锁

## 引言

### 概述

DBMS：由一个相互关联的数据集合和一组用以访问这些数据的程序组成，这个数据集合通常称作数据库（Database），其中包含了关于某个机构的信息。或简述为系统软件，对数据库进行统一管理和控制。

DBMS的主要功能包括数据定义功能、数据操作功能、数据库的运行管理和数据库的建立和维护功能.

数据：数据库中存储的基本对象。数据即**描述事物的符号记录**。数据的种类包括文字、图形、图像等。数据的特点在于与其语义是不可分的。

数据结构分为逻辑结构和物理结构：

- 逻辑结构：数据之间存在的逻辑关系。
- 物理结构：数据在计算机内的存储方式。

数据库是长期储存在计算机内、有组织的、可共享的大量数据集合。数据库的特征包括：

- 数据按一定的数据模型组织、描述和储存
- 可为各种用户共享
- 冗余度较小
- 易扩展

数据库系统（DBS）是指在计算机系统中引入数据库后的系统，由数据库、数据库管理系统、应用系统和数据库管理员构成。

四个基本概念：

- 数据（Data）
- 数据库（Database）
- 数据库管理系统（DBMS）
- 数据库系统（DBS）

### 数据管理的发展

数据管理经历了三个发展阶段：

1. 人工管理阶段（50年代中期以前）
2. 文件系统阶段（50年代后期-60年代中期）
3. 数据库系统阶段（60年代后期开始）

### 数据视图

数据库系统的一个主要目的是给用户提供数据的抽象视图，即隐藏关于数据存储和维护的某种细节。

型与值的关系：

- 型（模式，Schemas）是对数据的结构和属性的说明。
- 值（实例，Instance）是型的一个具体赋值。
- 型是相对稳定的，值是随时间不断变化的。

数据库将数据设计成三层结构：

- 视图层：只描述整个数据库的部分数据

  > 提供了防止用户访问数据库的某些部分的安全机制

- 逻辑层：描述数据库中的数据以及数据之间的关系，即数据的定义

- 物理层：描述数据存储

![三层结构](https://s2.loli.net/2023/06/14/tH63Bxq4dmC2GaS.png)

数据库模式就是数据库的整体设计，按照三层的划分，包括：

- 子模式/外模式：视图层描述的数据库设计，**逻辑模式的子集**
- 逻辑模式：逻辑层描述的数据库设计
- 物理模式：物理层描述的数据库设计

数据库的实例是指特定时刻存储在数据库中的信息的集合。

数据库的三级视图实现了数据独立性：

- 物理数据独立性：物理模式的改编而不会影响逻辑模式。当物理模式改变时，修改模式/内模式映像，使外模式保持不变，从而应用程序可以保持不变。
- 逻辑数据独立性：当模式改变时，修改外模式/模式映像，使外模式保持不变，从而应用程序可以保持不变，成为数据的逻辑独立性。

为了提高数据的物理独立性和逻辑独立性，使数据库的用户观点，即用户看到的数据库，与数据库的物理方面，即实际存储的数据库区分开来，数据库系统的模式是分级的，美国数据系统语言协商会提出模式、外模式、存储模式三级模式的概念。三级模式结构示意如下：

![三级模式](https://s2.loli.net/2023/06/14/JpOZRYsSofATVMW.png)

> 与独立性的联系：三级模式之间有两级映象。

### 数据模型

数据模型用来对数据的形式进行描述，包括：

- ER模型
- 层次模型
- 网状模型
- 关系模型
- 面向对象模型

### 数据库语言

DML（Data Manipulation Language）：**操纵**那些按照某种适当的数据模型组织起来的数据的语言。**SQL是使用最广泛的DML**。

DDL（Data Definition Langauge）：用于**定义数据库模式以及其他特征**的语言。

### DBMS的运行过程

![DBMS的运行过程](https://s2.loli.net/2023/06/14/pBJKsD6AnOEWlkS.png)

1. 用户向DBMS发出命令
2. DBMS对命令进行语法检查、语义检查、存取权限检查，决定是否执行该命令
3. DBMS执行查询优化，把命令转换为一串单记录的存取操作序列
4. 执行存取操作序列（反复执行以下各步，直至结束）
5. DBMS首先在缓冲区内查找记录，若找到转10，否则转6
6. DBMS查看存储模式，决定从哪个文件存取哪个物理记录
7. DBMS根据6的结果，向操作系统发出读取记录的命令
8. 操作系统执行读取数据的命令
9. 操作系统将数据从数据库存储区送到系统缓冲区
10. DBMS根据用户命令和数据字典的内容导出用户所要读取的数据格式
11. DBMS将数据记录从系统缓冲区传送到用户工作区
12. DBMS将执行状态信息返回给用户

### 数据存储和查询

存储管理器是一个程序模块，提供了数据库中存储的低层数据与应用程序以及向系统提交的查询之间的接口。

查询处理器包括：

- DDL解释器：解释DDL语句，将这些定义记录在数据字典中
- DML编译器：将查询语句中的DML语句翻译成为一个执行方案。
- 查询执行引擎：执行由DML编译器产生的低级指令。

## 关系模型

### 关系模型

#### 关系数据结构

关系：某一时刻对应某个关系模式的内容(元组的集合称作关系。现实世界的实体以及实体之间的各种联系均采用关系来表示。每个属性可能的取值范围（集合）叫做属性的域。属性的值（通常）要求为原子的。

域（Domain）是一组值的集合，这组值具有相同的数据类型。

笛卡尔积

笛卡尔积的子集叫做在域上的关系，用$R({D_1}, {D_2}, ...,{D_n})$表示。其中$R$是关系的名字，$n$是关系的度或目。关系是笛卡尔积中有意义的子集。

关系的性质：

- $n$元组的顺序是无关紧要的。

- 列是同质的。每一列中的分量来自同一个域，是同一类型的数据。

- 不同的列可以来自同一个域，每列必须有不同的属性名。

- 列的次序可以任意交换。

- 任意两个元组不能完全相同。

  > 这个性质由笛卡尔积的性质决定，但是许多关系数据库都没有遵循这一性质。

- 每一分量必须是不可再分的数据。满足这一条件的关系称作满足第一范式（1NF）的

**null（空值）**：是一个特殊的值，表示值未知或者不存在。空值给数据库访问和更新带来很多困难。他有两种含义：

- 无意义（数据项没有值）
- 值未知（值存在，但没有获取该信息）

空值参与算术运算、比较运算的结果都是Null，参与逻辑运算时除`Null or true = true`和`Null and false = false`以外均为Null。

### 关系模式

数据库由多个关系组成。

关系模式：关系的描述称作关系模式，包括关系名、关系中的属性名、属性向域的映象、 属性间的数据依赖关系等。关系模式可以表示为$R(UmDmdom,F)$：

- R：关系名
- U：组成该关系的属性名集合
- D：属性组U中属性所来自的域
- dom：属性向域的映像集合
- F：属性间的数据依赖关系集合

关系模型的组成包括：

- 关系数据结构
- 关系完整性约束
- 关系操作集合

关系：某一时刻对应某个关系模式的内容(元组的集合)称作关系。

#### 码

码是能唯一标识实体的属性集，他是整个实体集的性质，而不是单个实体的性质。

码的作用：必须有一种能够区分给定关系中的不同元组的方法。我一般用元组中的属性来表明，即一个元组的属性值必须是能够唯一区分元组的，一个关系中没有两个元组在所有属性上的取值都相同

| 码     | 作用                                                         |
| ------ | ------------------------------------------------------------ |
| 超码   | 超码是一个或者多个属性的集合，这些属性的组合可以使我们在一个关系中唯一地标识一个元组 |
| 候选码 | 最小的超码称为候选码，即超码的任意真子集都不能成为超码       |
| 主码   | 从一个关系的多个候选码中选定一个作为主码                     |
| 外码   | 一个关系模式$r_1$可能在它的属性中包含另一个关系模式$r_2$的主码，这个属性在上称作在$r_1$上参照$r_2$的外码（ $r_1$和$r_2$可以是同一个关系）。关系r1称作外码依赖的参照关系，关系r2称作外码的被参照关系 |

> 习惯上把主码属性放在其他属性前面，并且加下划线

#### 完整性约束

数据库完整性 (Database Integrity)）：是指数据库中数据的正确性和相容性。由各种各样的完整性约束来保证，数据库完整性约束可以通过DBMS或应用程序来实现

完整性包括三种：

- 实体完整性约束
- 参照完整性约束
- 用户定义的完整性约束

**实体完整性**要求关系的主码中的值不能为空值。实体完整性规则规定基本关系的所有**主属性**都不能取空值。

关系模型必须遵守实体完整性规则的原因：

1. 实体完整性规则是针对基本关系而言的。一个基本表通常对应现实世界的一个实体集或多对多联系
2. 现实世界中的实体和实体间的联系都是可区分的，即它们具有某种唯一性标识
3. 相应地，关系模型中以主码作为唯一性标识
4. 主码中的属性即主属性不能取空值

**参照完整性**要求如果关系$R_2$的外部码$F_k$与关系$R_1$的主码$P_k$相对应，则$R_2$中的每一个元组的$F_k$值或者等于$R_1$中某个元组的$P_k$值，或者为空值。

**用户定义的完整性**要求用户针对具体的应用环境定义的完整性约束条件。

抽象的查询语言：

- 关系代数：用对关系的运算来表达查询，需要指明所用
- 操作关系演算：用谓词来表达查询，只需描述所需信息的特性
- 元组关系演算：谓词变元的基本对象是元组变量
- 域关系演算：谓词变元的基本对象是域变量

## SQL

### 简介

SQL：Structured Query Language，SQL包含：

- 数据定义语言（DDL）
- 数据操纵语言（DML）
- 数据控制语言（DCL）
- 事务控制
- 嵌入式SQL和动态SQL

### SQL数据定义

SQL的数据定义语言（DDL）能够定义每个关系的信息，包括：

- 每个关系的模式
- 每个属性的值域
- 完整性约束
- 以及以后我们将要看到的一些信息，比如每个关系的索引集合，每个关系的安全性和权限信息，磁盘上每个关系的物理存储结构。

![SQL的数据定义语句](https://s2.loli.net/2023/06/21/QRFTO7XESNj8MsY.png)

一些SQL的基本数据类型：

|            类型            | 介绍                                                         |
| :------------------------: | :----------------------------------------------------------- |
|         `char(n)`          | 固定长度的字符串                                             |
|        `varchar(n)`        | 可变长度的字符串                                             |
|           `int`            | 整数类型（和机器相关的整数的有限子集）                       |
|         `smallint`         | 小整数类型（和机器相关的整数类型的子集）                     |
|       `numeric(p,d)`       | 定点数， 精度由用户指定 这个数有 p 位数字，其中 d 位数字在小数点右边 |
| `real`, `double precision` | 浮点数与双精度浮点数，精度与机器相关                         |
|         `float(n)`         | 浮点数与双精度浮点数，精度与机器相关                         |
|        `date/time`         | 日期（年、月、日）/时间（小时、分、秒）                      |

SQL建表示例：

```sql
CREATE TABLE S
	(sno CHAR(4),
   sname CHAR(8) NOT NULL,
   age SMALLINT,
   sex CHAR(1),
   constraint pk_s PRIMARY KEY(sno),
   CHECK(sex='0' OR sex='1')
  )
```

常用完整性约束有：

- 主码约束`PRIMARY KEY`
- 参照完整性约束`FOREIGN KEY (something) REFERENCES r`
- 唯一性约束`UNIQUE`
- 非空值约束`NOT NULL`

索引的目的：大部分的查询只涉及数据库中的少量数据，建立索引是加快查询速度的有效手段建立索引。有关说明：

- 可以动态地定义素引，即可以随时建立和删除索引
- 不允许用户在数据操作中指定素引，素引如何使用完全由DBMS决定
- 应该在使用频率高的、经常用于连接的列上建索引
- 一个表上可建名个素引。素引可以提高查询效率，但素引过多耗费空间，且降低了插人、州除、更新的效率
- 索引实现 （DBMS)：Bt树，散列 (hash）

### SQL数据查询

SQL的数据操纵语言 （DML） 提供从数据库中查询信息，以及在数据库中插入元组、删除元组、修改元组的能力 。

SQL允许在关系和查询结果中保留重复的元组，缺省为保留重复元组，也可用关键字`all`显式指明。若要去掉重复元组，可用关键字`distinct`或`unique`指明。

`from`子句列出查询的对象表（关系）当目标列取自多个表时，在不混淆的情况下可以不用显式指明来自哪个关系,相当于表的笛卡尔积，不是自然连接。

自然连接：自然连接只考虑两个关系模式中都出现的属性上取值相同的元组对，并且相同属性的列只保留一个副本。

> 自然连接中的危险：注意有些属性具有相同名称，但它们实际的意义是不同的，在这种情况下，它们可能错误的被认为是相同的属性 

排序使用`asc`和`desc	`关键字。当排序列含空值时，`asc`排序列为空值的元组最后显示，`desc`排序列为空值的元组最先显示。

### 聚集函数

聚集函数的性质：

- 聚集函数作用于集合/多重集，返回单值（—个或多个属性的关系，其中只包含一个元组）
- 聚集函数在输人为空集的情况下也返回一个关系 （一个属性一个元组的关系），关系中返回结果为null值。count两数例外，在输人为空集的情况下，count返回0
- 强制作用于集合，使用distinct
- 除了`count(*)`以外的聚集两数，均忽略`null`值。如果应用的需求不希望忽略`null`值，可以使用`nvl`函数进行处理

`count(*)`与`count(column_name)`的区别：

- `count(*)`返回满足条件的元组的总个数（即使一个元组的所有属性取值均为null也会被计算在内），`count(属性名)`返回该属性中取值不为`null`的总个数；
- 不允许在`count(*)`中使用`distinct`

`group by`子句：

- 将表中的元组按指定列上值相等的原则分组
- 在每一分组上使用聚集函数得到单一值

### 嵌套子查询

#### 检查元素是否在一个集合里

#### 跟集合中的一个或多个元素比较

#### 判断元素/元组是否是否存在于集合中

#### “全部”概念处理方法

“全部”概念在SQL中有三种写法：

- 超集superset：`NOT EXISTS (B EXCEPT A)`
- 关系代数（$$）：`NOT IN (NOT IN)`
- 关系演算：`NOT EXISTS (NOT EXISTS)`

#### 测试集合是否存在重复元组



### 视图

### 完整性约束

### JDBC

JDBC 是一个支持SQL 的Java API，用于与数据库系统通信。他与数据库的通信模型建立在：

1. 打开一个连接
2. 创建一个statement对象
3. 使用statement对象执行查询，发送查询并取回结果
4. 处理返回结果

##形式化关系查询语言

### 关系代数基本运算

关系代数包括六种基本运算符：

- 选择，selection
- 投影，project
- 更名，rename
- 并，union
- 集合差，set difference
- 笛卡尔积，Cartesian product

一些运算律：

- 投影和并可以分配
- 投影和差不可以分配

附加的关系运算：

| -            | -    | -    |
| ------------ | ---- | ---- |
| 交           |      |      |
| 自然连接     |      |      |
| 赋值         |      |      |
| 外连接       |      |      |
| $\theta$连接 |      |      |

> 各种外连接都是在自然连接的基础上扩展的

#### 除运算

#### 广义投影

#### 聚集函数

#### 空值处理

元组的一些属性可以表示为空值，含有null的算术表达式的结果为null，聚集函数直接忽空值。在各种运算中，null被视作相同的值。

### 拓展运算

#### 除运算

数据库考试中经常会出现关系运算题目而一般的加减乘运算相对比较简单，通常不会直接出题，比较容易乱的是除法：

![除运算的定义](https://s2.loli.net/2023/06/22/1gFNI5oLxOiCtTd.png)

详情参考<https://blog.csdn.net/t_1007/article/details/53036082>

## 数据库设计和ER模型

E-R图的作用：

- 帮助澄清用户数据需求，分析员和用户对数据需求达到高度一致
- 数据逻辑模型设计的标准

一个数据库可以建模为：一组实体集，多个实体间的相互关联。

实体：客观存在并可相互区分的事物叫实体（唯一标识）。实体是现实世界中可区别于所有其他对象的一个“事物”或“对象”。相关的概念有：

- 属性：实体集中每个成员具有的描述性性质。一个实体由若干个性质刻画。

  > 属性还可以分为简单属性（不可再分的属性）和复合属性（可以进一步划分的属性）。
  >
  > 属性的类型包括单值属性（每一个特定的实体在该属性上的取值唯一）、多值属性（个特定的实体在该属性上的有多于一个的取值，例如某个实体的联系方式）、派生属性（可以从其他相关的属性或实体派生出来的属性值）与基属性。数据库中，一般只存基属性值，而派生属性只存其定义或依赖关系，用到时再从基属性中计算出来。但是不排除基属性和派生属性均保存在数据库中的现象

- 域：属性的取值范围。

- 实体集的属性是将实体集映射到域的函数。

码有许多种：

- 超码：能唯一标识实体的属性或属性组。超码的任意超集也是超码

- 候选码：任意真子集都不能成为超码的最小超码。

- 主码：所有候选码中选定一个用来区别同一实体集中的不同实体。

  > 一个实体集中任两个实体在主码上的取值不能相同，实体集属性中作为主码的属性用下划线来标明。

实体集：是具有相同类型及共享相同性质（属性）的实体集合，如全体学生。一个实体集是相同类型，即具有相同性质（或属性）的实体集合。组成实体集的各实体称为实体集的外延（Extension）

联系集是相同类型联系的集合，是$n \geq 2$个实体集上的数学联系。联系也可以具有描述性属性。联系集的度是指联系集涉及的实体集数量，数据库系统中大部分联系集都是二元的。

联系集的码：参与联系的实体集的主码的集合形成了联系集的超码。例如(sno, cno) 构成了联系sc的超码。关系集的主码依赖于关系集映射的基数。

参与：实体集之间的关联称为参与，即实体参与联系。

角色(Role)：实体在联系中的作用称为实体的角色，由于参与一个联系的实体集通常是互异的，角色是隐含的一般不需要指定。当同一个实体集不止一次参与一个联系集时，为区别各实体的参与联系的方式，需要显式指明其角色。

> 如果实体集E中的每个实体都参与到联系集R中的至少一个联系，则称E全部参与R；如果实体集E中只有部分实体参与到联系集R的联系中，则称E部分参与R

映射基数表示一个实体通过一个联系集能关联到的实体的个数，在描述二元联系集时非常有用。对于二元联系集来说，映射基数必然是一下情况之一：

- 一对一
- 一对多
- 多对一
- 多对多

> 一对实体在特定联系集中至多有一个关系。因此决定候选码时必须考虑联系集的映射基数，在选择主键的时候必须考虑联系集的寓语义信息以防有多个候选码。

存在依赖：如果实体x的存在依赖于实体y的存在，则称x存在依赖于y。y称作支配实体，x称作从属实体。如果y被删除，则x也要被删除。

弱实体集：弱实体集是一类特殊的实体集，它们依赖于其它实体集而存在，如订单中的分项以及公司员工的家属等；相应的，可以独立存在的实体集被称为强实体集。强实体集的成员必然是支配实体，而弱实体集的成员是从属实体。弱实体集所依赖的实体集称为标识实体集（identifying entity set），相应的关系为标识联系（identifying relationship）。由于弱实体集依赖于标识实体集而存在，在组成联系时，必须是所有的实体全部参与，不允许部分参与。弱实体集与强实体集之间是一对多的联系。

弱实体集通常没有主键。弱实体集的属性中，用来与标识实体集的键结合以识别一个弱实体集的属性称为部分键（partial key）：弱实体集的主键 = 它的标识实体集的键+它的部分键。

### ER绘制

参考资料：

- <https://wenku.baidu.com/view/9b4cc3aadd3383c4bb4cd237.html?_wkts_=1687417935762>

### 向关系模式的转换

### 扩展E-R特性

#### 特殊化

自顶而下设计过程; 实体集可能包含一些子集，子集中的实体在某些方面区别于实体集中的其他实体；这些子集变为低层次的实体集，拥有不适用于高层次实体集的属性或一些部分参与的关系。

在E-R图中，特化用从特化实体指向另一个实体的空心箭头来表示，我们称这种关系为ISA关系 (E.g., instructor “is a” person)。

属性继承：高层实体集的属性可以被低层实体集继承。低层实体集（或子类）同时还继承地参与其高层实体（或超类）所参与的实体集

#### 概括/泛化

自底而上的设计过程– 多个实体集根据共同的特征综合成一个较高层的实体集。概化只不过是特化的逆过程; 在E-R图中，我们对概化和特化的表示不作区分。

实体可以是给定低层次实体集成员的约束。在一个概化中一个实体集是否可以属于多个低层实体集：

- 不相交：不相交约束要求一个实体至多属于一个低层实体集
- 重叠：同一个实体可以同时属于同一个概化中的多个低层实体集

完全性约束：定义高层实体集中的一个实体是否必须至少属于该概化/特化的一个低层实体集：

- 全部概化: 每个高层实体必须属于一个低层实体集。
- 部分概化: 允许一些高层实体不属于任何低层实体集。

#### 属性继承

#### 聚集

通过聚集来减少冗余：

- 将关系作为抽象实体
- 允许关系间存在关系
- 将关系抽象用于新实体

为了表示聚集，需要包含以下信息：

- 聚集关系的主键
- 相关实体集的外键
- 任何描述属性

## 关系数据库设计

### 好的设计模式

更大的模式会导致信息重复：信息冗余大、数据不一致等，因此我们会选择更小的模式。

原子域：域元素被认为是不可分的单元，称域是原子的。

### 函数依赖

**函数依赖**：在合法关系集合上的约束，要求一个特定的属性集合的值唯一决定另一个属性集合的值。一个函数依赖是一个码标识的概化。

> 函数依赖可以帮助我们确定哪些属性是冗余的，哪些属性是必要的，从而设计出更加高效和优化的数据库结构。

函数依赖与范式有密切的关系。范式是一种规范化的技术，用于帮助我们设计出符合规范的数据库结构。函数依赖是范式的一个重要概念，它被广泛应用于不同的范式中。例如，第一范式（1NF）要求关系中的每个属性都是原子的，不可再分的，而函数依赖可以帮助我们判断哪些属性是原子的，哪些属性可以进一步拆分。第二范式（2NF）要求关系中的每个非主属性都完全依赖于主属性，而函数依赖可以帮助我们判断哪些属性是主属性，哪些属性是非主属性，以及它们之间的依赖关系。

**部分函数依赖**是指在一个关系中，某个非主属性依赖于关系中的一部分而不是全部主属性。换句话说，如果一个非主属性可以通过关系中的部分主属性来确定，而不是全部主属性，那么就称这个非主属性存在部分函数依赖。

> 举个例子，假设有一个关系表格包含学生的信息，其中包括学生的学号、姓名、性别和年龄。如果我们知道了一个学生的学号和姓名，那么就能够唯一地确定这个学生的性别，因为同一学号和姓名的学生性别是唯一的。但是如果我们只知道学号或者只知道姓名，就不能确定学生的性别了。因此，性别就存在部分函数依赖，它依赖于学号和姓名这两个属性的组合而不是单独的学号或姓名。

部分函数依赖在数据库设计中是不推荐的，因为它会导致数据冗余和不一致性。为了避免部分函数依赖，我们需要对关系进行规范化，将非主属性拆分成更小的属性集合，以消除部分函数依赖。

**传递函数依赖**

给定函数依赖集F，必定有一些其他的函数依赖被F逻辑蕴涵。能够从给定F集合推导出的所有函数依赖的集合称为F的闭包，使用$F^+$表示。

对于属性集$\alpha$，我们将函数依赖集F下被$\alpha$函数确定的所有属性的集合称为F下$\alpha$的闭包

### 多值依赖

描述型：关系模式R(U)，、、  U，并且 = U –  – ，多值依赖  成立当且仅当对R(U)的任一关系实例r，给定的一对(1，1)值，有一组的值，这组值仅仅决定于值而与值无关

![多值依赖示意图](https://s2.loli.net/2023/06/22/GoTvk4IjwQlux3t.png)

### Armstrong公理系统

Armstrong公理系统是一种用于推导函数依赖的集合的公理系统。它由计算机科学家William W. Armstrong于1974年提出，是现代数据库理论中最重要的基础之一。

Armstrong公理系统包括三个基本公理和一个推论规则：

1. 自反律（Reflexivity）：如果X是一个属性集合，那么X → X是一个成立的函数依赖。

2. 增广律（Augmentation）：如果X → Y成立，那么对于任何属性集合Z，都有XZ → YZ成立。

3. 传递律（Transitivity）：如果X → Y成立，Y → Z成立，那么X → Z也成立。

4. 推论规则（Inference Rule）：如果X → Y成立，且Y → Z成立，那么可以推导出X → Z成立。

这些公理和推论规则可以用于推导出函数依赖的集合。例如，如果我们知道X → Y和Y → Z成立，那么我们可以使用第四个推论规则推导出X → Z也成立。

Armstrong公理系统是数据库理论中非常重要的理论基础，它可以帮助我们推导出函数依赖集合，进而进行关系规范化和数据库设计。

给定属性集，可以这样使用：

- 判断超码：计算$\alpha^+$检查$\alpha^+$是否包含$R$中的所有属性。
- 判断函数依赖：通过检查是否$\beta \subset \alpha^+$，我们可以检查函数依赖α   是否成立（换句话说，是否属于F+ ）
- 计算$F$的闭包

### 正则覆盖

函数依赖集可能拥有可从其他推导出来的冗余的依赖，F的正则覆盖是一个与F相等的**最小的函数依赖集**，没有冗余的依赖，也没有**无关属性**。

> 无关属性：如果去除一个函数依赖中的属性，不会改变该函数依赖集的闭包，则称该属性是无关的(extraneous)

无关属性的核心：能够被函数依赖集F逻辑蕴涵的函数依赖，不必出现在F中。

满足下列条件的函数依赖集F称为正则覆盖，记作$F_c$：

- Fc 与 F 等价（F逻辑蕴涵Fc,中的所有依赖； Fc 逻辑蕴涵 F 中的所有依赖）
- Fc 中任何函数依赖都不含无关属性
- Fc 中函数依赖的**左半部都是唯一**的

F的正则覆盖作用：

- 假设在一个关系模式上有一个函数依赖集F。当用户对于关系进行更新时，数据库系统将保证此操作不会破坏任何一个函数依赖
- 可以通过测试与给定函数依赖集有相同闭包的简化集的方式，来降低检测的开销

> 正则覆盖未必唯一。

### 范式

参考资料：

- <https://www.cnblogs.com/caiyishuai/p/10975736.html>
- <https://blog.csdn.net/sumaliqinghua/article/details/86246762>
- <https://blog.csdn.net/sumaliqinghua/article/details/85872446#commentBox>

首先要明白，范式具有包含关系，也就是2NF包含1NF，3NF包含2NF，以此类推。

1. 第一范式（1NF）：要求关系模式的每个属性都是原子型的，即不可再分解成更小的数据项。
2. 第二范式（2NF）：要求关系模式必须满足第一范式，并且不存在非主属性对主键的部分依赖。
3. 第三范式（3NF）：要求关系模式必须满足第二范式，并且不存在非主属性对主键的传递依赖。
4. 巴斯-科德范式（BCNF）：要求关系模式必须满足第一范式，并且不存在任何属性对候选键的部分依赖。
5. 第四范式（4NF）：要求关系模式必须满足BCNF，并且不存在多值依赖。
6. 第五范式（5NF）：要求关系模式必须满足第四范式，并且不存在联接依赖。

#### 1NF

如果某个域的元素被认为是不可再分的单元，那么这个域就是原子的(atomic)。如果一个关系模式R的所有的属性域都是原子的，我们称关系模式R属于第一范式(first normal form, 1NF)。

称一个关系模式R属于第一范式，如果R的所有属性的域都是原子的。

> 非原子值使得存储变复杂并且导致数据存贮的冗余。

#### 2NF

对于关系模式S(sno , sname, dno , dean , cno, score)，主码(sno, cno)，有不良特性：

- 插入异常：如果学生没有选课，关于他的个人信息及所在系的信息就无法插入
- 删除异常：如果删除学生的选课信息，则有关他的个人信息及所在系的信息也随之删除
- 更新异常：如果学生转系，若他选修了k门课，则需要修改k次
- 数据冗余：如果一个学生选修了k门课，则有关他的所在系的信息重复

教材定义：若$R\in 1NF$，且每个属性满足下列准则之一：

- 他出现在一个候选码中
- 他没有部分依赖于一个候选码

> 消除非主属性对码的部分依赖

所谓“**非主属性对主键的部分依赖**”，指的是关系模式中某个非主属性（即不属于主键的属性）依赖于主键的某一部分，而不是依赖于整个主键。换句话说，如果一个关系模式中存在非主属性只依赖于主键的某一部分，而不依赖于整个主键，那么这个关系模式就不符合2NF的要求。

例如，一个学生选课信息表的关系模式为（学生编号，课程编号，课程名称，学分），其中主键为（学生编号，课程编号）。如果课程名称只依赖于课程编号，而不依赖于学生编号，那么就存在非主属性对主键的部分依赖，这个关系模式就不符合2NF的要求。

为了满足2NF的要求，可以将上述关系模式拆分为两个关系模式：学生信息表（学生编号，学生姓名，学生性别）和选课信息表（学生编号，课程编号，学分）。这样，每个关系模式都只包含一个主键，不存在非主属性对主键的部分依赖，满足2NF的要求。也就是**把部分依赖的属性与自己依赖的属性拆出去**。

#### 3NF

对以下关系模式S_SD(sno , sname , dno , dean)，有不良特性：

- 插入异常：如果系中没有学生，则有关院系的信息就无法插入
- 删除异常：如果学生全部毕业了，则在删除学生信息的同时有关院系的信息也随之删除了
- 更新异常：如果学生转系，不但要修改dno，还要修改dean，如果换系主任，则该系每个学生元组都要做相应修改
- 数据冗余：每个学生存储了所在系主任的信息

教材定义为：关系模式$R<U, F>$中，$F^+$中所有函数依赖$\alpha \rightarrow \beta$，至少有以下之一成立，则称$R\in 3NF$：

- $\alpha \rightarrow \beta$是平凡的函数依赖
- $\alpha$是超码
- $\beta - \alpha$的每一个属性$A$都包含在$R$的候选码中

> 消除非主属性对码的传递依赖.
>
> 特别的，当R中所有属性都是主属性，$R\in 3NF$

所谓“**非主属性对主键的传递依赖**”，指的是一个非主属性依赖于另一个非主属性，而这个非主属性又依赖于主键的情况。

举个例子，假设有一个关系模式R(A, B, C)，其中A是主键。如果B依赖于A，而C又依赖于B，那么C就存在非主属性对主键的传递依赖。这是因为C并不直接依赖于主键A，而是通过非主属性B间接依赖于主键A。

这种情况下，如果修改B的值，会导致C的值发生变化，从而破坏数据的一致性和完整性。为了消除非主属性对主键的传递依赖，可以将关系模式R拆分为两个关系模式R1(A, B)和R2(B, C)，其中R1的主键为A，R2的主键为B。这样就可以消除非主属性对主键的传递依赖，使得数据更加规范化和完整。

#### BCNF

不良特性有：

- 插入异常：如果没有学生选修某位老师的任课，则该老师担任课程的信息就无法插入
- 删除异常：删除学生选课信息，会删除掉老师的任课信息
- 更新异常：如果老师所教授的课程有所改动，则所有选修该老师课程的学生元组都要做改动
- 数据冗余：每位学生都存储了有关老师所教授的课程的信息

原因：**主属性对码的不良依赖**。

教材定义为：关系模式$R<U, F>$中，$F^+$中所有函数依赖$\alpha \rightarrow \beta$，至少有以下之一成立，则称$R\in 3NF$：

- $\alpha \rightarrow \beta$是平凡的函数依赖
- $\alpha$是超码

BCNF的本质是在（只考虑函数依赖的前提下）只讲一件事，非码决定因素的相关决定关系讲述了另外一件事，有多个码是一件事的不同方面,本质仍是一件事。BCNF一定无损分解。

> BCNF比较抽象，略作解释：在学生信息表里，学号是一个候选码，学号可确定学生姓名；（班级，学生姓名）也是一组候选码，有(班级，学生姓名）->学号，因此在主属性问形成了传递依赖。

与3NF相比，BCNF增加了以下约束：

1. 每个非主属性必须完全依赖于主键。这意味着每个非主属性必须与主键形成一个候选键，而不仅仅是与主键的某一部分相关。
2. 不存在主属性之间的依赖。这意味着主属性之间不能存在函数依赖，否则需要将主属性拆分为多个关系模式。

简单验证: 检查关系模式R是否属于BCNF，仅需检查给定集合F中的函数依赖是否违反BCNF就足够了，不用检查F+中的所有函数依赖。

3NF与BCNF的特点：

|     3NF      |          BCNF          |
| :----------: | :--------------------: |
| 满足无损分解 |      满足无损分解      |
|   保持依赖   | **可能不满足保持依赖** |

### 模式分解

无损连接判别方法—快速法：将R分解成 R1 和 R2 ，若F+中存在以下至少一个依赖，则是无损分解：${R_1}\cap{R_2}\rightarrow {R_1}, {R_1}\cap{R_2}\rightarrow {R_2}$，以上的函数依赖是无损分解的充分条件；这些依赖是必要条件当且仅当所有约束都是函数依赖。

无损连接判别方法—表格法（充分必要条件）

保持函数依赖的判定：

![判定保持函数依赖的算法](https://s2.loli.net/2023/06/20/TS8yqE7MIgDGhB5.png)

#### 3NF分解

3NF分解分为保持依赖和无损连接两步，为了说明求解保持依赖，我们先要会求最小依赖集。口诀为：

```shell
右侧先拆单，依赖依次删。
还原即可删，再拆左非单。
```

![求解最小依赖集示例](https://s2.loli.net/2023/06/22/iUGfRmwMnhvZ19D.png)

3NF分解整体的口诀为：

```shell
保函依赖分解题，先求最小依赖集。
依赖两侧未出现，分成子集放一边，剩余依赖变子集。
若要连接成无损，再添候选做子集。
```

![3NF分解例1](https://s2.loli.net/2023/06/22/gkh5zAoy6CVNIGQ.png)

![3NF分解例题2](https://s2.loli.net/2023/06/22/j5w38KGb91IYsmJ.png)

#### BCNF分解

将关系模式R<U,F>分解为一个BCNF的基本步骤是：

```shell
先求最小依赖集，候码非码成子集。
余下左侧全候码，完成BCNF题。
```

![BCNF例1](https://s2.loli.net/2023/06/22/i438TjLPKRqvU9Y.png)

## 数据存储和数据访问

### 文件组织

文件组织：为了减少块访问时间，我们可以按照与预期的数据访问方式最接近的方式来组织磁盘上的块。包括两种方法：

- 定长记录
- 变长记录

文件中记录的组织方式有四种：

- 堆文件：一个记录可以放在文件中任何地方，只要有足够的空间。

- 顺序文件：记录根据”搜索码“的值顺序存储

  > 适用于需要对整个文件进行顺序处理的应用程序

- 哈希文件：在每条记录的某些属性上计算一个哈希函数

- 每个关系的记录存储在一个单独的文件中：在多表聚簇文件组织中一个文件可以存储多个不同关系的记录

  > 使用多表聚簇文件组织，一个文件可以存储几个关系，减少IO次数。

数据字典（系统目录）存储元数据，也就是关于数据的数据：

- 关系的有关信息：关系的名字，关系中属性的名字、类型、长度，视图的名字和定义，完整性约束
- 用户和帐号信息
- 统计和描述数据：每个关系中元组的总数
- 文件的组织信息：存储组织（顺序/哈希等）、存储位置
- 索引信息

数据字典通常存储成非规范的形式，以便快速存取

#### 定长记录

一种简单的实现方法是从字节$n \times (i-1)$开始存储记录$i$，$n$是每个记录的长度。

这种存储方式存在的缺点包括：

- 访问记录很容易，但是记录可能会分布在不同的块上。
- 删除记录困难。删除记录所占的空间必须由文件的其他记录来填充，或者我们自己必须用一种方法标记删除的记录，使得它可以被忽略 。因此删除有三种选择：
  1. 移动后面的记录
  2. 将记录`n`移到`i`
  3. 不移动记录, 但是链接所有的空闲记录到一个 free list

#### 变长记录

可变长度的记录的几种方式：

- 存储在一个文件中的记录有多个记录类型
- 记录类型允许记录中某些字段值的长度可变（例如`varchar`）

![](https://s2.loli.net/2023/06/16/gzTJu6vb5x1RLrE.png)

- 实际记录是从块的尾部开始排列
- 块中空闲空间是连续的，在块头数组的最后一个条目和第一条记录之间
- 如果插入一条记录，在空闲的尾部给这条记录分配空间，并且将包含这条记录大小和位置的条目添加到块头中
- 如果一条记录被删除，他所占用的空间被释放，并且他的条目被设置成删除状态。块中被删除记录之前的记录被移动，以此重用删除的空闲空间，并且所有的空闲空间中在块头数组的最后一个条目和第一条记录之间
- 块头中的空闲末尾指针也要做适当的调整

对于图片、音频等数据，这些数据比块大很多，可以使用 Blob和clob数据类型，大对象一般存储到一个特殊文件中，而不是与记录的其他属性存储在一起，然后一个指向该对象的指针存储到包含该大对象的记录中。

#### 数据库缓冲区

数据库系统尽量减少磁盘和内存之间的数据块传输数量。可以在主存中保留尽可能多的块来减少磁盘访问次数。

缓冲区：部分主存用于存储磁盘块的副本。

当程序需要从磁盘中得到一个块时，将调用缓冲区管理程序，检查磁盘块是否在缓冲区内。操作系统用过去块访问模式来预测未来的访问查询：系统替换掉那些最近最少使用的块 (LRU 策略)

### 索引

索引机制用于加快访问所需数据的速度。搜索码是用于在文件中查找记录的属性或属性集。在两种基本的索引类型：

- 顺序索引：基于搜索码值的顺序排序
- 散列索引：将值平均分布在若干散列桶中。

索引的评价指标包括：

- 能有效支持的访问类型
- 访问时间
- 插入时间
- 删除时间
- 空间开销

#### 顺序索引

在一个顺序索引中，索引项按照搜索码值的排序顺序存储。例如，图书馆作者目录。

- 主索引（聚集索引）：包含记录的文件按照某个搜索码指定的顺序排序，那么该搜索码对应的索引称为主索引。通常使用主码作为主索引。
- 辅助索引（非聚集索引）：搜索码指定的顺序与文件中记录的物理顺序不同的索引。

索引顺序⽂件: 在搜索码上有聚集索引的⽂件。

索引根据密度又可以分为：

- 稠密索引：文件中的每个搜索码值都有一个索引项

- 稀疏索引：只为搜索码的某些值建立索引项，只有索引是聚集索引时才能使用稀疏索引。

  > 此时要找到搜索码值为`K`的索引，需要找到搜索码值小于`K`的最大索引项，并从该记录开始逐个向后查找。因此，其所占空间较小，插入和删除时所需的维护开销也比较小，但定位时的速度更慢。
  >
  > 为文件中的每个块建立一个索引项的稀疏索引是一个很好的折中。

多级索引：如果主索引太大而不能放在内存中，访问效率将变低。解决方案就是把主索引当做一个连续的文件保留在磁盘上 ，创建一个它之上的稀疏索引。按主索引顺序对⽂件进⾏顺序扫描是⾮常有效地，但是使⽤辅助索引进⾏顺序扫描是很费时的（每读⼀条记录都可能从磁盘读取⼀个新的块）。

#### B+树索引

对索引顺序文件来讲，随着文件的增大索引查找性能和数据顺序扫描性能都会下降，这是由于许多溢出块会被创建，频繁重组整个文件是必须的。使用B+树索引文件，可以在数据插入和删除时通过小的自动调整来保持平衡，无需重组文件。

通过使用B+树索引可以解决索引文件退化问题。

#### 散列文件组织

使用散列文件组织可以避免对文件的多次访问。

## 查询处理和查询优化

### 查询处理

查询处理的基本步骤：

1. 语法分析与翻译：把查询语句翻译成系统的内部表示形式，也就是翻译成关系代数；语法分析器检查语法，验证关系
2. 优化：
3. 执行：查询执行引擎接收一个查询执行计划，执行该计划并把结果返回给查询

![查询处理的基本步骤](https://s2.loli.net/2023/06/16/tjzWxv3YPhQ5Rbg.png)

一个关系代数表达式可以有许多等价的表达式，也可以用多种不同的算法来执行一个关系代数运算，用于执行一个查询的原语操作序列称为**查询执行计划**；查询优化就是在所有等效执行计划中选择具有最小查询执行代价的计划。

查询处理的代价可以通过该查询对各种资源的使用情况进行度量，这些资源包括磁盘存取，执行一个查询所用 CPU 时间，甚至是网络通信代价。在磁盘上存取数据的代价通常是主要代价。通过以下指标来对其进行度量：

- 搜索磁盘次数（平均寻道时间）
- 读取的块数（平均块读取时间）
- 写入的块数（平均块写入时间）

> 写入一个块的代价通常大于块读取的代价，因为数据在写入后会被读取以确保写入正确

只使用传输磁盘块数和搜索磁盘次数来度量查询计算计划的代价，忽略CPU时间，不包括将操作写回磁盘的代价。

> 假设$t_T$为阐述一个块的时间$t_S$，传输$b$个块以及执行$s$次磁盘搜索的操作代价：$b\times {t_T} + s \times {t_S}$

### 关系代数运算的执行

#### 选择运算

- 算法A1（线性搜索）：在线性搜索中，系统扫描每一个文件块，对所有记录都进行测试，看它们是否满足选择条件。
- 算法A2（主索引，码属性等值比较）：对于具有主索引的码属性的等值比较，我们可以使用索引检索到满足相应等值条件的唯一一条记录。
- 算法A3（主索引，非码属性等值比较）
- 算法A4（辅助索引，等值比较）
- 算法A5（主索引，比较）
- 算法A6（主索引，比较）

#### 连接运算

连接运算有如下四种：

1. 嵌套循环连接（Nested Loop Join）：嵌套循环连接是一种简单但效率较低的连接算法。它通过两个嵌套的循环来执行连接操作。对于左表中的每一条记录，都会与右表中的所有记录进行比较，找出满足连接条件的记录。这种算法适用于小规模数据集，但对于大规模数据集来说效率较低。
2. 块嵌套循环连接（Block Nested Loop Join）：它是嵌套循环连接的一个变种，其中内层关系的每一块与外层关系的每一块对应，形成块对，在每一个块对中，一个块的每一个元组与另一个块的每一个元组形成组对，从而得到全体组对。适应于内存不能完全容纳任何一个关系时，如果我们以块的方式而不是以元组的方式处理关系，可以减少块读写次数。
3. 索引循环嵌套连接：
4. 归并连接（Merge Join）：归并连接是一种基于有序性的连接算法。它要求连接的两个表都已按连接属性进行排序。归并连接首先将两个表按连接属性进行合并排序，然后通过同时遍历两个排好序的表，找到满足连接条件的记录。这种算法适用于大规模数据集，但要求预先对表进行排序。
5. 散列连接（Hash Join）：散列连接是一种基于散列函数的连接算法。它将连接属性的值作为输入，通过散列函数将记录分散到不同的散列桶中。然后，对于每个散列桶，将左表和右表中的记录进行比较，找出满足连接条件的记录。散列连接适用于大规模数据集，可以通过并行处理来提高性能。

#### 表达式计算

计算一个完整表达式树有两种方法：

- 物化：输入一个关系或者已完成的计算，产生一个表达式的结果，在磁盘中物化它，重复此过程。
- 流水线：将一个正在执行的操作的部分结果传送到流水线的下一个操作，使得两操作可同时进行。

#### 物化

物化计算:  从最底层开始，执行树中的运算，计算每个中间结果，然后用于下一层运算。

在任何情况下，物化计算都是永远适用的。但是将结果写入磁盘和读取它们的代价是非常大的，此时可以使用双缓冲技术设定两个缓冲区，一个用于连续执行算法，一个用于写出结果，以此允许CPU活动与I/O活动的并行。

#### 流水线

流水线执行：同时执行多个操作，一个操作的结果传递到下一个。比实体化代价小很多，没有必要存储临时关系到磁盘，但并不总是可行的，例如对排序、散列连接。

流水线的执行方法有：

- 需求驱动/消极计算流水线：系统不停地向位于流水线顶端的操作发出需要元组的请求，为了输出自己的下一个元组，每个操作发出请求以获得来自孩子操作的下一个元组，迭代算子维护两次调用之间的执行“状态”，使得下一个`next()`调用请求可以获取下面的结果元组。
- 生产者驱动/积极计算流水线：各操作并不等待元组请求，而是积极地产生元组。

### 查询优化

一个执行计划准确地定义了每个运算应使用的算法，以及运算之间的执行应该如何协调。

基于代价的优化步骤：

1. 使用**等价规则**产生逻辑上的等价表达式
2. 注解结果表达式来得到替代查询计划
3. 基于**代价估计**选择代价最小的计划

如果两个关系代数表达式在所有有效数据库实例中都会产生相同的元组集，则称它们是等价的

> 不关心在违反完整性约束的数据库上是否产生不同的结果

基本的等价规则有：

1. 合取选择运算可以被分解为单个选择运算的序列
2. 选择运算满足交换律
3. 一系列投影中只有最后一个运算时必需的，其余的可以忽略
4. 选择操作可与笛卡尔积以及$\theta$连接相结合
5. $\theta$连接运算满足交换律。
6. 自然连接运算满足结合律。

#### 下推选择

尽可能早地执行选择操作以减小被连接的关系的大小

#### 下推投影



## 事务

### 事务

**事务(transaction)**是访问并可能更新各种数据项的一个程序执行单元(Unit)。事务通常由SQL或者高级程序设计语言通过嵌入式SQL的执行所引起。事务主要解决两个问题：

- 各种故障（硬件故障和系统故障）
- 多事务的并发执行

事务必须保证以下性质：

- **原子性（Atomicity）**：事务的所有操作在数据库中要么全部正确反映，要么全部不反映。原子性由**恢复系统**实现。

- **一致性（Consistency）**：隔离执行事务时（即，在没有其他事务并发执行的情况下）保持数据库的一致性。一致性状态由用户负责，由**并发控制系统**实现。

  > 在事务执行过程中，数据库可能会暂时的不一致；当事务执行完后，数据库必须是一致的。

- **隔离性（Isolation）**：尽管多个事务可能并发执行，但系统保证每个事务都感觉不到系统中有其他事务在并发的执行。中间事务结果对其他并发执行的事务是隐藏的。隔离性通过**并发控制系统**实现。

  > 隔离性可以保证串行地执行事务，但是并发执行事务能够显著提升性能。

- **持久性（Durability）**：一个事务成功完成后，它对数据库的改变是永久的，即使系统可能出现故障。持久性通过**恢复系统**实现。

使用事务状态图描述事务的执行状态，包括如下状态：

- 活动的（active）：初始状态
- 部分提交的（partially committed）：最后一条语句执行后的状态
- 失败（failed）：正常的执行无法继续
- 提交（commited）
- 中止的（aborted）：事务回滚并且数据库恢复到事务开始前的状态，此时系统有两种选择：
  - 重启事务
  - 杀死事务

![事务状态图示例](https://s2.loli.net/2023/06/18/qwhCVJ1YUufRx35.png)

事务通过这两个操作访问数据：

- read(X)：从数据库吧数据项X传送到执行事务的局部缓冲区
- Wtite(X)：将执行事务的局部缓冲区将数据项写会数据库

> 在实际数据库系统中，write操作不一定立即更新磁盘上的数据；write操作的结果可以临时存储在内存中，以后再写到磁盘上，当前假设write操作立即更新数据库，这是“恢复系统”提供的一系列机制

### 并发执行机制

多个事务可以在系统中并发运行，它的核心问题是**保证一致性的前提下最大限度提高并发度**，依靠并发控制机制（保证隔离性的一系列机制）实现。并发执行的优势有：

- 一个事务由**不同的步骤**组成，所涉及的系统资源也不同。这些步骤可以并发执行，以提高系统的**吞吐量(throughput)**。
- 系统中存在着周期不等的各种事务，串行会导致难于预测的延迟。如果各个事务所涉及的是**数据库的不同部分**，采用并行会减少**平均响应时间(average response time)**。

对于并发运行的多个事务，当这些事务操作数据库中相同的数据时，如果没有采取必要的隔离机制，就会导致各种并发问题，这些并发问题主要可以归纳为以下几类：

- 更新丢失：丢失修改是指事务1与事务2从数据库中读入同一数据并修改，事务2的提交结果破坏了事务1提交的结果，导致事务1的修改被丢失。 

  ![更新丢失](https://s2.loli.net/2023/06/18/a29FRUbXjY1tg3v.png)

- 脏读：事务1修改某一数据，并将其写回磁盘，事务2读取同一数据后，事务1由于某种原因被撤消，这时事务1已修改过的数据恢复原值，事务2读到的数据就与数据库中的数据不一致，是不正确的数据，又称为“脏”数据。

  ![脏读](https://s2.loli.net/2023/06/18/vMJwzcIF5Ulym2Z.png)

- 不可重复读：不可重复读是指事务1读取数据后，事务2执行更新操作，使事务1无法再现前一次读取结果。

  ![不可重复读](https://s2.loli.net/2023/06/18/AXmCQOlcBSEof2d.png)

- 幻读：事务1读取某一数据后：

  1. 事务2删除了其中部分记录，当事务1再次读取数据时，发现某些记录神密地消失了。
  2. 事务2插入了一些记录，当事务1再次按相同条件读取数据时，发现多了一些记录。

  > 也可以归结为不可重复读

事务的执行顺序称为一个**调度(schedule)**，表示事务的指令在系统中执行的时间顺序。一组事务的调度需要保证：

- 包含所有事务的操作指令
- 一个事务中指令的顺序必须保持不变

根据调度的执行方式，可以分为串行调度和并行调度。在串行调度中，属于同一事务的指令紧挨在一起，对于有$n$个事务的事务组，可以有$n!$个有效调度；在并行调度中，来自不同事务的指令可以交叉执行，并行调度等价于某个串行调度时，则称它是正确的。

### 冲突可串行化

数据库系统的调度应该保证任何调度执行后数据库总处于一致状态。如果一个并发调度等价于一个串行调度，则该调度是**可串行化**的。不同形式的等价调度：

- 冲突可串行化
- 视图可串行化

冲突指令：当两条指令是不同事务在相同数据项上的操作，并且其中至少有一个是write指令时，则称这两条指令是冲突的。非冲突指令交换次序不会影响调度的最终结果。

> 直观上指令之间的冲突，强迫它们之间具有时序顺序。

如果通过一系列非冲突指令的交换，调度 S 可以转换为调度 S´, 我们说 S 和 S´ 是**冲突等价(conflict equivalent)**的。我们说调度 S 是**冲突可串行化(conflict serializable )**的，如果它与一个串行调度冲突等价。

优先图：一个调度$S$的优先图是这样构造的：它是一个有向图$G =（V，E）$，$V$是顶点集，$E$是边集。顶点集由所有参与调度的事务组成，边集由满足下述条件之一的边${T_i}\rightarrow{T_j}$组成：

- 在${T_j}$执行`read(Q)`之前，${T_i}$执行`write(Q)`
- 在${T_j}$执行`write(Q)`之前，${T_i}$执行`read(Q)`
- 在${T_j}$执行`write(Q)`之前，${T_i}$执行`write(Q)`

如果优先图中存在边${T_i}\rightarrow{T_j}$，则在任何等价于$S$的串行调度$S'$中，$T_i$都必须出现在$T_j$之前。如果调度S的优先图中有环，则调度S是非冲突可串行化的。如果图中无环，则调度S是冲突可串行化的。

![优先图示例](https://s2.loli.net/2023/06/22/UhLyoZe8QmHdMaD.png)

事务的恢复：一个事务失败了，应该能够撤消该事务对数据库的影响。如果有其它事务读取了失败事务写入的数据，则该事务也应该撤消。

**可恢复的调度(Recoverable schedule )** ：对于每对事务T1与T2，如果T2读取了T1所写的数据，则T1必须先于T2提交。

- 级联回滚——单个事务失败，导致一系列事务回滚，会导致大量工作撤销。

- 无级联调度(Cascadeless schedules ) — 不会发生级联回滚; 对每一事务对 $T_i$ 和$T_j$ ，如果$T_j$ 读取了$T_i$ 所写的数据项, 则 Ti  必须在 Tj 的读操作之前提交。

  > 无级联调度必是可恢复调度

### 事务隔离

数据库必须提供一个机制，确保所有可能的调度或者是冲突可串行化，或者是视图可串行化，并且是可恢复的，最好是无级联的。

事务的隔离性实质上是数据库的并发性与一致性的函数。

按照隔离级别从低到高的顺序：

- 未提交读：允许读取未提交数据，这是SQL允许的最低级的隔离性（当事务A更新某条数据时，不容许其他事务来更新该数据，但可以读取。） 

  > 可能出现脏读

- 已提交读：只允许读取已提交数据，但不要求可重复读。连续读记录可能返回不同的值。比如，在同一个事务两次读取同一个数据项之间，其他事务可以更新并提交该数据项

  > 可能出现不可重复读

- 可重复读：只允许读取已提交数据，在两次读取相同数据项之间，不允许其他事务更新这些数据项（可以看到其他事务已经提交的新插入的记录，但是不能看到其他事务对已有记录的更新。）。至于和其他事务之间不一定保证可串行化。

  >  可能出现幻读

- 可串行化

![隔离级别](https://s2.loli.net/2023/06/18/vKB2I3kuLz6SAyr.png)

隔离级别的实现可以通过：

- 封锁：两阶段封锁，一种可以保证可串行化的技术
- 时间戳：为每个事务和每个数据项均分配时间戳，通过时间戳确定事务应该执行还是回滚
- 多版本和快照隔离：多版本：维护数据项的多个版本，允许事务读取数据项的旧版本；快照隔离：每个事务开始时拥有自己的数据库版本或者快照

## 并发控制

### 基于锁的协议

确保可串行化的方法之一是要求对数据项的访问以互斥的方式进行。

封锁就是事务T在对某个数据对象（例如表、记录等）操作之前，先向系统发出请求，对其加锁，加锁后事务T就对该数据对象有了一定的控制，在事务T释放它的锁之前，其它的事务不能更新此数据对象。封锁是控制对数据项的并发访问的机制。

数据库加锁有两种锁模式：

- **排他锁（X，exclusive模式，写锁）**：数据项既可以读又可以写。使用 `lock-X` 指令申请
- **共享锁（S，Shared模式，读锁）**：数据项是只读的。使用` lock-S` 指令申请

锁请求发送给并发控制管理器，只有在并发控制管理器授予所需锁后，事务才能继续其操作。任何数量的事务在一个数据项上可以同时持有共享锁。但是如果任何的事务持有排他锁，其他事务不能持有该数据项上的任何锁。

**封锁协议**是在请求和释放锁时的一系列规则，适用于所有事务。封锁协议限制了可能的调度集。

大多数封锁协议都可能导致死锁，难以避免；如果并发控制管理器设计的不好，还可能造成饥饿/饿死。要防止饥饿，可以对申请S锁的事务，如果有先于该事务且等待的加X锁的事务，令申请S锁的事务等待。

#### 两阶段封锁协议

定义：每个事务分两个阶段，提出加锁和解锁申请阶段。

1. 阶段 1: **增长阶段(growing phase)**。事务可以获取锁，事务不能释放锁。
2. 阶段 2: **收缩阶段(shrinking phase)** 。事务可以释放锁，事务不能获取锁。

**封锁点（lock point）**：事务最后加锁的位置，称为事务的封锁点, 记作$Lp(T)$。

该协议确保可串行化，可以证明事务按照它们的封锁点的顺序可串行化  (即事务获取最后一次封锁的时刻)。

并行执行的所有事务均遵守两段锁协议，则对这些事务的所有并行调度策略**都是可串行化**的。事务遵守两段锁协议是可串行化调度的充分条件，而不是必要条件。

1. **两阶段封锁不能确保避免死锁，可能产生级联回滚**。
2. 为避免级联回滚，采用称为**严格两阶段封锁协议**：事务必须持有所有排他锁，直到事务提交或中止。
3. **强两阶段封锁协议**更加严格：事务提交或终止前，不能释放任何锁。在该协议中，事务按照提交的顺序可串行化。

两阶段封锁协议的证明：

- 定理：两阶段保证调度冲突可串行化
- 证明：反证法，假设调度$\{{T_1}, {T_2}. ..., {T_n}\}$遵从两阶段封锁协议，但不冲突可串行化。
  1. 不冲突可串行化，优先图必有环。不妨假设${T_1}, {T_2}, ..., {T_m}, {T_1}$是一个环。
  2. 如果优先图有边${T_i}\rightarrow{T_j}$，一定有${Lp({T_i})} \lt {{T_i}.unlock(Q)} \lt {T_j}.lock(Q) \leq Lp({T_j})$。即$Lp({T_i}) \lt Lp(T_j)$。
  3. 即对上面的环，有$Lp({T_1}) \lt Lp({T_2}) \lt \cdots \lt Lp({T_m}) \lt Lp({T_1})$存在矛盾，故假设不成立。

锁转换包括：

- Upgrade：共享锁升级到拍他锁
- Downgrade：拍他锁降级到共享锁

> 锁升级只能发生在增长阶段，降级只能发生在缩减阶段。

#### 多粒度封锁

### 基于时间戳的协议

另一种解决事务可串行化的次序的方法是事先选定事务的次序。时间戳排序协议的目标：令调度冲突等价于按照事务开始早晚次序排序的串行调度。

时间戳排序协议的基本思想：

- 开始早的事务不能**读**开始晚的事务写的数据。
- 开始早的事务不能**写**开始晚的事务已经读过或写过的数据。

> 也就是数据的时间戳是不减的，允许时间戳相同的操作。

计数方式可以通过**系统时钟**或**逻辑计数器**。

每个数据项Q需要与两个时间戳值相关联：

- W-timestamp(Q)：表示成功执行write(Q)的所有事务的最大的时间戳
- R-timestamp(Q)：表示成功执行read(Q)的所有事务的最大的时间戳

时间戳排序协议保证任何有冲突的read或write操作按时间戳顺序进行：

1. 事务$T_i$发出read(Q)：

   - 如果TS(Ti)< W-timestamp(Q)，则Ti需读入的Q值已被覆盖。因此，read操作被拒绝，Ti回滚
   - 如果TS(Ti)>= W-timestamp(Q)，则执行read操作，R-timestamp(Q)被设为R-timestamp(Q)和TS(Ti)两者的最大值

   > 也就是如果事务时间戳小于上次写操作，则要读入的值已经被覆盖了，需要回滚重新执行。

2. 事务$T_i$发出write(Q)：

   - 如果TS(Ti)< R-timestamp(Q)，则Ti产生的Q值是先前所需要的值，且系统已假定该值不会被产生。因此，write操作被拒绝，Ti回滚
   - 如果TS(Ti)< W-timestamp(Q)，则Ti试图写入的Q值已过时。因此，write操作被拒绝，Ti回滚
   - 否则，执行write操作，将W-timestamp(Q)设为TS(Ti)

   > 也就是如果事务时间戳小于上次写（即将发生的写已经过时）或读操作，需要回滚重新执行。

时间戳排序协议与两阶段封锁协议的比较：

- 都有本协议下合法、另一协议下不合法的调度。
- 能被这两种协议调度的，一定是冲突可串行化的，优先图中一定没有圈。

解决级联回滚的方案：

- 使事务的结构是在**处理结束时执行所有写操作**(在事务末尾执行所有写操作）。事务的所有写操作构成一个原子动作；当一个事务正在写时其他事务都不能执行（在写操作正在执行时，任何事务都不允许访问已写好的任何数据项）。
- **限制形成封锁**：在**读数据前等待数据提交**（对未提交数据项的读操作，被推迟到更新该数据项的事务提交之后)。
- 使用**提交依赖**来确保可恢复性(事务$T_i$读取了其他事务所写的数据，只有在其他事务提交之后，$T_i$才能提交）。

### 死锁的处理

如果事务集中的每个事务在等待该集合中的另一个事务，则系统处于死锁状态。

预防死锁的方法有：

- 一次封锁法：要求每个事务必须⼀次将所有要使⽤的数据全部加锁，否则就不能继续执⾏。⼀次封锁法存在的问题：扩⼤封锁范围，降低并发度。 
- 顺序封锁法：预先对数据对象规定⼀个封锁顺序（施加偏序关系），所有事务都按这个顺序（偏序关系）封锁数据项(基于图的协议)。顺序封锁法存在的问题：维护成本⾼
- 抢占与事务回滚策略
- 基于超时的策略：事务等待封锁一定的时间，超时之后，事务回滚。死锁不会发生。可能产⽣饥饿，也难以确定合适的超时时间。优点：实现简单；缺点：有可能误判死锁；时限若设置得太⻓，死锁发⽣后不能及时发现 

因此，在操作系统中广为采用的预防死锁的策略并不很适合数据库的特点。

DBMS在解决死锁的问题上更普遍采用的是诊断并解除死锁的方法(允许死锁发生;解除死锁)。

死锁的检测：

- 超时法检测死锁：如果一个事务的等待时间超过了规定的时限，就认为发生了死锁。
- 事务等待图法：用等待图$G =(V,E)$描述事务。

发现死锁时,解除死锁：

- 一些事务必须回滚 (牺牲) 来去除死锁，选择成本最小的那些事务作为牺牲品
  - 事务已经计算了多久，在事务完成前该事务还需要计算多长时间
  - 该事务已经使用了多少数据项；为了完成该事务，还需要使用多少数据项
  - 回滚时将牵扯多少事务
- 回滚 – 回滚到何处
  - 完全回滚：中止事务，重启动
  - 部分回滚：回滚到解除死锁为止，这种方法更有效
- 如果同一事务总是被牺牲，会产生饥饿，把回滚次数作为成本因子，避免饥饿

#### 抢占与事务回滚策略

在抢占机制中，当事务Ti所申请的锁被事务Tj所持有时，授予Tj的锁可能通过回滚事务Tj被抢占，并将锁授予Ti。通过时间戳确定事务等待还是回滚，事务重启时，保持原有的时间戳。

- wait-die模式（非抢占）：当事务Ti申请的数据项当前被事务Tj持有时，仅当Ti的时间戳小于Tj的时间戳时，允许Ti等待，否则Ti回滚。老事务可以等待年轻事务释放数据项，年轻事务不等待老事务，而是直接回滚；在获取所需的数据项之前，事务或许会死若干次。
- wound-wait模式（抢占）：当事务Ti申请的数据项当前被事务Tj持有时，仅当Ti的时间戳大于Tj的时间戳时，允许Ti等待，否则Tj回滚。老事务不等待年轻事务，直接强制其回滚，年轻事务等待老事务；可能比 wait-die 模式回滚次数少。

在wait-die 和wound-wait 策略下,回滚事务用它最初的时间戳重启，从而老事务比新事务有更高的优先级，避免了“饿死”。

二者的共同问题是：发生不必要的回滚

- 在wait-die机制中，较老的事务必须等待较新的事务释放它所持有的数据项。因此，事务越老，越要等待。在wound-die机制中，老事务从不等待
- 在wait-die机制中，如果事务Ti由于申请的数据项当前被Tj持有而死亡并回滚，则当事务Ti重启时，它可能重新发出相同的申请序列。如果该数据项仍被Tj持有，则Ti将再度死亡。所以，wound-die机制中，回滚可能较少

## 恢复系统

### 故障分类

事务故障：

- 逻辑错误：因为某些内部错误条件导致事务不能完成
- 系统错误：因为某种错误条件(如死锁)导致数据库系统终止一个活跃事务

系统崩溃：停电故障或者其他软硬件故障导致系统崩溃。

> 故障-停止假设: 假设非易失性存储器的内容不会因系统崩溃而破坏。数据库系统通过许多完整性检查来防止磁盘数据被破坏。

磁盘故障：磁头损坏或类似的磁盘故障可能破坏全部或部分磁盘存储器。

> 假设损坏是可以检测到的: 磁盘驱动器使用校验和来检测故障

恢复算法：在正常事务处理时采取措施，保证有足够的信息用于故障恢复；故障发生后采取措施，将数据库内容恢复到某个保证数据库一致性、事务原子性及持久性的状态。

### 数据访问

- 物理块：位于磁盘上的块
- 缓冲块：临时位于主存中的块

磁盘和主存之间的块移动通过下列两个操作来引发：

- `input(B)`：将物理块B传入主存。
- `output(B)`：将缓冲块B传到磁盘，并且替换相应的物理块。

每个事务$T_i$都有自己的私有工作区, 用来保存它存取和更新的所有数据项的局部副本，对应数据项X的局部副本记为$X_i$。事务使用下列操作来在系统缓冲块和它的私有工作区之间传送数据项：

- `read(X)`：将数据项X的值赋给局部变量$X_i$。

- `write(X)`：将局部变量$X_i$的值赋给缓冲块中的数据项X。

  > `output({B_X})`不必紧随`write(X)`，系统可以选择在他认为适当的时机执行。

事务第一次访问X之间必须先执行`read(X)`，后续操作对局部副本进行；事务完成之前的任何时间可以执行`write(X)`。

![缓冲区示意图](https://s2.loli.net/2023/06/22/boESGBxUL1iPqFV.png)

### 恢复与原子性

为了在故障的情况下仍确保原子性, 我们首先向稳定存储器输出描述更新的信息（日志—）， 而不更新数据库本身。

日志保存在稳定存储器上，日志是日志记录的序列，用来记录对数据库的更新活动。先写日志，后写数据库。

日志的组成包括：

- 事务标识符
- 数据项标识符
- 旧值
- 新值

当一个事务的commit日志记录输出到稳定存储器后，我们就说这个事务提交了。这时，所有更早的日志记录都已经输出到稳定存储器中。

> 不是在一个事务提交时必须将包含该事务修改的数据项的块输出到稳定存储器中，可以在以后的某个时间再输出。事务提交后虽然没真正写入，但可以认为完成了该事务。

数据库系统通过Undo和Redo操作配合日志实现恢复：

- Undo 对于日志记录<Ti, X,  V1,  V2> 将X写为旧值V1
- Redo 对于日志记录<Ti, X,  V1,  V2> 将X写为新值V2

对于事务$T_i$，他的Undo和Redo是：

- $undo(T_i)$将事务$T_i$更新过的所有数据项的值都恢复成旧值，从最后一条日志记录向前进行。每将数据项X恢复到旧值V，就写一个特别的日志记录<Ti , X, V>。undo操作完成后写下日志记录<Ti abort>。
- $redo(T_i)$将所有Ti 更新过的数据项的值都设置成新值, 从Ti 的第一条日志记录向后进行，不产生日志。

当事务启动时，会写入一条登记的日志记录<Ti  start>，并在每次执行`write(X)`之前写入日志记录，完成最后一条语句后写入<Ti commit>。

基于日志的恢复有两种方法：

- 延迟修改：只有在事务提交时才执行更新（也就是commit时才write），简化了恢复的一些方面，但存在存储部分拷贝的开销。
- 立即修改：允许在事务提交之前，对还未提交的事务进行更新。

系统发生故障时，检查日志，决定哪些事务需要Redo，哪些事务需要Undo，原则上需要搜索整个日志，但有下列问题：搜索过程太耗时；大多数需要Redo的事务已经写入了数据库，此时Redo不会产生不良后果，但是会使得恢复过程太长。

检查点，由系统周期性地执行检查点，需要执行下列操作：

1. 将当前位于主存的所有日志记录输出到稳定存储器
2. 将所有修改了的缓冲块输出到磁盘
3. 将一个日志记录<checkpoint L>输出到稳定存储器，其中L是执行检查点时正活跃的事务的列表

> 检查点执行过程中，不允许事务执行更新操作。在检查点之前提交的事务，不予考虑。

恢复时仅需考虑在检查点之前启动的最近的事务$T_i$，以及在$T_i$之后启动的事务。从日志末尾反向扫描, 找到最近的<checkpoint L>记录。只对在L中的事务，或是在需要重做或撤销的检查点之后开始的事务。

![检查点示例](https://s2.loli.net/2023/06/22/UkcCNDLZy7FTVuQ.png)

系统由后向前扫描⽇志，直⾄发现第⼀个<checkpoint>：

- Redo-List：对每一个形如<Ti commit>的记录，将下加入Redo-list
- Undo-List：对每一个形如<Ti start>的记录，如果Ti不属于Redo-list， 将丁加入undo-list

#### 延迟修改

延迟的数据库修改(deferred-modification technique)，事务中所有的`write`操作，在事务部分提交时才修改数据库的执行，日志中只记录新值。

![延迟修改日志示例](https://s2.loli.net/2023/06/22/Ce3YzJdTiMqkU8m.png)

事务$T_i$需要Redo操作，当且仅当日志中既包含记录$<T_i, start>$又包含记录$<T_i, commit>$。

延迟修改的恢复机制：

- $Redo(T_i)$: 将事务$T_i$更新的所有数据项的值设为新值。
- Redo操作必须是幂等的。

![延迟修改恢复示例](https://s2.loli.net/2023/06/22/iE2h1Urm5CfzTFs.png)

#### 立即修改

立即的数据库修改：允许数据库修改在事务处于活动状态时就输出到数据库中。

- 事务$T_i$需要Redo操作，当且仅当日志中既包含记录$<T_i, start>$**又包含**记录$<T_i, commit>$。
- 事务$T_i$需要Undo操作，当且仅当日志中既包含记录$<T_i, start>$**不包含**记录$<T_i, commit>$。

![立即修改的恢复机制](https://s2.loli.net/2023/06/22/2EeFpN4iHWzUdmC.png)

### 恢复算法

要确定系统如何从故障中恢复，首先需要确定用于存储数据库设备的故障状态，其次考虑这些故障状态对数据库内容有什么影响。然后设计在故障发生后仍保障数据库一致性以及事务原子性的算法，这些算法称为恢复算法。