
---
layout: post
title: SQLite 中 B-tree 的实现
category: 数据库
tags: sqlite b-tree
keywords: B-tree
description:
---

# SQLite 中 B-tree 的实现

B-tree 在 SQLite 中占有举足轻重的地位。SQLite 的结构图如图 1 所示。B-Tree 位于虚拟机和页缓存之间。虚拟机负责执行程序，负责操纵数据库文件。数据库的存储用 B 树实现。B 树向磁盘请求信息，以固定大小的块。系统默认的块的大小是 1024 字节，块大小应在 512 字节到 65536 字节之间。B 树从页缓存中请求页，当修改页或者提交，或者回滚操作时，通知页缓存。修改页时，如果使用传统的回滚日志，pager 首先将原始页复制到日志文件。同样，page 在 B-tree 完成写操作时收到通知，并基于所处的事务状态决定如何处理。

![SQLiteBlockGraph](/assets/img/post/20170821/SQLiteBlockGraph.jpg)

## SQLite中的B-tree

SQLite中每个数据库完全存储在单个磁盘文件中，因为B树进行数据的查找、删除、添加速度快，所以这些数据以B树数据结构的形式存储在磁盘上（实现代码在btree.c源文件中）。INGRES那样的DBMS，也用树结构（如B树）来实现存储，以支持借助于多级目录结构的按关键字存取。B树的典型结构如图2所示。B+树是应文件系统所需而出的一种B-树的变型树。B+树可以进行两种查找算法，第一种，从最小关键字起顺序查找；第二种，从根结点开始，进行随机查找。B-tree应用到数据库的每个表和索引中。所有B-tree存储在相同的磁盘文件中。

![典型的B-tree](/assets/img/post/20170821/典型的B-tree.jpg)


B-tree为SQLiteVDBE提供O（㏒N）级时间复杂度的查询和插入，通过遍历记录实现O（1）级时间复杂度的删除。B-tree是自平衡的，并能够对碎片清理和内存再分配进行自动管理。B-tree对如何读写磁盘没有限定，只是关注页之间的关系。

B-tree的职责就是排序，它维护着多个页之间错综复杂的关系，这些关系能够保证快速定位并找到一切有联系的数据。B-tree将页面组织成树状结构，这些组织结构很适合搜索，页面就是树的叶子。Pager帮助B-tree管理页面，它负责传输。B-tree是为查询而高度优化了的。它使用特殊的算法来预测将来要使用哪些页，从而让B-tree保存该页面以便尽可能快地工作。

数据库中所有的页都是以1开始顺序编号的。一个数据库是由多个B-tree组成的——每张表以及每个索引各对应一个B-tree。数据库中每张表或索引都以根页面作为第一页。所有的索引和表的根页面都存储在sqlite_master表中。

B-tree中的页由一系列B-tree记录组成，这些记录也称为有效载荷。这些记录不是传统的数据库记录（表中有多个列的那种格式），而是更为原始的格式。一个B-tree记录（有效载荷）仅由两个域组成：键值域和数据域。键值域是每个数据库表中所包含的ROWID值或主键值；在B-tree中，数据域可以包含任意类型的内容。最终，数据库的记录信息存储在数据域中。B-tree用来保持记录有序并方便记录的查询，同时，键值域能够完成B-tree的主要工作。此外，记录（有效载荷）的大小是可变的，这取决于内部键值域和数据域的大小。一般而言，每个页拥有多个有效载荷，但如果一个页的有效载荷超出了一个页的大小，将会出现跨越多个页的情况（包括blob类型的记录）。

B-tree记录按键值顺序存储。所有的键值在一个B-tree中必须唯一（由于键值对应于rowid主键，主键具有唯一性）。表使用B+tree定义在内部页中，不包含表数据（数据库记录）。图3为B+tree表示一个表的实例。

![3B+tree组织结构](/assets/img/post/20170821/3B+tree组织结构.jpg)

B+tree的根页面和内部结点页都用于搜索导航。这些页中的数据域均指向下一层页，这些页只包含键值。所有的数据库记录都存储在叶子页中，在叶子页层，记录和页按键值顺序排列，以便B-tree游标能够遍历记录（水平遍历）。

## SQLite层次化数据组织

基本上，具体的数据单元由堆栈中的各模块处理，如图4所示。自下而上，数据变得更加精确、详细。自上而下，数据变得更集中、模糊。具体地说，C API负责域值，VDBE负责处理记录，B-tree负责处理键值和数据，pager负责处理页，操作系统接口负责处理二进制数据和原始数据存储。 每个模块负责维护自身在数据库中对应的数据部分，然后依靠底层提供所需信息的初始数据，并从中提取需要的内容。

![3B+tree组织结构](/assets/img/post/20170821/4hierarchicalorganization.jpg)

## 页面溢出

有效载荷及其内容可有不同的大小。然而，页面大小是固定不变的。因此，给定的有效载荷总有可能超出单页装载大小。这种情况发生时，额外的有效载荷将添加到溢出页面的链接链表上。由此看来，有效载荷将在有序的链接链表中显示，如图5所示。

![5overflow](/assets/img/post/20170821/5overflow.jpg)

图5中第4个有效载荷超出了当前页所能装载的大小。因此，B-tree模块创建了溢出页来装载。实际上，一个溢出页也不能装载，因此，又链接了第二个溢出页。这实际上就是处理二进制大对象的方法。使用真正的大字段时，最后都采用页链接链表来存储。如果blob字段太大，这种方式效率很低，此时，可考虑创建外部文件来存储blob数据，并将外部文件名保存在记录中。

## 实现的技术细节

```c
static int btreeCreateTable(Btree *p, int *piTable, int createTabFlags)
```

1. 创建一个btree表格
2. 移动现有数据库为新表的根页面腾出空间
3. 用新根页更新映射寄存器和metadata
4. 新表根页的页号放入PgnoRoot并写入piTable

流程图如下：

![btreeCreateTable](/assets/img/post/20170821/btreeCreateTable.jpg)

```c
int sqlite3BtreeInsert(  BtCursor *pCur, const void *pKey, i64 nKey,  const void *pData, int nData, int nZero,  int appendBias, int seekResult)
```

向Btree中插入一个新记录：

1. 关键字按照(pKey,nKey)给定，数据按照(pData,nData)给定。
2. 找到结点插入位置
3. 分配内存空间
4. 插入结点

流程图如下：

![Insert](/assets/img/post/20170821/Insert.jpg)

```c
static int btreeDropTable(Btree *p, Pgno iTable, int *piMoved)
```

1. 删除表中所有信息
2. 将表的根页放入自由列表
3. 游标不能为打开状态
4. 更新最大的根页

流程图如下：

![droptable](/assets/img/post/20170821/droptable.jpg)
