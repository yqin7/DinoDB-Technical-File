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
    edges    map[*Transaction][]*Transaction  // 记录事务间的等待关系
    mtx      sync.RWMutex                    // 保护等待图的并发访问
}
```

- 功能：记录事务等待关系，用于死锁检测

- 作用：通过检测图中是否存在环来识别死锁

- 实现：edges存储的是key为Transaction和value为Transaction指针的切片（类似于动态数组，Java中的List），用于记录事务之间的等待关系，

- WaitsForGraph示例：

```go
waitsForGraph = {
    edges: {
        Transaction1: [Transaction2, Transaction3],  // 事务1等待事务2和事务3
        Transaction2: [Transaction3],               // 事务2等待事务3
        Transaction3: [Transaction1],               // 事务3等待事务1，形成死锁
    },
    mtx: sync.RWMutex{} 
}
```

- 死锁检测示例：

```go
// 事务1请求资源A（假设资源A被事务2持有）
事务1: 请求资源A
    1.1 添加等待边：事务1 -> 事务2
    1.2 检测是否形成环

// 事务2请求资源B（假设资源B被事务1持有）
事务2: 请求资源B
    2.1 添加等待边：事务2 -> 事务1
    2.2 检测到环：事务1 -> 事务2 -> 事务1
    2.3 发现死锁，回滚事务2
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