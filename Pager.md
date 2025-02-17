# Pager

## 重要问题

### LRU体现

1. 新访问的页面放入 pinnedList 尾部（最近使用）

   不再使用的页面放入 unpinnedList 尾部（最近使用）（候选淘汰）

   获得新page，freeList没有空闲时从 unpinnedList 头部获取（最久未使用）
   
2. 注意：freeList不会补充，它只作为初始的空闲页面池使用。freeList用完为空之后，如果获取新页面从unpinnedList刷盘后获取。

## 重要参数

### freeList

```go
freeList     *list.List // List是链表，每个节点是L
```

预分配但未使用的页面列表，这些页面是已经分配了内存但还没被使用过的

### unpinnedList 

```go
unpinnedList *list.List
```

未固定的页面列表，这些页面已在内存中但当前没有被使用，可以被淘汰

### pinnedList      

```go
pinnedList   *list.List
```

固定的页面列表，这些页面当前正在被数据库使用中的页面

### pageTable

```go
pageTable map[int64]*list.Link
```

pageTable 是一个哈希表（map）数据结构

它的结构是：

- key: int64 类型，表示页面号(pagenum)
- value: *list.Link 类型，指向链表节点的指针，每个链表节点的value就是一个页

## 完整参数

```go
type Pager struct {
    file         *os.File   // File descriptor for the file that backs this pager on disk.
    numPages     int64      // The number of pages that this page has access to (both on disk and in memory).
    freeList     *list.List // A list of pre-allocated (but unused) pages.
    unpinnedList *list.List // The list of pages in memory that have yet to be evicted, but are not currently in use.
    pinnedList   *list.List // The list of in-memory pages currently being used by the database.
    // The page table, which maps pagenums to their corresponding pages (stored in a link belonging to the list the page is in).
    pageTable map[int64]*list.Link
    ptMtx     sync.Mutex // Mutex for protecting the Page table for concurrent use.
}
```

```go
type Page struct {
    pager      *Pager       // Pointer to the pager that this page belongs to
    pagenum    int64        // Unique identifier for the page also denoting it's position stored in the pager's file
    pinCount   atomic.Int64 // The number of active references to this page
    dirty      bool         // Flag on whether the page's data has changed and needs to be written to disk
    rwlock     sync.RWMutex // Reader-writer lock on the page struct itself
    data       []byte       // Serialized data (the actual 4096 bytes of the page)
    updateLock sync.Mutex   // Mutex for updating the page's data
}
```

## Function

### NewPage

```go
func (pager *Pager) newPage(pagenum int64) (newPage *Page, err error) 
```

1. 优先从 freeList(空闲页面列表)获取可用页面
2. 如果 freeList 为空，尝试从 unpinnedList(未固定页面列表)淘汰一个页面
  - 获取页面后需要先将其数据刷新到磁盘
  - 从pageTable中删除该页面的旧映射

3. 如果以上两种方式都无法获取页面，返回 ErrRanOutOfPages 错误

4. 成功获取页面后:
  - 更新页面编号
  - 重置脏页标记
  - 将引用计数设为1

### **GetNewPage**

```go
func (pager *Pager) GetNewPage() (page *Page, err error)
```

1. 加锁保护并设置延迟解锁

2. 创建新页面，使用当前 numPages 作为页面号

3. 标记为脏页以确保数据持久化

4. 将页面加入pinnedList的尾部并更新页表映射

5. 更新pageTable，建立页面号到页的映射关系
6. 增加总页面数计

### GetPage

```go
func (pager *Pager) GetPage(pagenum int64) (page *Page, err error)
```

1. 验证页面号(pagenum)合法性

2. 在 pageTable 中查找页面：

- 如果存在且在 unpinnedList 中，转移到 pinnedList尾部
- 更新 pageTable 映射关系
- 增加引用计数后返回

3. 页面不在 pageTable 中：

- 创建新页面
- 从磁盘加载数据
- 读取失败则放入 freeList
- 读取成功则加入 pinnedList 尾部并更新 pageTable
- 返回加载的页面

### PutPage

```go
func (pager *Pager) PutPage(page *Page) (err error)
```

​      使用场景：当某个操作（查询、更新）完成对页面的访问，需要释放对该页面的引用，避免无限占用内存

1. 减少页面的引用计数(pinCount)

2. 如果引用计数变为0：

- 页面从 pinnedList 移动到 unpinnedList
- 更新 pageTable，将pageNum和新的在unpinnedList的节点构成key-value映射。

3. 检查引用计数是否有效(不能小于0)

### FlushPage

```go
func (pager *Pager) FlushPage(page *Page)
```

1. 单个页面的刷盘，当页面被修改后需要保存到磁盘时调用
2. 写入位置 = 页面号 * 页面大小，写入内容 = 页面数据

### FlushAllPages

```go
func (pager *Pager) FlushAllPages()
```

1. 批量数据刷盘，在系统关闭或做检查点(checkpoint)时调用，确保所有修改过的数据都保存到磁盘。
2. 对pinnedList和unpinnedList进行刷盘

 