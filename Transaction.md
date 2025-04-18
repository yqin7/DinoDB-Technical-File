# Transaction

# 1. 重要概念

## 1.1 严格两阶段锁定（Strict 2Phase Locking）协议工作原理

**1. 两个阶段**：

- **增长阶段**：事务只能获取锁，不能释放任何锁

- **缩减阶段**：事务只能释放锁，不能获取新锁

**2. 严格特性**

- 所有的锁都必须持有到事务结束（提交或中止）
- 不同于基本2PL，严格2PL不允许在事务过程中提前释放任何锁

**3. 锁类型**：

- **读锁（共享锁）**：多个事务可同时持有同一资源的读锁
- **写锁（排他锁）**：只能由一个事务独占，与其他任何锁都互斥

**4. 锁操作**：

- **获取锁**：事务尝试获取资源锁，如果资源已被锁定且不兼容，则等待
- **检测死锁**：通过等待图识别潜在的死锁情况
- **回滚**：检测到死锁时，选择当前事务回滚，释放其持有的所有锁

## 1.2 Strict 2PL如何保证ACID特性

**1. 原子性（Atomicity）**：

- 事务要么完全提交，要么完全回滚
- 当发生错误（如死锁）时，事务管理器执行回滚，释放所有锁

**2. 一致性（Consistency）**：

- 锁机制确保事务只能以一致的方式修改数据
- 资源级别的锁保护确保数据从一个一致状态转换到另一个一致状态

**3. 隔离性（Isolation）**：

- 严格2PL提供可串行化隔离级别
- 事务间的干扰被最小化，每个事务看到的是数据库的一致快照
- 防止了脏读、不可重复读和幻读问题

**4. 持久性（Durability）**：

- 事务提交后，锁被释放，更改永久保存
- 结合日志机制确保即使系统崩溃也能恢复到正确状态

## 1.3 严格两阶段锁定协议 vs 普通两阶段锁定协议：关键区别

**1. 主要区别**:

- **普通2PL**: 锁可以在事务执行过程中释放，只要遵循"先获取后释放"的原则
- **严格2PL**: 所有锁必须持有到事务结束(提交或中止)后才能释放

**2. 普通2PL的级联回滚问题**: 当事务A释放了对资源X的锁，事务B获取了X的锁并修改了X，此时如果事务A提交失败需要回滚，会导致：

1. A已修改的数据需要恢复
2. B读取了A未提交(最终会回滚)的数据，产生了依赖
3. B也必须回滚，即使B本身没有错误
4. 如果C依赖B，C也需要回滚，形成级联效应

**3. 严格2PL避免级联回滚的原理**: 严格2PL通过持有所有锁到事务结束，确保其他事务无法读取或修改未提交的数据，因此不会产生依赖关系，避免了级联回滚问题。

**4. 严格2PL获取锁图示**

![strict_2PL](/images/strict_2PL.png)

**5. 普通2PL获取锁图示**

![strict_2PL](/images/2PL.png)

# 2. 核心字段

```go
type TransactionManager struct {
	resourceLockManager *ResourceLockManager       // Maps every resource to it's corresponding mutex
	waitsForGraph       *WaitsForGraph             // Identifies deadlocks through cycle detection
	transactions        map[uuid.UUID]*Transaction // Identifies the Transaction for a particular client
	mtx                 sync.RWMutex               // concurrency control for transactions
}
```

## 2.1 ResourceLockManager

```go
type ResourceLockManager struct {
    locks map[Resource]*sync.RWMutex // 哈希表，键是Resource，值是锁的类型
    mtx   sync.Mutex // 保护 locks map 的并发访问，是互斥锁，只有锁定和非锁定两种状态
}
type Resource struct {
    tableName string  // 数据库表名
    key       int64   // 记录的主键值
}
```

- 功能：维护资源与对应互斥锁的映射关系

- 作用：为每个数据库资源提供读写锁控制，确保并发安全。
- 实现：locks存储的是Resource为key和锁为value的哈希表。mtx 是互斥的，一次只能被一个事务持有，事务必须等待其他食物释放mtx后才能获得ResourceLockManager

- ResourceLockManager示例：

```go
// ResourceLockManager 可能的内容
resourceLockManager = {
    locks: {
        Resource{tableName: "users", key: 1}: &sync.RWMutex{}, // 存储的是一个读写锁对象的指针
        Resource{tableName: "orders", key: 100}: &sync.RWMutex{}, // 具体是读锁还是写锁在Transaction中记录
    },
    mtx: sync.Mutex{state: locked}  // 互斥锁：被当前事务锁定
}
```

- 获取mtx示例：

```go
// 事务1先获得mtx锁，事务2等待
事务1: 获取用户表id=1的写锁
    1.1 获取 mtx 锁           // mtx 被事务1锁定
    1.2 在 locks map 中找到/创建对应的 RWMutex
    1.3 释放 mtx 锁           // mtx 被释放

// 此时事务2才能获取 mtx
事务2: 获取订单表id=100的写锁
    2.1 获取 mtx 锁           // mtx 被事务2锁定
    2.2 在 locks map 中找到/创建对应的 RWMutex
    2.3 释放 mtx 锁           // mtx 被释放
```

## 2.2 WaitsForGraph

```go
// 等待图，用于检测事务间的死锁
type WaitsForGraph struct {
    edges []Edge           // 存储所有的等待边（事务间的等待关系）
    mtx   sync.RWMutex    // 保护 edges 的并发访问
}

type Edge struct {
    from *Transaction    // 等待资源的事务
    to   *Transaction    // 持有资源的事务
}
```

- 功能：记录事务等待关系，用于死锁检测

- 作用：通过检测图中是否存在环来识别死锁

- 实现：edges 是一个 Edge 结构体的切片，每个 Edge 表示一个事务等待另一个事务的关系（类似于动态数组，Java中的List），用于记录事务之间的等待关系

- WaitsForGraph示例：

```go
waitsForGraph = {
    edges: [
        Edge{from: transaction1, to: transaction2},  // 事务1等待事务2
        Edge{from: transaction2, to: transaction3},  // 事务2等待事务3
        Edge{from: transaction3, to: transaction1},  // 事务3等待事务1，形成环，表示死锁
    ],
    mtx: sync.RWMutex{} 
}
```

- 死锁检测示例：

```go
// 1. 事务A请求资源X（被事务B持有）
AddEdge(事务A, 事务B)    // 添加等待边：A -> B

// 2. 事务B请求资源Y（被事务A持有）
AddEdge(事务B, 事务A)    // 添加等待边：B -> A
                        // 此时形成环 A -> B -> A
                        // DetectCycle() 返回 true，表示检测到死锁
```

## 2.3 transactions

```go
// 事务哈希表
transactions map[uuid.UUID]*Transaction

type Transaction struct {
    clientId        uuid.UUID                  // 客户端ID
    lockedResources map[Resource]LockType      // 当前事务持有的资源锁
    mtx             sync.RWMutex               // 保护事务内部状态
}
```

- 功能：维护所有活跃事务
- 作用：跟踪每个客户端当前正在执行的事务
- 实现：使用客户端ID作为key，事务对象作为value的哈希表

- transactions示例：

```go
transactions = {
    "client-uuid-1": &Transaction{
        clientId: "client-uuid-1",
        lockedResources: {
            Resource{"users", 1}: W_LOCK,     // 持有users表id=1的写锁，这里才记录具体锁类型
            Resource{"orders", 100}: R_LOCK,  // 持有orders表id=100的读锁
        }
    },
    "client-uuid-2": &Transaction{
        clientId: "client-uuid-2",
        lockedResources: {
            Resource{"products", 50}: W_LOCK  // 持有products表id=50的写锁
        }
    }
}
```

- 事务操作示例

```go
// 开始新事务
Begin("client-uuid-1"):
    1.1 检查客户端是否已有活跃事务
    1.2 创建新的Transaction对象
    1.3 添加到transactions映射表

// 提交事务
Commit("client-uuid-1"):
    1.1 获取客户端的事务Transaction对象
    1.2 释放所有持有的资源锁
    1.3 从transactions哈希表中删除事务
```

## 2.4 sync.RWMutex

- 功能：读写锁专门保护上面的 transactions map 的并发访问
- 注意：**每个哈希表都有自己专门的锁来保护并发访问，这是 Go 中常见的并发安全设计模式。**之前的lockedResources哈希表、edges哈希表、locks哈希表都用这种模式控制并发

# 3. 核心函数

## 3.1 Begin

### A. 参数介绍

```go
func (tm *TransactionManager) Begin(clientId uuid.UUID) error
```

- 参数：clientId - 客户端的唯一标识符
- 返回：error - 如果已存在事务则返回错误，否则返回 nil
- 目的：为指定客户端创建并启动一个新的事务

### B. 完整流程

**1. 并发控制**

- 先对transactions事务哈希表添加写锁
- 使用defer确保锁一定释放

**2. 检查事务存在性**

- 根据客户端id查找有没有存在的事务。注意一个客户端同时只有一个活跃事务。

**3. 创建新事务**

- 如果当前客户端id没有事务存在，创建新的事务键值对放入transactions map中，键为客户端id，值为初始化的Transaction结构体。

**4. 返回值**

- 如果事务存在返回`"transaction already began"`
- 如果创建成功返回nil

## 3.2 Lock

```go
func (tm *TransactionManager) Lock(clientId uuid.UUID, table database.Index, resourceKey int64, lType LockType) error
```

### A. 参数介绍

- 参数：
  - clientId - 客户端的唯一标识符
  - table - 数据库索引
  - resourceKey - 要锁定的数据库索引的key
  - lType - 锁定类型

- 返回 err - 如果事务未找到、事务持有资源升级锁、检测到死锁、获取资源锁失败返回错误，否则返回nil
- 目的：为指定的数据库索引的键上读锁或者写锁。

### B. 完整流程

**1. 获取事务**

- 先对transactions事务哈希表添加读锁，之后如果找不到事务、检测到事务要求升级锁、检测到死锁都会释放该读锁。

- 根据客户端id获取事务，如果找不到返回错误

- 从transactions哈希表中找到当前客户端的事务，创建新的事务对象`t`

  `t, found := tm.GetTransaction(clientId)`

**2. 构造资源标识**

- 根据数据库table和key构建Resource结构体

**3. 检查是否持有锁**

- 为当前事务`t`加读锁，该步骤检查完毕后解锁

- 如果当前客户端已经持有读锁，根据strict 2PL的约束，不可以在执行事务时升级锁
- 如果当前客户端已经持有了相同类型的锁，则无需重复加锁，直接返回nil

**4. 冲突事务检测**

- 调用`tm.conflictingTransactions(resource, lType)`，返回与指定资源存在冲突的所有事务，冲突的情况有：
  - 某个事务持有该资源的写锁
  - 某个事务持有该资源的读锁，而当前请求的是写锁

- 遍历所有冲突的事务，调用`tm.waitsForGraph.AddEdge(t, conflictingTxn)`向waitsForGraph的edges切片添加等待边，waitsForGraph示意：

```go
waitsForGraph = {
    edges: [
        Edge{from: transaction1, to: transaction2},
        Edge{from: transaction2, to: transaction3},
        Edge{from: transaction3, to: transaction1}  // 形成环
    ]
}
    T1[事务1] --> T2[事务2]
    T2[事务2] --> T3[事务3]
    T3[事务3] --> T1[事务1]
```

- 在添加等待边时使用了 `defer tm.waitsForGraph.RemoveEdge(t, conflictingTxn)`，确保了后续操作是否成功，等待边都会被移除。

**5. 死锁检测**

- 调用`tm.waitsForGraph.DetectCycle()`
  - 使用DFS对存在等待环的事务进行死锁检测。
- 一旦发现当前事务与其他事务形成等待环，调用`tm.Rollback(clientId)`回滚当前事务，释放当前事务的所有锁，并且从事务管理器中移除当前事务。

**6. 获取资源锁**

- 释放transactions事务哈希表的读锁，此时已经完成对transactions事务哈希表读取的访问。
- 调用`tm.resourceLockManager.Lock(resource, lType)`对资源进行上锁。
  - 首先用互斥锁锁定locks map，保证线程安全。在locks map中查找资源对应的锁，如果不存在则创建新的读写锁。最后释放locks map
  - 对要访问的资源加读锁或写锁：
    - 对于添加读锁（R_LOCK）：如果存在写锁会等待
    - 对于添加写锁（W_LOCK）：如果存在任何锁都会等待

**7. 更新事务状态**

- 对当前事务进行lockedResources锁定资源的哈希表进行上锁，使用defer在之后解锁
- 在当前事务`t.lockedResources`哈希表中更新锁定的资源以及锁的类型

**8. 返回值**

- 事务未找到：`"transaction not found"`
- 试图升级锁：`"cannot upgrade from read lock to write lock in the middle of transaction"`
- 检测到死锁：`"deadlock detected"`
- 获取资源锁失败：来自 `resourceLockManager.Lock()` 的错误

## 3.3 Unlock

```go
func (tm *TransactionManager) Unlock(clientId uuid.UUID, table database.Index, resourceKey int64, lType LockType)
```

### **A. 参数介绍**

- 参数：
  - clientId - 客户端的唯一标识符
  - table - 数据库索引
  - resourceKey - 要解锁的数据库索引的key
  - lType - 解锁类型
- 返回 err - 如果事务未找到、事务未持有该锁、解锁类型与持有的锁类型不匹配、释放资源锁失败返回错误，否则返回nil
- 目的：释放指定数据库索引键上的读锁或者写锁
- 说明：解锁过程中不会出现阻塞，因为解锁本身是非阻塞的，不需要任何等待条件。

### **B. 完整流程** 

**1. 获取事务**

- 先对transactions事务哈希表添加读锁，获取完事务后释放读锁

- 根据客户端id获取事务，如果找不到返回错误

- 从transactions中找到当前客户端的事务，创建新的事务对象 `t`

  `t, found := tm.GetTransaction(clientId)`

**2. 构造资源标识**

- 根据数据库table和key构建Resource结构体 `r`

**3. 检查事务是否持有该资源的锁**

- 对当前事务`t`加写锁，用defer确保后续解锁

- 在事务对象 `t` 的lockedResources表中查找是否持有该资源的锁，如果不存在返回报错

**4. 检查锁类型是否匹配**

- 如果要求释放的锁和资源被上的锁类型不匹配，则无法完成解锁
  - 例如资源本身是被事务的写锁占有，此时要求释放的是读锁则无法完成该操作

**5. 删除锁和释放资源管理器中的锁**

- 调用`delete(t.lockedResources, resource)`，删除事务lockedResources哈希表中的资源和相对应的锁
- 调用`tm.resourceLockManager.Unlock(r, lType)`
  - 首先用互斥锁锁定locks map，保证线程安全。
  - 如果资源不存在，返回错误
  - 根据锁类型调用RUnlock()或Unlock()释放对应类型的锁

**6. 返回值**

- 事务未找到：`"transaction not found"`
- 事务未持有该锁：`"trying to unlock a resource that was not locked"`
- 锁类型不匹配：`"incorrect unlock type"`
- 释放资源锁失败：来自 `resourceLockManager.Unlock()` 的错误

## 3.4 Commit

```go
func (tm *TransactionManager) Commit(clientId uuid.UUID)
```

### **A. 参数介绍**

- 参数：
  - clientId - 客户端的唯一标识符
- 返回 err - 如果事务未找到、释放资源锁失败返回错误，否则返回nil
- 目的：提交事务，释放事务持有的所有锁，并从事务管理器中移除该事务

### B. 完整流程

**1. 获取事务**

- 对transactions事务哈希表添加写锁，使用defer确保后续解锁
- 根据客户端id获取事务，如果找不到返回错误
- 从transactions中找到当前客户端的事务对象 `t`

**2. 释放资源管理器中的锁**

- 调用`tm.resourceLockManager.Unlock(r, lType)`
  - 首先用互斥锁锁定locks map，保证线程安全
  - 如果资源不存在，返回错误
  - 根据锁类型调用RUnlock()或Unlock()释放对应类型的锁

**3. 删除事务**

- 从事务管理器的 transactions 哈希表中删除该事务 `delete(tm.transactions, clientId)`

- 由于已经transactions哈希表添加了写锁，所以这个delete操作是线程安全的

**4. 返回值**

- 事务未找到：`"no transaction running for specified client"`

- 释放资源锁失败：来自 `resourceLockManager.Unlock()` 的错误

- 提交成功：返回 nil

## 3.5 Rollback

### **A. 参数介绍**

- 参数：
  - clientId - 客户端的唯一标识符
- 返回 err - 如果事务未找到、释放资源锁失败返回错误，否则返回nil
- 目的：回滚事务，释放事务持有的所有锁，并从事务管理器中移除该事务（与Commit类似，但语义上表示事务失败）

### **B. 完整流程**

**1. 获取事务**

- 对transactions事务哈希表添加写锁，使用defer确保后续解锁
- 根据客户端id获取事务，如果找不到返回错误
- 从transactions中找到当前客户端的事务对象 `t`

**2.释放资源管理器中的锁**

- 对事务对象 `t` 加读锁，以安全访问 lockedResources，这一步结束后释放读锁
- 遍历事务的 lockedResources 哈希表，释放每个资源的锁：
  - 对每个资源调用 `tm.resourceLockManager.Unlock(r, lType)`，同`Commit`第二步
  - 如果释放某个资源锁失败，立即返回错误

**3. 移除事务**

- 从事务管理器的 transactions 哈希表中删除该事务 `delete(tm.transactions, clientId)`
- 由于已经持有 transactions 的写锁，这个操作是线程安全的

**4. 返回值**

- 事务未找到：`"no transaction running for specified client"`
- 释放资源锁失败：来自 `resourceLockManager.Unlock()` 的错误
- 回滚成功：返回 nil

# 4. 测试

## 4.1 测试框架

```go
测试代码                     handleTransactionThread
   |                                |
   |-- 创建通道 ch1 ---------------> |
   |                                |
   |-- sendWithDelay             等待命令 (<-ch)
   |     |                           |
   |     |-延迟                      |
   |     |-发送命令------------->  执行命令
   |                                 |
   |                                 |
   |                                 |
   |-- sendWithDelay ------------->等待下一个命令
```

## 4.2 基础功能测试

- `testTransactionBasic`: 测试基本的写锁获取
- `testTransactionWriteUnlock`: 测试写锁的加锁解锁
- `testTransactionReadUnlock`: 测试读锁的加锁解锁
- `testTransactionCommitsReleaseLocks`: 测试提交时释放所有锁

## 4.3 锁类型和兼容性测试

- `testTransactionReadLockNoCycle`: 多个读锁共存测试
- `testTransactionDontUpgradeLocks`: 禁止读锁升级为写锁
- `testTransactionDontDowngradeLocks`: 锁降级场景测试
- `testTransactionLockIdempotency`: 重复加锁的幂等性测试

## 4.4 错误处理测试

- `testTransactionWrongUnlockLockType`: 错误的解锁类型处理
- `testTransactionDeadlock`: 死锁检测和处理

## 4.5 并发场景测试

- `testTransactionDAGNoCycle`: 有向无环图并发场景
- `testTransactionDeadlock`: 死锁并发场景

## 4.6 完整事务流程测试

`TestCompleteTransaction`: 测试完整的事务生命周期

- 场景1: 正常执行和提交
- 场景2: 锁升级失败和回滚
- 场景3: 新事务获取已回滚事务的资源

## 4.7 压力测试

`TestStress`: 高并发场景下的系统稳定性测试

- 目的：测试系统在高负载下的表现和稳定性
- 实现：
  - 启动100个并发协程，每个执行1000次事务操作
  - 资源数量为100个
  - 每个事务随机执行多次读写操作
  - 添加随机延时，模拟真实场景
- 测试点：
  - 并发事务处理能力
  - 死锁检测和处理
  - 错误恢复机制
  - 系统稳定性

## 4.8 资源使用测试

`TestResourceUsage`: 系统资源消耗监控测试

- 目的：监控系统在大量操作下的资源使用情况
- 实现：
  - 在一个客户端串行执行10000次的事务操作
  - 每个事务执行完整的加锁-解锁-提交流程
- 测试点：
  - 内存使用情况（仅产生16.5KB内存开销）
  - 资源释放正确性（每个事务结束后都正确释放资源）
  - 内存泄漏检测（多次GC后内存增长稳定）

- 性能指标：

  - 平均每个事务产生约1.65字节的内存开销，系统的内存管理非常高效，对比MySQL每个事务需要几KB到几十KB

  - 串行处理下每秒约3800个事务操作

  - 内存使用随事务数量线性增长，没有泄漏

## 4.9 REPL命令执行流程

![transactionREPL](./images/transactionREPL.jpg)

# 5. 存在的问题

## 5.1 压力测试中的死锁问题

### A. 问题表现

在压力测试中，当多个事务并发执行时，出现了资源锁定但无法释放的情况。具体表现为：

- 多个事务同时请求同一资源的写锁
- 第一个事务获取锁后，其他事务一直等待
- 实际调用了解锁操作，未看到任何解锁操作的日志，系统最终陷入死锁状态

### B. 问题分析

- 锁请求堆积

  - 大量事务同时请求同一资源

  - 后续事务持续等待，形成请求队列

  - 可能由随机数生成的集中性导致

- 解锁机制失效

  - 资源锁获取成功后，释放操作未能正确执行

  - 死锁检测算法（dfs函数）实现有缺陷，无法正确识别环形等待

- 解决难点：

  - 尝试记录操作日志，但日志打印的顺序可能不能准确反映代码执行的实际顺序，因为：

    - 日志打印延迟：日志记录本身有开销，特别是在写入终端或文件时

    - 线程调度：多线程环境中的线程调度是不确定的

    - 并发操作：多个操作可能在非常接近的时间点同时发生

```
=== RUN   TestStress
=== PAUSE TestStress
=== CONT  TestStress
=== 事务[c69bf968]请求获取资源[3002996722.db.1]的写锁 ===
✅ 找到事务[c69bf968]
📦 构造资源标识: 3002996722.db.1
✅ 事务[c69bf968]成功获取资源[3002996722.db.1]的写锁
✅ 更新事务[c69bf968]的锁记录
=== 事务[c69bf968]请求释放资源[3002996722.db.1]的写锁 ===
=== 事务[fc1e9f98]请求获取资源[3002996722.db.1]的写锁 ===
=== 事务[c312fc49]请求获取资源[3002996722.db.1]的写锁 ===
=== 事务[cfdfa504]请求获取资源[3002996722.db.1]的读锁 ===
```

### C. 修改方案

- 使用拓扑排序代替dfs进行死锁检测，拓扑排序避免了DFS的递归调用和回溯，减少栈空间使用

- 修改`tm.resourceLockManager.Unlock(r, lType)`的逻辑
