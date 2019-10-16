---
layout: post
title: "MySQL 笔记 - 关于 change buffer"
date: 2019-10-16
excerpt: "普通索引的更新详解"
tags: [mysql]
comments: false
feature: https://cm-alimama-upload-1253836176.file.myqcloud.com/txb/resource/156767337013.png
---

![image](http://img.qingtingip.com/crawler/article/201971/dc70a3ac18bca10b9d3695664384367b)

``change buffer`` 是一种特殊的数据结构，占用 ``buffer pool`` 内存，用于缓存不在 ``buffer pool`` 中的**非唯一约束二级索引**的数据页的修改。

## 普通索引和唯一索引的更新区别

1. 当索引的数据页在 ``buffer pool`` 中：
    - 唯一索引：直接获取 ``buffer pool`` 缓存数据，判断是否有冲突后，直接修改，结束。
    - 普通索引：直接获取 ``buffer pool`` 缓存数据，直接修改，结束。

    这种情况，都是直接对内存进行修改，差别只是一个判断，性能差距忽略不计。

2. 当索引的数据页不在 ``buffer pool`` 中：
    - 唯一索引：随机 IO 读磁盘，将数据页读入 ``buffer pool`` 中，判断是否有冲突后，直接修改，结束。
    - 普通索引：直接更新操作 ``change buffer`` 中，结束。

    这种情况，唯一索引由于要判断唯一性，必须将数据从磁盘载入内存。

    **将数据从磁盘读入内存涉及随机 IO 的访问，是数据库里面成本最高的操作之一。**

    **因此 change buffer 针对于普通索引的修改，并减少了修改成的随机访问磁盘，明显提高了性能。**

## change buffer 和 redo log

现在，我们要在表上执行这个插入语句：
```
mysql> insert into t(id,k) values (id1,k1),(id2,k2);
```
这里，假设 k 是普通索引，k1 所在的索引数据页在 ``buffer pool``中，为 page1，k2 不在 ``buffer pool`` 中，为 page2，下面为此普通索引的更新过程图：

![image](https://static001.geekbang.org/resource/image/98/a3/980a2b786f0ea7adabef2e64fb4c4ca3.png)

执行过程：
1. page1 在 ``buffer pool`` 中，直接更新内存
2. page2 不在 ``buffer pool`` 中，在 ``change buffer`` 记录下“我要在 page2 插入这行”的信息
3. ``redo log`` 记录索引这两个动作

虚线说明：
1. ``change buffer`` 持久化，同步到 ``ibdata1`` 系统表空间
2. 将 ``buffer pool`` 更新的数据（脏页）刷回（purge）磁盘，而不是根据 ``redo log``

接下来，我们立刻查询刚刚插入的数据：
```
mysql> select * from t where k in (k1,k2);
```
下面读数据的过程图：

![image](https://static001.geekbang.org/resource/image/6d/8e/6dc743577af1dbcbb8550bddbfc5f98e.png)

执行过程：
1. page1 直接从内存中读取
2. page2 先从磁盘中读入内存中，然后结合 ``change buffer`` 记录，生成正确的 page2，这个过程称为 ``merge``。

可以看出，直到 page2 被读取时，这个数据页才被读入内存。

## buffer pool 中的索引数据页什么时候执行 purge 到磁盘，并将 change buffer 持久化到 ibdata

1. 访问数据页，数据页不在``buffer pool``时，先 ``merge``，后 ``purge``。
2. 有一个定时后台线程，认为数据库空闲时
3. 数据库 ``buffer pool`` 空间不足
4. 数据库正常关闭时
5. ``redo log`` 写满时

> ``redo log`` 记录了更新的动作，并依次执行刷脏页流程，实现批量顺序 IO。
>真正对磁盘索引页数据修改，是通过将内存中的脏页刷回磁盘来完成的

> ``redo log`` 节省随机写磁盘的 IO 消耗（转为顺序写），而 ``change buffer`` 节省随机读磁盘的 IO 消耗。

## change buffer 恢复

如果掉电，持久化的 ``change buffer`` 已经 ``purge``，不用恢复。主要分析没有持久化的数据，以下情况：

1. ``change buffer`` 写入，``redo log`` 虽然做了 fsync 但未 commit ，``binlog``未 fsync 到磁盘,这部分数据丢失
2. ``change buffer`` 写入，``redo log`` 写入但没有 commit，``binlog`` 以及 fsync到磁盘，先从 ``binlog`` 恢复 ``redo log``，再从 ``redo log`` 恢复 ``change buffer``
3. ``change buffer`` 写入，``redo log`` 和``binlog`` 都已经 fsync。那么直接从 ``redo log`` 里恢复。

## 配置 change buffer

当新增，更新和删除操作在表中执行，索引列的值（尤其是非主键索引列的值）通常是无序的，如果需要立即更新这些非主键索引会要求大量的随机 IO。当跟这些操作（增删改）相关的非主键索引数据页不在内存中的时候，``change buffer`` 会将相关的修改缓存起来，从而避免了昂贵的随机 IO。当数据页加载进 ``buffer pool`` 后，``change buffer`` 缓存的修改将会合并到数据页中，随后会刷新到磁盘中。当 MySQL 空闲或者正常关闭的过程中，InnoDB 的 ``main`` 线程会合并 ``change buffer`` 的更改。

由于 ``change buffer`` 可以减少磁盘 IO，所以 ``change buffer`` 适合于频繁增删改的场景

然而，``change buffer`` 占用的是 ``buffer pool`` 的空间，这会相对减少缓存数据页的空间。如果数据页基本都在 ``buffer pool`` 中，或者表的非主键索引比较少，那么建议关闭 ``change buffer``。

通过参数 ``innodb_change_buffering`` 可以控制InnoDB``change buffer`` 缓存的类型

值 | 说明|
---|---|
all |   默认值，缓存插入、删除标记操作和清除。|
none |  不缓存任何操作|
inserts |   缓存插入操作|
deletes |   缓存删除操作|
changes |   缓存插入和删除操作|
purages | 缓存发生在后台的物理删除操作|

可以通过MySQL配置文件（my.cnf 或 my.ini）或使用 ``set global`` 语句动态设置 ``innodb_change_buffering``，这要求 MySQL 账户拥有足够的权限（root 用户）。修改参数后不会影响 ``change buffer`` 中已缓存的修改。

## 配置 change buffer 的最大值

通过参数``innodb_change_buffer_max_size``参数，可以设置``change buffer``在``buffer pool``的最大值（以百分比的形式）。默认情况下，``innodb_change_buffer_max_size``的值为25，最大设置的值是50。

当MySQL需要承受大量的新增、更新或删除操作的时候，可以考虑增大``change_buffer``在``buffer pool``中的比例。

当 MySQL 主要用于静态数据报表之类，或者``change buffer``占用``buffer pool``的空间过大导致内存命中率下降的时候，可以考虑减少``change_buffer``在``buffer pool``中的比例。

参数``innodb_change_buffer_max_size``得设置是动态的，修改后不需要重启 MySQL。

---
参考：
-  [缓冲池(buffer pool)，这次彻底懂了！！！](https://juejin.im/post/5d11a79ee51d4555e372a624#heading-4)
- [MySQL学习笔记 - 3 - 关于change buffer的补充](https://juejin.im/post/5ce78e71f265da1b7a4b4e8d)
- [普通索引和唯一索引，应该怎么选择？](https://time.geekbang.org/column/article/70848)
