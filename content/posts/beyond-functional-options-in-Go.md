---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: 【译】Go 中函数式选项之外
subtitle: ""
date: 2023-05-11
lastmod: 2023-05-11
categories: []
tags:
  - Go
draft: false
---

[原文](https://www.felesatra.moe/blog/2023/04/23/beyond-functional-options)

2014 年，戴夫-切尼（Dave Cheney）谈到了函数式选项模式（functional option pattern），
从那时起，这种模式在 Go 生态系统中开始流行起来。它们有一些优势：

- 它们提供了一个面向未来的 API，可以自由扩展。
- 它们使带有非零默认值的可选设置更容易处理。
- 它们可以直接操作要初始化的对象。

我已经实现并使用了许多函数式选项的 API，随着时间的推移，我对它们的热情已经大大降温。在实践中，函数式选项几乎总是被用来设置传统的选项结构体（traditional options strucs）。选项本身初始化对象的能力（我称之为 "真正的函数式选项"）当然很聪明，但聪明并不是工程中的一个积极特征。

函数式选项有很多缺陷。确保真正的函数式选项正确地互动（例如，一个选项如果以错误的顺序传递，可能会无意中覆盖另一个选项的一部分）是一个令人头痛的问题，而用一个选项结构体就可以避免。另外，许多选项不能直接在一个对象上初始化，而是要和其他选项一起考虑。这就是为什么在实践中函数式选项的实现一般都会回到选项结构体中去。因此实现通常使用一个没有任何公共方法的接口，与 Dave 的原始例子不同：

```go
type FooOption interface {
	// only private methods, so other packages cannot implement options
}

func NewFoo(o ...FooOption) (*Foo, error) {...}
```

如果我们无论如何都要使用选项结构体，那么我们最好不要把它们打扮成函数式选项。毕竟，选项结构体真的那么糟糕吗？让我们来对比一下函数式选项的所谓优势。

如果你遵循 Go 的向后兼容准则，函数式选项并不能使 API 更适合未来的发展。就像你可以添加一个新的函数式选项一样，你可以在选项结构中添加一个新的字段。你仍然需要支持任何以前添加的选项，并确保它们与可能取代它们的新选项正确互动。事实上，确保与废弃选项的正确交互是函数式选项实现在实践中都使用选项结构体的原因之一。

对于第二点，非零的缺省值确实很难用选项结构来处理，但它们通常可以被容纳。如果零值是无效的，那么它可以被转换为默认值。如果零值是有效的，它可以被映射为一个神奇的值，比如 -1。可以在字段注释中记录这一行为。如果其他的都失败了，你可以添加一个布尔字段来表示零值：

```go
type Options struct {
	// Name of Foo.  Defaults to "Foo".
	Name string
	// If true, use empty string for Name rather than the default value.
	UseEmptyName bool
}
```

这不是一个最漂亮的 API，但对于一个罕见的用例来说，这真的那么糟糕吗？

最后，选项结构体确实不能直接模拟真正的函数式选项。然而，正如前面所描述的，真正的函数式选项很少有用。

你可能会说，即使选项结构体不比函数式选项差，那也没有理由不使用函数式选项。那么，函数式选项也有一些缺点。它们增加了开销，因为它们在为每个选项创建闭包对象时增加了开销。它们还阻碍了对可用选项的发现，即使使用 IDE 的功能可以使之更加容易。如果你问我，我是想为它们的文档注释寻找一打函数式的选项构造函数，还是想处理尴尬的默认值，如果这意味着我可以在一个地方看到所有的选项，我会很乐意选择后者！

如果你真的不相信，那么我可以建议一个中间方案：接口结构体选项。Go 中接口的好处是，任何实现了接口的值都可以作为接口使用，所以我们不必拘泥于函数值：

```go
type FooOption interface {
	fooOption()
}

type FooOpts struct {
	SomeTimeout int
}

func (*FooOpts) fooOption() {}

func NewFoo(o ...FooOption) (*Foo, error) {
	opt := &FooOpts{}
	for _, o := range o {
		switch o := o.(type) {
		case *FooOpts:
			opt = o
		}
	}
	// ...
}

func main() {
	f, err := NewFoo(&FooOpts{
		SomeTimeout: 5,
	})
}
```
