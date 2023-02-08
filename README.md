# Go语言学习笔记

## 环境变量配置

```
export PATH=$PATH:/usr/local/go/bin
```

## 代理配置

```
export GOPROXY=https://goproxy.cn,direct;
export GOPRIVATE=git.inspii.com;
export GONOSUMDB=git.inspii.com;
export GOSUMDB=sum.golang.google.cn;
```

```
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOPRIVATE=git.inspii.com
go env -w GONOSUMDB=git.inspii.com
go env -w GOSUMDB=sum.golang.google.cn
```
