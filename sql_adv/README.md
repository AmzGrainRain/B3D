# SQL 进阶

## MySQL 体系结构
![MySQL 体系结构图](./images/sql_architecture.png)

### 连接层
最上层是一些客户端和链接服务，主要完成一些类似于连接处理、授权认证、及相关的安全方案。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

### 服务层
第二层架构主要完成大多数的核心服务功能，如 SQL 接口，并完成缓存的查询，SQL 的分析和优化，部分内置函数的执行。所有跨存储引擎的功能也在这一层实现，如过程、函数等。

### 引擎层
存储引擎真正的负责了 MySQL 中数据的存储和提取，服务器通过API和存储引擎进行通信。不同的存储引擎具有不同的功能，这样我们可以根据自己的需要，来选取合适的存储引擎。

### 存储层
主要是将数据存储在文件系统之上，并完成与存储引擎的交互。

## 存储引擎

### 查看数据库支持的引擎
通过此方式查看数据库支持的存储引擎：
``` sql
SHOW ENGINES;
```

### MyISAM
MyISAM 是 MySQL 5.5 之前的默认数据库引擎。由早期的 ISAM 所改良。性能极佳，在这几年的发展下，InnoDB 数据库引擎以强化参照完整性与并发违规处理机制，在一些方面逐渐取代了 MyISAM 数据库引擎。

#### 特点
- 支持全文索引，在涉及全文索引领域的查询效率上 MyISAM 速度更高。（5.7 以后的 InnoDB 也支持全文索引）
- 遇到错误，必须完整扫描后才能重建索引或修正未写入硬盘的错误。且 MyISAM 的修复时间与数据量的多少成正比。
- MyISAM 引擎的数据表被压缩后仍然可以进行查询操作。
- 执行 `SELECT COUNT(*) FROM 数据表` 时不需要全表扫描，因为 MyISAM 保存了表的具体行数。执行上述语句时如果加有 WHERE 条件，就会退化为全表扫描。
- 读写互相堵塞。在 MyISM 类型的表中，向表中写入数据的同时另一个会话无法向该表中写入数据，也不允许其他的会话读取该表中的数据。只允许多个会话同时读取数据表中的数据。
- 只会缓存索引，不会缓存数据。所谓缓存，就是指数据库在访问磁盘数据时，将更多的数据读取进入内存，这样可以使得当访问这些数据时，直接从内存中读取而不是再次访问硬盘。MyISAM 可以通过 key_buffer_size 缓存索引，以减少磁盘 I/O，提升访问性能。但数据表并不会被缓存。
- 不支持行级锁，仅支持表级锁定。即发生数据更新时，会锁定整个表，以防止其他会话对该表中数据的同时修改所导致的混乱。这样做可以使得操作简单，但是会减少并发量。
- 不支持外键约束和事务。

#### 存储形式
MyISAM 表在磁盘上对应着三个文件，他们以表名作为文件名：`.frm` 存储了表的结构；`.MYD` 存储了表的数据；`.MYI` 存储了表的索引。

#### 创建方法
``` sql
CREATE TABLE 表名 (
  字段 字段类型 [约束条件],
  字段 字段类型 [约束条件]
) ENGINE=MyISAM;
```

### InnoDB
**InnoDB 是一种兼顾高可靠性和高性能的通用存储引擎，支持事务、行级锁、外键约束。并在 MySQL 5.5 之后成为 MySQL 的默认存储引擎。**

#### 特点
- InnoDB 是聚集索引，使用 B+Tree 作为索引结构，数据文件是和（主键）索引绑在一起的（表数据文件本身就是按 B+Tree 组织的一个索引结构），必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。
- InnoDB 可借由事务记录档（Transaction Log）来恢复程序崩溃，或非预期结束所造成的数据错误（修复时间，大略都是固定的）相对而言，随着数据量的增加，InnoDB 会有较佳的稳定性。
- InnoDB 有自己的读写缓存管理机制。（InnoDB 不会将被修改的数据页立即交给操作系统）因此在某些情况下，InnoDB 的数据访问会更快。
- 支持行级锁，并发性能优异。
- 支持外键约束和事务。

#### 逻辑存储结构
表空间 -> 段 -> 区 -> 页 -> 行
![InnoDB 逻辑存储结构](./images/innodb_logical_structure.png)

有以下几点需要注意：
- 区的大小是固定的 1MB 。
- 页的大小是固定的 16KB。
- 一个区最多存储 64 个页。
- 磁盘操作的最小单元是页。

#### 存储形式
> ibd 对应着[逻辑存储结构](#逻辑存储结构)中的表空间

查看 INNODB_FILE_PER_TABLE 状态：
``` sql
SHOW VARIABLES LIKE 'INNODB_FILE_PER_TABLE';
```

创建 InnoDB 表时，分两种情况：  
- 如果通过 `SET GLOBAL INNODB_FILE_PER_TABLE = 1;` 将 INNODB_FILE_PER_TABLE 设置为 ON，然后再去创建一个 InnoDB 表，创建表的同时，磁盘上也会出现两个文件，他们以表名作为文件名：`.ibd` 内部存储了表的数据，`.frm` 内部存储了表的结构和索引。
- 如果通过 `SET GLOBAL INNODB_FILE_PER_TABLE = 0;` 将 INNODB_FILE_PER_TABLE 设置为 OFF，然后再去创建一个 InnoDB 表，创建表的同时，磁盘上也会出现两个文件，后缀为 `.ibd`。内部存储了数据表的结构、数据和索引。

我们可以通过命令查看 `.ibd` 内部存储的表结构：
``` bash
ibd2sdi 表名.ibd
```

#### 创建方法
InnoDB 是 MySQL 的默认引擎，创建表时不指定引擎参数，MySQL 会默认创建 InnoDB 引擎的数据表：
``` sql
CREATE TABLE 表名 (
  字段 字段类型 [约束条件],
  字段 字段类型 [约束条件]
);
```

### MRG_MYISAM
**MERGE 存储引擎，也被称为 MRG_MYISAM 引擎（又叫分表）**是一个相同的可以被当作一个来用的 MyISAM 表的集合。“相同”意味着所有表结构、索引、字段顺序都需要一致。而且，任何或者所有的表都可以用 myisampack 来压缩。表选项的差异，比如 AVG_ROW_LENGTH，MAX_ROWS 或 PACK_KEYS 都不重要。 

#### 特点
当你创建 MRG_MYISAM 表时，你必须指定 `UNION=(子表1, 子表2, ...)` 子句，它用来说明你要把哪些表当作一个来用。MRG_MYISAM 表默认是只读的，如果你想要自己对 MRG_MYISAM 表的插入操作生效，那么可以指定 `INSERT_METHOD` 选项，它支持三个值：
- `NO`：只读（默认值）
- `FIRST`：插入操作发生在第一个子表上
- `LAST`：插入操作发生在最后一个子表上

如果你没有指定 `INSERT_METHOD` 选项，那么 MRG_MYISAM 表将会是只读的。  
你可以对表的集合使用 `SELECT`, `DELETE`, `UPDATE` 和 `INSERT`，并且你必须拥有所有子表的 `SELECT`, `DELETE`, `UPDATE` 和 `INSERT` 权限。对 MRG_MYISAM 表做出的操作只能生效于子表的第一个或最后一个。  
删除 MRG_MYISAM 表不会影响子表。

#### 存储形式
MRG_MYISAM 表在磁盘上对应着两个文件，他们以表名作为文件名：`.frm` 后缀名的文件文件存储表定义；`.mrg` 后缀名的文件包含被当作一个来用的表的名字。（这些子表作为 MMRG_MYISAM 表自身，不必要在同一个数据库中）

#### 创建方法
> 所有子表与主表的结构必须一致。  

先创建三个子表：
``` sql
CREATE TABLE 子表1 (
  字段 字段类型 [约束条件],
  字段 字段类型 [约束条件]
) ENGINE=MyISAM;

-- 复制表操作
CREATE TABLE 子表2 LIKE 子表1;
CREATE TABLE 子表3 LIKE 子表1;
```

创建主表（只读）：
``` sql
CREATE TABLE 主表 (
  字段 字段类型 [约束条件],
  字段 字段类型 [约束条件]
) ENGINE=MRG_MyISAM INSERT_METHOD=NO UNION(子表1, 子表2, 子表3);
```

### MEMORY
Memory 引擎的表数据是存储在内存中的，受硬件问题、断电问题的影响，只能将这些表作为临时表或缓存使用。

#### 特点
- 数据单独存放，索引上保存的是数据的位置，该方式称之为堆组织表。
- 不支持行锁，支持表锁。
- 表的锁力度过大，在处理并发事务时性能较低。
- 数据存放在内存中，如果数据库重启，表中的数据将会被清除，无法将数据持久化。
- 当表的数据大于 `max_heap_table_size` 设定的容量大小时，会转换超出的数据存储到磁盘上，因此性能会大打折扣。
- 使用哈希散列索引把数据保存在内存中，因此具有极快的速度，适合作为中小型缓存数据库使用。
- 不允许使用 `xxxTEXT` 和 `xxxBLOB` 数据类型；**只精确搜索记录**；不支持 `AUTO_INCREMENT`；只允许对非空数据列进行索引。
- 如果是复制的某数据表，则复制之后所有主键、索引、自增等格式将不复存在，如果需要的话，请重新添加主键和索引。

#### max_heap_table_size
我们可以根据实际情况调整 max_heap_table_size，例如在 `my.cnf` 文件中的 [mysqld] 下面加入：
``` cnf
max_heap_table_size = 2048M
```

#### 存储形式
MEMORY 表在磁盘上对应着一个文件，以表名作为文件名，后缀为 `.frm`，内部存储的是表结构。

#### 创建方法
> 在建表时，我们可以通过 MAX_ROWS 来控制表的记录数。

正常情况：
``` sql
CREATE TABLE 表名 (
  字段 字段类型 [约束条件],
  字段 字段类型 [约束条件]
) ENGINE=MEMORY;
```

创建一个最大存储 100 行数据的 Memory 表：
``` sql
CREATE TABLE 表名 (
  字段 字段类型 [约束条件],
  字段 字段类型 [约束条件]
) ENGINE=MEMORY MAX_ROWS=100;
```

### CSV
CSV 存储引擎可以将 csv 文件作为表进行处理。存储格式就是普通的 csv 文件。

#### 特点
- 直接将使用逗号分隔值格式的文本数据存储到表。
- 不支持索引、事务、查询下推等，一般用于日志表的数据存储或者作为数据转换的中间表。
- 可以直接导入 EXCEL 表或者 CSV 文件，使用起来非常方便。
- 所有的列必须都是非空的。
- 可以对数据文件直接编辑（保存文本文件内容）

#### 存储形式
创建 CSV 表时，其对应着磁盘上的三个文件，文件名以表的名字开始：`.CSV` 后缀名的文件存储表内容；`.CSM` 后缀名的文件存储表的元数据如表状态和数据量；`.frm` 后缀名的文件存储表结构信息。
如果是 8.0 版本的数据库，则没有 `.frm` 文件，取而代之的是 `.sdi` 文件。

#### 创建方法
``` sql
CREATE TABLE 表名 (
  字段 字段类型 NOT NULL [约束条件],
  字段 字段类型 NOT NULL [约束条件]
) ENGINE=CSV;
```

### FEDERATED
MySQL 提供了一个类似 Oracle 中的数据库链接（DBLINK）功能的存储引擎——FEDERATED，说人话就是“远程表”。本地只创建一个表定义文件（`.frm`），数据在服务端。自 5.1.26 版本开始，默认情况下不启用 FEDERATED 存储引擎。

#### 安装方法
FEDERATED 存储引擎支持动态安装（命令在 SQL 里面执行）：
``` sql
-- 全平台（推荐）
INSTALL PLUGIN FEDERATED SONAME 'ha_federated';

-- Windows
INSTALL PLUGIN FEDERATED SONAME 'ha_federated.dll';

-- Linux
INSTALL PLUGIN FEDERATED SONAME 'ha_federated.so';
```
安装完毕后再次查询即可发现 FEDERATED 的 supports 列已经变为了 yes

#### 特点
- 远程服务器必须是一个MySQL服务器。
- FEDERATED 对其它数据库引擎的支持可能会在将来被添加，目前不支持其他数据库，跨服务器远程连接其他类型数据库可以采用创建远程连接服务器的方式。
- FEDERATED 表指向的远程表在你通过 FEDERATED 表访问它之前必须存在。
- FEDERATED 表指向另一个 FEDERATED 表是可能的，小心不要创建一个循环。
- 不支持事务。
- 如果修改 FEDERATED 表，那么对应的远程表也会被改变。
- 如果远程表已经改变，对 FEDERATED 引擎的表而言是无法预知的。
- 您必须拥有目标数据库的权限，才能进行操作。
- FEDERATED 存储引擎支持 SELECT、INSERT、UPDATE、DELETE 和索引。
- 不支持 ALTER TABLE、DROP TABLE 或任何其它的 DDL 语句。且当前的实现不使用预先准备好的语句。
- FEDERATED 表不能对查询缓存不起作用。

#### 存储形式
创建 FEDERATED 表时，其对应着磁盘上的一个文件，文件名以表的名字开始，后缀名为 `.frm`，存储表的定义信息。

#### 创建方法
本地的 FEDERATED 与远程的表机构必须一致（表名可以不一样），CONNECTION 后面的是一个数据库连接字符串：
``` sql
CREATE TABLE 表名 (
  字段 字段类型 [约束条件],
  字段 字段类型 [约束条件]
) ENGINE=FEDERATED CONNECTION='mysql://user:host@ip:port/database/table'
```

### PERFORMANCE_SCHEMA
PERFORMANCE_SCHEMA 是运行在较低级别的用于监控 MySQL 运行过程中的资源消耗、资源等待等情况的一个功能特性。也是一个存储引擎。

#### 特点
- 提供了一种在数据库运行时，实时检查数据库服务器内部执行情况的方法。
- 可监控任何事情以及对应的时间消耗，利用这些信息来判断数据库服务器中的相关资源消耗。
- 只被记录在本地数据库服务器的 PERFORMANCE_SCHEMA 中，其表中数据发生变化时不会被写入 binlog 中，也不会通过复制机制被复制到其他数据库服务器中。
- 对于这些表可使用 SELECT 语句查询，也可以使用 SQL 语句更新 PERFORMANCE_SCHEMA 数据库中的表记录，但不建议更新，会影响后续的数据收集。
- 表中数据不会持久化存储在磁盘中，而是保存在内存中，一旦服务器重启，这些数据就会丢失。
- 它不会导致数据库服务器的行为发生变化（查询，优化等）。
- 总体上开销有限也不会影响性能。
- 对某事件监测失败，不影响数据库服务器正常运行。
- 当针对一个数据，同时被 PERFORMANCE_SCHEMA 收集和查询，则**收集优先于查询**。
- 事件监测点可进行配置。

#### 启用方法
查看数据库是否启用了 PERFORMANCE_SCHEMA ：
``` sql
SHOW VARIABLES LIKE 'PERFORMANCE_SCHEMA'
```

### BLACKHOLE

### ARCHIVE