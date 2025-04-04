# Hash

# 1. 重要概念

## 1.1 可扩展哈希(Extendible Hashing)

- 可扩展哈希是一种动态哈希技术，能够在不需要完全重建的情况下**根据数据量增长而扩展**
- 本项目实现的可扩展哈希使用两个关键概念：**目录（Directory）**和**桶（Bucket）**
- 特点：
  - **渐进式扩展**：只有发生溢出的桶才会分裂，不是整个表重建
  - **动态平衡**：负载均衡自动调整，减少碰撞概率
  - **高效查找**：无论数据量多大，查找操作都是常数时间复杂度O(1)

## 1.2 全局深度与局部深度

- **全局深度 (Global Depth)**：
  - 定义：整个哈希表的目录中使用哈希值的位数
  - 作用：决定了目录的大小（2^全局深度），随表扩展而增加
  - 存储位置：哈希表元数据中
  ```go
  type HashTable struct {
      globalDepth int64    // 哈希表的全局深度
      buckets     []int64  // 桶的页号数组
      // ...其他字段
  }
  ```

- **局部深度 (Local Depth)**：
  - 定义：单个桶使用哈希值的位数
  - 作用：指示一个桶分裂了多少次
  - 存储位置：每个桶的页面头部中
  ```go
  type HashBucket struct {
      localDepth int64     // 桶的局部深度
      numKeys    int64     // 桶中的键数量
      // ...其他字段
  }
  ```

- **深度关系**：
  - 任何桶的局部深度 ≤ 哈希表全局深度
  - 当桶的局部深度达到全局深度且需要分裂时，必须先增加全局深度

## 1.3 目录项、哈希值和桶的关系

- **目录项(Directory Entry)**是哈希表索引结构的基本单元，存储在`buckets`数组中
- **目录项内容**：存储物理桶的页号，而非桶对象的直接引用
- **映射过程**：键 → 哈希值 → 目录索引 → 页号 → 物理桶

```
              哈希表
         +----------------+
         |  globalDepth=2 |
         +----------------+      目录项(二进制)     物理桶
         |    buckets     | ---> [00] ---> 页号0 ---> 桶0 (localDepth=1) 
         |                |      [01] ---> 页号1 ---> 桶1 (localDepth=2)
         |                |      [10] ---> 页号0 ---> 桶0 (与[00]指向同一桶)
         |                |      [11] ---> 页号2 ---> 桶2 (localDepth=2)
         +----------------+
```

- **（！重要）多对一关系（多个目录项指向一个物理bucket）**：
  - 上图中，目录项[00]和[10]指向同一个物理桶(桶0)
  - 原因：局部深度小于全局深度时候，意为该桶只关心它的最低位匹配key，忽略高位。全局深度等于多少，必然有2^全局深度个目录项。
    - 桶0的局部深度=1，只使用哈希值的最低1位进行区分
  - 目录项[01]和[11]分别指向不同桶，因为这些桶的局部深度=2，使用全部2位
- **哈希前缀匹配**：
  - 对于深度为d的桶，所有共享相同d位前缀的哈希值都映射到该桶
  - 例如：当桶的localDepth=1时，哈希值的最低1位相同的键都映射到同一个桶

## 1.4 桶存储结构

```go
const DEPTH_OFFSET int64 = 0
const DEPTH_SIZE int64 = binary.MaxVarintLen64
const NUM_KEYS_OFFSET int64 = DEPTH_OFFSET + DEPTH_SIZE
const NUM_KEYS_SIZE int64 = binary.MaxVarintLen64
const BUCKET_HEADER_SIZE int64 = DEPTH_SIZE + NUM_KEYS_SIZE
const ENTRYSIZE int64 = binary.MaxVarintLen64 * 2                        
const MAX_BUCKET_SIZE int64 = (PAGESIZE - BUCKET_HEADER_SIZE) / ENTRYSIZE
```

- 所有桶都存储在固定大小的页面中，结构如下：

  - **桶头部**：
    - `localDepth` (8字节)：桶的局部深度
    - `numKeys` (8字节)：桶中当前存储的键数量

  - **条目区域**：
    - 每个条目16字节(key 8字节, value 8字节)
    - 根据页面大小，一个桶通常可容纳约250-300个条目

- 页面布局示例（4096字节页面）：
  ```
  +------------------------+
  | localDepth (8字节)     |  binary.MaxVarintLen64 = 10字节
  +------------------------+                                ===>头部等于20字节
  | numKeys (8字节)        |  binary.MaxVarintLen64 = 10字节
  +------------------------+
  | entry 0 (16字节)       |
  | key (8字节) | val (8字节) |
  +------------------------+
  | entry 1 (16字节)       |
  | key (8字节) | val (8字节) |
  +------------------------+
  |        ...            |
  +------------------------+
  | entry 254 (16字节)     |
  | key (8字节) | val (8字节) |
  +------------------------+
  ```

# 2. 核心字段

## 2.1 HashTable

```go
type HashTable struct {
    globalDepth int64        // 哈希表的全局深度
    buckets     []int64      // 桶页号数组，索引（二进制形式）对应哈希表中的查找键
    pager       *pager.Pager // 与哈希表关联的分页器
    rwlock      sync.RWMutex // 哈希表的读写锁
}
```

### A. globalDepth

- 功能：记录整个哈希表的全局深度，即使用哈希值的位数
- 作用：
  - 决定哈希表的目录大小 (2^globalDepth)
  - 限制桶的最大局部深度
  - 指导哈希值的计算和桶的查找
- 示例：
  ```go
  // 计算给定键的哈希值，使用全局深度
  hash := Hasher(key, table.globalDepth)
  ```

### B. buckets

- 功能：存储所有桶页面的页号数组，实际上是哈希表的目录
- 作用：
  - 将哈希值映射到对应的桶
  - 支持多个目录项指向同一个桶（共享前缀的桶）
  - 当表扩展时，数组大小会翻倍
- 示例：
  ```go
  // 根据哈希值获取对应的桶页号
  pagenum := table.buckets[hash]
  bucket, err := table.GetBucketByPN(pagenum)
  ```

### C. pager

- 功能：指向页式存储管理器的指针
- 作用：
  - 管理磁盘和内存间的页面传输
  - 提供页面的创建、获取和释放操作
  - 处理缓存和脏页刷新
- 示例：
  ```go
  // 获取新页面
  newPage, err := pager.GetNewPage()
  // 获取现有页面
  page, err := pager.GetPage(pageNum)
  // 释放页面
  pager.PutPage(page)
  ```

### D. rwlock

- 功能：哈希表级别的读写锁
- 作用：
  - 保护对哈希表结构的并发访问
  - 允许多个读操作并发进行
  - 保证写操作（如分裂）的互斥性
- 示例：
  ```go
  // 获取读锁
  table.RLock()
  // 释放读锁
  table.RUnlock()
  // 获取写锁
  table.WLock()
  // 释放写锁
  table.WUnlock()
  ```

## 2.2 HashBucket

```go
type HashBucket struct {
    localDepth int64       // 桶的局部深度
    numKeys    int64       // 桶中的键/条目数量
    page       *pager.Page // 包含桶数据的页面
}
```

### A. localDepth

- 功能：记录桶的局部深度，表示该桶用于区分元素的哈希前缀长度
- 作用：
  - 指示桶分裂了多少次
  - 控制哪些目录项指向此桶
  - 决定分裂时记录重新分配的方式
- 示例：
  ```go
  // 增加桶的局部深度（分裂时）
  bucket.updateLocalDepth(bucket.localDepth + 1)
  ```

### B. numKeys

- 功能：记录桶中当前存储的键值对数量
- 作用：
  - 跟踪桶的填充程度
  - 确定是否需要分裂（当numKeys >= MAX_BUCKET_SIZE）
  - 帮助遍历桶内所有条目
- 示例：
  ```go
  // 判断插入后是否需要分裂
  split := bucket.numKeys >= MAX_BUCKET_SIZE
  ```

### C. page

- 功能：指向存储桶数据的物理页面
- 作用：
  - 提供对桶数据的直接访问
  - 管理页面锁定和释放
  - 跟踪页面是否被修改（脏页）
- 示例：
  ```go
  // 获取页面号
  pageNum := bucket.page.GetPageNum()
  // 更新页面数据
  bucket.page.Update(data, offset, size)
  ```

## 2.3 HashIndex

```go
type HashIndex struct {
    table *HashTable   // 底层哈希表
    pager *pager.Pager // 支持该索引/哈希表的分页器
}
```

### A. table

- 功能：指向底层哈希表的指针
- 作用：提供对哈希表核心操作的访问

### B. pager

- 功能：指向分页器的指针
- 作用：管理表的磁盘存储和内存缓存

# 3. 核心函数

## 3.1 Insert插入操作

```go
func (table *HashTable) Insert(key int64, value int64) error
```

### A. 参数介绍

- 参数：
  - `key int64`：要插入的键
  - `value int64`：关联的值
- 返回：
  - `error`：如果插入失败，返回错误；否则返回nil
- 目的：
  - 将新的键值对插入到哈希表中，如果需要，触发桶分裂

### B. 完整流程

**1. 获取表级写锁**

- 调用`table.WLock()`获取哈希表的写锁，保证整个插入过程的线程安全
- 使用`defer table.WUnlock()`确保函数结束时释放锁

**2. 计算哈希值**

- 使用全局深度计算键的哈希值：`hash := Hasher(key, table.globalDepth)`
- 哈希函数确保值在目录大小范围内（0到2^globalDepth-1）

**3. 获取目标桶**

- 根据哈希值从目录找到对应桶的页号：`table.buckets[hash]`
- 获取桶并锁定：`bucket, err := table.GetAndLockBucket(hash, WRITE_LOCK)`
- 确保函数结束时释放资源：`defer table.pager.PutPage(bucket.page)`和`defer bucket.WUnlock()`

**4. 执行插入**

- 调用桶的插入方法：`split := bucket.Insert(key, value)`
- 插入方法将条目添加到桶中，并返回是否需要分裂

**5. 处理桶分裂**

- 如果不需要分裂（split为false），直接返回
- 如果需要分裂，调用`table.split(bucket, hash)`处理分裂逻辑

### C. 示例

- 执行`table.Insert(42, 100)`，插入键42，值100
- 假设键42的哈希值为5，则查找页号为`table.buckets[5]`的桶
- 在该桶中插入条目(42, 100)
- 如果插入后桶已满，则触发分裂操作

## 3.2 Split桶分裂

```go
func (table *HashTable) split(bucket *HashBucket, hash int64) error
```

### A. 参数介绍

- 参数：
  - `bucket *HashBucket`：需要分裂的桶
  - `hash int64`：触发分裂的哈希值
- 返回：
  - `error`：如果分裂失败，返回错误；否则返回nil
- 目的：
  - 将一个已满的桶分裂成两个，并重新分配其中的条目

### B. 完整流程

**1. 计算新旧哈希值**

- 计算旧哈希后缀：`oldHash := (hash % powInt(2, bucket.localDepth))`
- 计算新哈希后缀：`newHash := oldHash + powInt(2, bucket.localDepth)`
- 这些后缀值用于确定哪些桶指针需要更新

**2. 检查并扩展表**

- 如果桶的局部深度等于表的全局深度：`bucket.localDepth == table.globalDepth`
- 则调用`table.ExtendTable()`增加表的全局深度并加倍目录大小

**3. 创建新桶**

- 增加原桶的局部深度：`bucket.updateLocalDepth(bucket.localDepth + 1)`
- 创建同样局部深度的新桶：`newBucket, err := newHashBucket(table.pager, bucket.localDepth)`
- 确保函数结束时释放新桶的页面：`defer table.pager.PutPage(newBucket.page)`

**4. 重新分配条目**

- 临时存储所有条目：`tmpEntries := make([]entry.Entry, bucket.numKeys)`
- 遍历条目并根据新的哈希深度重新分配：
  ```go
  for _, entry := range tmpEntries {
    if Hasher(entry.Key, bucket.localDepth) == newHash {
        newBucket.modifyEntry(newNKeys, entry)
        newNKeys++
    } else {
        bucket.modifyEntry(oldNKeys, entry)
        oldNKeys++
    }
  }
  ```
- 更新两个桶的键数量：
  ```go
  bucket.updateNumKeys(oldNKeys)
  newBucket.updateNumKeys(newNKeys)
  ```
  
- **分配规则**：
  - **如果条目的新哈希值与新哈希前缀匹配（通常是最高位为1），就移动到新桶**
  - **如果条目的新哈希值与原哈希前缀匹配（通常是最高位为0），就留在原桶**
  - 这确保了条目仍然可以通过相同的哈希前缀找到

**5. 更新目录指针**

- 更新所有指向新哈希后缀的目录项指向新桶：
  ```go
  power := bucket.localDepth
  for i := newHash; i < powInt(2, table.globalDepth); i += powInt(2, power) {
    table.buckets[i] = newBucket.page.GetPageNum()
  }
  ```

**6. 递归处理极端情况**

- 检查分裂后的桶是否仍然溢出：
  ```go
  if oldNKeys >= MAX_BUCKET_SIZE {
    return table.split(bucket, oldHash)
  }
  if newNKeys >= MAX_BUCKET_SIZE {
    return table.split(newBucket, newHash)
  }
  ```
- 如果任一桶仍溢出，递归调用split处理

### C. 设计考量

**1. 为什么需要递归分裂**

- **数据分布不均**：哈希函数可能导致大量键映射到相同的桶
- **极端情况处理**：分裂后可能所有或大部分元素仍在同一个桶
- **健壮性**：递归确保无论数据分布如何，都能达到稳定状态

### D. 桶分裂图示解释

```
分裂前:
              哈希表
         +----------------+
         |  globalDepth=2 |
         +----------------+      目录项(二进制)     物理桶
         |    buckets     | ---> [00] ---> 页号0 ---> 桶0 (localDepth=2, 已满)
         |                |      [01] ---> 页号1 ---> 桶1
         |                |      [10] ---> 页号2 ---> 桶2
         |                |      [11] ---> 页号3 ---> 桶3
         +----------------+

分裂后(桶0分裂，全局深度增加):
              哈希表
         +----------------+
         |  globalDepth=3 |
         +----------------+      目录项(二进制)     物理桶
         |    buckets     | ---> [000] ---> 页号0 ---> 桶0' (localDepth=3)
         |                |      [001] ---> 页号1 ---> 桶1
         |                |      [010] ---> 页号2 ---> 桶2
         |                |      [011] ---> 页号3 ---> 桶3
         |                |      [100] ---> 页号4 ---> 桶4 (新桶, localDepth=3)
         |                |      [101] ---> 页号1 ---> 桶1
         |                |      [110] ---> 页号2 ---> 桶2
         |                |      [111] ---> 页号3 ---> 桶3
         +----------------+
```

- 上图展示了可扩展哈希表的桶分裂过程：

  - **初始状态**：全局深度为2，有4个目录项(00,01,10,11)指向不同桶

  - **桶分裂时机**：当桶满时bucket.numKeys >= MAX_BUCKET_SIZE，这里MAX_BUCKET_SIZE = (PAGESIZE - BUCKET_HEADER_SIZE) / ENTRYSIZE = (4096 - 20) / 20 = 203
    - 增加桶的局部深度
    - 创建新桶
    - 根据增加的位数重新分配记录
    - 如果局部深度等于全局深度，则可能需要加倍目录大小

  - **分裂后状态**：
    - 原始桶和新桶基于更长的哈希前缀区分
    - 目录可能增长以适应更多桶
    - 一些目录项可能指向同一个桶（如果它们共享相同的哈希前缀）

## 3.3 Find查找操作

```go
func (table *HashTable) Find(key int64) (entry.Entry, error)
```

### A. 参数介绍

- 参数：
  - `key int64`：要查找的键
- 返回：
  - `entry.Entry`：找到的条目对象
  - `error`：如果键不存在，返回错误；否则返回nil
- 目的：
  - 根据键在哈希表中查找对应的条目

### B. 完整流程

**1. 获取表级读锁**

- 调用`table.RLock()`获取哈希表的读锁
- 确保在返回时释放锁

**2. 计算哈希值**

- 计算键的哈希值：`hash := Hasher(key, table.globalDepth)`
- 检查哈希值是否有效

**3. 获取目标桶**

- 获取并锁定桶：`bucket, err := table.GetAndLockBucket(hash, READ_LOCK)`
- 释放表级读锁，允许其他操作继续
- 确保函数结束时释放资源：`defer table.pager.PutPage(bucket.page)`和`defer bucket.RUnlock()`

**4. 在桶中查找**

- 调用桶的查找方法：`foundEntry, found := bucket.Find(key)`
- 如果未找到，返回错误
- 如果找到，返回条目

## 3.4 Update更新操作

```go
func (table *HashTable) Update(key int64, value int64) error
```

### A. 参数介绍

- 参数：
  - `key int64`：要更新的键
  - `value int64`：新的值
- 返回：
  - `error`：如果更新失败，返回错误；否则返回nil
- 目的：
  - 更新哈希表中现有键的值

### B. 完整流程

**1. 获取表级读锁**

- 调用`table.RLock()`获取哈希表的读锁
- 在获取桶后释放表锁

**2. 计算哈希值并获取桶**

- 计算键的哈希值
- 获取并锁定桶：`bucket, err := table.GetAndLockBucket(hash, WRITE_LOCK)`

**3. 执行更新**

- 调用桶的更新方法：`err2 := bucket.Update(key, value)`
- 如果键不存在，返回错误

## 3.5 Delete删除操作

```go
func (table *HashTable) Delete(key int64) error
```

### A. 参数介绍

- 参数：
  - `key int64`：要删除的键
- 返回：
  - `error`：如果删除失败，返回错误；否则返回nil
- 目的：
  - 从哈希表中删除指定的键值对

### B. 完整流程

与Update类似，但调用桶的Delete方法：
```go
err2 := bucket.Delete(key)
```

# 4. 并发控制

## 4.1 锁定策略

- **双层锁定**：

  - 表级锁：保护整个哈希表结构
  - 桶级锁：保护单个桶的内容

- **锁类型**：

  ```go
  type BucketLockType int
  const (
      NO_LOCK    BucketLockType = 0
      WRITE_LOCK BucketLockType = 1
      READ_LOCK  BucketLockType = 2
  )
  ```

- **分级锁定**：

  - 读操作：先获取表读锁，再获取桶读锁，然后释放表锁
  - 写操作：获取表写锁，再获取桶写锁
  - 分裂操作：保持表写锁，获取桶写锁

## 4.2 基本操作使用的锁类型

| 操作             | 表级锁      | 桶级锁      | 锁持有策略                           |
| ---------------- | ----------- | ----------- | ------------------------------------ |
| **Find(查找)**   | 读锁(RLock) | 读锁(RLock) | 获取桶后释放表锁，操作完成后释放桶锁 |
| **Insert(插入)** | 写锁(WLock) | 写锁(WLock) | 整个操作过程保持表锁和桶锁，直到完成 |
| **Update(更新)** | 读锁(RLock) | 写锁(WLock) | 获取桶后释放表锁，操作完成后释放桶锁 |
| **Delete(删除)** | 读锁(RLock) | 写锁(WLock) | 获取桶后释放表锁，操作完成后释放桶锁 |

**特别说明**：

- Insert操作使用表级写锁是因为可能触发桶分裂和表扩展
- Update和Delete只修改单个桶内容，不改变表结构，因此只需要表读锁

## 4.3 锁使用模式

**1. 表级锁方法**

```go
func (table *HashTable) WLock()   { table.rwlock.Lock() }
func (table *HashTable) WUnlock() { table.rwlock.Unlock() }
func (table *HashTable) RLock()   { table.rwlock.RLock() }
func (table *HashTable) RUnlock() { table.rwlock.RUnlock() }
```

**2. 桶级锁方法**

```go
func (bucket *HashBucket) WLock()   { bucket.page.WLock() }
func (bucket *HashBucket) WUnlock() { bucket.page.WUnlock() }
func (bucket *HashBucket) RLock()   { bucket.page.RLock() }
func (bucket *HashBucket) RUnlock() { bucket.page.RUnlock() }
```

**3. 安全获取桶**

```go
func (table *HashTable) GetAndLockBucket(hash int64, lock BucketLockType) (*HashBucket, error)
```

# 5. 持久化

## 5.1 结构持久化

- **哈希表元数据**：
  - 存储在`<tablename>.meta`文件中
  - 包含全局深度和桶页号数组
  
- **桶数据**：
  - 存储在`<tablename>`主文件中
  - 每个桶占一个页面

## 5.2 读取和写入

- **读取哈希表**：
  ```go
  func ReadHashTable(bucketPager *pager.Pager) (*HashTable, error)
  ```
  
- **写入哈希表**：
  ```go
  func WriteHashTable(bucketPager *pager.Pager, table *HashTable) error
  ```

# 6. 哈希表 vs B+树

**哈希表优势:**

1. **查找性能**
   - 哈希表：O(1)常数时间复杂度，不受记录数量影响
   - B+树：O(log n)对数时间复杂度，随记录增多而增加

2. **简单操作**
   - 哈希表：简单键值查找非常高效
   - B+树：需要维护复杂的树结构和平衡

**哈希表劣势:**

1. **范围查询**
   - 哈希表：不支持范围查询，需要全表扫描
   - B+树：天然支持范围查询，叶节点链表可快速遍历

2. **排序迭代**
   - 哈希表：无法保证顺序访问
   - B+树：支持按键顺序迭代数据

3. **空间利用率**
   - 哈希表：桶可能未被充分利用，尤其是分裂后
   - B+树：节点填充因子通常较高(>50%)

4. **扩展开销**
   - 哈希表：扩展时可能需要加倍目录大小
   - B+树：渐进式增长，无需大规模结构变更

**应用场景选择:**

- 使用哈希表的场景：
  - 点查询为主的工作负载
  - 键值对简单存储
  - 无序数据存储
  
- 使用B+树的场景：
  - 范围查询频繁
  - 需要排序访问数据
  - 需要前缀搜索功能
  - 空间效率要求高