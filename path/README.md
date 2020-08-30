# 路径

只支持以`/`分割的路径。对于文件路径，需使用`filepath`包。

## 函数

- `Clean(path string) string`

格式化路径：

1. 去除多余分隔符`/`
2. 去除当前目录符号`.`
3. 去除内部的`..`符号，及该符号之前的其他符号
4. 去除根节点后的`..`符号

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

## 源码解析

- 格式化路径

```go
func Clean(path string) string {
    // 空路径，返回`.`
    if path == "" {
        return "."
    }

    rooted := path[0] == '/'
    n := len(path)

    // r 待处理字符下标; w 待写入字符下标；dotdot为不能继续回退的下标。
    out := lazybuf{s: path}
    r, dotdot := 0, 0
    if rooted {
        out.append('/')
        r, dotdot = 1, 1
    }

    for r < n {
        switch {
        case path[r] == '/':
            // 分隔符，继续
            r++
        case path[r] == '.' && (r+1 == n || path[r+1] == '/'):
            // `.`元素，继续
            r++
        case path[r] == '.' && path[r+1] == '.' && (r+2 == n || path[r+2] == '/'):
            // `..`元素，回退，如果无法回退则直接添加`..`
            r += 2
            switch {
            case out.w > dotdot:
                // 可以回退
                out.w--
                for out.w > dotdot && out.index(out.w) != '/' {
                    out.w--
                }
            case !rooted:
                // 不能回退，而且不包含根目录，只能添加`..`
                if out.w > 0 {
                    out.append('/')
                }
                out.append('.')
                out.append('.')
                dotdot = out.w // 更新不能继续回退的下标
            }
        default:
            // 不是路径开头，需要添加`/`
            if rooted && out.w != 1 || !rooted && out.w != 0 {
                out.append('/')
            }
            // 拷贝元素直到`/`
            for ; r < n && path[r] != '/'; r++ {
                out.append(path[r])
            }
        }
    }

    // 空路径，返回`.`
    if out.w == 0 {
        return "."
    }

    return out.string()
}
```

- 匹配路径

```go
// 判断路径`name`是否匹配`pattern`。
// 
//    符号:
//        '*'         匹配任何非`/`字符
//        '?'         匹配单个任何非`/`字符
//        '[' [ '^' ] { character-range } ']'
//                  非空字符组
//        c           匹配字符c，且c不能是'*', '?', '\\', '['
//        '\\' c      匹配字符c
//
//    字符列表:
//        c           匹配字符c，且c不能是'\\', '-', ']'
//        '\\' c      匹配字符c
//        lo '-' hi   匹配字符 c，且 lo <= c <= hi
func Match(pattern, name string) (matched bool, err error) {
Pattern:
    for len(pattern) > 0 {
        var star bool
        var chunk string
        star, chunk, pattern = scanChunk(pattern)
        if star && chunk == "" {
            // 模式只有一个`*`，只需判断不包含`/`
            return !strings.Contains(name, "/"), nil
        }
        // 检查`name`头部是否匹配`chunk`
        t, ok, err := matchChunk(chunk, name)
        if ok && (len(t) == 0 || len(pattern) > 0) {
            // `name`头部匹配，去除头部后，重新匹配
            name = t
            continue
        }
        if err != nil {
            // `name`头部不匹配，退出
            return false, err
        }
        
        if star {
            // 如果模式为`*`，匹配到`/`
            for i := 0; i < len(name) && name[i] != '/'; i++ {
                t, ok, err := matchChunk(chunk, name[i+1:])
                if ok {
                    if len(pattern) == 0 && len(t) > 0 {
                        // 如果是最后一个chunk，需要把`name`匹配完
                        continue
                    }
                    name = t
                    continue Pattern
                }
                if err != nil {
                    // 匹配失败
                    return false, err
                }
            }
        }
        return false, nil
    }
    return len(name) == 0, nil
}

// 获取下一个不包含`*`的模式片段
// 该模式片段，只支持常量，`?`，及字符列表
func scanChunk(pattern string) (star bool, chunk, rest string) {
    for len(pattern) > 0 && pattern[0] == '*' {
        // 去除开头的`*`，并标识`star`=true
        pattern = pattern[1:]
        star = true
    }
    inrange := false // 是否在字符列表中，即在`[]`中 
    var i int
Scan:
    for i = 0; i < len(pattern); i++ {
        switch pattern[i] {
        case '\\':
            if i+1 < len(pattern) {
                // 跳过转义字符
                i++
            }
        case '[':
            inrange = true
        case ']':
            inrange = false
        case '*':
            if !inrange {
                // 未在字符列表中，找到下一个`*`
                break Scan
            }
        }
    }
    return star, pattern[0:i], pattern[i:]
}

// 检查`s`头部是否匹配`chunk`，如果匹配，返回`s`的尾部。
func matchChunk(chunk, s string) (rest string, ok bool, err error) {
    for len(chunk) > 0 {
        if len(s) == 0 {
            // chunk不为空，但s为空，返回失败
            return
        }
        
        switch chunk[0] {
        case '[':
            // 取出一个字符
            r, n := utf8.DecodeRuneInString(s)
            s = s[n:]
            chunk = chunk[1:]
            
            // 是否取反
            notNegated := true
            if len(chunk) > 0 && chunk[0] == '^' {
                notNegated = false
                chunk = chunk[1:]
            }
            
            // 匹配字符列表
            match := false
            nrange := 0
            for {
                if len(chunk) > 0 && chunk[0] == ']' && nrange > 0 {
                    // 字符列表结束，退出循环
                    chunk = chunk[1:]
                    break
                }
                
                // 获取字符范围，并进行匹配
                var lo, hi rune
                if lo, chunk, err = getEsc(chunk); err != nil {
                    return
                }
                hi = lo
                if chunk[0] == '-' {
                    if hi, chunk, err = getEsc(chunk[1:]); err != nil {
                        return
                    }
                }
                if lo <= r && r <= hi {
                    match = true
                }
                nrange++
            }
            if match != notNegated {
                return
            }

        case '?':
            if s[0] == '/' {
                // 不允许`/`
                return
            }
            // 匹配成功，移动一个字符
            _, n := utf8.DecodeRuneInString(s)
            s = s[n:]
            chunk = chunk[1:]

        case '\\':
            // 移动到被转义的字符，进入`default`分支进行匹配
            chunk = chunk[1:]
            if len(chunk) == 0 {
                err = ErrBadPattern
                return
            }
            fallthrough

        default:
            if chunk[0] != s[0] {
                return
            }
            // 匹配成功，移动一个字符
            s = s[1:]
            chunk = chunk[1:]
        }
    }
    return s, true, nil
}

// 获取一个未转义的字符
func getEsc(chunk string) (r rune, nchunk string, err error) {
    if len(chunk) == 0 || chunk[0] == '-' || chunk[0] == ']' {
        err = ErrBadPattern
        return
    }
    if chunk[0] == '\\' {
        chunk = chunk[1:]
        if len(chunk) == 0 {
            err = ErrBadPattern
            return
        }
    }
    r, n := utf8.DecodeRuneInString(chunk)
    if r == utf8.RuneError && n == 1 {
        err = ErrBadPattern
    }
    nchunk = chunk[n:]
    if len(nchunk) == 0 {
        err = ErrBadPattern
    }
    return
}
```