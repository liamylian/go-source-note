# 堆

堆（Heap）是计算机科学中一类特殊的数据结构的统称。堆通常是一个可以被看做一棵树的数组对象。
在队列中，调度程序反复提取队列中第一个作业并运行，因为实际情况中某些时间较短的任务将等待很长时间才能结束，
或者某些不短小，但具有重要性的作业，同样应当具有优先权。堆即为解决此类问题设计的一种数据结构

堆具有以下特性：

- 任意节点小于（或大于）它的所有后裔，最小元（或最大元）在堆的根上（堆序性）
- 堆总是一棵完全树。即除了最底层，其他层的节点都被元素填满，且最底层尽可能地从左到右填入

将根节点最大的堆叫做最大堆或大根堆，根节点最小的堆叫做最小堆或小根堆。

由于堆是完全二叉树，所以可以用顺序数组来表示。

## 依赖类型

```go
// 可排序类型
type Interface interface {
	// 元素个数
	Len() int
	// `元素i`是否应该排在`元素j`之前
	Less(i, j int) bool
	// 交换`元素i`及`元素j`的位置
	Swap(i, j int)
}
```
## 类型

```go
// 堆类型（最小堆）
type Interface interface {
	sort.Interface
	Push(x interface{}) // 往尾部添加元素
	Pop() interface{}   // 移除尾部元素，并返回
}
```

## 函数

- `Init(h Interface)`

初始化为最小堆。

- `Push(h Interface, x interface{})`

往堆中放入元素。

- `Pop(h Interface) interface{}`

从堆中取出最小元素。

- `Remove(h Interface, i int) interface{}`

移除堆中指定位置的元素。

- `Fix(h Interface, i int)`

元素修改值后，修复堆。比删除再添加元素效率高。

## 详解

```go

func Init(h Interface) {
	n := h.Len()
	// 从最底层的非叶子节点开始，将该节点进行下沉。
	for i := n/2 - 1; i >= 0; i-- {
		down(h, i, n)
	}
}

// 将`元素i0`下沉到该元素对应子树的合适位置，如果下沉了返回`true`。
// 执行完后保证`i0`及其子节点已满足最小堆。
func down(h Interface, i0, n int) bool {
	i := i0
	for {
		j1 := 2*i + 1
		if j1 >= n || j1 < 0 { // 当int溢出时`j1`会小于0
			break
		}
		
		// 找出最小子节点j
		j := j1 
		if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
			j = j2 
		}
		
		if !h.Less(j, i) {
			// 因为Init时，是从最末尾的子树开始的。
			// 所以，如果节点已经小于它的两个子节，那么该树已经满足最小堆，可以直接退出
			break 
		}
		h.Swap(i, j) // 将最小子节点，与父节点进行替换
		i = j // 继续从替换后的位置开始，将`i0`进行下沉
	}
	return i > i0
}

// 将元素放入最小堆
func Push(h Interface, x interface{}) {
	h.Push(x) // 往底部放入元素
	up(h, h.Len()-1) // 将底部的元素上浮到合适位置
}

func up(h Interface, j int) {
	for {
		i := (j - 1) / 2 // 父节点
		if i == j || !h.Less(j, i) {
			// 如果大于父节点，不再上浮，退出
			break
		}
		
		h.Swap(i, j) // 与父节点进行替换
		j = i // 继续上浮
	}
}

// 移除最小元素并返回
func Pop(h Interface) interface{} {
	n := h.Len() - 1
	h.Swap(0, n)
	down(h, 0, n)
	return h.Pop()
}

// 移除第`i`小的元素，并返回
func Remove(h Interface, i int) interface{} {
	n := h.Len() - 1
	if n != i { 
		h.Swap(i, n) // 将第`i`个元素与底部元素互换
		if !down(h, i, n) { // 将替换后的第`i`个元素进行下沉
		    // 如果替换后的第`i`个元素下沉失败，那么进行上浮
			up(h, i)
		}
	}
	return h.Pop() // 移除底部元素并返回
}

// 当弟`i`个元素的值发生改变时，使用该方法进行修复。
func Fix(h Interface, i int) {
	// 将第`i`个元素进行下沉 
	// 如果下沉失败，那么进行上浮
	if !down(h, i, h.Len()) {
		up(h, i)
	}
}
```