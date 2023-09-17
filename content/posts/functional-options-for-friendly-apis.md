---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: 【译】友好 API 的函数式选项
subtitle: ""
date: 2023-05-09
lastmod: 2023-05-09
categories: []
tags:
  - Go
draft: false
---
[原文](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/aa140d7e24da8d4eb37fb7ec95d79a57.png)

我想要以一个故事开始我的演讲。

现在是 2014 年底，贵公司正在推出一个革命性的新型分布式社交网络。明智的是，你的团队选择了 Go 作为该产品的语言。

你的任务是编写关键的服务器组件。可能看起来有点像这样。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/461b0dad7d1946fb7c73944a241e092c.png)

有一些未导出的字段需要初始化，并且必须启动 goroutine 才能为传入的请求提供服务。

该软件包有一个简单的 API，非常易于使用。

但是，有一个问题。在你宣布第一个测试版后不久，功能请求就会开始滚滚而来。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/29f8de5c06c6d715a5e56f7e9377e3ad.png)

移动客户端通常响应缓慢，或者完全停止响应 - 你需要添加对断开这些慢速客户端的连接的支持。

在安全意识提高的当下，你的错误跟踪器开始充满了支持安全连接的要求。

然后，你收到一个用户的报告，他在一个非常小的 VPS 上运行你的服务器。他们需要一种方法来限制并行客户端的数量。

接下来是要求限制来自一群被僵尸网络盯上的用户的并发连接数。

......就这样继续下去。

现在，你需要改变你的 API 以纳入所有这些功能请求。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/64e77c906a2023b4c9e1ffa1f55d61a0.png)

当函数无法放在一张幻灯片上时，往往意味着事情不太妙。

请举手，谁使用过这样的 API？

谁写过这样的 API？

谁的代码在依赖这样的 API 时出现过故障？

很明显，这种解决方案很麻烦，很脆弱。它也不太容易被发现。

刚开始使用你的包的人不知道哪些参数是可选的，哪些是必须的。

例如，如果我想创建一个用于测试的服务器实例，我是否需要提供一个真正的 TLS 证书？如果不需要，我应该提供什么来代替？

如果我不关心 maxconns 或 maxconcurrent，我应该使用什么值？我是否使用零？零听起来很合理，但取决于该功能是如何实现的，这可能会限制你的总并发连接为零。

在我看来，编写这样的 API 是很容易的；只要你让调用者负责正确使用它。

虽然这个例子可以被认为是夸大其词，是恶意构建的，并且由于文档不完善而变得更加复杂，但我相信它展示了像这样的华丽、脆弱的 API 的一个真正问题。

所以，现在我已经定义了这个问题，让我们来看看一些解决方案。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/5829465564a4305eb16d0830a121dd6b.png)

与其试图提供一个必须满足每一种变体的单一函数，一个解决方案可能是创建一组函数。

通过这种方法，当调用者需要一个安全的服务器时，他们可以调用 TLS 的变量。

当他们需要为空闲连接建立一个最大的持续时间时，他们可以使用需要超时的变体。

不幸的是，正如你所看到的，提供每一种可能的变体很快就会变得不堪重负。

让我们继续讨论使你的 API 可配置的其他方法。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/76b56dcae71d5b682223be7ad42ee18d.png)

一个非常常见的解决方案是使用 `Config` 结构体。

这有一些好处。

使用这种方法，`Config` 结构体可以随着时间的推移增加新的选项，而创建服务器的公共 API 本身则保持不变。

这种方法可以带来更好的文档。

曾经在 NewServer 函数上的大量注释块，变成了一个很好的文档化的结构体。

它还可能使调用者能够使用零值来表示他们想要某个特定配置选项的默认行为。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/2bdc26b38e036a3837e5a9fbf9999eaa.png)

然而，这种模式并不完美。

它在默认值方面有问题，特别是当零值有一个很好理解的含义时。

例如，在这里显示的配置结构中，当 Port 没有被提供时，NewServer 将返回一个 `*Server`，用于监听 8080 端口。

但这样做的缺点是，你不能再明确地将 Port 设置为 0，并让操作系统自动选择一个空闲的端口，因为这个明确的 0 与字段的 0 值是无法区分的。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/312d81bf2fb24f4da4456a9bafd5632a.png)

大多数时候，你的 API 的用户会期望使用其默认行为。

即使他们不打算改变任何配置参数，这些调用者仍然需要为第二个参数传递一些东西。

因此，当人们阅读你的测试或你的示例代码，试图找出如何使用你的包时，他们会看到这个神奇的空值。

对我来说，这感觉是错误的。

为什么你的 API 的用户要被要求构造一个空值，仅仅是为了满足函数的签名？

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/50c651f31108b6dc733da5f9e9a248c2.png)

解决这个空值问题的一个常见方法是传递一个指向该值的指针，从而使调用者能够使用 nil 而不是构造一个空值。

在我看来，这种模式具有前面例子的所有问题，而且还增加了一些问题。

我们仍然要为这个函数的第二个参数传递一些东西，但现在这个值可能是 nil，而且对于那些想要默认行为的人来说，大多数时候都是 nil。

这就提出了一个问题，传递 nil 和传递一个指向空值的指针之间有区别吗？

对于包的作者和它的调用者来说，更重要的是服务器和调用者现在可以共享对同一个配置值的引用。这就产生了这样的问题：如果这个值在被传递给 NewServer 函数后被改变了，会发生什么？

我相信写得好的 API 不应该要求调用者创建假值来满足那些罕见的使用情况。

我相信，作为 Go 程序员，我们应该努力工作，确保 nil 永远不会成为需要传递给任何公共函数的参数。

而当我们确实想传递配置信息时，它应该是不言自明的，并尽可能地具有表现力。

所以，现在考虑到这些要点，我想谈谈我认为的一些更好的解决方案。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/b30ccd325674b839fa459d93e32d5084.png)

为了消除这个强制性的、但经常不使用的配置值的问题，我们可以改变 NewServer 函数以接受可变数量的参数。

该函数的可变性意味着你根本不需要传递任何东西，而不是传递 nil 或一些零值，作为你想要默认值的信号。

在我看来，这解决了两个大问题。

首先，默认行为的调用变得尽可能的简洁。

其次，NewServer 现在只接受 Config 值，而不是指向 config 值的指针，删除了 nil 这个可能的参数，并确保调用者不能保留对服务器内部配置的引用。

我认为这是一个很大的改进。

但是，如果我们要讲究的话，它仍然有一些问题。

很明显，我们期望你最多提供一个配置值。但由于函数签名是可变的，所以必须编写实现来应对调用者传递多个可能相互矛盾的配置结构。

有没有办法在需要时使用可变参数函数签名并提高配置参数的表现力？

我认为是有的。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/1dc13cb14bc336596af35ec876702009.png)

在这一点上，我想说明的是，功能选项的概念来自于一篇题为 [Self referential functions and design](http://commandcenter.blogspot.com.au/2014/01/self-referential-functions-and-design.html) 罗伯-派克（Rob Pike）于今年 1 月发表。我鼓励在座的各位阅读它。

<strong>
	与前面的例子，以及到目前为止的所有例子的关键区别在于，对服务器的定制不是通过存储在结构中的配置参数来完成的，而是通过对服务器值本身进行操作的函数。
</strong>

和以前一样，函数签名的可变性质给了我们默认情况下的紧凑行为。

当需要配置时，我把对服务器值进行操作的函数作为一个参数传递给 NewServer。

timeout 函数简单地改变了传递给它的任何 `*Server` 值的超时域。

tls 函数则更复杂一些。它接收一个 `*Server` 值，并将原始的监听器值包裹在一个 tls.Listener 中，从而将其转化为一个安全的监听器。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/66c79ff670f7dae617fd662a1839fd40.png)

在 NewServer 内部，应用这些选项是很直接的。

在打开一个 net.Listener 之后，我们使用该监听器声明一个服务器实例。

然后，对于提供给 NewServer 的每个选项函数，我们都会调用该函数，并传入一个指向刚刚声明的服务器值的指针。

很明显，如果没有提供任何选项函数，那么在这个循环中就没有任何工作要做，所以 srv 是不变的。

这就是它的全部内容了。

使用这种模式，我们可以制作一个具有以下特点的 API

- 合理的默认值
- 高度可配置
- 可以随着时间的推移而增长
- 自我记录
- 对新来者安全
- 不需要 nil 或空值来让编译器满意

在我剩下的几分钟时间里，我想向你展示我是如何改进我自己的一个包，把它转换为使用函数式选项。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/49e09e0d4059742b35f8751bf09b19d2.png)

我是一个业余的硬件黑客，我工作中的许多设备都使用 USB 串行接口。所以几个月前我写了一个终端处理包。

在这个包的先前版本中，要打开一个串行设备，改变速度并将终端设置为原始模式，你必须单独进行这些步骤，在每个阶段检查错误。

尽管这个软件包试图在一个更低级的界面上提供一个更友好的界面，但它仍然给用户留下了太多的程序性缺陷。

让我们来看看应用功能选项模式后的包。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/3c8cfcb1e9f06241207c00f0d13461fc.png)

通过将 Open 函数转换为使用函数值的可变参数，我们得到了一个更干净的 API。

事实上，不仅仅是 Open API 得到了改善，设置一个选项、检查一个错误、设置下一个选项、检查错误的磨人过程也没有了。

默认情况下，仍然只需要一个参数，即设备的名称。

对于更复杂的用例，在术语包中定义的配置函数被传递给 Open 函数，并在返回前依次应用。

这与我们在前面的例子中看到的模式相同，唯一不同的是这些函数不是匿名的，而是公共的。在所有其他方面，它们的操作是相同的。

我们将在下一张幻灯片上看一下 Speed、RawMode 和 Open 是如何实现的。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/1b3a74246cc9e9c76900fe6b839baf7a.png)

RawMode 是最容易解释的。它只是一个签名与 Open 兼容的函数。

因为 RawMode 与 Term 声明在同一个包中，它可以访问私有字段并调用在 Term 类型上声明的私有方法，在本例中调用私有的 setRawMode 帮助器。

Speed 也只是一个普通的函数，但是它不符合 Open 要求的签名。这是因为 Speed 本身需要一个参数 -- 波特率。

Speed 返回一个匿名函数，该函数与 Open 函数的签名兼容，它关闭了波特率参数（有点闭包的感觉），在以后应用该函数时捕获它。

在对 Open 的调用中，我们首先用 openTerm 帮助器打开终端设备。

接下来，就像以前一样，我们在选项函数的片断范围内，依次调用每一个函数，并传入 t，即我们的 term.Term 值的指针。

如果在应用任何一个函数时出现错误，那么我们就在这一点上停止，进行清理并将错误返回给调用者。

否则，从函数返回时，我们已经按照调用者的要求创建并配置了一个 Term 值。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/ab6f2c0bf52874a1ac88151537e68a96.png)

综上所述

- 功能性选项让你编写的 API 可以随着时间的推移而增长。
- 它们使默认的用例成为最简单的。
- 它们提供了有意义的配置参数。
- 最后，它们让你可以使用整个语言的力量来初始化复杂的值。

在这次演讲中，我介绍了许多现有的配置模式，那些被认为是习以为常的、目前普遍使用的配置模式，并在每个阶段提出了一些问题，如："这能更简单吗？

- 这可以更简单吗？
- 那个参数有必要吗？
- 这个函数的签名是否能让它安全地被使用？
- API 是否包含会使人沮丧的陷阱或混乱的误导？

我希望我已经启发了你做同样的事情。重新审视你过去写过的代码，向自己提出这些同样的问题，从而改进它。

谢谢！

相关博文：

- [How to include C code in your Go package](https://dave.cheney.net/2013/09/07/how-to-include-c-code-in-your-go-package)
- [Do not fear first class functions](https://dave.cheney.net/2016/11/13/do-not-fear-first-class-functions)
- [Using go test, build and install](https://dave.cheney.net/2014/01/21/using-go-test-build-and-install)
- [Simple profiling package moved, updated](https://dave.cheney.net/2014/10/22/simple-profiling-package-moved-updated)
