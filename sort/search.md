# Search

## 函数

- `Search(n int, f func(int) bool) int`

在升序数组/切片中查找第一个最小数据。

- `SearchInts(a []int, x int) int`

在升序整型切片中查找第一个最小数据。

- `SearchFloat64s(a []float64, x float64) int`

在升序浮点切片中查找第一个最小数据。

- `SearchStrings(a []string, x string) int`

在升序字符串切片中查找第一个最小数据。

## 源码解析

- 防止溢出

```
func Search(n int, f func(int) bool) int {
	i, j := 0, n
	for i < j {
		h := int(uint(i+j) >> 1) // 防止溢出
		if !f(h) {
			i = h + 1
		} else {
			j = h 
		}
	}
}
```