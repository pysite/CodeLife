> https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/database-storage

数据库存储层

# Disk Manager 简介

传统的 DBMS 架构都属于 **disk-oriented architecture**，即假设数据主要存储在非易失的磁盘（non-volatile disk）上。于是 DBMS 中一般都有磁盘管理模块（disk manager），它主要负责数据在非易失与易失（volatile）的存储器之间的移动。

这里需要理解两点：

- 

  为什么需要将数据在不同的存储器之间移动？

- 

  为什么要自己来做数据移动的管理，而非利用 OS 自带的磁盘管理模块？

## 计算机存储体系

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LY_HB8UaEfE1efciC8V%2F-LY_K0SNgM4yb-lsVBlJ%2FScreen%20Shot%202019-02-13%20at%201.28.29%20PM.jpg?alt=media&token=8cd28260-ebb5-4729-8a41-732675a64afc)

存储体系示意图

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LY_HB8UaEfE1efciC8V%2F-LY_Kgs6xp4XVNA9n-FF%2FScreen%20Shot%202019-02-13%20at%201.31.21%20PM.jpg?alt=media&token=f4dade9f-4870-4c87-83bb-bd419e087ce1)

不同存储器的数据获取时间对照表

磁盘管理模块的存在就是为了**同时获得易失性存储器的性能和非易失性存储器的容量，让 DBMS 的数据看起来像在内存中一样。**

## **为什么不使用 OS 自带的磁盘管理模块**

OS 为开发者提供了如 mmap 这样的系统调用，使开发者能够依赖 OS 自动管理数据在内外存之间的移动，那么 DBMS 为什么要重复造这样的轮子？主要原因在于，OS 的磁盘管理模块并没有、也不可能会有 DBMS 中的领域知识，因此 DBMS 比 OS 拥有更多、更充分的知识来决定数据移动的时机和数量，具体包括：

- 

  将 dirty pages 按正确地顺序写到磁盘

- 

  根据具体情况预获取数据

- 

  定制化缓存置换（buffer replacement）策略

## 磁盘管理模块的核心问题

DBMS 的磁盘管理模块主要解决两个问题：

1. 1.

   （本节）如何使用磁盘文件来表示数据库的数据（元数据、索引、数据表等）

2. 2.

   如何管理数据在内存与磁盘之间的移动

## 本节大纲：

- 

  File Storage

- 

  Page Layout

- 

  Tuple Layout

- 

  Data Representation

- 

  System Catalogs

- 

  Storage Models

# 在文件中表示数据库

## File Storage

DBMS 通常将自己的所有数据作为一个或多个文件存储在磁盘中，而 OS 只当它们是普通文件，并不知道如何解读这些文件。虽然 DBMS 自己造了磁盘管理模块，但 DBMS 一般不会自己造文件系统，主要原因如下：

- 

  通过 DIY 文件系统获得的性能提升在 10% - 15% 之间

- 

  使用 DIY 文件系统将使得 DBMS 的可移植性大大下降

综合考虑，这不值得。

### Database Pages

OS 的文件系统通常将文件切分成 pages 进行管理，DBMS 也不例外。通常 page 是固定大小的一块数据，每个 page 内部可能存储着 tuples、meta-data、indexes 以及 logs 等等，大多数 DBMS 不会把不同类型数据存储在同一个 page 上。每个 page 带着一个唯一的 id，DBMS 使用一个 indirection layer 将 page id 与数据实际存储的物理位置关联起来。

注意：有几个不同的 page 概念需要分清楚

- 

  Hardware Page：通常大小为 4KB

- 

  OS Page: 通常大小为 4KB

- 

  Database Page：(1-16KB)

不同 DBMS 管理 pages 的方式不同，主要分为以下几种：

- 

  Heap File Organization

- 

  Sequential/Sorted File Organization

- 

  Hashing File Organization

***Heap File Organization***

heap file 指的是一个无序的 pages 集合，pages 管理模块需要记录哪些 pages 已经被使用，而哪些 pages 尚未被使用。那么具体如何来记录和管理呢？主要有以下两种方法 Linked List 和 Page Directory。

*Linked List*

pages 管理模块维护一个 header page，后者维护两个 page 列表：

- 

  free page list

- 

  data page list

如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYeb0gfcjnlFpoRuRX7%2F-LYebEz88ooTeVuMFLIh%2FScreen%20Shot%202019-02-14%20at%201.23.35%20PM.jpg?alt=media&token=cb33ff58-154b-4259-bd66-08bb8856e17b)

page file organization: linked list

*Page Directory*

pages 管理模块维护着一些特殊的 pages（directory pages），它们负责记录 data pages 的使用情况，DBMS 需要保证 directory pages 与 data pages 同步。

## Page Layout

每个 page 被分为两个部分：header 和 data，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYebNTlaoNsGvaj71_I%2F-LYebQTLUy5MwoUGLw-9%2FScreen%20Shot%202019-02-14%20at%201.29.06%20PM.jpg?alt=media&token=084e9d6e-5ff9-4968-a6f4-189e5a23c986)

page layout

header 中通常包含以下信息：

- 

  Page Size

- 

  Checksum

- 

  DBMS Version

- 

  Transaction Visibility

- 

  Compression Information

## Data Layout

data 中记录着真正存储的数据，数据记录的形式主要有两种：

- 

  Tuple-oriented：记录数据本身

- 

  Log-structured：记录数据的操作日志

### Tuple Oriented

*Strawman Idea:* 

在 header 中记录 tuple 的个数，然后不断的往下 append 即可，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYebNTlaoNsGvaj71_I%2F-LYebUJNjx5lPE-GdhLA%2FScreen%20Shot%202019-02-14%20at%201.42.13%20PM.jpg?alt=media&token=2110560c-88a8-4ca7-826b-d913128e246b)

data layout: tuple-oriented

这种方法有明显的两个缺点：

- 

  一旦出现删除操作，每次插入就需要遍历一遍，寻找空位，否则就会出现空间浪费

- 

  无法处理变长的数据记录（tuple）

为了解决这两个问题，就产生了 slotted pages。

*Slotted Pages*

如下图所示，header 中的 slot array 记录每个 slot 的信息，如大小、位移等

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYebNTlaoNsGvaj71_I%2F-LYebZrfUAimy4IrF1DG%2FScreen%20Shot%202019-02-14%20at%201.45.58%20PM.jpg?alt=media&token=76c626f4-fd5f-40bf-b7c2-2ee0acb03b1b)

data layout: slotted pages

- 

  新增记录时：在 slot array 中新增一条记录，记录着改记录的入口地址。slot array 与 data 从 page 的两端向中间生长，二者相遇时，就认为这个 page 已经满了

- 

  删除记录时：假设删除 tuple #3，可以将 slot array 中的第三条记录删除，并将 tuple #4 及其以后的数据都都向下移动，填补 tuple #3 的空位。而这些细节对于 page 的使用者来说是透明的

- 

  处理定长和变长 tuple 数据都游刃有余

目前大部分 DBMS 都采用这种结构的 pages。

### Log Structured

log-structured 只存储日志记录，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYebNTlaoNsGvaj71_I%2F-LYebc_D1CpzNH1QhiNS%2FScreen%20Shot%202019-02-14%20at%201.55.58%20PM.jpg?alt=media&token=f241bc9c-5b9a-4398-af62-328424b8b32e)

log-structured: append

每次记录新的操作日志即可，增删改的操作都很快，但有得必有失，在查询场景下，就需要遍历 page 信息来生成数据才能返回查询结果。为了加快查询效率，通常会对操作日志在记录 id 上建立索引，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYebNTlaoNsGvaj71_I%2F-LYebgBrjCRjYP4bXt1a%2FScreen%20Shot%202019-02-14%20at%201.59.39%20PM.jpg?alt=media&token=8c4081b9-b71a-413c-84dd-cf143bc19d2a)

log-structured: build indexes

当然，定期压缩日志也是不可或缺的

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYebNTlaoNsGvaj71_I%2F-LYebjzhcR8dxiQBLFof%2FScreen%20Shot%202019-02-14%20at%202.01.41%20PM.jpg?alt=media&token=3b288148-58c5-431f-99bf-62c029dba3dd)

log-structured: compress logs

## Tuple Layout

上节讨论了 page 的 layout 可以分成 header 与 data 两部分，而 data 部分又分为 tuple-oriented 和 log structured 两种，那么在 tuple-oriented 的 layout 中，DMBS 如何存储 tuple 本身呢？

由于 tuple 本身还有别的元信息，如：

- 

  Visibility Info (concurrency control)：即 tuple 粒度的权限和并发控制

- 

  Bit Map for **NULL** values

因此不难猜到，tuple 中还可以分为 header 和 data 两部分，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYg_v6jCZnflR2OgqQ1%2F-LYgb3nbBH2gBw-5o265%2FScreen%20Shot%202019-02-14%20at%2011.24.43%20PM.jpg?alt=media&token=d3cf2e74-bafb-40fe-b6d7-0369bf8e38e2)

tuple-layout

通常 DBMS 会按照你在建表时候指定的顺序（并不绝对）来存储 tuple 的 attribute data，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYg_v6jCZnflR2OgqQ1%2F-LYgbSRuJhUrF1Aq9A8j%2FScreen%20Shot%202019-02-14%20at%2011.26.11%20PM.jpg?alt=media&token=61e29767-d358-4571-98fb-827988f3c350)

order of tuple attribute data

有时候，为了提高操作性能，DBMS 会在存储层面上将有关联的表的数据预先 join 起来，称作 denormalize，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYg_v6jCZnflR2OgqQ1%2F-LYgc28CII_KL-Kf0GAd%2FScreen%20Shot%202019-02-14%20at%2011.28.48%20PM.jpg?alt=media&token=e3ad57dd-1233-4871-a561-73c0791c30ac)

related tables

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYg_v6jCZnflR2OgqQ1%2F-LYgc6IfM4tZC2Zln-uU%2FScreen%20Shot%202019-02-14%20at%2011.28.54%20PM.jpg?alt=media&token=f5c795e7-27d2-46a6-9056-3883ef9699cd)

table foo and table bar

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYg_v6jCZnflR2OgqQ1%2F-LYgcGKlBaw_2AKf4XWi%2FScreen%20Shot%202019-02-14%20at%2011.29.00%20PM.jpg?alt=media&token=9737baf6-bac1-4ac6-8b69-b725b8f2e1f4)

pre-join of table foo and bar

如果表 bar 与表 foo 经常需要被 join 起来，那么二者可以在存储阶段就预先 join 到一起，这么做当然有利有弊：

- 

  利：减少 I/O

- 

  弊：更新操作复杂化

在 DBMS 层面上，由于它需要跟踪每一个 tuple，因此通常会给每个 tuple 赋予一个唯一的标识符（record identifier），通常这个标识符由 page_id + offset/slot 组成，有时候也会包含文件信息。这属于 DBMS 实现的细节，虽然它是唯一的，但 DBMS 可能随时修改它，因此 DBMS 上层的应用不应该依赖于它去实现自己的功能。

## Tuple Storage

在文件中，一个 tuple 无非就是一串字节，而如何解读这些字节中隐含的数据类型和数据本身，就是 DBMS 的工作。DBMS 的 catelogs 保存着数据表的 schema 信息，有了这些信息，DBMS 就能够解读 tuple 数据。

### Data Representation

数据库支持的数据类型主要包含：

类型

实现

INTEGER/BIGINT/SMALLINT/TINYINT

C/C++ Representation

FLOAT/REAL vs. NUMERIC/DECIMAL

IEEE-754 Standard / Fixed-point Decimals

VARCHAR/VARBINARY/TEXT/BLOB

Header with length, followed by data bytes

TIME/DATE/TIMESTAMP

32/64-bit integer of (micro) seconds since Unix epoch

*FLOAT/REAL/DOUBLE vs. NUMERIC/DECIMAL*

float, real, double 类型的数字按照 IEEE-754 标准存储，它们都是 fixed-precision，在 range 和 precision 上作了取舍，无法保证精确度要求很高的计算的正确性，如：

\#include <stdio.h>



int main(int argc, char* argv[]) {

​    float x = 0.1;

​    float y = 0.2;

​    printf("x+y = %.20f\n", x+y)

​    printf("0.3 = %.20f\n", 0.3)

}



// =>

// x+y = 0.30000001192092895508

// 0.3 = 0.29999999999999998890

如果希望允许数据精确到任意精度（arbitrary precision），则可以使用 numeric/decimal 类型类存储，它们就像 VARCHAR 一般，长度不定，以 Postgres 的 NUMERIC 为例，它的实际数据结构如下所示：

typedef unsigned char NumericDigit;

typedef struct {

​    int ndigits;           // # of Digits

​    int weight;            // Weight of 1st Digit

​    int scale;             // Scale Factor

​    int sign;              // Positive/Negative/NaN

​    NumericDigit * digits; // Digit Storage

} numeric;

*Large Values*

大部分 DBMSs 不允许一个 tuple 中的数据超过一个 page 大小，如果想要存储这样的 large values，DBMS 通常会使用 **overflow/TOAST page：**

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LYz1kPQAXFP6nsJROMN%2F-LYz7jTGzXcXun510WMo%2FScreen%20Shot%202019-02-18%20at%201.44.53%20PM.jpg?alt=media&token=a00af1fa-2fc8-46d1-90f8-a6143b45d302)

overflow storage page

*External Value Storage*

一些 DBMSs 甚至允许你在外部存储二进制大文件，即 BLOB，但 DBMSs 一般不会负责管理外部文件，没有 durability protections 和 transaction protections，而且数据库的 dump 操作不会对外部文件起作用，转移数据时需要单独处理。

### System Catalogs

除了数据本身，DBMS 还需要存储数据的元数据，即数据字典，它们包括：

- 

  Table, columns, indexes, views

- 

  Users, permissions

- 

  Internal statistics

几乎所有 DBMSs 都将这些元数据也存储在一个特定的数据库中，它们本身也会被存储为 table、tuple。根据 SQL-92 标准，你可以通过 INFORMATION_SCHEMA 数据库来查询这些数据库的元信息，但一般 DBMSs 都会提供更便捷的命令来查询这些信息，示例如下：

SQL-92

Postgres

MySQL

SQLite

SELECT *

  FROM INFORMATION_SCHEMA.TABLES

 WHERE table_catalog = '<db name>';

 

SELECT *

  FROM INFORMATION_SCHEMA.TABLES

 WHERE table_name = 'student';

### OLTP & OLAP

数据库的应用场景大体可以用两个维度来描述：操作复杂度和读写分布，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LZ4Wq8LNTEzFCmrje83%2F-LZ4X-RjcFWNScWa57u0%2FScreen%20Shot%202019-02-19%20at%2012.50.59%20PM.jpg?alt=media&token=cdee7e3d-45e3-40cf-9959-7e582072d617)

坐标轴左下角是 OLTP（On-line Transaction Processing），OLTP 场景包含简单的读写语句，且每个语句都只操作数据库中的一小部分数据，举例如下：

OLTP-queries.sql

SELECT P.*, R.*

  FROM pages AS P

 INNER JOIN revisions AS R

​    ON P.latest = R.revID

 WHERE P.pageID = ?;

 

UPDATE useracct

   SET lastLogin = NOW(),

​       hostname = ?

 WHERE userID = ?

 

 INSERT INTO revisions

 VALUES (?,?...,?)

在坐标轴右上角是 OLAP（On-line Analytical Processing），OLAP 主要处理复杂的，需要检索大量数据并聚合的操作，举例如下：

OLAP-queries.sql

SELECT COUNT(U.lastLogin)

​       EXTRACT (month FROM U.lastLogin) AS month

  FROM useracct AS U

 WHERE U.hostname LIKE '%.gov'

 GROUP BY EXTRACT(month FROM U.lastLogin);

*通常 OLAP 操作的就是 OLTP 应用搜集的数据*

### **Data Storage Models**

Relational Data Model 将数据的 attributes 组合成 tuple，将结构相似的 tuple 组合成 relation，但它并没有指定这些 relation 中的 tuple，以及 tuple 的 attributes 的存储方式。一个 tuple 的所有 attributes 并不需要都存储在同一个 page 中，它们的实际存储方式可以根据数据库应用场景优化，如 OLTP 和 OLAP。

目前常见的 Data Storage Models 包括：

- 

  行存储：N-ary Storage Model (NSM)

- 

  列存储：Decomposition Storage Model (DSM)

*NSM*

NSM 将一个 tuple 的所有 attributes 在 page 中连续地存储，这种存储方式非常适合 OLTP 场景，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LZ4Wq8LNTEzFCmrje83%2F-LZ4X3qmsw30ZvGZeX1R%2FScreen%20Shot%202019-02-19%20at%207.04.56%20PM.jpg?alt=media&token=09808184-37cb-4faf-bdb4-70eeb97df9bc)

DBMS 针对一些常用 attributes 建立 Index，如例子中的 userID，一个查询语句通过 Index 找到相应的 tuples，返回查询结果，流程如下：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LZ4Wq8LNTEzFCmrje83%2F-LZ4X6-FRYb-oWOL_7Br%2FScreen%20Shot%202019-02-19%20at%207.07.09%20PM.jpg?alt=media&token=4c8a8ac9-e4e0-4019-89bc-52067c3fb9bd)

但对于一个典型的 OLAP 查询，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LZ4Wq8LNTEzFCmrje83%2F-LZ4X8I7f3f-3Id2W9Dc%2FScreen%20Shot%202019-02-19%20at%207.09.38%20PM.jpg?alt=media&token=1bdd9fef-e389-4bad-9966-91b29f5653e1)

尽管整个查询只涉及到 tuple 的 hostname 与 lastLogin 两个 attributes，但查询过程中仍然需要读取 tuple 的所有 attributes。

总结一下，NSM 的优缺点如下：

- 

  Advantages

  - 

    高效插入、更新、删除，涉及表中小部分 tuples

  - 

    有利于需要整个 tuple （所有 attributes）的查询

- 

  Disadvantages

  - 

    不利于需要检索表内大部分 tuples，或者只需要一小部分 attributes 的查询

*DSM*

DSM 将所有 tuples 的单个 attribute 连续地存储在一个 page 中，这种存储方式特别适用于 OLAP 场景，如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LZ4Wq8LNTEzFCmrje83%2F-LZ4XAhoVE_WLpjY6lBh%2FScreen%20Shot%202019-02-19%20at%207.17.24%20PM.jpg?alt=media&token=15f47600-d449-4c56-9053-b0e6b3e347c6)

这时候，就可以优雅地处理 OLAP 查询浪费 I/O 的问题：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LZ4Wq8LNTEzFCmrje83%2F-LZ4XDfuuJnIRk5wzeoG%2FScreen%20Shot%202019-02-19%20at%207.20.23%20PM.jpg?alt=media&token=16ef9d5a-4de7-4ba9-95ea-1e4b08f80a45)

由于 DSM 把 attributes 分开存储，也引入了新的问题，比如：

如何跟踪每个 tuple 的不同 attributes？可能的解决方案有：

1. 1.

   Fixed-length Offsets：每个 attribute 都是定长的，直接靠 offset 来跟踪（常用）

2. 2.

   Embedded Tuple Ids：在每个 attribute 前面都加上 tupleID

如下图所示：

![img](https://2836672763-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LMjQD5UezC9P8miypMG%2F-LZ4Wq8LNTEzFCmrje83%2F-LZ4XFUANT4RYk-sof-s%2FScreen%20Shot%202019-02-19%20at%207.25.51%20PM.jpg?alt=media&token=1aa0ba25-6167-4b37-9085-4c30a6607847)

总结一下，DSM 的优缺点如下：

- 

  Advantages

  - 

    减少 I/O 操作

  - 

    更好的查询处理和数据压缩支持

- 

  Disadvantages

  - 

    涉及少量 tuples、多数 attributes 的查询低效