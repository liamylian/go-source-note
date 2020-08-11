# 环形链表

环形链表没有头和尾，任意一个节点都可以作为环形链表的指针。空指针即为空的环形链表，空值环形链表为一个节点的环形链表。

## 类型

```go
type Ring struct {
    next, prev *Ring
    Value      interface{} 
}
```

## 函数

- `New(n int) *Ring`

创建带`n`个节点的环形链表。

## `Ring`方法

- `Next() *Ring `

返回下一个节点。

- `Prev() *Ring`

返回上一个节点。

- `Move(n int) *Ring`

移动`n % r.Len()`个元素，如果`n<0`则向后移动，`n>0`则向前移动。

- `Link(s *Ring) *Ring`

将环形链表加入到当前链表，并返回移动前的下一个节点。

- `Unlink(n int) *Ring`

从下一个节点开始，移除`n % r.Len()`个节点，并返回移除掉的环形链表。

- `Len() int`

返回节点个数

- `Do(f func(interface{}))`

遍历节点。

## 源码解读

```go
// 初始化
func (r *Ring) init() *Ring {
    r.next = r
    r.prev = r
    return r
}

// 创建包含n个节点的环形链表
func New(n int) *Ring {
    if n <= 0 {
        return nil
    }
    r := new(Ring) // 第一个节点
    p := r // 当前节点
    for i := 1; i < n; i++ {
        p.next = &Ring{prev: p} // 创建节点，并将当前节点的下一个指向该节点。
        p = p.next // 将当前节点设置为上面创建的节点，继续循环创建节点。
    }
    
    p.next = r // 将最后创建节点的下一个指向第一个节点
    r.prev = p // 将第一个节点的上一个指向最后一个节点
    return r
}

func (r *Ring) Next() *Ring {
    if r.next == nil {
        // 如果还没有初始化，则初始化
        // r必须为非空环形链表，否则r.init()会panic
        return r.init() // 可能不是使用New创建的，节点个数为1的环形链表
    }
    return r.next
}

func (r *Ring) Move(n int) *Ring {
    if r.next == nil {
        return r.init()
    }
    switch {
    case n < 0:
        for ; n < 0; n++ {
            r = r.prev
        }
    case n > 0:
        for ; n > 0; n-- {
            r = r.next
        }
    }
    return r
}

func (r *Ring) Do(f func(interface{})) {
    if r != nil {
        f(r.Value)
        for p := r.Next(); p != r; p = p.next {
            f(p.Value)
        }
    }
}

// 如果`r`和`s`为同一个环形链表，那么`r`和`s`之间的元素（不包含）将被移除并返回。
func (r *Ring) Link(s *Ring) *Ring {
    n := r.Next()
    if s != nil {
        p := s.Prev()
        r.next = s
        s.prev = r
        n.prev = p
        p.next = n
    }
    return n
}
```