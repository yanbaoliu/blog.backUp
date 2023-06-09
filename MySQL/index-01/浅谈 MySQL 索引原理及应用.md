

## 前言

> 本篇文章主要胡侃一下 MySQL 中的索引，重点剖析了 InnoDB 存储引擎的 B+ 树
>
> 首先，在索引概述中，介绍了索引是什么？有哪些特点？是怎么分类的？然后，通过一个示例，剖析了聚集索引和非聚集索引的结构、特点。最后，总结了 MySQL 中一些索引失效的场景、设计索引的原则等实践应用



## **一、索引概述**

- 数据库系统除了存储数据，还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据结构上实现高级查找算法，这种数据结构就是**索引**（Index）
- 索引是存储引擎用于快速查找记录的一种数据结构，即**索引是数据结构**
- 在 MySQL 中，索引是在存储引擎层而不是服务器层实现的。所以，不同存储引擎的索引的工作方式不一样，也不是所有的存储引擎都支持所有类型的索引。MyISAM 和 InnoDB 存储引擎的表默认创建的都是 B+ 树索引



#### 1、索引的特点

- 可以减少需要扫描的数据数量，快速地定位到表的指定位置，提高查询的速度
- 创建和维护索引需要耗费时间；索引需要占用磁盘空间；当对数据库表中的数据进行更新（增加、修改、删除）时，索引也需要动态地维护，这样降低了数据的维护速度



#### 2、**索引的分类**

- 按照功能逻辑划分：普通索引、唯一索引、主键索引、全文索引
- 按照物理实现划分：聚集索引（聚簇索引）和非聚集索引（非聚簇索引、辅助索引、二级索引）
- 按照索引列个数划分：单列索引和联合索引（复合索引、多列索引）
- 按照数据结构划分：B+ 树索引、哈希索引、全文索引、空间数据索引



#### **3、索引 SQL 语句**

```sql
-- 查看索引
SHOW INDEX FROM 表名;

-- 创建索引
ALTER TABLE 表名 ADD INDEX 索引名(字段列表); 
或 CREATE INDEX 索引名 ON 表名(字段列表);

-- 删除索引
DROP INDEX 索引名 ON 表名;
```





## 二、**InnoDB 中** **B+ 树索引**

#### 1、数据准备

###### （1）创建数据库表

```
CREATE TABLE table_name(
col1 INT,
col2 INT,
col3 CHAR(1),
PRIMARY KEY(col1),
INDEX idx_key(col2)
) ROW_FORMAT = COMPACT;
```

###### （2）往表中插入数据记录

| col1 | col2 | col3 |
| :--: | :--: | :--: |
|  1   | 200  | 'D'  |
|  2   | 200  | 'K'  |
|  3   | 100  | 'N'  |
|  6   | 300  | 'O'  |
|  8   | 500  | 'T'  |
|  10  | 400  | 'E'  |
|  12  | 600  | 'W'  |
|  13  | 800  | 'Z'  |
|  14  | 900  | 'A'  |
|  15  | 700  | 'S'  |
|  16  | 550  | 'S'  |



#### 2、表的行格式示意图

<img src="D:\ResourcesData\blog.backUp\MySQL\index-01\img\mysql-record.png" style="zoom:40%;" />

- record_type：表示当前记录的类型。0 是普通记录，1 是 B+ 树非叶子节点的目录项记录，2 是 Infimun 记录，3 是 Supremum 记录
- next_record：表示下一条记录的相对位置。即表示从当前记录的真实数据到下一条记录的真实数据的距离。如果该属性值为正数，说明当前记录的下一条记录在当前记录的后面；如果该属性值为负数，说明当前记录的下一条记录在当前记录的前面
- 各个列的值：数据库用户实际存储的数据列
- 其他信息：隐藏列以及记录的额外信息



#### 3、**以主键为键值建立的 B+ 树索引——**聚集索引

<img src="D:\ResourcesData\blog.backUp\MySQL\index-01\img\mysql_B+tree1.png" style="zoom:150%;" />

###### **（1）B+ 树**聚集索引的结构说明

- 下一个数据页中用户记录的主键值必须大于上一个数据页中用户记录的主键值
- 给所有的页建立一个目录项。每个页对应一个目录项，每个目录项包括两个部分：页的用户记录中最小的主键值，用 key 表示；页号，用 page_no 表示
- 叶子节点是存储真正的用户记录的（完整的用户记录）；非叶子节点是存储目录项记录的

###### **（2）B+ 树聚集索引的特点**

- 页（包括叶子节点和非叶子节点）内的记录按照主键的大小顺序排成一个单向链表，页内的记录被划分成若干个组，每个组中主键值最大的记录在页内的偏移量会被当作槽依次存放在页目录中
- 各个存放用户记录的页根据页中用户记录的主键大小顺序排成一个双向链表
- 存放目录项记录的页分为不同的层级，在同一层级中的页也是根据页中目录项记录的主键大小顺序排成一个双向链表

###### **（3）根据主键值查找一条用户记录的步骤**

1. 确定存储目录项记录的页。根据根页面定位到用户记录所在的目录项记录所在的页
2. 通过存储目录项记录的页确定用户记录真正所在的页
   - 1）通过二分法确定目录项记录所在分组对应的槽，并找到该槽所在分组中主键值最小的那条目录项记录
   - 2）通过目录项记录的 next_record 属性遍历该槽所在的组中的各个目录项记录，从而确定用户记录真正所在的页
3. 在真正存储用户记录的页中定位到具体的记录
   - 1）通过二分法确定用户记录所在分组对应的槽，并找到该槽所在分组中主键值最小的那条用户记录
   - 2）通过用户记录的 next_record 属性遍历该槽所在的组中的各个用户记录，从而定位到具体的用户记录



#### 4、**以非主键列为键值建立的 B+ 树索引——**非聚集索引

<img src="D:\ResourcesData\blog.backUp\MySQL\index-01\img\mysql_B+tree2.png" style="zoom:150%;" />

###### （1）B+ 树非聚集索引的结构说明

- 下一个数据页中用户记录的索引列值必须大于或等于上一个页中用户记录的索引列值
- 给所有的页建立一个目录项。每个页对应一个目录项，每个目录项包括三个部分：页的用户记录中最小的索引列值，用 key 表示；主键值；页号，用 page_no 表示（目录项包括“主键值”部分是为了保证 B+ 树每一层节点中各条目录项记录除“页号”部分外是唯一的，即保证目录项记录的唯一性）
- 叶子节点是存储真正的用户记录的（不完整，只存储索引列值和主键值）；非叶子节点是存储目录项记录的

###### **（2）B+ 树非聚集索引的特点**

- 页（包括叶子节点和非叶子节点）内的记录按照索引列（图中 col2 列）的大小顺序排成一个单向链表，页内的记录被划分成若干个组，每个组中索引列值最大的记录在页内的偏移量会被当作槽依次存放在页目录中
- 各个存放用户记录的页根据页中用户记录的索引列大小顺序排成一个双向链表
- 存放目录项记录的页分为不同的层级，在同一层级中的页也是根据页中目录项记录的索引列大小顺序排成一个双向链表
- 对于非聚集索引记录，是先按照非聚集索引列的值进行排序，在非聚集索引列值相同的情况下，再按照主键值进行排序。例如，图中为 col2 列建立非聚集索引，相当于为 (col2, col1) 列建立了一个联合索引

###### **（3）根据索引列值（非主键值）查找一条用户记录的步骤**

1. 确定存储目录项记录的页。根据根页面定位到用户记录所在的目录项记录所在的页
2. 通过存储目录项记录的页确定用户记录真正所在的页
   - 1）通过二分法确定目录项记录所在分组对应的槽，并找到该槽所在分组中索引列值最小的那条目录项记录
   - 2）通过目录项记录的 next_record 属性遍历该槽所在的组中的各个目录项记录，从而确定用户记录真正所在的页
3. 在真正存储用户记录的页中定位到具体的记录
   - 1）通过二分法确定用户记录所在分组对应的槽，并找到该槽所在分组中索引列值最小的那条用户记录
   - 2）通过用户记录的 next_record 属性遍历该槽所在的组中的各个用户记录，从而定位到具体的用户记录
4. 根据用户记录中的主键信息到聚集索引中查找完整的用户记录



#### 5、**总结：聚集索引和**非聚集索引

- InnoDB 的索引可以分为聚集索引（Clustered Inex）和非聚集索引（Secondary Index）两种，其内部都是 B+ 树
- 聚集索引与非聚集索引不同的是，叶子节点存放的是否是完整的用户记录。即*聚集索引记录要存储用户定义的所有列以及隐藏列，而非聚集索引只需要存放索引列和主键*

###### **（1）聚集索引**（Clustered Inex，聚簇索引）

- InnoDB 表是索引组织表，即表中数据按照主键顺序存放。聚集索引是按照每张表的主键构造一棵 B+ 树，其叶子节点中存储的是完整的用户记录，非叶子节点中存储的是目录项记录（主键值和页的编号）
- 在 InnoDB 中，聚集索引就是数据的存储方式（所有的用户记录都存储在叶子节点）。索引即数据，数据即索引
- InnoDB 会自动创建聚集索引，不需要用户显式创建
- 由于实际的数据页只能按照一棵 B+ 树进行排序，因此每张表只能拥有一个聚集索引
- 聚集索引是以主键（若没有显式定义主键，InnoDB 会先判断表中是否有非空的唯一索引，否则自动添加隐式列作为主键）的大小为排序规则而建立的 B+ 树
- 索引聚集索引是以主键值的大小作为页和记录的排序规则，在叶子节点存储的记录包含了表中所有的列
- 主键索引属于聚集索引

###### （2）非聚集索引（Secondary Index，聚簇索引，辅助索引，二级索引）

- 非聚集索引即索引结构和数据分开存放的索引。非聚集索引是以非主键索引列的大小为排序规则而建立的 B+ 树索引
- 非聚集索引的叶子节点并不包含行记录的全部数据，叶子节点只包含索引列和主键
- 在 InnoDB 中，非聚集索引叶子节点保存的不是指向行的物理位置的指针，而是聚集索引键值（行的主键值）——理由是：减少当出现行移动或者数据页分裂时非聚集索引的维护工作。使用主键值作为指针会让非聚集索引占用更多的空间，但在移动行时无须更新非聚集索引中的主键值
- 非聚集索引的存在并不影响数据在聚集索引中的组织，因此每张表上可以有多个非聚集索引
- 当通过非聚集索引来查找数据时，InnoDB 首先通过非聚集索引获得指向聚集索引的主键值，然后再通过聚集索引来找到一个完整的行记录
- 非聚集索引是以非主键索引列的大小作为页和记录的排序规则，在叶子节点存储的记录内容是索引列和主键
- 唯一索引、普通索引、前缀索引、联合索引等索引属于非聚集索引



#### *6、*覆盖索引（Covering index）

- 覆盖索引：如果一个索引包含（或覆盖）所有需要查询的字段的值
- 由于非聚集索引只存储主键的值，如果使用非聚集索引查找数据就必须先从非聚集索引查询到主键的值，再使用主键的值去聚集索引上查询，直到找到叶子节点上完整的用户数据返回，这个过程称为“**回表**”
- InnoDB 支持覆盖索引，即从非聚集索引中就可以得到查询的记录，而不需要查询聚集索引中的记录，即不需要回表查询。如果非聚集索引包含（覆盖）所有需要查询的字段的值，就称为＂覆盖索引”
- 覆盖索引的优点：非聚集索引不包含整行记录的所有信息，故其大小要远小于聚集索引，因此可以减少的 IO 操作。覆盖索引可以避免回表操作带来的性能损耗



#### 7、**使用 B+ 树索引的原因**

- 当索引本身很大时，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。于是索引查找过程中就要产生磁盘 I/O 消耗
- B+ 树具有更少的检索次数。B+ 树的特点是每层节点数目非常多、层数很少，而且除了叶子节点外其他节点不存储完整的用户记录数据（InnoDB 中页的默认大小是 16 KB，如果不存储完整的用户记录数据，就可以存储更多的目录项记录，减少树的高度）。即磁盘 IO 次数少，磁盘 I/O 带来的性能损耗就越小，查询的效率就越高
- B+ 树的所有叶子节点之间通过双向链表连接，利于范围查询



## 三、B+ 树索引的应用

#### 1、使用 B+ 树索引的好处

- 可以减少需要扫描的数据记录数量，快速地定位到表的指定位置
- 可以用于排序和分组

- 可以帮助服务器避免排序和临时表
- 可以将随机 I/O 变为顺序 I/O
- 可以通过唯一索引，保证数据库表中记录的唯一性



#### 2、使用 B+ 树索引的代价

###### （1）空间上

- 每建立一个索引，都要为它建立一棵 B+ 树，每一棵 B+ 树的每一个节点都是一个数据页
- 一个数据页默认会占用 16KB 的存储空间

###### （2）时间上

- 每当对表中的数据进行增删改操作时，需要修改各个 B+ 树索引
- B+ 树中的每层节点都按照索引列的值从小到大的顺序排序组成了双向链表。无论是叶子节点中的记录还是非叶子节点中的记录（即无论是用户记录还是目录项记录），都按照索引列的值从小到大的顺序形成了一个单向链表
- 增删改操作可能会对节点和记录的排序造成破坏，所以存储引擎需要额外的时间进行页面分裂、页面回收等操作，以维护节点和记录的排序

###### （3）执行计划的耗时

- 在执行查询语句前，会生成一个执行计划。在生成执行计划时需要计算使用不同索引执行查询时所需的成本，最后选取成本最低的那个索引执行查询
- 如果建立了过多的索引，可能会导致成本分析过程耗时太多，从而影响查询语句的执行性能



#### 3、使用 B+ 树索引的经典场景

- **全值匹配**：对索引中所有列都指定具体值，即是对索引中的所有列都有等值匹配的条件
- **匹配最左前缀**：对于联合索引（多列索引），使用索引必须按照索引列的顺序，依次从最左边的索引列进行匹配，一旦跳过某个索引列，那么后面的索引列都无法使用。即从索引的最左边列开始且不跳过索引中的列
- **匹配列前缀**：和索引中的某一列的值的开头部分
- **匹配范围值**：和索引中列在某个范围内匹配
- **精确匹配某一列并范围匹配另外一列**：例如第一列全匹配，第二列范围匹配
- **只访问索引的查询**：查询只需要访问索引，而无须访问数据行



#### 4、**B+ 树索引失效**

- 使用全表扫描比索引查询还快，则可能不使用索引（查询优化器的执行计划会选择）
- 使用联合索引时，违背最左前缀原则，则不会使用索引
- 在索引列上进行了计算、函数、类型转换等操作，则不会使用索引
- 使用联合索引时，索引列中范围条件右边的列都不会使用索引
- LIKE 以通配符 % 开头，则不会使用索引。如果满足使用覆盖索引的条件，会使用覆盖索引，避免全表扫描
- 字符串不加单引号，则不会使用索引——会进行自动类型转换
- 条件中有 OR，如果 OR 连接的列存在任意一个没有使用到索引或者不满足索引合并要求，则不会使用索引，会进行全表扫描——若要利用索引，要求 OR 连接的每个条件列都使用到索引，且符合索引合并的要求
- 不等于（!= 或者 <>），不会使用索引。如果满足使用覆盖索引的条件，会使用覆盖索引，避免全表扫描
- IS NOT NULL 不会使用索引（IS NULL 可以使用索引）



#### 5、**B+ 树索引的**设计原则

- 只为用于搜索、排序或分组的列创建索引。只为出现在 WHERE 子句中的列、连接子句中的连接列、ORDER BY 或 GROUP BY 子句中的列，或者 DISTINCT 中的列等创建索引
- 考虑索引列中不重复值的个数
- 索引列的数据类型尽量小。数据类型越小，索引占用的存储空间就越少，在一个数据页内就可以存放更多的记录，磁盘 I/O 带来的性能损耗也就越小
- 只为索引列前缀建立索引。可以减少索引占用的存储空间，但无法支持使用前缀索引进行排序（GROUP BY 子句）的需求
- 覆盖索引可以避免回表操作带来的性能损耗
- 让索引列以列名的形式在搜索条件中单独出现，不要在索引列上进行了计算、函数、类型转换等操作
- 避免创建冗余和重复索引
- 新插入记录时主键大小对效率的影响。尽量使新插入记录的主键值依次增大，这样每插满一个数据页就换到下一个数据页继续插入。如果新插入记录的主键值忽大忽小，可能会导致页面分裂，使性能损耗
- 对于 InnoDB 存储引擎的表，尽量手工指定主键
- 限制索引的数量，建议每个表的索引数量不超过 6 个

