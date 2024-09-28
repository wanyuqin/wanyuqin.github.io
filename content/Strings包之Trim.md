---
title: Strings包之Trim方法
 
categories: golang
comments: true
---

# Strings包之Trim相关方法

## 简介
Go 开发人员在使用 strings 包时常犯的一个错误是混合使用 TrimRight 和 TrimSuffix。这两个函数都有相似的用途，并且很容易混淆它们。让我们来进一步对这两个方法进行了解
<!--more-->
## TrimRight

```go   
func TrimRightDemo() {
    // 123
    fmt.Println(strings.TrimRight("123oxo", "xo"))
}
```
![20230705204610](https://img.ethanleo.top/20230705204610.png)

TrimRight是会将源字符串的每个字符从右边开始进行迭代并且进行判断，检查是否可以与需要被截取的字符串中的字符匹配，如果匹配到了则进行截取并且进行下一次判断。当有一次的判断当前字符不存在目标截取的字符串内容中时，退出循环，返回剩下的字符串

### 源码分析
```go
func TrimRight(s, cutset string) string {
	if s == "" || cutset == "" {
		return s
	}
	if len(cutset) == 1 && cutset[0] < utf8.RuneSelf {
		return trimRightByte(s, cutset[0])
	}
    // 判断是否所有需要被切的字符可以在ascii码中找到
	if as, ok := makeASCIISet(cutset); ok {
		return trimRightASCII(s, &as)
	}
    // 使用unicode编码进行匹配
	return trimRightUnicode(s, cutset)
}


type asciiSet [8]uint32

// 
func makeASCIISet(chars string) (as asciiSet, ok bool) {
	for i := 0; i < len(chars); i++ {
		c := chars[i]
		if c >= utf8.RuneSelf {
			return as, false
		}
		as[c/32] |= 1 << (c % 32)
	}
	return as, true
}

func trimRightByte(s string, c byte) string {
	for len(s) > 0 && s[len(s)-1] == c {
		s = s[:len(s)-1]
	}
	return s
}

func trimRightASCII(s string, as *asciiSet) string {
	for len(s) > 0 {
		if !as.contains(s[len(s)-1]) {
			break
		}
		s = s[:len(s)-1]
	}
	return s
}

// 判断字节是否在asciiSet中
func (as *asciiSet) contains(c byte) bool {
	return (as[c/32] & (1 << (c % 32))) != 0
}

func trimRightUnicode(s, cutset string) string {
	for len(s) > 0 {
		// 获取传入的字符串的最后一个字符
		r, n := rune(s[len(s)-1]), 1
		// 如果r的值大于ascii编码的最大值
		if r >= utf8.RuneSelf {
			// 使用utf8对s的最后一个字符进行编码 unicode
			r, n = utf8.DecodeLastRuneInString(s)
		}
		//判断编码出来的是否在传入的cutset中
		if !ContainsRune(cutset, r) {
			break
		}
		s = s[:len(s)-n]
	}
	return s
}



// ContainsRune reports whether the Unicode code point r is within s.
func ContainsRune(s string, r rune) bool {
	return IndexRune(s, r) >= 0
}


func IndexRune(s string, r rune) int {
	switch {
	case 0 <= r && r < utf8.RuneSelf:
	// 是acsii编码，直接判断是否在s中，如果不在s中返回一个-1
		return IndexByte(s, byte(r))
	case r == utf8.RuneError:
		for i, r := range s {
			if r == utf8.RuneError {
				return i
			}
		}
		return -1
		// 如果是无效的utf8编码 返回-1
	case !utf8.ValidRune(r):
		return -1
	default:
		return Index(s, string(r))
	}
}


```
### ASCII和Unicode的区别
ASCII（American Standard Code for Information Interchange）是一种基于拉丁字母的字符编码标准，用于表示英语及其他西欧语言中常用的字符。它使用7位二进制数表示128个字符，包括数字、字母、标点符号和一些控制字符。

Unicode是一种字符编码方案，它是用于表示世界上所有语言的字符的标准。Unicode编码为每个字符分配一个唯一的数字，使用16位或32位二进制数表示。Unicode字符集也包括了ASCII字符。

区别如下：
1. 编码范围：ASCII只涵盖128个字符，而Unicode包括了世界上几乎所有的语言字符。
2. 编码方式：ASCII使用7位二进制数表示字符，而Unicode使用16位或32位二进制数表示字符。
3. 字符表示：ASCII只能表示英语及一些西欧语言的字符，而Unicode可以表示世界上所有语言的字符，包括汉字、日语、韩语等。
4. 兼容性：Unicode字符集包含了ASCII字符集，因此ASCII编码的文本可以被解释为Unicode编码。


## TrimSuffix
```go
func TrimSuffixDemo() {
    // 123x
	fmt.Println(strings.TrimSuffix("123oxo", "xo"))
}
```
TrimSuffix则直接判断，输入的源字符串后几个字符是否和需要截取的字符相等，如果相等则进行截取，否则不进行操作

### 源码分析

```go

func TrimSuffix(s, suffix string) string {
	if HasSuffix(s, suffix) {
		return s[:len(s)-len(suffix)]
	}
	return s
}

// 直接判断字符串最后几位和需要被截取的是否相同
func HasSuffix(s, suffix string) bool {
	return len(s) >= len(suffix) && s[len(s)-len(suffix):] == suffix
}

```