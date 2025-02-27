# B+ Tree

# 1. 重要概念

## **1.1 本项目中B+树度数(degree)为202说明:**

- 每个非叶子节点最多有201个key索引和202个子节点(page)，每个叶子节点最多存201个key - value键值对，计算规则见3.2
- 每个非叶子节点(内部节点)至少有⌈203/2⌉个子节点
- 根节点至少有两个子节点,除非它是叶子节点
- 具有 k个子节点的内部节点包含k-1个键值

一个完整degree为3的 B+ 树例子：

```
             [10,  20]           <- Internal Node (根节点)       <---  
          /      |     \                                        <---
     [5,7]    [15,17]  [25,30]   <- Internal Nodes              <---            
    /  |  \    /  |  \   /  |  \                                <--- BTreeIndex
   L1  L2  L3  L4  L5 L6 L7  L8 L9  <- Leaf Nodes (存储实际数据)  <---
   |   |   |   |   |   |  |   |   |                             <---
   ------>----->----->--->--->---->  <- 叶子节点链表（单向链表）    <---
```

## 1.2 B+树的容量计算

- 在B+树中，假设degree为x，树的高度为n层（根节点为第1层），我们可以计算出这棵树能存储的最大键值对数量，对于一个B+树：

  - 每个内部节点最多有x个子节点指针和x-1个键

  - 每个叶子节点最多存储x-1个键值对

  - 所有叶子节点都位于同一层（第n层）

- 整个B+树最多可以存储的键值对总数公式：

![公式](https://latex.codecogs.com/svg.latex?(x-1)%20\cdot%20x^{n-1})

- 例如，当degree=202时：

  - 两层B+树：201 × 202 = 40,602个键值对

  - 三层B+树：201 × 202² = 8,201,604个键值对

## 1.3 重要Helper Function

### 1.3.1 getKeyAt

- 输入index后，由于每个key_size相同，因此我们可以通过对数据切片获取数据
- 类似对页数据的读取和修改的function都是通过这种切片的方法

```go
[KEYS_OFFSET(11字节)][KEY(10字节)][KEY2(10字节)]...
                    |<-- 切片 -->|
                    11           21
// getKeyAt 从内部节点的指定索引位置获取存储的键值
// 并发注意事项：调用此函数前，InternalNode的页面至少需要获得读锁
func (node *InternalNode) getKeyAt(index int64) int64 {
	// 1. 计算键值在页面中的起始位置
	// 使用 keyPos 函数计算第 index 个键值的偏移量
	// startPos = KEYS_OFFSET + index*KEY_SIZE
	startPos := keyPos(index)

	// 2. 从页面数据中读取并解码键值
	// - node.page.GetData() 获取整个页面的字节数据
	// - [startPos : startPos+KEY_SIZE] 截取出键值的字节片段
	// - binary.Varint 将字节序列解码成 int64 类型的键值
	key, _ := binary.Varint(node.page.GetData()[startPos : startPos+KEY_SIZE])

	// 3. 返回解码后的键值
	return key
}
```

# 2. B+树索引

## 2.1 相关结构体介绍

### 2.1.1 BTreeIndex 结构体

```go
// Index 接口定义的方法
type Index interface {
    Close() error
    GetName() string
    GetPager() *pager.Pager
    Find(int64) (entry.Entry, error)
    Insert(int64, int64) error
    Update(int64, int64) error
    Delete(int64) error
    Select() ([]entry.Entry, error)
    Print(io.Writer)
    PrintPN(int, io.Writer)
    CursorAtStart() (cursor.Cursor, error)
}

// BTreeIndex 实现了所有这些方法
type BTreeIndex struct {
    pager  *pager.Pager // The pager used to store the B+Tree's data.
    rootPN int64        // The pagenum of this B+Tree's root node.
}
func (index *BTreeIndex) Close() error { ... }
func (index *BTreeIndex) GetName() string { ... }
func (index *BTreeIndex) GetPager() *pager.Pager { ... }
func (index *BTreeIndex) Find(key int64) (entry.Entry, error) { ... }
func (index *BTreeIndex) Insert(key int64, value int64) error { ... }
func (index *BTreeIndex) Update(key int64, value int64) error { ... }
func (index *BTreeIndex) Delete(key int64) error { ... }
func (index *BTreeIndex) Select() ([]entry.Entry, error) { ... }
func (index *BTreeIndex) Print(w io.Writer) { ... }
func (index *BTreeIndex) PrintPN(pagenum int, w io.Writer) { ... }
func (index *BTreeIndex) CursorAtStart() (cursor.Cursor, error) { ... }
```

- 包含了页管理器和初始化的root Page Number = 0。

- 这里的BTreeIndex是Index接口的实现。BtreeIndex实现了Index接口所有的方法。在go中，只要一个类型实现接口定义的所有方法，它就隐式自动实现了这个接口，并不需要关心实现类的参数attributes有哪些。

- Java：显式implements实现接口，实现类实现接口所有方法，不同的实现类可以有不同的参数。

- Go：隐式实现接口，实现类实现接口所有方法，不同的实现类可以有不同的参数。

### 2.1.2 Split 结构体

```
type Split struct {
    isSplit bool  // 标志是否发生了分裂
    key     int64 // 需要上推到父节点的中间键
    leftPN  int64 // 左子节点的页号
    rightPN int64 // 右子节点的页号
}
```

- Split结构体在节点分裂时用于向上传递分裂信息。当一个节点分裂时，它会返回一个包含分隔键和两个子节点页号的Split结构体，供父节点处理分裂并更新指针关系。这种设计实现了B+树自底向上的分裂机制，确保树结构始终保持平衡。

### 2.1.2 Node接口

```go
type Node interface {
    insert(key int64, value int64, update bool) (Split, error)
    delete(key int64)
    get(key int64) (value int64, found bool)
    search(searchKey int64) int64
    printNode(io.Writer, string, string)
    getPage() *pager.Page
    getNodeType() NodeType
}
```

- Node接口定义了内部节点和叶子节点共享的基本操作，并由InternalNode和LeafNode结构体实现，支持Go语言的多态特性。

## 2.2 BtreeIndex Function

### 2.2.1 Insert

```go
func (index *BTreeIndex) Insert(key int64, value int64)
```

#### A. 参数介绍

- 参数：
  - key - 要插入的键
  - value - 要插入的值
- 返回：err - 可能返回的错误包括：插入重复键、页面分配失败、根节点分裂异常等，插入成功返回nil
- 目的：向B+树索引中插入entry键值对

以插入 key5 为例，初始结构

               [key2,key3]
              /     |     \
        [key1]->[key2]->[key3,key4]

#### **B. 插入过程的调用链**

1. `BTreeIndex.Insert(5)`
2. -> `InternalNode[key2,key3].insert(5)`
3. -> `LeafNode[key3,key4].insert(5)`

#### **C. 完整流程**  

1. 获取和锁定根节点：

   - 获取根页面：`rootPage, err := index.pager.GetPage(index.rootPN)`

   - 锁定根节点：`lockRoot(rootPage)`

   - 转换为节点：`rootNode := pageToNode(rootPage)`

   - 设置延迟释放：`defer index.pager.PutPage(rootPage)`

   - 目的：确保函数结束时释放根页面，防止内存泄漏

2. 叶子节点插入和分裂

   - 调用链：`LeafNode[key3,key4].insert(5) -> node.split()`


   - 分裂结果：`Split{key:key4, leftPN:key3的page, rightPN:key4的page}`，**这里返回的Split信息是上一层内部节点中插入的新的分隔键和指针（子节点页号）**


```
叶子层分裂：[key3,key4] -> [key3,key4,key5] -> [key3] | [key4,key5]

分裂后：
           [key2,key3]
          /     |     \
    [key1]->[key2]->[key3]->[key4,key5]   
```

3. 内部节点处理分裂

   - 收到叶子节点的分隔键和指针信息：`Split{key:key4, leftPN:key3的page, rightPN:key4的page}`


   - 调用链：`InternalNode[key2,key3].insertSplit(Split{key:key4, leftPN:key3的page, rightPN:key4的page})`


      - 在 [key2,key3] 中**插入 key4，插入指向叶子节点分裂信息中rightPN的指针（子节点页号）**


      - 内部节点变为 [key2,key3,key4]

        ```
              [ key2  key3   key4 ]   
              /     |      |      \
        [key1]->[key2]->[key3]->[key4,key5]   
        ```


      - 因超过节点最大容量（degree=3时最多2个键），需要分裂

4. 内部节点分裂（见图示）

   - 调用链：`InternalNode.split()`


   - 分裂过程：

     a. 计算分裂点

     - `midpoint = (3-1)/2 = 1`，即 key3 的位置

     b. 创建新节点并转移数据，新的节点

     ```
         [key4]
        /      \
     [key3]->[key4 key5]
     ```

     c. 原节点数据处理

     - 执行 `node.updateNumKeys(midpoint)`，设置键数量为1
     - 数据特点：
       - 页中实际数据仍然是 `[key2,key3,key4]`
       - 因为 numKeys=1，只能访问到 key2
       - key3, key4 和其指向的节点虽然物理存在，但逻辑上不可访问
       - 这些"不可见"数据区域会在将来被新数据覆盖
     
     d. 分裂结果：
     
     - `Split{key:key3, leftPN: 原节点page, rightPN: 新节点的page}`，这里返回的Split信息是新的上层节点的key和指针（子节点页号）

5. 组装新的根节点

   - 因为是根节点分裂，需要创建新的根节点


      - 特殊处理：根节点必须保持在页面0


      - page 0保存key3


      - 到page0保存指向key2页面和key4页面的指针


      - 更新numKeys


      - 最终树组装完成👇

        ```
                  [key3]         (页面0)
                  /     \
              [key2]   [key4]    (其他页面)
              /    \    /    \
        [key1]->[key2]->[key3]->[key4,key5]
        ```

6. 图示

![b_tree_insert](./images/b_tree_insert.jpg)

### 2.2.2 Select

```go
func (index *BTreeIndex) Select() ([]entry.Entry, error)
```

#### **A. 参数介绍**

- 参数：无
- 返回：
  - []entry.Entry - 包含B+树中所有条目的切片，按键排序
  - error - 可能的错误，如获取游标失败、读取条目失败等
- 目的：返回B+树中所有条目，按键顺序排列

以遍历整棵树为例，初始结构:

```
          [key3]         (页面0)
          /     \
      [key2]   [key4]    (其他页面)
      /    \    /    \
[key1]->[key2]->[key3]->[key4,key5]
```

#### **B. 遍历过程的调用链**

1. `BTreeIndex.Select()`
2. -> `BTreeIndex.CursorAtStart()`
3. -> `BTreeCursor.GetEntry()` + `BTreeCursor.Next()`循环

#### **C. 完整流程**

1. 获取起始位置的游标 
   - 调用链： `CursorAtStart() -> 从根向左遍历 -> 到达最左叶子节点`
   - 调用`cursor.Close()`，确保在函数结束时释放游标持有的锁

```
         [key3]         
          /               从根开始
      [key2]              向左遍历      
      /     
[key1]->                  到达最左叶子
```

2. 遍历所有叶子节点数据

   - 创建动态切片 entries 存储结果

   - 循环处理:

     - `cursor.GetEntry()` 获取当前条目
     - 添加到结果集 `entries = append(entries, entry)`
     - `cursor.Next()` 移动到下一个条目。这里分为游标在节点内或者节点间移动。节点间移动只需要将curIndex++即可。在节点间移动通过右邻居页号找到下一个节点，并初始化新起点的位置。 


3. 返回结果
   - 最终结果为动态切片 entries，包含所有叶子节点的键值对


### 2.2.3 SelectRange

```go
func (index *BTreeIndex) SelectRange(startKey int64, endKey int64) ([]entry.Entry, error)
```

#### **A. 参数介绍**

- 参数：
  - startKey - 范围的起始键（包含）
  - endKey - 范围的结束键（不包含）
- 返回：
  - []entry.Entry - 包含键值在[startKey, endKey)范围内的所有条目的切片
  - error - 可能的错误，包括参数无效（startKey >= endKey）、获取游标失败、读取条目失败等
- 目的：查询特定键范围内的所有条目，区间为[startKey, endKey)

以遍历查找startKey = key2, endKey = key4为例，初始结构:

```
           [key3]         (页面0)
          /     \
      [key2]   [key4]    (其他页面)
      /    \    /    \
[key1]->[key2]->[key3]->[key4,key5]
```

#### **B. 遍历过程的调用链**

1. `BTreeIndex.SelectRange()`

2. -> `BTreeIndex.CursorAt(startKey)`

3. -> `BTreeCursor.GetEntry()` + `BTreeCursor.Next()`循环

#### **C. 完整流程**

1. 参数校验与初始化

   - 检查区间合法性：startKey < endKey

   - 创建动态切片存储结果集


2. 定位起始位置 
   - 调用链：`c = CursorAt(startKey) -> 从根向下查找 -> 定位到 startKey`，让游标`c`指向startKey位置
   - 调用`c.Close()`，确保在函数结束时释放游标持有的锁

```
         [key3]
          /
     [key2]
     /    \
[key1]->[key2]  <-   游标指向Key2
```

3. 遍历收集区间数据 

   - 循环处理直到遇到 endKey 或 B+ 树末尾：

     - `cursor.GetEntry()` 获取当前条目
     - 检查是否到达区间末尾（endKey > checkEntry.Key）
     - 如未到达末尾：
     
       - 添加到结果集
     
       - `cursor.Next()` 移动到下一个位置


4. 返回结果
   - 最终结果和Select类似为动态切片，包含特定范围的entries


### 2.2.4 find

```go
func (index *BTreeIndex) Find(key int64) (entry.Entry, error)
```

#### **A. 参数介绍**

- 参数：
  - key - 要查找的键
- 返回：
  - entry.Entry - 找到的条目，包含键和关联的值
  - error - 键不存在时返回"no entry with key %d was found"错误，或者可能的页面获取失败错误
- 目的：在B+树中查找指定键的条目

以查找 key4 为例，初始结构:

```
           [key3]         (页面0)
          /     \
      [key2]   [key4]    (其他页面)
      /    \    /    \
[key1]->[key2]->[key3]->[key4,key5]
```

#### **B. 查找过程的调用链**

1. `BTreeIndex.Find()`
2. -> `Node.get()` （接口多态，实际执行的是内部节点或叶子节点的 `get` 方法）
3. -> 递归调用直到叶子节点

#### **C. 完整流程** 

1. 获取和锁定根节点
   - `index.pager.GetPage(index.rootPN)` 获取根页面`rootPage`(永远在页面0)，将根页面`rootPage`转换为根节点`rootNode`
   - 锁定根节点：`lockRoot(rootPage)`
   - 转换为节点：`rootNode := pageToNode(rootPage)`
   - 设置延迟释放：`defer index.pager.PutPage(rootPage)`


2. 从根节点开始查找
   - `rootNode.get(key)`接口多态执行内部节点或者叶子节点`get`方法。


```
       [key3]         
          \               key4 > key3
      --->  [key4]        向右子树查找      
            /     
        [key3]->[key4,key5]   找到目标叶子节点
```

3. 在内部节点中定位

   - `node.search(key)`，二分查找定位位置，使用二分查找找到第一个大于 key 的位置`childIndex`

   - 调用`getAndLockChildAt(childindex)`
     - 通过childIndex获得子节点的的页号，再从页面管理器获得该页
     - 将页面转换为子节点
   
   
      - 递归向下再次调用`child.get(key)`，直到到达叶子节点
   


4. 在叶子节点中定位 

   - `node.search(key)`，二分查找定位位置，定位到key的位置
   
   
      - `node.getEntry(index)` 获取条目
   


5. 返回结果

   - 找到：返回对应的 Entry
   
   
      - 未找到：返回错误 "no entry with key %d was found"
   


### 2.2.5 update

```go
func (index *BTreeIndex) Update(key int64, value int64) error
```

#### **A. 参数介绍**

- 参数：
  - key - 要更新的键
  - value - 要设置的新值
- 返回：
  - error - 键不存在时返回"cannot update non-existent entry"错误，或者可能的页面获取/更新失败错误；更新成功返回nil
- 目的：更新B+树中指定键的值，不改变树的结构

#### **B. 更新过程的调用链**

1. `BTreeIndex.Update()`
2. -> `Node.insert(key, value, update=true)` （复用`insert`方法，但设置 update 标志为 true）
3. -> 递归调用直到叶子节点

#### C. 完整流程

1. 获取和锁定根节点，设置延迟释放，和`insert(),find()`一样

2. 直接调用`Node.insert(key, value, update=true)`

3. 只修改值，不改变树结构

4. 在页面上更新数据

5. 返回结果

   - 更新成功：返回 nil


   - 键不存在：返回错误 "cannot update non-existent entry"


# 3. InternalNode

## 3.1 InternalNode (内部节点) 存储布局

假设一个包含2个键和3个子节点指针的内部节点：

- 键：`[10, 20]`
- 子节点指针：`[P1, P2, P3]`
- 子节点指针pointer实际存储的就是page number。
- **子节点指针比键多一个，因为P1指向 < 10的子树，P2指向 10 <= 子树 < 20，P3指向>=20的子树**
- **一个page4KB，非叶子节点不存放数据，只存放key作为索引**

```
+------------------------+  偏移量
| NodeHeader             |  0
| - nodeType = INTERNAL  |  // 1字节
| - numKeys = 202        |  // 10字节
+------------------------+  NODE_HEADER_SIZE = 11
| Keys Array             |
| - key1 = 10            |  // 10字节
| - key2 = 20            |  // 10字节
| - key3 = 30            |  // 10字节
|       ...              |
| - key201 = 2010        |  // 10字节
+------------------------+  KEYS_OFFSET(11) + 202*KEY_SIZE
| Page Number Array      |
| - page1                |  // 10字节
| - page2                |  // 10字节
| - page3                |  // 10字节
|       ...              |
| - page202              |  // 10字节
+------------------------+  PNS_OFFSET(2041) + 203*PN_SIZE

关键偏移量：
- NODE_HEADER_SIZE = 11  (1 + 10)
- KEYS_OFFSET = NODE_HEADER_SIZE = 11
- PNS_OFFSET = KEYS_OFFSET + (201 * 10)
- 总大小 = 4096字节(一个页面)
```

## 3.2 内部节点最多key数量公式

- 步骤1: 计算可用空间
  ptrSpace = pager.Pagesize - INTERNAL_NODE_HEADER_SIZE - KEY_SIZE
          = 4096 - 11 - 10
          = 4075

- 步骤2: 计算键数量

​	PN_SIZE：每个指针大小

​	**KEYS_PER_INTERNAL_NODE = (ptrSpace / (KEY_SIZE + PN_SIZE)) - 1 = (4075 / (10 + 10)) - 1 = 202个，实际代码中第202个键会分裂，最多存储201个键**

## 3.3 InternalNode Function

### 3.3.1 Insert

```go
func (node *InternalNode) insert(key int64, value int64, update bool) (Split, error)
```

#### A. 参数介绍

- 参数：

  - key - 要插入的键

  - value - 要插入的值

  - update - 是否为更新操作（true=更新，false=插入）

- 返回：

  - Split - 如果发生节点分裂，返回分裂信息，包含提升的键和左右子节点页号。如果没有分裂返回空Split结构体

  - error - 可能的错误，如页面获取失败或子节点插入错误

- 目的：
  - 在B+树内部节点中递归查找并执行键值对的插入操作；如果子节点分裂，调用`node.insertSplit(result)`在当前节点插入新的分隔键和指针；插入指针后如果需要分裂，由`node.insertSplit(result)`调用`node.split()`将被提升的Key的 Split结构体传递给上层父节点，维护B+树的平衡性。

#### B. 插入过程调用链

1. InternalNode.insert(key, value, update)

2. -> 找到目标子节点 `node.search(key)` + `node.getAndLockChildAt(childIdx)`

3. -> 递归调用子节点的 `child.insert(key, value, update)`

4. -> 处理可能的子节点分裂 `node.insertSplit(result)`

5. -> 可能触发当前节点分裂 `node.split()`

#### C. 完整流程

1. 查找插入位置：

   - 使用`node.search(key)`二分查找找到第一个大于目标key的子节点位置`childIndex`。

     - 使用Go标准库的`sort.Search`实现二分查找，返回第一个满足条件的索引。在这个实现中，判断函数是 `func(idx int) bool { return node.getKeyAt(int64(idx)) > key }`，用于找到第一个大于指定 `key` 的位置。

       ```go
       func (node *InternalNode) search(key int64) int64 {
       	// 使用二分查找找到第一个大于 key 的位置
       	minIndex := sort.Search(
       		int(node.numKeys), // 在 [0,numKeys) 范围内搜索
       		func(idx int) bool {
       			// 比较函数：返回 true 表示找到目标位置
       			// getKeyAt(idx) 获取节点中 idx 位置的键值
       			return node.getKeyAt(int64(idx)) > key
       		},
       	)
       	return int64(minIndex)
       }
       ```

     - 示例：现有键值对 `[2 3]`，插入 `5` → `childIdex = 2`。这里获得的key的方法通过切片操作，计算公式是从Internal_Node_Header_Size + index * KEY_SIZE到Internal_Node_Header_Size + index * KEY_SIZE + KEY_SIZE。

     - 具体的字节布局：


     ```
     [页面头部(11字节)][Key1(10字节)][Key(10字节)]...
                     |<-- 切片 -->|
                     11          21
     ```

2. 获得目标子节点：

   - 通过刚才找到的`childIndex`子节点下标，调用`node.getAndLockChildAt(childIdx)`获得到孩子节点的pagenum页号。页号的计算和上一步的切片一样，通过PNS_OFFSET + index * PN_SIZE 到 PNS_OFFSET + index * PN_SIZE + PN_SIZE切片获得页号。
   - 孩子是通过找到子节点的页号，获得到页号就能找到页，将页转换成leafNode或者internalNode。
   - 根据go的语法，因为和internalNode共享一个node interface，所以可以找到的是leafNode，也可以是internalNode。

3. 获得页面管理器：
   - 确保在函数返回时候释放页面资源，将没有被引用的页放入unpinnedList等待被刷盘后再次写入。

4. 执行递归插入：
   - 在子节点中递归执行插入操作，调用本身`child.insert(key, value, update)`方法，得到result是一个split结构体。
   - 如果刚才第二步获得的节点是leafNode节点，这里调用的是leafNode里的insert方法。反之调用internalNode的insert继续递归执行。
   - 这里的base case是leafNode的insert方法，因为无论如何最后都会在叶子节点中插入。

5. 插入完成处理子节点分裂情况：
   - 如果子节点发生了分裂，需要执行internalNode的`node.insertSplit(result)`插入新的键作为索引和子节点页的指针。
   - 如果插入后检查当前节点超过了一个页面最多key数限制，当前节点需要执行internalNode的`node.split()`方法，返回得到一个Split结构体，这个Split结构体中包含了向上提升的键和左页号与右页号。
   - 到返回Split结构体就已经结束，后面由`btreeIndex.insert()`函数完成新的树的组装。

6. 返回结果：
   - 子节点插入成功且未分裂：返回 `Split{}, nil`
   - 子节点插入导致分裂，但当前节点插入分隔键后未分裂：返回 `Split{}, nil`
   - 子节点分裂导致当前节点也需分裂：返回 `Split{isSplit: true, key: 提升的键, leftPN: 左子节点页号, rightPN: 右子节点页号}, nil`
   - 子节点插入失败：返回子节点返回的错误 `Split{}, childErr`


### 3.3.2 InsertSplit

```go
func (node *InternalNode) insertSplit(split Split) (Split, error)
```

#### **A. 参数介绍**

- 参数：
  - split - Split结构体，包含由子节点分裂产生的新键(key)和右子节点页号(rightPN)
- 返回：
  - Split - 如果当前节点也发生分裂，返回分裂信息；否则返回空的Split结构体
  - error - 可能的错误，主要是在分裂过程中可能出现的页面分配错误
- 目的：处理子节点分裂后，在当前内部节点中插入新的分隔键和指针；如果必要时对当前节点进行分裂，调用`node.split()`将被提升的Key的 Split结构体传递给上层父节点，维护B+树的平衡性。

#### B. 插入分裂键调用链

1. 当子节点分裂后，需要在父节点(当前内部节点)中插入新的分隔键和右子节点指针时调用
2. 由`InternalNode.insert`方法在检测到子节点分裂时候调用

#### C. 完整流程

1. 查找插入位置：

   - 使用`node.search(key)`二分查找找到第一个大于目标key的子节点位置`childIndex`。
   - 示例：节点 `[key2,key3]`，插入 `key4` → `insertPos = 2`

2. 移动现有键和指针：

   - 为新键和指针腾出空间，从右向左移动数据，避免覆盖

   - 移动键：

     ```
     [key2,key3] → [key2,key3,_]  // _表示空位
     ```

   - 移动指针：

     ```
     [page1,page2,page3] → [page1,page2,page3,_] // _表示空位
     ```

3. 插入新键和指针：

   - 在腾出的位置插入新键：`node.updateKeyAt(insertPos, split.key)`
   - 插入新页号：`node.updatePNAt(insertPos+1, split.rightPN)`
   - 更新节点键数量：`node.updateNumKeys(node.numKeys + 1)`

4. 检查是否需要分裂：

   - 如果键数量超过限制（degree=3时最多2个键），调用 `node.split()`

5. 返回结果：
   - 如果当前节点已分裂，返回包含分裂信息的Split结构体（包括isSplit=true、提升的键、左右子节点页号）
   - 如果当前节点未分裂，返回空的Split结构体（isSplit=false）和nil错误
   - 分裂信息将被传递给上层节点，可能触发级联分裂直到根节点

### 3.3.3 Split

```go
func (node *InternalNode) split() (Split, error)
```

#### **A. 参数介绍**

- 参数：无
- 返回：
  - Split - 包含分裂信息的结构体，包括提升的键、左右子节点页号
  - error - 可能的错误，主要是在创建新节点时的页面分配错误
- 目的：将当前已满的内部节点分裂成两个节点以页的形式，选择中间键提升到父节点，保持B+树的平衡性

#### **B. 调用时机**

1. 在`node.insertSplit()`方法中，当节点键数量达到最大值时被调用
2. 自底向上的分裂传递过程中，作为中间环节处理节点溢出

#### C. 完整流程

1. 创建新节点：

   - 通过页面管理器创建新的内部节点，设置延迟释放节点
   - 示例：分裂 `[key2,key3,key4]`，指针 `[page1,page2,page3,page4]`

2. 计算分裂点：

   - `midpoint = (numKeys-1)/2`
   - 示例：`(3-1)/2 = 1`，key3 的位置

3. 转移数据到新节点：

   - 转移分裂点之后的键和指针到新节点：

     ```
     - 原节点：[key2,key3,key4] → [key2]
     - 新节点：[] → [key4]
     ```

   - 数据存储特点：

     * 原节点通过 `updateNumKeys(midpoint)` 设置键数量
     * 页面保留原数据，但因 numKeys=1 只能访问 key2
     * key3、key4 虽然存在但逻辑上不可见
     * 这些数据区域将在未来写入时被覆盖

4. 处理中间键：

   - 中间键(key3)不会被复制到任何子节点
   - 它将被提升到父节点作为分隔键

5. 页号分配：

   - 左子节点保留原节点页号
   - 右子节点获得新分配的页号
   - 页号关系保持正确以确保树遍历的一致性

6. 返回分裂信息：

   ```go
   return Split{
       isSplit: true,
       key:     middleKey,    // key3，将被提升到父节点
       leftPN:  node.page.GetPageNum(),  // 包含key2的页号
       rightPN: newNode.page.GetPageNum(),  // 包含key4的页号
   }
   ```
   
   - 分裂结果说明：
   
     - 左节点（原节点）：`[key2]`，指针：`[page1,page2]`
   
     - 提升键：`key3`
   
     - 右节点（新节点）：`[key4]`，指针：`[page3,page4]`
   

# 4. LeafNode

## 4.1 LeafNode (叶子节点) 存储布局

- 假设一个包含3个键值对的叶子节点：`(5,100), (8,200), (12,300)`

- 一个page 4KB

```css
+------------------------+  偏移量
| NodeHeader            |  0
| - nodeType = LEAF     |
| - numKeys = 3         |
+------------------------+  NODE_HEADER_SIZE = 11
| rightSiblingPN = 789  |  // 右兄弟页号
+------------------------+  RIGHT_SIBLING_PN_OFFSET(11) + RIGHT_SIBLING_PN_SIZE(10)
| Entry 1: (5,100)      |  // 第一个键值对
+------------------------+  LEAF_NODE_HEADER_SIZE(21) + ENTRYSIZE(20)
| Entry 2: (8,200)      |  // 第二个键值对
+------------------------+  LEAF_NODE_HEADER_SIZE + 2*ENTRYSIZE
| Entry 3: (12,300)     |  // 第三个键值对
+------------------------+  LEAF_NODE_HEADER_SIZE + 3*ENTRYSIZE
```

- 访问叶子节点数据的代码示例：

```go
// 获取键值对的位置
entryPos := LEAF_NODE_HEADER_SIZE + index*ENTRYSIZE
    
// 读取键值对
entry := entry.UnmarshalEntry(page.GetData()[entryPos : entryPos+ENTRYSIZE])
// page.GetData()[entryPos : entryPos+ENTRYSIZE]做了一个切片操作，获得20B特定index的键值对
[页面头部(21字节)][Entry1(20字节)][Entry2(20字节)]...
                  |<--  切片  -->|
                  21            41
// entry.Key = 5, 8, 或 12
// entry.Value = 100, 200, 或 300
```

## 4.2 每个叶子节点包含多少键值对公式

```go
ENTRIES_PER_LEAF_NODE = ((pager.Pagesize - LEAF_NODE_HEADER_SIZE) / ENTRYSIZE) - 1
```

`pager.Pagesize`: 整个页面的大小（4KB = 4096B）

`LEAF_NODE_HEADER_SIZE`: 叶子节点头部大小，包含NodeHeader（11B）、右兄弟指针（10B）（11 + 10 = 21B）

`ENTRYSIZE`: 每个键值对的大小（20B）

**ENTRIES_PER_LEAF_NODE = ((4096 - 21) / (20)) - 1 = 202个，实际代码中第202个键会分裂，最多存储201个键**

## 4.3 LeafNode Function

### 4.3.1 Insert

```go
func (node *LeafNode) insert(key int64, value int64, update bool) (Split, error)
```

#### **A. 参数介绍**

- 参数：
  - key - 要插入或更新的键
  - value - 要关联的值
  - update - 操作模式标志（true=更新模式，false=插入模式）
- 返回：
  - Split - 如果发生节点分裂，返回分裂信息；否则返回空的Split结构体
  - error - 可能的错误，包括：重复键错误（插入模式）、键不存在错误（更新模式）
- 目的：在叶子节点中插入新的键值对或更新已存在键的值；如果节点已满，执行分裂操作并返回分裂信息供父节点处理

#### **B. 插入过程调用链**

1. `LeafNode.insert(key, value, update)` - 执行叶子节点插入操作
2. -> 使用 `node.search(key)` 确定插入位置
3. -> 如果需要插入新键，移动现有元素并插入
4. -> 如果节点已满，调用 `node.split()` 执行分裂
5. -> 返回分裂信息或操作结果

#### C. 完整流程

1. 查找插入位置：

   - 使用二分查找确定新键值对的插入位置 `insertPos`。
   - 示例：现有键值对 `[10, 20, 30]`，插入 `25` → `insertPos = 2`。

2. 检查键是否存在：
   - 如果 `insertPos < numKeys` 且 `node.getKeyAt(insertPos) == key`，表示键已存在。
     - 如果 `update = true`，更新现有键的值调用node.updateValueAt()并返回成功。
     - 如果 `update = false`，返回重复键错误。
   - 如果键不存在且 `update = true`，返回键不存在错误。

3. 插入新键值对：
   - 从 `insertPos` 开始，将后续元素右移一位，腾出插入空间。
   - 在 `insertPos` 插入新条目。
   - 更新条目数量 `numKeys`。
   - 注意：所有键值对操作直接修改页面数据，而非LeafNode结构体。通过 `updateKeyAt()`, `updateValueAt()`, `modifyEntry()` 等方法修改页面数据

4. 检查是否需要分裂：

   - 如果 `numKeys >= ENTRIES_PER_LEAF_NODE`（默认202），触发分裂操作：
     
     - 调用 `node.split()` 执行分裂过程
     - 返回分裂信息给父节点处理
     
     - 如果未分裂，返回成功（空的Split结构体和nil错误）

5. 返回结果：
   - 插入成功且未分裂：返回 `Split{}, nil`
   - 插入导致分裂：返回 `Split{isSplit: true, key: 分隔键, leftPN: 原节点页号, rightPN: 新节点页号}, nil`
   - 键已存在（非更新模式）：返回 `Split{}, errors.New("cannot insert duplicate key")`
   - 键不存在（更新模式）：返回 `Split{}, errors.New("cannot update non-existent entry")`

### 4.3.2 Split

```go
func (node *LeafNode) split() (Split, error)
```

#### **A. 参数介绍**

- 参数：无
- 返回：
  - Split - 包含分裂信息的结构体，包括提升键、左右子节点页号
  - error - 可能的错误，主要是在创建新节点时的页面分配错误
- 目的：将一个已满的叶子节点分裂成两个节点，迁移一半数据到新节点，维护B+树的横向叶子节点链表结构，并返回分裂信息供父节点处理

#### B. 调用时机

1. 调用时机：在叶子节点插入新键值对后，条目数量达到最大值(`ENTRIES_PER_LEAF_NODE`)时被调用

2. 作为B+树自底向上分裂传递的起点

#### C. 完整流程

1. 创建新叶子节点：

   - 通过页管理器分配新页，调用 `createLeafNode(pager)` 初始化新节点。
   - 错误处理：若创建失败（如无可用页），返回错误。
   - 资源释放：通过 `defer pager.PutPage()` 确保新节点的页最终被释放。

2. 调整兄弟指针：

   - 原节点右兄弟：指向新节点（`newNode.page.GetPageNum()`）。

   - 新节点右兄弟：继承原节点的旧右兄弟（`prevSiblingPN`）。

   - 目的：维护叶子节点的横向链表结构。

     ```go
     prevSiblingPN := node.setRightSibling(newNode.page.GetPageNum())
     newNode.setRightSibling(prevSiblingPN)
     ```

3. 计算分裂中点：

   - 公式：`midpoint := node.numKeys / 2`。
   - 作用：若原节点键数为偶数，分裂为两个相等节点；若为奇数，新节点多承载一个键。

4. 迁移条目到新节点：

   - 循环范围：从 `midpoint` 到 `node.numKeys - 1`。

   - 操作：

     ```go
     for i := midpoint; i < node.numKeys; i++ {
         newNode.updateKeyAt(newNode.numKeys, node.getKeyAt(i))     // 复制键到新节点末尾
         newNode.updateValueAt(newNode.numKeys, node.getValueAt(i)) // 复制值到新节点末尾
         newNode.updateNumKeys(newNode.numKeys + 1)                 // 递增新节点条目数
     }
     ```

   - 底层数据操作：

     - `updateKeyAt` 和 `updateValueAt` 直接修改新节点的页数据。
     - 示例：原节点 `[10:100, 20:200, 25:250, 30:300]` → 迁移后新节点 `[25:250, 30:300]`。

5. 更新原节点条目数：

   - 操作：`node.updateNumKeys(midpoint)`。
   - 目的：原节点仅保留前半部分键值对，之后新的数据对后半部分的key和指针进行覆盖。
   - 示例：原节点 `numKeys` 从 `3` 更新为 `1`。

6. 返回分裂信息：

   - 结构体字段：

     ```go
     return Split{
         isSplit: true,
         key:     newNode.getKeyAt(0), // 新节点的第一个键（提升键）
         leftPN:  node.page.GetPageNum(),  // 原节点页号
         rightPN: newNode.page.GetPageNum(), // 新节点页号
     }, nil
     ```

   - 提升键逻辑：新节点的第一个键作为父节点的新分隔键（B+树特性）。

   - 页号信息：父节点需要这些信息来更新其子节点指针

# 5. Cursor

## 5.1 BTreeCursor结构体

```go
type BTreeCursor struct {
   index    *BTreeIndex // 此游标遍历的 B+ 树索引
   curNode  *LeafNode   // 游标当前指向的叶子节点
   curIndex int64       // 游标在当前节点内指向的索引位置
}
```

## 5.2 BTreeCursor Function

### 5.2.1 CursorAtStart

```go
func (index *BTreeIndex) CursorAtStart() (cursor.Cursor, error)
```

#### **A. 参数介绍**

- 参数：无
- 返回：
  - cursor.Cursor - 指向B+树第一个条目的游标接口
  - error - 可能的错误，如获取页面失败或所有叶子节点为空
- 目的：创建一个指向B+树最左侧叶子节点第一个条目的游标，为顺序遍历做准备

#### **B. 调用链与调用时机**

1. 主要由`BTreeIndex.Select()`调用，用于初始化B+树的顺序遍历

2. 按键值升序获取所有条目时使用

#### C. 完整流程

1. 获取根节点：

   - 通过页面管理器获取根页面(index.rootPN = 0)，设置延时释放页面

   - 转换为节点格式


2. 向左遍历直至叶子节点：

   - 从根节点开始循环向下遍历

   - 每次获取最左子节点(索引0位置的子页号)

   - 直到遇到叶子节点类型为止


3. 创建并返回游标：

   - 初始化游标，将 curNode 指向找到的叶子节点

   - 设置 curIndex = 0 指向第一个条目

   - 如果是空节点，尝试移动到下一个非空节点


4. 返回游标：

```go
cursor := &BTreeCursor{
    index:    index,
    curIndex: 0, // 指向节点的第一个条目
    curNode:  leftmostNode,
}
```

### 5.2.2 Next

```go
func (cursor *BTreeCursor) Next() (atEnd bool)
```

#### **A. 参数介绍**

- 参数：无
- 返回：
  - atEnd bool - 如果已到达B+树末尾返回true，否则返回false
- 目的：将游标移动到下一个条目，可以是同一节点内移动，也可以是跨节点移动，通过rightSiblingPN 实现叶子节点间的顺序遍历。

#### **B. 调用链与调用时机**

1. 由`BTreeIndex.Select()`和`BTreeIndex.SelectRange()`在遍历过程中调用

2. 由`BTreeCursor.CursorAtStart()`在处理空节点时调用

#### **C. 完整流程**

1. 检查是否需要移动到下一个节点: 
   - 如果`curIndex + 1 >= curNode.numKeys`，当前节点已遍历完
   - 获取右兄弟节点的页号`nextPN = curNode.rightSiblingPN`
   - 如果没有右兄弟(页号<0)，返回true表示到达末尾

2. 节点内移动：
   - 如果 curIndex+1 < numKeys，直接增加索引


3. 节点间移动：

   - 获取右兄弟节点页面
   - 使用锁爬行技术：先锁住新节点，再释放旧节点
   - 重置游标：`curIndex = 0, curNode = nextNode`
   - 处理空节点：如果新节点为空，递归调用Next()


4. 返回值

   - 返回true的情况：
     - 当前节点没有右兄弟节点(rightSiblingPN < 0)
     - 获取下一个节点页面失败

   - 返回false的情况：
     - 在当前节点内移动成功(curIndex++)
     - 成功切换到右兄弟节点

### 5.2.3 GetEntry

```go
func (cursor *BTreeCursor) GetEntry() (entry.Entry, error)
```

#### **A. 参数介绍**

- 参数：无
- 返回：
  - entry.Entry - 游标当前位置的条目
  - error - 可能的错误，如游标位置无效或节点为空
- 目的：获取游标当前位置的条目数据，同时进行边界检查确保访问安全

#### **B. 调用链与调用时机**

1. 由`BTreeIndex.Select()`和`BTreeIndex.SelectRange()`在遍历每个条目时调用

2. 在查询操作中获取实际数据

#### **C. 完整流程**

1. 边界检查：
   - 检查`curIndex > curNode.numKeys`是否成立，若成立则返回错误
   - 检查节点是否为空(`curNode.numKeys == 0`)，若为空则返回错误
2. 获取数据：
   - 调用`curNode.getEntry(curIndex)`获取条目
   - 返回条目数据和nil错误

3. 返回值：

   - 成功情况：
     - 返回`entry.Entry{Key: key, Value: value}, nil`，包含游标当前位置的键值对

   - 错误情况：

     - 索引越界：`entry.Entry{}, errors.New("getEntry: cursor is not pointing at a valid entry")`

     - 空节点：`entry.Entry{}, errors.New("getEntry: cursor is in an empty node :(")`

### 5.2.4 CursorAt

```
func (index *BTreeIndex) CursorAt(key int64) (cursor.Cursor, error)
```

#### **A. 参数介绍**

- 参数：
  - key int64 - 要查找的键值
- 返回：
  - cursor.Cursor - 指向找到的键位置或下一个更大键的游标
  - error - 可能的错误，如获取页面失败
- 目的：返回指向指定key的游标或找不到时指向下一个更大key的游标

#### **B. 调用链与调用时机**

1. 主要由`BTreeIndex.SelectRange()`调用，用于定位范围查询的起始位置

2. 按指定范围查询条目时使用

#### **C. 完整流程**

1. 获取根节点：
   - 通过页面管理器获取根页面并加读锁，
   - 转换为节点格式
2. 查找包含目标key的叶子节点：
   - 使用for循环从根向下遍历
   - 在每个内部节点使用`search(key)`二分查找确定要走的子节点路径
   - 使用锁爬行技术确保并发安全，锁住子节点，释放父节点
   - 直到到达叶子节点(非InternalNode)
   
3. 创建游标结构体

```go
cursor := &BTreeCursor{
    index:    index,
    curIndex: curNode.search(key), // 在叶子节点中查找 key 的位置
    curNode:  curNode.(*LeafNode),
}
```

4. 处理 key 不在当前节点的情况

   - 如果`curIndex >= curNode.numKeys`，表示key不在当前节点

   - 调用cursor.Next()移动到下一个叶子节点

5. 返回值

   - 成功情况：

     - 返回`cursor.Cursor`接口，实际类型为`*BTreeCursor`，指向包含目标key的位置

     - 如果目标key不存在，则指向下一个更大key的位置

    - 错误情况：
      - 获取页面失败：`nil, err`
      - 遍历过程中出现其他错误：`nil, err`
   