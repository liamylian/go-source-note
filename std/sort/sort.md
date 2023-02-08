# Sort

## 接口

```go
// 可排序类型
type Interface interface {
    // 元素数量
	Len() int
    // 元素i是否小于元素j
	Less(i, j int) bool
    // 交换元素i和j
	Swap(i, j int)
}
```

## 函数

- `Sort(data Interface)`

对数据进行排序。

- `Reverse(data Interface) Interface`

返回反向的可排序类型。

- `IsSorted(data Interface) bool`

是否已排序。

- `func Ints(a []int)`

对整型切片进行排序。

- `Float64s(a []float64)`

对浮点切片进行排序。

- `Strings(a []string)`

对字符串切片进行排序。

- `IntsAreSorted(a []int) bool`

整型切片是否已排序。

- `Float64sAreSorted(a []float64) bool `

浮点切片是否已排序。

- `StringsAreSorted(a []string) bool`

字符串切片是否已排序。

- `Stable(data Interface)`

对数据进行稳定排序。

## 源码解析

- 插入排序

```go
func insertionSort(data Interface, a, b int) {
	for i := a + 1; i < b; i++ {
		for j := i; j > a && data.Less(j, j-1); j-- {
			data.Swap(j, j-1)
		}
	}
}
```

- 堆排序

```go
// 在第[lo, hi)元素间进行下次沉，first为第一个节点索引。
func siftDown(data Interface, lo, hi, first int) {
	root := lo
	for {
		child := 2*root + 1
		if child >= hi {
			break // 没有叶子节点了，返回
		}
		if child+1 < hi && data.Less(first+child, first+child+1) {
			child++ // 找出最大子节点
		}
		if !data.Less(first+root, first+child) {
            // 最大子节点已经小于当前根节点，下沉完成，退出（heapSort中已经确保从最底层的非叶子节点开始下沉）
			return
		}
    
        // 需要交换子节点与当前根节点，并且从子节点继续下沉
		data.Swap(first+root, first+child)
		root = child
	}
}

func heapSort(data Interface, a, b int) {
	first := a
	lo := 0
	hi := b - a

    // 从最后一个非叶子节点开始下沉，构建最大堆
	for i := (hi - 1) / 2; i >= 0; i-- {
		siftDown(data, i, hi, first)
	}

    // 从最大堆取数据，进行排序
	for i := hi - 1; i >= 0; i-- {
		data.Swap(first, first+i)  // 取出最大数据，放入尾部
		siftDown(data, lo, i, first) // 从新整理堆
	}
}
```

- 快速排序

```go
func quickSort(data Interface, a, b, maxDepth int) {
	for b-a > 12 { 
        // 元素数量大于12
		if maxDepth == 0 { 
			heapSort(data, a, b) // 深度为0，使用堆排序，并结束排序
			return
		}
		maxDepth-- // 循环一次，降低一次深度。避免递归层级过深，从而保证栈深度最多为lg(b-a)

		mlo, mhi := doPivot(data, a, b)
		if mlo-a < b-mhi {
			quickSort(data, a, mlo, maxDepth)
			a = mhi // i.e., quickSort(data, mhi, b)
		} else {
			quickSort(data, mhi, b, maxDepth)
			b = mlo // i.e., quickSort(data, a, mlo)
		}
	}

    // 元素数量小于12，使用希尔排序
	if b-a > 1 {
        // 使用gap=6进行希尔排序（因为b-a<=12）
		for i := a + 6; i < b; i++ {
			if data.Less(i, i-6) {
				data.Swap(i, i-6)
			}
		}
        // 使用插入排序
		insertionSort(data, a, b)
	}
}

func doPivot(data Interface, lo, hi int) (midlo, midhi int) {
	m := int(uint(lo+hi) >> 1) // 首先用位运算的方式求中间点，防止溢出

    //  多数取中
	if hi-lo > 40 {
		// Tukey's ``Ninther,'' median of three medians of three.
		s := (hi - lo) / 8
		medianOfThree(data, lo, lo+s, lo+2*s)
		medianOfThree(data, m, m-s, m+s)
		medianOfThree(data, hi-1, hi-1-s, hi-1-2*s)
	}
	medianOfThree(data, lo, m, hi-1)

	// 接下来要对数据达成以下划分结果
	//	data[lo] = pivot (set up by ChoosePivot)
	//	data[lo < i < a] < pivot
	//	data[a <= i < b] <= pivot
	//	data[b <= i < c] unexamined
	//	data[c <= i < hi-1] > pivot
	//	data[hi-1] >= pivot
	pivot := lo
	a, c := lo+1, hi-1

	for ; a < c && data.Less(a, pivot); a++ {
	}
	b := a
	for {
		for ; b < c && !data.Less(pivot, b); b++ { // data[b] <= pivot
		}
		for ; b < c && data.Less(pivot, c-1); c-- { // data[c-1] > pivot
		}
		if b >= c {
			break
		}
		// data[b] > pivot; data[c-1] <= pivot
		data.Swap(b, c-1)
		b++
		c--
	}
    
    // 如果hi-c<3 这表明数据中有重复的数，
    // 这里保守一些，认为hi-c<5 为边界，如果重复的数较多，会以直接扫描跳过的方式把pivot左右两边的区间缩小
	protect := hi-c < 5
	if !protect && hi-c < (hi-lo)/4 {
		// Lets test some points for equality to pivot
		dups := 0
		if !data.Less(pivot, hi-1) { // data[hi-1] = pivot
			data.Swap(c, hi-1)
			c++
			dups++
		}
		if !data.Less(b-1, pivot) { // data[b-1] = pivot
			b--
			dups++
		}
		// m-lo = (hi-lo)/2 > 6
		// b-lo > (hi-lo)*3/4-1 > 8
		// ==> m < b ==> data[m] <= pivot
		if !data.Less(m, pivot) { // data[m] = pivot
			data.Swap(m, b-1)
			b--
			dups++
		}
		// if at least 2 points are equal to pivot, assume skewed distribution
		protect = dups > 1
	}
	if protect {
        // 有很多数据重复的情况
		// 对数据进行划分:
		//	data[a <= i < b] unexamined
		//	data[b <= i < c] = pivot
		for {
			for ; a < b && !data.Less(b-1, pivot); b-- { // data[b] == pivot
			}
			for ; a < b && data.Less(a, pivot); a++ { // data[a] < pivot
			}
			if a >= b {
				break
			}
			// data[a] == pivot; data[b-1] < pivot
			data.Swap(a, b-1)
			a++
			b--
		}
	}
	// Swap pivot into middle
	data.Swap(pivot, b-1)
	return b - 1, c
}
```

- 稳定排序

```go
func stable(data Interface, n int) {
    // 定义最小merge的块大小为20
	blockSize := 20 // must be > 0
	a, b := 0, blockSize
	for b <= n {
        // 调用插入排序将每一块数据内部排序
		insertionSort(data, a, b)
		a = b
		b += blockSize
	}
	insertionSort(data, a, n) // //最后不足20个的块特殊处理

	for blockSize < n {
		// 以blocksize作为基数，相邻两块为一组进行归并
		a, b = 0, 2*blockSize
		for b <= n {
			symMerge(data, a, a+blockSize, b)
			a = b
			b += 2 * blockSize
		}
		//剩余的不足一组的，如果个数大于基数blocksize，特殊处理
		if m := a + blockSize; m < n {
			symMerge(data, a, m, n)
		}
		//扩大基数，进行下一次循环
		blockSize *= 2
	}
}

// 实现了Stable Minimum Storage Merging
// by Symmetric Comparisons 算法
func symMerge(data Interface, a, m, b int) {
    // 当要归并的前面一组只有一个元素时，直接进行二分查找插入
	if m-a == 1 {
		i := m
		j := b
		for i < j {
			h := int(uint(i+j) >> 1)
			if data.Less(h, a) {
				i = h + 1
			} else {
				j = h
			}
		}
		for k := a; k < i-1; k++ {
			data.Swap(k, k+1)
		}
		return
	}

	// 当要归并的后面一组只有一个元素时，直接进行二分查找插入
	if b-m == 1 {
		i := a
		j := m
		for i < j {
			h := int(uint(i+j) >> 1)
			if !data.Less(m, h) {
				i = h + 1
			} else {
				j = h
			}
		}
		for k := m; k > i; k-- {
			data.Swap(k, k-1)
		}
		return
	}

	// 首先，在两个有序段中，以较短的段长为准，在mid周围找到另一个与之等长的有序段。
	mid := int(uint(a+b) >> 1)
	n := mid + m
	var start, r int
	if m > mid {
		start = n - b
		r = mid
	} else {
		// 说明a，b的mid在m的右侧。即a：m是比较短的一段
		start = a
		r = m
	}
	p := n - 1

	for start < r {
        // c 是折半查找时的下标，所以在m<mid时c的取值范围就是 start<c<r即，a<c<m
        c := int(uint(start+r) >> 1)
        // p-c=mid+m-1-c
        // 不明白p-c是指什么，可以将c的取值范围带入
        // 得到 p-c 的取值范围是 mid+m-1-m < p-c < mid+m-1-a
        // 可以看出来p-c其实是对应 mid-1：mid-1+(m-a)的一段
        // 而c的取值范围可以写成 a<c<a+(m-a)
        // 所以p-c与c其实分别在两段长度都等于m-a的有序段中
        if !data.Less(p-c, c) {
            start = c + 1
        } else {
            r = c
        }
	} //上面这个循环的解释写在这里：
    // 这个循环看起来像是个折半查找，但是target不是一个确定的量，而是p-c这个下标的元素
    // 而c是一个变量，p-c与c分别属于位于两个等长的段a:a+m-a，mid-1:mid-1+m-a
    // c取最大值时p-c取最小值，可以看出来，这段其实是正文中提到的Stable Minimum Storage Merging
    // by Symmetric Comparisons 算法的一个实现
    // 即在a:m中从后往前，找到连续的n个元素每一个都大于mid-1:mid-1+m-a中从前往后的n个元素
       
       
    // 上面的循环是在两个等长的有序段找到可以替换的部分
    // 而实际情况是m：mid-1这一段元素也小于a:m的后n个元素
    // 所以可以交换这两段不等长的数据
	end := n - start
	if start < m && m < end {
		rotate(data, start, m, end)
	}
	// 交换后的两段本身都是有序的，交换后与没有变动的元素组成了两组
	// 即a:start，start:mid 和mid:end，end:b
	// 分别对两段再进行归并
	if a < start && start < mid {
		symMerge(data, a, start, mid)
	}
	if mid < end && end < b {
		symMerge(data, mid, end, b)
	}
}

// 不等量前后对调
func rotate(data Interface, a, m, b int) {
   	// 计算 两段需要对调的段长
	i := m - a
	j := b - m

	// 两段不等长时
	// 先将较短的一段移动到目标位置
	for i != j {
		if i > j {
            //此时前一段比较长
            //将从m开始的j个元素与从m-i开始的j个元素对调
            swapRange(data, m-i, m, j)
            //此时原来后面j个元素已经到达最前面
            //而原来前面的i个元素的前j个元素跑到了最后面
            //调整i
            i -= j
            //在下一个循环对调前面i，j个元素
		} else {
			//此时后一段比较长
			//将从m-i开始的i个元素与从m+j-i开始的i个元素对调
			swapRange(data, m-i, m+j-i, i)
			//此时原来前面的i个元素已经到达最后面
			//而原来后面j个元素的后i个元素跑到了最前面
			//调整j
			j -= i
			//在下一个循环对调前面i，j个元素
		}
	}
	// i==j 循环结束时，直接将m前后的i个元素对调
	swapRange(data, m-i, m, i)
}

// 前后等量对调
func swapRange(data Interface, a, b, n int) {
	for i := 0; i < n; i++ {
		data.Swap(a+i, b+i)
	}
}
```