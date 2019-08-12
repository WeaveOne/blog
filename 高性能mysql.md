## 高性能mysql

### ALTER 

```sql
# alter很慢 将进行表的重建，相当于拷贝到了一张新表
ALTER TABLE table
MONDIFY COLUMN xxx INT(3) NOT NULL DEFAULT 5
# alter很快 直接修改.frm文件
ALTER TABLE xxx
ALTER COLUMN xxx SET  DEFAULT 5
```

1. 以下操作是有可能不需要重建表的

   1. 移除（非增加）一个列的AUTO_INCREMENT属性
   2. 增加、移除或更改ENUM和SET常亮。如果移除的是已经有行数据用到其值的常量，查询将返回一个空串
   3. 基本技术是创建一个新的.frm文件然后替换原来的
   4. 思路

   ```sql
   #创建新表
   CREATE TABLE xxx_new like xxx
   # 修改新表的属性 比如增加enum的一个参数
   ALTER TABLE xxx_new 
   MODIFY COLUMN xxx ENUM('xx','xx','xx_new') DEFAULT 'xx'
   # 关闭所有正在使用的表，并且禁止任何表被打开（锁表）
   FLUSH TABLE WITH READ LOCK
   # 通过操作系统命令交换.frm文件
   /var/lib/mysql/xxx# mv xxx.frm xxx_tmp.frm
   mv xxx_new.frm xxx.frm
   mv xxx_tmp xxx_new.frm
   #解锁
   UNLOCK TABLES
   ```

   ### B -Tree组合索引的限制

   1. 如果不是按索引的第一个开始查找，则无法使用
   2. 不能跳过索引中的列
   3. 如果有范围查询，则其右边所有列都无法使用索引优化查找。列如 `where name = 'xxx'  and address like 'j%' and age = 10` 这个查询只能使用索引前两列

### 哈希索引

   1. 只有memory引擎支持哈希索引，同时也支持B-Tree索引
   2. 如果哈希值相同会使用链表的方式记录

### B-Tree使用伪哈希索引

   1. 列如存储大量URL，并根据URL进行搜索查找，如果使用B-Tree存储内容会很大，因为URL很长。

   2. 这样可以使用一个被索引的url_crc列，使用crc32做哈希`select id from url where url='xxx' and url_crc = CRC32("xxxx")`

   3. 缺陷需要维护哈希值，可以使用**触发器**实现

      ```sql
      DELIMITER
      CREATE TRIGGER xxx_crc_ins BEFORE INSERT ON xxx FOR EACH ROW BEGIN SET NEW.url_crc = crc32(NEW.url)
      END;
      DELIMITER
      CREATE TRIGGER xxx_crc_ins BEFORE UPDATE ON xxx FOR EACH ROW BEGIN SET NEW.url_crc = crc32(NEW.url)
      END;
      ```

   4. 不要使用SHA1和MD5，这两个计算出来的哈希值太长。

   5. 如果数据表很大CRC32会出现大量哈希冲突

   6. 处理哈希冲突时，把原url带入即可

   7. 可以使用如FNV64作为哈希函数

   ps: 如果数据为TB级别可以建立一个块级别元数据信息表来替代索引

   ### 高性能的索引策略（索引按最左前缀查询）

   1. 独立的列：始终将索引列单独放在比较符号的一侧

   2. 前缀索引和索引选择性：索引选择性是指，不重复的索引值和数据表的记录总数的比值。索引的选择性越高则查询效率越高，因为可以再查找时过滤掉更多的行。唯一索引的选择性是1，是最好的索引选择性，性能也是最好的。对于BLOB、TEXT或者很长得VARCHAR必须使用前缀索引，因为mysql不允许索引这些列的完整长度。诀窍在于选择足够长的前缀以保证较高的选择性；

      ```sql
      # 计算完整列的选择性
      select count(distinct xxx)/count(*) from table;
      # 通过截取前缀来尽量达到完整列的选择性
      select count(distinct left(xxx,3))/count(*) from table;
      ```

      缺点：无法使用前缀索引做order by 和group by 和 覆盖扫描

   3. 覆盖索引

      1. 覆盖索引必须要存储索引列的值
      2. 一个被索引覆盖的查询（索引覆盖查询）在EXPLAIN的Extra列可以看到`Usering index`信息

   4. 使用索引扫描来排序

      1. 文件排序`use filesort`
      2. 第一列提供常量条件，使用第二列进行排序，这样两列组合在一起就形成了索引的最左前缀
      3. **组合索引有列是范围条件时，无法使用索引其余列进行索引扫描排序**
      4. 使用join有可能由于优化问题造成join左边的表被当做关联的第二张表导致无法使用左边表的所有进行排序

   5. 技巧：

      1. 按用户查询习惯或者需求来设计索引，即时 列的索引选择性低，但是每次都会使用可以考虑将其设为第一个索引

      2. 组合索引 如sex 只有m和f两种，当没有选择sex查询时，可以使用 sex in (m,f)来能够匹配索引左前缀，如果IN列表太长则不适用

      3. 范围查询所有尽量放在索引列的最后面，以便优化器使用更多的索引列，因为范围查询列会使后面的列索引失效

      4. 对于范围条件查询(> < 等)，mysql是无法使用后面的其他索引列，但多于In()（多个等值条件查询）则没有这个限制

      5. **延迟关联**（通过覆盖索引查询，可以减少mysql扫描那些需要丢弃的行数）

         ```sql
         select col from tab join (
         	select primary key from tab where xx = xx order by xx limit 100000,10
         ) AS using(primary key)
         ```


### 维护索引和表

1. 统计信息在打开某些INFORMATION_SCHEMA表或者使用SHOW TABLE STATUS 和SHOW INDEX触发，关闭innodb_stats_on_metadata来避免触发索引信息采样更新时可能会导致的大量的锁,必要时手动触发(ANALYSIS TABLE XXX)

#### 减少索引和数据碎片

1. 碎片会导致随机IO，有三种类型的数据碎片
   1. 行碎片：这种碎片指的是数据行被存储为多个地方的多个片段中。即时查询只从索引中访问一行记录，行碎片也会导致性能下降(innodb不会出现)
   2. 行间碎片：指逻辑上的顺序的页，或者行在磁盘上不是顺序存储的。行间碎片对诸如全表扫描和聚簇索引扫描之类的操作有很大的影响，因为这些操作原本能够从磁盘上顺序存储的数据中获益
   3. 剩余空间碎片：指数据页中有大量的空余空间。导致服务器读取大量不需要的数据，从而造成浪费
2. 解决：
   1. 执行OPTIMIZE TABLE XX
   2. 导出再导入
   3. 对于新版本innodb可以先删除索引在创建来消除碎片化

### 查询性能优化

#### 衡量查询开销的三个指标

响应时间、扫描的行数、返回的行数

1. 响应时间
   1. 服务时间和排队时间
2. 扫描的行数和返回的行数
3. 扫描的行数和访问类型
   1. 在索引中使用where条件来过滤不匹配的记录。在存储引擎层完成
   2. 使用索引扫描(Using index) 来返回记录
   3. 从数据表返回数据，然后过滤不满足条件的记录(Using Where)。这在MySQL服务器层完成，MySQL需要先从数据读出记录然后过滤。
   4. count()查询不能减少需要扫描的行数
   5. 如果发现查询需要扫描大量数据但只返回少数的行，可以尝试下面的技巧
      1. 使用索引覆盖扫描，把索引需要的列都放到索引中
      2. 改变库表结构，列如使用单独的汇总表
      3. 重写这个复杂的查询，让MySQL优化器能够以更优化的方式执行查询

#### 重构查询方式

 	1. 讲一个大查询分解为多个小查询是很有必要的，需要好好衡量一下这样做是不是会减少工作量
 	2. 比如删除数据，定期清除大量数据时，如果用一个大的语句一次性完成的话，则可能需要一次锁住很多数据、占满整个事务日志、耗尽系统资源、阻塞很多小的但重要的查询。将一个大的DELETE语句切分成多个较小的查询可以尽可能小地影响MySQL性能，同时还可以减少MySQL复制的延迟。可以每次删除的时候暂停一下再做下一次删除，可以将压力分散到一个长的时间段中，大大降低对服务器的影响，还可以大大减少删除时锁得持有时间。
 	3. 分解关联查询，在应用程序中进行关联优势如下：
      	1. 缓存效率更高
      	2. 执行单个查询可以减少锁（共享锁）得竞争
      	3. 应用层关联，更容易堆数据库进行拆分，更容易做到高性能和可扩展
      	4. 查询本身效率可以有所替代，**使用IN代替关联查询，可以让MySQL按照ID顺序查询**，这可能比随机的关联更高效
      	5. 相当于在应用中实现了哈希关联

#### 查询执行的基础





