# Pager

# 1. 重要概念

## 1.1 LRU体现

1. 新访问的页面放入 pinnedList 尾部（最近使用）

   不再使用的页面从pinnedList 取出，放入 unpinnedList 尾部（最近使用）（候选淘汰）

   获得新page，freeList没有空闲时，从 unpinnedList 头部获取（最久未使用）

2. 数据库通常采用"脏页延迟写回"策略。当页面被修改后，不会立即写回磁盘，而是标记为脏页放在unpinnedList并保留在内存中尽可能长时间，之后如果需要再次读取可以从内存中读取。LRU能保证最近修改的热点页面尽量留在内存中。

3. 注意：freeList不会补充，它只作为初始的空闲页面池使用。freeList用完为空之后，如果获取新页面从unpinnedList刷盘后获取。

## 1.2 不使用LFU原因

1. LFU容易出现"频率陷阱"：过去频繁访问但最近不再使用的页面会因高频率计数而长期占用内存。

2. LFU对新加入但重要的页面不友好，也就是说新加入到unpinnedList 的重要页面一开始频率统计次数为1，之后调用`GetNewPage()`时候会立即让这页刷盘在内存中移除。之后要对该页面访问又必须从磁盘中访问。
3. 实现更复杂，需要额外的数据结构维护频率计数和排序

# 2. 重要参数

## 2.1 freeList

```go
freeList     *list.List // List是链表，每个节点是L
```

预分配但未使用的页面列表，这些页面是已经分配了内存但还没被使用过的

## 2.2 unpinnedList 

```go
unpinnedList *list.List
```

未固定的页面列表，这些页面已在内存中但当前没有被使用（已经被释放，pinCount = 0），可以被淘汰

## 2.3 pinnedList      

```go
pinnedList   *list.List
```

固定的页面列表，这些页面当前正在被数据库使用中的页面（还未被释放，(pinCount > 0)）

## 2.4 pageTable

```go
pageTable map[int64]*list.Link
```

PageTable是内存缓存索引，它跟踪所有在内存中的页面。它不区分页面是否已刷盘，只关心页面是否在内存中

它的结构是：

- key: int64 类型，表示页面号(pagenum)
- value: *list.Link 类型，指向链表节点的指针，每个链表节点的value就是一个页

# 3. 完整参数

```go
type Pager struct {
    file         *os.File   // 磁盘上支持分页器的文件描述符
    numPages     int64      // 分页器可访问的页面总数（包括磁盘和内存中的）
    freeList     *list.List // 预分配但尚未使用的空闲页面列表
    unpinnedList *list.List // 内存中尚未驱逐但当前未被使用的页面列表
    pinnedList   *list.List // 数据库当前正在使用的内存页面列表
    // 页面表，将页面号映射到对应的页面（存储在该页面所属链表的节点中）
    pageTable map[int64]*list.Link
    ptMtx     sync.Mutex    // 保护页面表并发访问的互斥锁
}
```

```go
type Page struct {
    pager      *Pager       // 指向该页面所属的分页器的指针
    pagenum    int64        // 页面的唯一标识符，也表示其在分页器文件中的存储位置
    pinCount   atomic.Int64 // 该页面的活跃引用计数
    dirty      bool         // 标志页面的数据是否已更改并需要写入磁盘
    rwlock     sync.RWMutex // 页面结构体本身的读写锁
    data       []byte       // 序列化数据（实际的4096字节页面内容）
    updateLock sync.Mutex   // 用于更新页面数据的互斥锁
}
```

# 4. Function

## 4.1 GetNewPage获得新页主函数

```go
func (pager *Pager) GetNewPage() (page *Page, err error)
```

### A. 参数介绍

- 参数：
  - 无
- 返回：
  - `page *Page`：新分配的页面
  - `error`：如果分配失败，返回错误；否则返回nil
- 目的：
  - 分配一个新的页面，使用下一个可用的页面号

### B. 完整流程

**1. 获取互斥锁**

- 调用`pager.ptMtx.Lock()`获取页表互斥锁，确保并发安全
- 使用`defer pager.ptMtx.Unlock()`确保函数结束时释放锁

**2. 申请新页面**

- 调用内部函数`pager.newPage(pager.numPages)`请求一个新页面
- 参数为当前的`numPages`，作为新页面的页面号
- 如果没有可用页面，返回错误

**3. 初始化页面**

- 将页面标记为脏页（dirty = true），确保数据最终会写入磁盘
- 设置页面号为当前的numPages值
- **引用计数已在newPage中设置为1**

**4. 更新数据结构**

- 将新页面添加到pinnedList尾部
- 在pageTable中创建从页面号到页面的映射
- 增加pager的numPages计数，为下一个页面分配准备

**5. 返回结果**

- 返回新分配的页面和nil错误

## 4.2 NewPage获取新页内部函数

```go
func (pager *Pager) newPage(pagenum int64) (newPage *Page, err error) 
```

### A. 参数介绍

- 参数：
  - `filePath string`：数据库文件的路径
- 返回：
  - `pager *Pager`：创建的Pager对象
  - `error`：如果创建失败，返回错误；否则返回nil
- 目的：
  - 创建并初始化一个新的Pager对象，打开或创建指定的数据库文件

### B. 完整流程

1. 优先从 freeList(空闲页面列表)获取可用页面
2. 如果 freeList 为空，尝试从 unpinnedList(未固定页面列表)淘汰一个页面

     - **获取页面后需要先将其数据刷新到磁盘（这里是页面管理器唯一刷盘时机，本项目采用惰性刷盘）**

     - 从pageTable中删除该页面的旧映射


3. 如果以上两种方式都无法获取页面，返回 ErrRanOutOfPages 错误

4. 成功获取页面后:

     - 更新页面编号

     - 重置脏页标记

     - 将引用计数设为1


## 4.3 GetPage 获取存在的页面

```go
func (pager *Pager) GetPage(pagenum int64) (page *Page, err error)
```

### A. 参数介绍

- 参数：
  - `pagenum int64`：要获取的页面号
- 返回：
  - `page *Page`：获取的页面
  - `error`：如果获取失败，返回错误；否则返回nil
- 目的：
  - 获取指定页面号的页面，如果内存中没有则从磁盘加载

### B. 完整流程

**1. 获取互斥锁**

- 调用`pager.ptMtx.Lock()`获取页表互斥锁
- 使用`defer pager.ptMtx.Unlock()`确保函数结束时释放锁

**2. 验证页面号(pagenum)合法性**

**3. 在内存(pageTable)中查找页面：**

- 如果存在且在 unpinnedList 中，转移到 pinnedList尾部
  - 更新 pageTable 映射关系
  - **调用`page.Get()`增加引用计数后返回**


- 在pinnedList中不做改动

**4. 页面不在内存(pageTable)中：**

- **调用`pager.newPage(pagenum)`获取一个新页面框架，在`newPage(pagenum)`函数中会设置引用计数为1**
- 设置页面号和脏页标记
- 调用`pager.FillPageFromDisk(page)`从磁盘加载数据
- 如果加载失败，将页面返回到freeList并返回错误
- 将页面添加到pinnedList并更新pageTable

## 4.4 PutPage释放页面引用

```go
func (pager *Pager) PutPage(page *Page) (err error)
```

### A. 参数介绍

- 参数：
  - `page *Page`：要释放的页面
- 返回：
  - `error`：如果释放失败，返回错误；否则返回nil
- 目的：
  - 减少页面的引用计数，如果计数为0则将页面从pinnedList移到unpinnedLis

- 使用场景：当某个操作（查询、更新）完成对页面的访问，需要释放对该页面的引用，避免无限占用内存

### B. 完整流程

**1. 获取互斥锁**

- 调用`pager.ptMtx.Lock()`获取页表互斥锁
- 使用`defer pager.ptMtx.Unlock()`确保函数结束时释放锁

**2. 减少引用计数**

- 调用`page.Put()`减少页面的引用计数并获取返回值
- 检查返回值是否小于0，小于0表示错误（引用计数不平衡）

**3. 检查引用计数为0的情况**

- 获取页面在pageTable中对应的链表节点

- 从pinnedList中移除该节点

- 将页面添加到unpinnedList尾部

- 更新 pageTable，将pageNum和新的在unpinnedList的节点构成key-value映射。

## 4.5 FlushPage将页面写回磁盘

```go
func (pager *Pager) FlushPage(page *Page)
```

### A. 参数介绍

- 参数：
  - `page *Page`：要刷新的页面
- 返回：
  - 无
- 目的：
  - 如果页面被标记为脏页，将其内容写回磁盘

### B. 完整流程

**1. 检查页面是否为脏页**

- 调用`page.IsDirty()`检查页面是否被修改过

- 如果页面不是脏页，函数直接返回，不进行任何操作

**2. 写入磁盘**

- 计算页面在文件的偏移位置（pagenum * Pagesize），写入位置 = 页面号 * 页面大小，写入内容 = 页面数据
- 使用`pager.file.WriteAt()`将页面数据写入正确的文件位置
- 写入的数据是页面的完整内容（`page.data`）

**3. 重置脏页标记**

- 调用`page.SetDirty(false)`清除脏页标记
- 表示页面的内存内容和磁盘内容现在一致

## 4.6 FlushAllPages

```go
func (pager *Pager) FlushAllPages()
```

### A. 参数介绍

- 参数：
  - 无
- 返回：
  - 无
- 目的：
  - 将所有修改过的页面（脏页）写回磁盘
  - 批量数据刷盘，在系统关闭或做检查点(checkpoint)时调用，确保所有修改过的数据都保存到磁盘。

### B. 完整流程

**1. 定义处理函数**

- 创建一个函数用于处理每个页面链表节点
- 函数从节点获取页面对象
- 对每个页面调用`pager.FlushPage(page)`

**2. 遍历已固定页面列表**

- 调用`pager.pinnedList.Map(writer)`对pinnedList中的所有页面应用处理函数
- 检查并刷新所有正在使用中的脏页

**3. 遍历未固定页面列表**

- 调用`pager.unpinnedList.Map(writer)`对unpinnedList中的所有页面应用处理函数
- 检查并刷新所有可替换但尚未替换的脏页

**4. 完成刷新**

- 所有脏页都被写入磁盘
- 所有页面的脏页标记被重置