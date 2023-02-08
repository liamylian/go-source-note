# 临时文件

## 函数

- `TempFile(dir, pattern string) (f *os.File, err error)`

在目录dir下创建一个临时文件。如果dir为空则使用系统临时目录。

- `TempDir(dir, pattern string) (name string, err error)`

在目录dir下创建一个临时文件夹。如果dir为空则使用系统临时目录。
