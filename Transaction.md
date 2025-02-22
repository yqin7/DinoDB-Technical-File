# 1. 核心字段

```go
type TransactionManager struct {
	resourceLockManager *ResourceLockManager       // Maps every resource to it's corresponding mutex
	waitsForGraph       *WaitsForGraph             // Identifies deadlocks through cycle detection
	transactions        map[uuid.UUID]*Transaction // Identifies the Transaction for a particular client
	mtx                 sync.RWMutex               // concurrency control for transactions
}
```

## 1.1 ResourceLockManager

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

## 1.2 WaitsForGraph

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

## 1.3 transactions

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

## 1.4 sync.RWMutex

- 功能：专门保护上面的 transactions map 的并发访问
- 注意：**每个哈希表都有自己专门的锁来保护并发访问，这是 Go 中常见的并发安全设计模式。**之前的lockedResources哈希表、edges哈希表、locks哈希表都用这种模式控制并发

# 2. 核心函数

## 2.1 Begin

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

## 2.2 Lock

```go
func (tm *TransactionManager) Lock(clientId uuid.UUID, table database.Index, resourceKey int64, lType LockType) error
```

### A. 参数介绍

- 参数
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

- 从transactions中找到当前客户端的事务，创建新的事务对象t

  `t, found := tm.GetTransaction(clientId)`

**2. 构造资源标识**

- 根据数据库table和key构建Resource结构体

**3. 检查是否持有锁**

- 为当前事务t加读锁，该步骤检查完毕后解锁

- 如果当前客户端已经持有读锁，根据strict 2PL的约束，不可以在执行事务时升级锁，返回错误`"cannot upgrade from read lock to write lock in the middle of transaction"`
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

- 调用`tm.waitsForGraph.DetectCycle()`，使用DFS对存在等待环的事务进行死锁检测。
- 一旦发现当前事务与其他事务形成等待环，调用`tm.Rollback(clientId)`回滚当前事务，释放当前事务的所有锁，并且从事务管理器中移除当前事务。

**6. 获取资源锁**

- 释放transactions事务哈希表的读锁，此时已经完成对transactions事务哈希表读取的访问。
- 调用`tm.resourceLockManager.Lock(resource, lType)`对资源进行上锁。
  - 首先用写锁锁定locks map，保证线程安全。在locks map中查找资源对应的锁，如果不存在则创建新的读写锁。最后释放locks map
  - 对要访问的资源加读锁或写锁：
    - 对于添加读锁（R_LOCK）：如果存在写锁会等待
    - 对于添加写锁（W_LOCK）：如果存在任何锁都会等待

**7. 更新事务状态**

- 对当前事务进行lockedResources锁定资源的哈希表进行上锁，使用defer在之后解锁
- 在lockedResources哈希表中更新锁定的资源以及锁的类型

**8. 返回值**

- 事务未找到：`"transaction not found"`
- 试图升级锁：`"cannot upgrade from read lock to write lock in the middle of transaction"`
- 检测到死锁：`"deadlock detected"`
- 获取资源锁失败：来自 `resourceLockManager.Lock()` 的错误
