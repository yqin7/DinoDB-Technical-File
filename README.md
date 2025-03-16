# 🌟 DinoDB-Technical-File

DinoDB是一个简单而高效的数据库系统实现，主要关注数据库的核心组件的开发。

> **注意**: 本仓库包含的是DinoDB系统的技术文档和可执行成果物，而非完整源代码。这些文档详细说明了系统架构、算法实现和关键组件的工作原理。仓库中的run_me_exe_files文件下的可执行文件（dinodb.exe, dinodb_client.exe, dinodb_stress.exe）可直接运行使用，无需额外编译。
>
> **推荐使用Typora阅读本项目所有文档。**

## 📚 文档说明

本项目包含以下技术文档：

- **B+ Tree.md** - 详细介绍B+树索引结构实现，包括节点结构、插入、查询、删除和更新
- **Concurrency for B+ Tree.md** - 说明B+树的并发控制实现，重点介绍悲观锁爬行策略
- **Join.md** - 讲解基于布隆过滤器的哈希连接算法实现和优化
- **Pager.md** - 描述页面管理器的设计，包括LRU缓存机制和页面替换策略
- **Transaction.md** - 阐述事务管理器和并发控制设计，包括2PL协议和死锁检测
- **RecoveryManager.md -** 详解基于WAL和简化版ARIES协议的故障恢复机制

## 🚀 项目特点
* **📊 数据结构**：基于B+树的索引结构，支持高效的查询和范围扫描
* **⚡ 并发控制**：采用悲观锁爬行策略实现B+树高效并发访问，支持读-读并发，读-写互斥
* **🔄 事务管理**：使用事务管理器实现严格的2PL（两阶段锁定）协议，确保事务的隔离性
* **💾 页面管理**：实现LRU（最近最少使用）缓存机制的页面管理器，优化内存使用
* **🔍 连接算法**：基于分区哈希连接的高效查询处理，使用布隆过滤器优化性能
* **🔁 故障恢复**：基于WAL（预写日志）和ARIES协议实现数据库恢复机制，确保数据库的持久性和一致性

## 🧩 核心组件

### 🌲 B+树索引
B+树是一种自平衡的树形数据结构，支持插入、删除和范围查询操作。在本项目中实现的B+树具有以下特点：
* 度数(degree)为202
* 所有数据存储在叶子节点，内部节点只存储索引键
* 叶子节点通过右兄弟指针连接，支持的顺序扫描

支持的操作：
* 插入（Insert）
* 查找（Find）
* 更新（Update）
* 删除（Delete）
* 范围查询（SelectRange）
* 全量查询（Select）

**并发控制：** 采用悲观锁爬行策略，允许多个读操作并发执行，同时与写操作互斥。通过"螃蟹锁"(Lock-Crabbing)机制，在树的遍历过程中动态获取和释放锁：

- 遍历时先锁住父节点，再锁住子节点，当确认子节点不会分裂时释放父节点锁
- 读操作使用读锁(RLock)，写操作使用写锁(WLock)，保证读-读并发
- 使用节点分裂传递机制处理高并发环境下的树结构动态平衡
- 在多线程环境下比单线程B+树实现提高118倍并发吞吐量

### 🔄 连接算法（Join）

基于分区哈希连接的实现，特点：

* 通过哈希函数对表数据进行分区
* 使用布隆过滤器优化探测阶段性能
* 支持key-key, key-value, value-key等多种连接模式

工作流程：

1. 构建阶段：创建左右表的临时哈希索引
2. 探测阶段：使用布隆过滤器和哈希表查找匹配记录

### 📝 页面管理器（Pager）

Pager负责管理内存中的数据页与磁盘文件的交互，主要特点：
* 实现LRU缓存机制，优化内存使用
* 通过PinnedList、UnpinnedList和FreeList管理页面状态
* 提供页面获取、释放和刷新接口

LRU策略：
* 新访问的页面放入pinnedList尾部（最近使用）
* 不再使用的页面放入unpinnedList尾部（最近使用但候选淘汰）
* 需要新页面时从freeList或unpinnedList头部（最久未使用）获取

### 🔐 事务管理器（Transaction Manager）
实现基于严格2PL协议的事务管理，确保数据库的ACID特性：
* 通过ResourceLockManager管理资源与锁的映射关系
* 使用WaitsForGraph进行死锁检测
* 支持事务的开始、提交、回滚以及锁定/解锁操作

主要组件：
* ResourceLockManager：管理资源与对应互斥锁的映射
* WaitsForGraph：通过图算法检测事务间的死锁
* Transactions Map：维护活跃事务及其持有的资源锁

### 📝 恢复管理器（Recovery Manager）

基于WAL（预写日志）和简化版ARIES协议实现故障恢复：

- 遵循"先写日志，后执行操作"的WAL原则
- 通过单一日志文件记录所有数据修改操作
- 实现三阶段恢复过程：分析、重做和撤销

主要功能：

- 日志记录：跟踪所有表创建、数据修改和事务状态变化
- 检查点：定期创建数据库的一致性快照，减少恢复时间
- 崩溃恢复：系统故障后通过日志重建数据库到一致状态
  - 分析阶段：识别崩溃时活跃的事务
  - 重做阶段：重新执行所有事务的操作
  - 撤销阶段：回滚未提交事务的操作

## 🔧 使用方法

### 运行环境

DinoDB是使用Go语言开发的自包含数据库系统，提供的二进制文件（在run_me_exe_files文件夹里）可以直接运行：

- **操作系统**：支持Linux、macOS和Windows
- **依赖**：不需要额外安装Go语言环境或其他数据库系统
- **权限**：可能需要为可执行文件添加执行权限（Linux/Mac下使用`chmod +x`）

### 编译

如果需要从源代码编译（非必需，可以直接使用提供的预编译二进制文件）：

```bash
# 编译服务器端
go build -buildvcs=false -o dinodb ./cmd/dinodb

# 编译客户端
go build -buildvcs=false -o dinodb_client ./cmd/dinodb_client

# 编译压力测试工具
go build -buildvcs=false -o dinodb_stress ./cmd/dinodb_stress
```

### 启动服务器

```bash
# 启动DinoDB服务器，指定项目和端口
./dinodb -project recovery -p 8335
```

服务器成功启动后会显示：
```
dinodb server started listening on localhost:8335
```

### 启动客户端

在另一个终端窗口中启动客户端：

```bash
# 连接到指定端口的DinoDB服务器
./dinodb_client -p 8335
```

连接成功后，您将看到REPL界面：
```
Welcome to the dinodb REPL! Please type '.help' to see the list of available commands.
dinodb> 
```

### 可用命令
使用 `.help` 命令查看所有可用命令：

```
dinodb> .help
transaction: Handle transactions. usage: transaction <begin|commit>
create: Create a table. usage: create <btree|hash> table <table>
select: Select elements from a table. usage: select from <table>
find: Find an element. usage: find <key> from <table>
checkpoint: Saves a checkpoint of the current database state and running transactions. usage: checkpoint
abort: Simulate an abort of the current transaction. usage: abort
pretty: Print out the internal data representation. usage: pretty
insert: Insert an element. usage: insert <key> <value> into <table>
update: Update en element. usage: update <table> <key> <value>
crash: Crash the database. usage: crash
delete: Delete an element. usage: delete <key> from <table>
lock: Grabs a write lock on a resource. usage: lock <table> <key>
```

### 命令使用示例

以下是一些常用操作的示例：

#### 创建表
```
dinodb> create btree table test
```

#### 只读操作
以下操作不需要事务包装，因为它们只是读取数据不修改数据库状态：
```
dinodb> find 2 from test
dinodb> select from test
dinodb> pretty from test
```

#### 查看数据结构
```
dinodb> pretty from test
[0] Leaf (root) size: 4
 |--> (1, 10)
 |--> (2, 20)
 |--> (3, 30)
 |--> (5, 500)
```

#### 修改数据的事务操作
所有修改数据的操作都需要在事务中执行：

##### 插入数据
```
dinodb> transaction begin
dinodb> insert 1 10 into test
dinodb> insert 2 20 into test
dinodb> insert 3 30 into test
dinodb> transaction commit
```

##### 更新数据
```
dinodb> transaction begin
dinodb> update test 2 25
dinodb> transaction commit
```

##### 删除数据
```
dinodb> transaction begin
dinodb> delete 3 from test
dinodb> transaction commit
```

##### 多操作事务
```
dinodb> transaction begin
dinodb> insert 5 500 into test
dinodb> update test 1 15
dinodb> delete 3 from test
dinodb> transaction commit
```

#### 使用锁（显式锁定）
```
dinodb> transaction begin
dinodb> lock test 1
dinodb> lock test 2
// 执行其他需要访问这些资源的操作
// 注意：通常不需要显式加锁，因为操作如update和delete会自动获取必要的锁
dinodb> transaction commit
```

#### 数据库恢复操作

##### 模拟数据库崩溃

```
dinodb> crash
Connection to server lost. Please restart the client.
```

##### 模拟事务中止

```
dinodb> transaction begin
dinodb> insert 7 700 into test
dinodb> abort
Transaction aborted.
```

##### 恢复流程展示

```
# 1. 创建表并添加数据
dinodb> transaction begin
dinodb> insert 1 100 into test
dinodb> insert 2 200 into test
dinodb> transaction commit

# 2. 创建检查点
dinodb> checkpoint

# 3. 更多操作
dinodb> transaction begin
dinodb> insert 3 300 into test
dinodb> update test 1 150
dinodb> transaction commit

# 4. 模拟崩溃
dinodb> crash

# 5. 重启服务器
# 在新终端中:
./dinodb -project recovery -p 8335

# 6. 重新连接客户端
# 在另一个终端中:
./dinodb_client -p 8335

# 7. 查看恢复后的数据
dinodb> select from test
```

### 测试

```bash
# 完整测试（无代码不能测试）
go test './test/concurrency/...' -race -timeout 180s -v

# 压力测试，并发线程数为8
./dinodb_stress -index=btree -workload=workloads/i-a-sm.txt -n=8 -verify

# 压力测试，如果上述命令报错，建议使用绝对路径
./dinodb_stress -index=btree -workload="C:\Users\huo00\OneDrive\Documents\DinoDB-Technical-File\workloads\i-i-md.txt" -n=8 -verify
```

## 📊 性能测试结果
* **B+树插入**：在多线程环境下，顺序插入比乱序插入表现更好
* **Join连接操作**：随着数据量增加，执行时间呈线性增长，对不同匹配率表现稳定
* **Select操作**：在1-8线程间性能较为稳定，但16线程时出现性能下降

## 🔮 未来展望
* 添加区间操作的命令，比如大于小于等。

## 📫 获取代码
由于课程要求（不能对未来学弟学妹公开代码），源代码暂时不能公开。如果你对项目感兴趣，请发送邮件至 huo000311@outlook.com 索取代码。
