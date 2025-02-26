# DinoDB-Technical-File

DinoDB是一个简单而高效的数据库系统实现，主要关注数据库的核心组件和并发控制机制。

## 项目特点

- **数据结构**：基于B+树的索引结构，支持高效的查询和范围扫描
- **并发控制**：使用事务管理器实现严格的2PL（两阶段锁定）协议，确保事务的隔离性
- **页面管理**：实现LRU（最近最少使用）缓存机制的页面管理器，优化内存使用
- **连接算法**：基于分区哈希连接的高效查询处理，使用布隆过滤器优化性能

## 核心组件

### B+树索引

B+树是一种自平衡的树形数据结构，支持高效的插入、删除和范围查询操作。在本项目中实现的B+树具有以下特点：

- 度数(degree)为202，允许更高的分支因子减少树高
- 所有数据存储在叶子节点，内部节点只存储索引键
- 叶子节点通过右兄弟指针连接，支持高效的顺序扫描

支持的操作：
- 插入（Insert）
- 查找（Find）
- 更新（Update）
- 删除（Delete）
- 范围查询（SelectRange）
- 全量查询（Select）

**并发控制：** 采用悲观锁爬行策略，允许多个读操作并发执行，同时与写操作互斥。

### 页面管理器（Pager）

Pager负责管理内存中的数据页与磁盘文件的交互，主要特点：

- 实现LRU缓存机制，优化内存使用
- 通过PinnedList、UnpinnedList和FreeList管理页面状态
- 提供页面获取、释放和刷新接口

LRU策略：
- 新访问的页面放入pinnedList尾部（最近使用）
- 不再使用的页面放入unpinnedList尾部（最近使用但候选淘汰）
- 需要新页面时从freeList或unpinnedList头部（最久未使用）获取

### 事务管理器（Transaction Manager）

实现基于严格2PL协议的事务管理，确保数据库的ACID特性：

- 通过ResourceLockManager管理资源与锁的映射关系
- 使用WaitsForGraph进行死锁检测
- 支持事务的开始、提交、回滚以及锁定/解锁操作

主要组件：
- **ResourceLockManager**：管理资源与对应互斥锁的映射
- **WaitsForGraph**：通过图算法检测事务间的死锁
- **Transactions Map**：维护活跃事务及其持有的资源锁

### 连接算法（Join）

基于分区哈希连接的实现，特点：

- 通过哈希函数对表数据进行分区
- 使用布隆过滤器优化探测阶段性能
- 支持key-key, key-value, value-key等多种连接模式

工作流程：
1. 构建阶段：创建左右表的临时哈希索引
2. 探测阶段：使用布隆过滤器和哈希表查找匹配记录

## 项目特色

- **高效性能**：B+树索引和哈希连接确保查询处理的高效性
- **良好的并发支持**：通过悲观锁爬行和两阶段锁定实现并发控制
- **内存优化**：LRU页面缓存机制有效降低磁盘I/O
- **死锁处理**：通过等待图进行死锁检测和处理

## 使用方法

### 编译

```bash
# 编译服务器端
go build -buildvcs=false -o dinodb ./cmd/dinodb

# 编译客户端
go build -buildvcs=false -o dinodb_client ./cmd/dinodb_client

# 编译压力测试工具
go build -buildvcs=false -o dinodb_stress ./cmd/dinodb_stress
```

### 测试

```bash
# 完整测试
go test './test/concurrency/...' -race -timeout 180s -v

# 压力测试，并发线程数为8
./dinodb_stress -index=btree -workload=workloads/i-a-sm.txt -n=8 -verify
```

## 性能测试结果

- **B+树插入**：在多线程环境下，乱序插入比顺序插入表现更好
- **连接操作**：随着数据量增加，执行时间呈线性增长，对不同匹配率表现稳定
- **Select操作**：在1-8线程间性能较为稳定，但16线程时出现性能下降

## 项目结构

- **BTreeIndex**：B+树索引实现
- **Pager**：页面管理器
- **TransactionManager**：事务管理和并发控制
- **Join**：哈希连接算法实现

## 未来展望

- 添加更多索引类型支持
- 优化大规模数据处理性能
- 增强分布式数据处理能力