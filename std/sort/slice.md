# Slice

## 函数

- `Slice(slice interface{}, less func(i, j int) bool)`

对切片进行排序（使用快速排序）。

- `SliceStable(slice interface{}, less func(i, j int) bool)`

对切片进行排序（使用稳定排序）。

- `SliceIsSorted(slice interface{}, less func(i, j int) bool) bool`

判断切片是否已排序。
