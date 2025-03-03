# RecoveryManager

# 1. 重要概念

## 1.1 WAL和ARIES恢复协议

- WAL（Write-Ahead Logging）和ARIES（Algorithm for Recovery and Isolation Exploiting Semantics）是数据库恢复系统的核心概念。本项目实现了简化版的ARIES协议，包含以下三个阶段：

### Phase #1 – 分析阶段 (Analysis)

- 从最近的检查点checkpoint（类似于ARIES中的MasterRecord）开始，向前扫描日志
- 识别崩溃时处于活跃状态的事务
- 确定哪些表需要恢复，哪些事务需要撤销
- 构建事务状态表，追踪每个事务的操作记录

### Phase #2 – 重做阶段 (Redo)

- 在当前实现中，仅对已提交的事务执行重做，而不是所有操作
- 确保所有持久化的操作都被正确应用到数据库状态

### Phase #3 – 撤销阶段 (Undo)

- **逆序撤销所有未提交事务的操作直到崩溃时活跃事务的最早日志记录**
- 为每个撤销操作写入相应的日志记录
- 最后标记所有未提交事务为已回滚

## 1.2 ARIES恢复图示解释

![Aries](./images/Aries.png)

- 图中展示了ARIES恢复协议的关键元素：

  - **WAL日志**: 垂直的记录列表，按时间顺序从上到下排列

  - **TIME箭头**: 表示时间流向，从上到下

  - **Start of last checkpoint**: 最近的检查点位置，是恢复重做阶段的起点

  - **Oldest log record of txn active at crash**: 崩溃时仍活跃（未提交）事务的最早日志记录
    - 这是ARIES中的关键概念，它可能早于最近的检查点
    - 表示撤销阶段可能需要回滚到多远的历史
    - 例如：如果一个长时间运行的事务在检查点前就开始了，尽管其数据修改已经在检查点时刷新到磁盘，但由于事务未提交，恢复时仍需要**完全撤销该事务的所有操作**

## 1.3 WAL原则

- 任何修改数据库状态的操作**必须先写入日志，再执行实际操作**
- 实际代码示例：

```go
func HandleInsert(db *database.Database, tm *concurrency.TransactionManager, rm *RecoveryManager, payload string, clientId uuid.UUID) (err error) {
    // ...参数解析省略...
    // 先写日志
    err = rm.Edit(clientId, table, INSERT_ACTION, int64(key), 0, int64(newval))
    if err != nil {
        return err
    }
    
    // 后执行实际插入操作
    err = concurrency.HandleInsert(db, tm, payload, clientId)
    // ...错误处理...
    
    return err
}
```

## 1.4 日志文件结构

- 所有类型的日志（表创建、编辑、事务开始/提交、检查点）都按时间顺序写入**同一个日志文件, 所有表的所有日志都写入同一个表**

- 日志以文本形式存储，每条记录占一行
- 每次写入操作后立即调用`Sync()`确保日志持久化存储

- 日志示例：

  ```
  < create btree table students >
  < 123e4567-e89b-12d3-a456-426614174000 start >
  < 123e4567-e89b-12d3-a456-426614174000, students, INSERT, 10, 0, 100 >
  < 123e4567-e89b-12d3-a456-426614174000, students, UPDATE, 10, 100, 200 >
  < 123e4567-e89b-12d3-a456-426614174000 commit >
  < 456e7890-e89b-12d3-a456-426614174000 start >
  < 456e7890-e89b-12d3-a456-426614174000, students, INSERT, 20, 0, 300 >
  < 123e4567-e89b-12d3-a456-426614174000, 456e7890-e89b-12d3-a456-426614174000 checkpoint >
  < 789e0123-e89b-12d3-a456-426614174000 start >
  < 789e0123-e89b-12d3-a456-426614174000, students, INSERT, 30, 0, 400 >
  < 456e7890-e89b-12d3-a456-426614174000, students, DELETE, 20, 300, 0 >
  < 456e7890-e89b-12d3-a456-426614174000 commit >
  < 789e0123-e89b-12d3-a456-426614174000, students, UPDATE, 30, 400, 500 >
  ```

- 在恢复时，系统会从检查点checkpoint开始分析，识别活跃事务（此例中为事务789e0123），然后重做（此例中为事务456e7890）已提交事务的操作，最后撤销未提交（活跃）事务的操作。

# 2. 核心字段

```go
type RecoveryManager struct {
	db *database.Database              // 该恢复管理器负责的底层数据库
	tm *concurrency.TransactionManager // 用于该数据库的事务管理器

	// 跟踪所有未提交事务的操作
	// 将每个客户端/事务ID映射到日志栈
	txStack map[uuid.UUID][]editLog

	logFile *os.File   // 存储预写日志(WAL)的日志文件
	mtx     sync.Mutex // 用于保证该结构体可以被安全地并发使用的互斥锁
}
```

## 2.1 db

```go
db *database.Database
```

- 功能：指向底层数据库实例的指针

- 作用：允许RecoveryManager访问数据库的表、索引和其他数据结构

- 实现：在初始化RecoveryManager时传入，用于数据恢复和数据操作

- 示例：

```go
// 使用db字段访问数据库
tables := rm.db.GetTables()
table, err := rm.db.GetTable("users")
```

## 2.2 tm

```go
tm *concurrency.TransactionManager
```

- 功能：指向事务管理器的指针

- 作用：负责管理数据库事务，包括锁的获取和释放

- 实现：在初始化RecoveryManager时传入，用于事务操作

- 示例：

```go
// 使用tm字段管理事务
err := rm.tm.Begin(clientId)
err := rm.tm.Lock(clientId, table, key, concurrency.W_LOCK)
err := rm.tm.Commit(clientId)
```

## 2.3 txStack 

```go
txStack map[uuid.UUID][]editLog

type editLog struct {
    id        uuid.UUID // 事务ID
    tablename string    // 表名
    action    action    // 操作类型(INSERT/UPDATE/DELETE)
    key       int64     // 操作的键
    oldval    int64     // 操作前的值
    newval    int64     // 操作后的值
}
```

- 功能：维护所有未提交（活跃）事务的操作日志哈希表

- 作用：用于回滚操作和崩溃恢复，记录每个事务的所有修改操作

- 实现：使用事务ID作为键，editLog数组作为值的哈希表

- 结构示例：

```go
txStack = {
    "uuid1": [
        editLog{id: "uuid1", tablename: "users", action: INSERT_ACTION, key: 1, oldval: 0, newval: 100},
        editLog{id: "uuid1", tablename: "users", action: UPDATE_ACTION, key: 2, oldval: 50, newval: 150},
    ],
    "uuid2": [
        editLog{id: "uuid2", tablename: "orders", action: DELETE_ACTION, key: 5, oldval: 200, newval: 0},
    ]
}
```

- 操作示例：

```go
// 添加日志到事务栈
rm.txStack[clientId] = append(rm.txStack[clientId], el) // el = editLog

// 事务提交时删除对应日志栈
delete(rm.txStack, clientId)
```

## 2.4 logFile

```go
logFile *os.File
```

- 功能：指向预写日志(WAL)文件的文件指针

- 作用：用于持久化存储所有数据修改操作，确保崩溃恢复

- 实现：在初始化RecoveryManager时打开，使用追加模式写入

- 示例：

```go
// 写入日志到文件
_, err := rm.logFile.WriteString(log.toString())
err = rm.logFile.Sync() // 确保数据刷新到磁盘
```

## 2.5 mtx

- 功能：互斥锁，主要用于保护 txStack map 的并发访问
- 作用：确保多个事务不会同时读写 txStack，避免并发修改导致的数据不一致或程序崩溃
- 实现：标准 Go 互斥锁，用于同步访问

# 3. 核心函数

## 3.1 Table

```go
func (rm *RecoveryManager) Table(tblType string, tblName string) error
```

### A. 参数介绍

- 参数：

  - `tblType string`：表的类型，通常为 "btree" 或 "hash"

  - `tblName string`：表的名称

- 返回：
  - `error`：如果日志写入失败，返回错误；否则返回 nil

- 目的：
  - 将**表创建操作**记录到预写日志(WAL)中，确保在系统崩溃和恢复时能重新创建表结构

- 说明：
  - 表创建日志不关联任何特定事务，因此不会被添加到 txStack 中
  - 此方法应在实际创建表后立即调用，确保表结构能被正确记录

### B. 完整流程

**1. 并发控制**

- 获取 `RecoveryManager`的互斥锁，保护对日志文件的写入日志到磁盘操作的线程安全

- 使用 defer 语句确保函数结束时释放锁，防止死锁

**2. 创建日志对象**

- 创建 tableLog 结构体实例

- 将表类型和表名填入日志对象

**3. 写入日志到磁盘**

- 调用 `flushLog()`方法将日志序列化并写入日志文件，确保日志在实际数据修改前持久化存储（预写日志核心原则）

**4. 返回结果**

- 如果一切正常，返回 nil 表示操作成功

### C. 记录实例

- 执行`rm.Table("btree", "students")`，创建一个B+树类型的"students"表，日志文件会记录

  ```
  < create btree table students >

## 3.2 Start

```go
func (rm *RecoveryManager) Start(clientId uuid.UUID) error
```

### A. 参数介绍

- 参数：
  - `clientId uuid.UUID`：要启动的事务/客户端的唯一标识符
- 返回：
  - `error`：如果日志写入失败，返回错误；否则返回 nil
- 目的：
  - 标记一个事务的开始，并将此事件记录到预写日志(WAL)中
  - 创建一个空的事务操作键值对，用于跟踪该事务的所有修改操作
- 说明：
  - 每个事务在执行任何数据修改前必须先调用此方法
  - 开始日志在恢复过程中用于确定崩溃时哪些事务处于活跃状态

### B. 完整流程

**1. 并发控制**

- 获取 `RecoveryManager`的互斥锁，保护对共享资源的访问；用 defer 语句确保函数结束释放锁防止死锁

**2. 创建日志对象**

- 创建 startLog 结构体实例
- 将客户端（事务）ID填入日志对象

**3. 写入日志到磁盘**

- 调用 `flushLog()`方法将日志序列化并写入日志文件落盘

**4. 初始化事务状态**

- 在 txStack 映射中为该事务创建一个空的编辑日志数组条目
- 此键值对将用于跟踪事务中的所有操作，以便在需要时进行回滚

**5. 返回结果**

- 如果操作成功，返回 nil
- 如果有错误发生，返回带上下文信息的错误

### C. 记录实例

- 执行`rm.Start(uuid.MustParse("123e4567-e89b-12d3-a456-426614174000"))`，启动一个新事务，日志文件会记录

  ```
  < 123e4567-e89b-12d3-a456-426614174000 start >
  ```

- 注：这里假设UUID值`123e4567-e89b-12d3-a456-426614174000`代表事务ID(tx1)

## 3.3 Edit

```go
func (rm *RecoveryManager) Edit(clientId uuid.UUID, table database.Index, action action, key int64, oldval int64, newval int64) error
```

### A. 参数介绍

- 参数：
  - `clientId uuid.UUID`：客户端/事务的唯一标识符
  - `table database.Index`：要修改的表对象
  - `action action`：操作类型（INSERT_ACTION, UPDATE_ACTION, DELETE_ACTION）
  - `key int64`：要修改的记录键值
  - `oldval int64`：修改前的值（对于插入操作通常为0）
  - `newval int64`：修改后的值（对于删除操作通常为0）
- 返回：
  - `error`：如果日志写入失败，返回错误；否则返回 nil
- 目的：
  - 将数据修改操作记录到预写日志(WAL)中，确保在系统崩溃时能够恢复或回滚操作
- 说明：
  - 遵循WAL原则，必须先记录日志，再执行实际修改操作
  - 每个编辑日志都与特定事务关联，并存储在txStack中用于可能的回滚操作

### B. 完整流程

**1. 并发控制**

- 获取 `RecoveryManager`的互斥锁，保护对共享资源的访问；用 defer 语句确保函数结束释放锁防止死锁

**2. 创建日志对象**

- 创建 editLog 结构体实例，填充事务ID、表名、操作类型、键值以及新旧值信息

**3. 写入日志到磁盘**

- 调用 `flushLog()`方法将日志序列化并立即写入日志文件落盘

**4. 更新事务状态**

- 检查txStack中是否已存在该事务的日志记录，如不存在则初始化
- 将当前编辑日志添加到对应事务的日志栈中，用于潜在的事务回滚

**5. 返回结果**

- 如果操作成功，返回 nil
- 如有错误发生，返回带上下文信息的错误

### C. 记录实例

- 执行插入操作：`rm.Edit(tx1, studentsTable, INSERT_ACTION, 10, 0, 100)`，日志文件会记录

  ```
  < 123e4567-e89b-12d3-a456-426614174000, students, INSERT, 10, 0, 100 >
  ```

- 执行更新操作：`rm.Edit(tx1, studentsTable, UPDATE_ACTION, 10, 100, 200)`，日志文件会记录

  ```
  < 123e4567-e89b-12d3-a456-426614174000, students, UPDATE, 10, 100, 200 >
  ```

- 执行删除操作：`rm.Edit(tx1, studentsTable, DELETE_ACTION, 10, 200, 0)`，日志文件会记录

  ```
  < 123e4567-e89b-12d3-a456-426614174000, students, DELETE, 10, 200, 0 >
  ```

- 注：这里假设UUID值`123e4567-e89b-12d3-a456-426614174000`代表事务ID(tx1)

## 3.4 Commit

```go
func (rm *RecoveryManager) Commit(clientId uuid.UUID) error
```

### A. 参数介绍

- 参数：
  - `clientId uuid.UUID`：要提交的事务/客户端的唯一标识符
- 返回：
  - `error`：如果日志写入失败，返回错误；否则返回 nil
- 目的：
  - 标记一个事务的成功完成，并将此事件记录到预写日志(WAL)中
  - 从事务跟踪系统中移除该事务的操作记录
- 说明：
  - 每个事务完成所有操作后调用此方法确认事务已成功完成
  - 提交日志在恢复过程中用于确定哪些事务在崩溃前已成功完成
  - 提交操作不会立即将所有数据写入磁盘，但确保所有操作将在恢复时重做

### B. 完整流程

**1. 并发控制**

- 获取 `RecoveryManager`的互斥锁，保护对共享资源的访问；用 defer 语句确保函数结束释放锁防止死锁

**2. 创建日志对象**

- 创建 commitLog 结构体实例
- 将客户端（事务ID）填入日志对象

**3. 写入日志到磁盘**

- 调用 `flushLog()`方法将日志序列化并写入日志文件落盘

**4. 清理事务状态**

- 从 txStack 哈希表中删除该事务的条目
- 这表明该事务已成功完成，其操作不需要在回滚时撤销

**5. 返回结果**

- 如果操作成功，返回 nil
- 如果有错误发生，返回带上下文信息的错误

### C. 记录实例

- 执行`rm.Commit(uuid.MustParse("123e4567-e89b-12d3-a456-426614174000"))`，提交一个事务，日志文件会记录

  ```
  < 123e4567-e89b-12d3-a456-426614174000 commit >
  ```

## 3.5 Checkpoint

### A. 参数介绍

- 参数：
  - 无参数
- 返回：
  - `error`：如果日志写入或页面刷新失败，返回错误；否则返回 nil
- 目的：
  - 创建数据库的一致性快照，将内存中的所有修改刷新到磁盘
  - 记录当前所有活跃（未提交）事务，为恢复提供起点
  - 备份数据库状态用于潜在的恢复
- 说明：
  - 减少系统崩溃后的恢复时间，只需重放检查点之后的日志
  - 与传统WAL实现不同，此实现不清理或截断日志文件

### B. 完整流程

**1. 并发控制**

- 获取 `RecoveryManager`的互斥锁，保护对共享资源的访问；用 defer 语句确保函数结束释放锁防止死锁

**2. 获取所有表**

- 调用 `db.GetTables()` 获取数据库中所有的表

**3. 刷新所有数据页到磁盘**

- 对每个表，获取其页面管理器(Pager)
- 锁定所有页面，进行刷新操作，然后解锁
- 确保所有内存中的变更都持久化到磁盘

**4. 收集活跃事务**

- 创建空的活跃事务ID列表
- 遍历 txStack 哈希表，收集所有未提交事务的ID

**5. 创建并写入检查点日志**

- 创建包含所有活跃事务ID的检查点日志对象
- 调用 `flushLog()` 确保日志被持久化到磁盘

**6. 备份数据库**

- 调用 `delta()` 方法创建数据库当前状态的备份
  - `delta()`函数实际创建了整个数据库文件的物理副本
- 复制所有数据文件到恢复目录，以便未来恢复使用

**7. 返回结果**

- 如果操作成功，返回 nil
- 如果有错误发生，返回带上下文信息的错误

### C. 记录实例

- 执行`rm.Checkpoint()`，当事务"123e4567"和"456e7890"活跃时，日志文件会记录

  ```
  < 123e4567-e89b-12d3-a456-426614174000, 456e7890-e89b-12d3-a456-426614174000 checkpoint >
  ```

- 如果没有活跃事务，则记录

  ```
  < checkpoint >
  ```

  