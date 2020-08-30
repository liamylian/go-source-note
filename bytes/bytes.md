# 字节

## 函数

- `Equal(a, b []byte) bool`

判断字节切片是否相等

- `Compare(a, b []byte) int`

判断字节切片大小。返回0则相等，-1则a<b，1则a>b

- `Count(s, sep []byte) int`

统计s中sep中的字节串出现次数。如果sep为空，则返回1加上utf-8字符数。

- `Contains(b, subslice []byte) bool`

判断b是否包含了字节串subslice。

- `ContainsAny(b []byte, chars string) bool`

判断b中是否包含chars中的任意utf-8字符。

- `ContainsRune(b []byte, r rune) bool`

判断b中是否包含chars中的字符。

- `IndexByte(b []byte, c byte) int`

查找c在b中的索引。

- `LastIndex(s, sep []byte) int`

查找字节串c在字节串b中的最后一个索引。

- `LastIndexByte(s []byte, c byte) int`

查找c在s中的最后一个索引。

- `IndexRune(s []byte, r rune) int`

查找r在s的索引。

- `IndexAny(s []byte, chars string) int`

查找chars中任意一个字符，出现在s的位置。

- `LastIndexAny(s []byte, chars string) int`

查找chars中任意一个字符，最后出现在s的位置。

- `SplitN(s, sep []byte, n int) [][]byte`

将字节串s，按sep分割成最多n个子串。

- `SplitAfterN(s, sep []byte, n int) [][]byte`

将字节串s，按sep分割成最多n个子串（子串中包括sep）。

- `Split(s, sep []byte) [][]byte`

将字节串s，按sep分割成多个子串。

- `SplitAfter(s, sep []byte) [][]byte`

将字节串s，按sep分割成多个子串（子串中包括sep）。

- `Fields(s []byte) [][]byte`

将字节串，分割成不带空字符的子串。

- `FieldsFunc(s []byte, f func(rune) bool) [][]byte`

将字节串，按f分割成子串。

- `Join(s [][]byte, sep []byte) []byte`

拼接字节串。

- `HasPrefix(s, prefix []byte) bool`

字符串是否以prefix开头。

- `HasSuffix(s, suffix []byte) bool`

字符串是否以suffix结尾。

- `Map(mapping func(r rune) rune, s []byte) []byte`

按字典进行字符替换。

- `Repeat(b []byte, count int) []byte`

构造重复多次的字节串。

- `ToUpper(s []byte) []byte`

转换为大写字符。

- `ToLower(s []byte) []byte`

转换为小写字符。

- `ToTitle(s []byte) []byte`

转换为首字母大写。

- `ToUpperSpecial(c unicode.SpecialCase, s []byte) []byte`

转换为大写字符。

- `ToLowerSpecial(c unicode.SpecialCase, s []byte) []byte`

转换为小写字符。

- `ToTitleSpecial(c unicode.SpecialCase, s []byte) []byte`

转换为首字母大写。

- `ToValidUTF8(s, replacement []byte) []byte`

转换为UTF8

- `Title(s []byte) []byte`

转换为首字母大写。

- `TrimLeftFunc(s []byte, f func(r rune) bool) []byte`

删除符合条件的前缀。

- `TrimRightFunc(s []byte, f func(r rune) bool) []byte`

删除符合条件的后缀。

- `TrimFunc(s []byte, f func(r rune) bool) []byte`

删除符合条件的前缀和后缀。

- `TrimPrefix(s, prefix []byte) []byte`

删除前缀。

- `TrimSuffix(s, suffix []byte) []byte`

删除后缀。

- `IndexFunc(s []byte, f func(r rune) bool) int`

查找索引。

- `LastIndexFunc(s []byte, f func(r rune) bool) int`

查找最后的索引。

- `Trim(s []byte, cutset string) []byte`

截取掉cutset中的字符。

- `TrimLeft(s []byte, cutset string) []byte`

左边截取掉cutset中的字符。

- `TrimRight(s []byte, cutset string) []byte`

右边截取掉cutset中的字符。

- `TrimSpace(s []byte) []byte`

截取掉空字符。

- `Runes(s []byte) []rune`

转换为rune串。

- `Replace(s, old, new []byte, n int) []byte`

替换字节串。

- `ReplaceAll(s, old, new []byte) []byte`

替换所有字节串。

- `EqualFold(s, t []byte) bool`

大小写不敏感是否相等。

- `Index(s, sep []byte) int`

sep在s中的索引。

## 源码解析

- 子串查找算法

```go
// 哈希基数
const primeRK = 16777619

// Rabin-Karp算法
func indexRabinKarp(s, sep []byte) int {
    hashsep, pow := hashStr(sep)
    n := len(sep)
    var h uint32
    for i := 0; i < n; i++ {
        h = h*primeRK + uint32(s[i])
    }
    if h == hashsep && Equal(s[:n], sep) { 
        return 0
    }
    for i := n; i < len(s); {
        h *= primeRK
        h += uint32(s[i])
        h -= pow * uint32(s[i-n])
        i++
        if h == hashsep && Equal(s[i-n:i], sep) {
            // 即使哈希值相等，还需要比较子串
            return i - n
        }
    }
    return -1
}

// 返回子串哈希值及系数
func hashStr(sep []byte) (uint32, uint32) {
    hash := uint32(0)
    for i := 0; i < len(sep); i++ {
        hash = hash*primeRK + uint32(sep[i])
    }
    var pow, sq uint32 = 1, primeRK
    for i := len(sep); i > 0; i >>= 1 {
        if i&1 != 0 {
            pow *= sq
        }
        sq *= sq
    }
    return hash, pow
}
```