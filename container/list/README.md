# 双向链表

## 类型

```go
// 链表节点
type Element struct {
    // 上下一个节点指针
    next, prev *Element

    // 链表指针
    list *List

    // 节点值
    Value interface{}
}

// 空值表示空链表
type List struct {
    root Element // 哨兵节点 
    len  int     // 节点个数，不包含哨兵节点
}
```

## 函数

- `New() *List`

创建链表

### List对象方法

- `Init() *List`

初始化链表/清空链表。

- `Len() int`

返回链表节点个数。

- `Front() *Element`

返回首个节点。

- `Back() *Element`

返回末尾节点。

- `Remove(e *Element) interface{}`

删除节点并返回。

- `PushFront(v interface{}) *Element`

往头部插入值为v的节点，并返回该节点。

- `PushBack(v interface{}) *Element`

往末尾插入值为v的节点，并返回该节点。

- `InsertBefore(v interface{}, mark *Element)`

往节点`mark`前插入值为v的节点。

- `InsertAfter(v interface{}, mark *Element) *Element`

往节点`mark`后插入值为v的节点。

- `MoveToFront(e *Element)`

将节点移动到链表头部。

- `MoveToBack(e *Element)`

将节点移动到链表尾部。

- `MoveBefore(e, mark *Element)`

将节点`e`移动到节点`mark`之前。

- `MoveAfter(e, mark *Element)`

将节点`e`移动到节点`mark`之后。

- `PushBackList(other *List)`

将其他链表`other`插入到链表之后。

- `PushFrontList(other *List)`

将其他链表`other`插入到链表之前。

## 详解

```go
func (l *List) Init() *List {
    // 初始化，将哨兵节点的前后节点都指向自己
    l.root.next = &l.root
    l.root.prev = &l.root
    l.len = 0
    return l
}

// 只有当链表未初始化过时才进行初始化，即没有使用`New`创建队列，Init还没有调用过。
func (l *List) lazyInit() {
    if l.root.next == nil {
        l.Init()
    }
}

// 在节点`at`后插入节点`e`，并增加节点个数
func (l *List) insert(e, at *Element) *Element {
    n := at.next
    at.next = e
    e.prev = at
    e.next = n
    n.prev = e
    e.list = l
    l.len++
    return e
}

// 在节点`at`后插入值为v的节点，并返回该节点
func (l *List) insertValue(v interface{}, at *Element) *Element {
    return l.insert(&Element{Value: v}, at)
}

// 将节点`e`从链表中删除
func (l *List) remove(e *Element) *Element {
    e.prev.next = e.next
    e.next.prev = e.prev
    e.next = nil // 防止内存泄漏
    e.prev = nil // 防止内存泄漏
    e.list = nil
    l.len--
    return e
}

// 将节点`e`移动到节点`at`之后，并返回节点`e`
func (l *List) move(e, at *Element) *Element {
    if e == at {
        return e
    }
    e.prev.next = e.next
    e.next.prev = e.prev

    n := at.next
    at.next = e
    e.prev = at
    e.next = n
    n.prev = e

    return e
}

// 往链表头部插入值为v的节点（如果链表未初始化，则初始化）
func (l *List) PushFront(v interface{}) *Element {
    l.lazyInit()
    return l.insertValue(v, &l.root)
}

// 往链表尾部插入值为v的节点（如果链表未初始化，则初始化）
func (l *List) PushBack(v interface{}) *Element {
    l.lazyInit()
    return l.insertValue(v, l.root.prev)
}

func (l *List) Remove(e *Element) interface{} {
    if e.list == l {
        // 链表必须初始化过，否则会崩溃（既然e存在，一般情况是初始化过的，因为`PushFront`，`PushBack`等都会懒初始化）
        // 只有节点在该链表中，才会进行移除
        l.remove(e)
    }
    return e.Value
}
```