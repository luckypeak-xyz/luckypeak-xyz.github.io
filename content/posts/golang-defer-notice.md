---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: Golang 中 defer 关键字的注意点
subtitle: 
date: 2023-09-17
lastmod: 2023-09-17
categories: []
tags:
  - Go
draft: false
featuredImagePreview: https://s2.loli.net/2023/09/17/SUVkZh1MWbAogix.png
featuredImage: https://s2.loli.net/2023/09/17/SUVkZh1MWbAogix.png
---

有关 Golang 中的 `defer` 有几个需要注意的点：

- 与 `go` 语句相同 `defer` 将函数调用（而非函数声明）作为参数
- `defer` 必须记录 `defer` 参数的闭包，所以存在一定的性能损失？
- `defer` 会在返回之前调用，但是注意 **`return` 语句不是原子的**

<!--more-->

首先，`defer` 的参数是函数调用而非函数申明很好理解，也不容易出错，因为这是语法错误，编辑器会提示，例如：

```go
func FuncCallParam() {
	defer foo // error: expression in defer must be function call
}

func foo() {
	fmt.Println("foo")
}
```

其次，`defer` 必须记录 `defer` 参数的闭包，也就是说：

```go
defer mu.Unlock()
```

等价于：

```go
defer func() {
	mu.Unlock()
}()
```

那这会存在性能损失吗？损失大吗？

```go

type SyncMap struct {
	m map[int]int
	mu sync.RWMutex
}

func NewSyncMap() *SyncMap {
	return &SyncMap{m: make(map[int]int)}
}

func (m *SyncMap) SetDoDefer(key int, value int) {
	m.mu.Lock()
	defer m.mu.Unlock()
	
	m.m[key] = value
}

func (m *SyncMap) SetDoNotDefer(key int, value int) {
	m.mu.Lock()

	m.m[key] = value

	m.mu.Unlock()
}

var MapDoDefer = NewSyncMap()

// BenchmarkSyncMapDoDefer-8       10617040               149.5 ns/op            57 B/op          0 allocs/op
// BenchmarkSyncMapDoDefer-8       10884612               149.2 ns/op            55 B/op          0 allocs/op
func BenchmarkSyncMapDoDefer(b *testing.B) {
	for i := 0; i < b.N; i++ {
		MapDoDefer.SetDoDefer(i, i+1)
	}
}

var MapDoNotDefer = NewSyncMap()

// BenchmarkSyncMapDoNotDefer-8     8368516               155.6 ns/op            71 B/op          0 allocs/op
// BenchmarkSyncMapDoNotDefer-8     8572152               149.6 ns/op            69 B/op          0 allocs/op
func BenchmarkSyncMapDoNotDefer(b *testing.B) {
	for i := 0; i < b.N; i++ {
		MapDoNotDefer.SetDoNotDefer(i, i+1)
	}
}
```

通过上述代码看，似乎 `defer` 并没有带来太大的性能损失。但是，我们或多或少对于 defer 的性能损失有所耳闻，这里为什么几乎没有性能损失了呢？这里测试用的 Go 版本是 1.21.1，Go defer 的性能是如何一步步变强的呢？

最后，`defer` 会在返回前调用没错，且有[官方文档](https://go.dev/ref/spec#Defer_statements)为证，但是要注意 `return` 语句不是原子的：

```go
func TestLDefer(t *testing.T) {
	fmt.Println("test1: ", test1()) // 0 0
	fmt.Println("test2: ", test2()) // 0 0
	fmt.Println("test3: ", test3()) // 0 3
	fmt.Println("test4: ", test4()) // 4 4
	fmt.Println("test5: ", test5()) // 0 5
	fmt.Println("test6: ", test6()) // 6 6
	fmt.Println("test7: ", test7()) // 0 0 
	fmt.Println("test8: ", test8()) // 0 8
	fmt.Println("test9: ", test9()) // 9 0
	fmt.Println("test10: ", test10()) // 0 10
}

func test1() (r int) {
	defer fmt.Println("defer1: ", r) // 0
	return r
}

func test2() (r int) {
	defer func() {
		fmt.Println("defer2: ", r) // 0
	}()
	return r
}

func test3() (r int) {
	defer fmt.Println("defer3: ", r) // 0 值传递，下一句 r = 3 不影响
	r = 3
	return r
}

func test4() (r int) {
	defer func() {
		fmt.Println("defer4: ", r) // 4 闭包，引用传递，下一句 r = 4 有影响
	}()
	r = 4
	return r
}

func test5() (r int) {
	fmt.Println("defer5: ",r) // 0 值传递，return 5 不影响
	return 5
}

func test6() (r int) {
	defer func() {
		fmt.Println("defer6: ", r) // 6
	}()
	return 6
}
// return 6 不是原子的，test6 等价于
// func test6() (r int) {
//	r = 6
//	func() {
//		fmt.Println("defer6: ", r) // 6
//	}()
//	return
// }

func test7() (r int) {
	defer func(r int) {
		fmt.Println("defer7: ", r) // 0
	}(r)
	return r
}

func test8() (r int) {
	defer func(r int) {
		fmt.Println("defer8: ", r) // 0 值传递，下一句 r = 8 不影响
	}(r)
	r = 8
	return r
}

func test9() (r int) {
	defer func(r int) {
		r = 9
		fmt.Println("defer9: ", r) // 9
	}(r)
	return r
}

func test10() (r int) {
	defer func(r int) {
		fmt.Println("defer10: ", r) // 0 值传递，下一句 return10 不影响
	}(r)
	return 10
}
```