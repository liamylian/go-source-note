# 文件路径

## 函数

- `Clean(path string) string`

格式化路径：

1. 去除多余分隔符`/`
2. 去除当前目录符号`.`
3. 去除内部的`..`符号，及该符号之前的其他符号
4. 去除根节点后的`..`符号

- `ToSlash(path string) string`

将系统路径分隔符替换为`/`。

- `FromSlash(path string) string`

将`/`替换为系统路径分隔符。

- `EvalSymlinks(path string) (string, error)`

返回快捷方式实际路径。

- `Abs(path string) (string, error)`

返回绝对路径。

- `Rel(basepath, targpath string) (string, error)`

返回相对路径。

- `Walk(root string, walkFn WalkFunc) error`

遍历路径。

- `VolumeName(path string) string`

返回分区名称（Windows）。

- `SplitList(path string) []string`

将路径拆列表拆分为路径数组。

- `Split(path string) (dir, file string)`

将路径拆分为目录和文件。

- `Join(elem ...string) string`

拼接路径。

- `Ext(path string) strin`

返回文件后缀。

- `Base(path string) string`

获取路径的最后一个元素。

- `IsAbs(path string) bool`

是否绝对路径。

- `Dir(path string) string`

返回目录。

- `Match(pattern, name string) (matched bool, err error)`

判断路径是否符合shell表达式。

- `Glob(pattern string) (matches []string, err error)`

返回所有匹配表达式的所有文件。