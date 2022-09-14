# A Gentle Introduction To Rust

![Rust](PPrustS.png)

[thanks to David Marino](http://leftoversalad.com/c/015_programmingpeople/)

## Why learn a new Programming Language?

这个指南的目标是带你读和写足够的Rust代码以感谢线上优秀的学习资源，
特别是 [The Book](https://doc.rust-lang.org/stable/book/)。这是个机会在
买前尝试，得到这个语言威力的足够感觉以继续深入。

就像爱因斯坦也许会说，"尽可能小心，但不是更小心"。有很多新人来学习，
要获得一些思想家具的重新安排区别很大。用'gentle'我意思是这些功能实际
用例子描述；当我们遇到困难，我向你展示如何用Rust解决问题。在解决问题
前先理解问题很重要。将它当作华而不实的语言前，我们将先去山地远足，
我将用少量几课指出路上一些有趣的岩层。有些更高但视野已激活；社区不
常乐意和高兴去帮助。有 [Rust Users Forum](https://users.rust-lang.org/)
和[subreddit](https://www.reddit.com/r/rust/) ，它不太适中。
如果你有规范问题[FAQ](https://www.rust-lang.org/en-US/faq.html)是很好的资源。

首先，为什么要学习新语言？这是对时间和能力投资的正当理由。即便你没有立即
用这语言找到一个好工作，它拉伸了思想肌肉并让你成为更好的程序员。这看起来
有很差的ROI但如果你没有学习真正的新东西你将停滞并像用10年经验一直重复做
同样事情。

## Where Rust Shines

Rust是个静态的、强类型的系统语言。_静态的_ 意味着所有类型在编译期就知道，
_强_ 类型意味着这些类型设计出来让你很难写错程序。一次成功的编译意味着你
比牛仔语言C有了更好的正确保障。_系统_ 意思是生成了最好的机器代码并完全控制
了内存使用。所以用途非常核心: 操作系统，设备驱动和没有操作系统的嵌入式系统。
尽管如此，它实际上是个非常友好的语言也用来编写常规应用程序。

Rust与C&C++的重大区别是 _默认安全_；所有访问的内存都是检查过的，它不太
可能偶然内存崩溃。

Rust背后的独特原理是:

  - 严格强制对数据的安全借用
  - 用函数，方法和闭包来操作数据
  - 用元组、结构体和枚举类聚合数据
  - 用模式匹配来选择和解构出数据
  - 用特性定义了数据上的 _行为_ 

通过Cargo有快速增长的生态库但这里我们通过学习使用标准库来聚焦于
语言的核心原理。我的建议是写很多小程序，学习直接使用`rustc`是核心
技能。当尝试这里的例子时我们定义了一个脚本`rrun`来编译并运行目标
程序:

```
rustc $1.rs && ./$1
```

## Setting Up

This tutorial assumes that you have Rust installed locally. Fortunately this is
[very straightforward](https://www.rust-lang.org/en-US/downloads.html).

```
$ curl https://sh.rustup.rs -sSf | sh
$ rustup component add rust-docs
```
I would recommend getting the default stable version; it's easy to download
unstable versions later and to switch between.

This gets the compiler, the Cargo package manager, the API documentation, and the Rust Book.
The journey of a thousand miles starts with one step, and this first step is painless.

`rustup` is the command you use to manage your Rust installation. When a new stable release
appears, you just have to say `rustup update` to upgrade. `rustup doc` will open
the offline documentation in your browser.

You will probably already have an editor you like, and [basic Rust support](https://areweideyet.com/)
is good. I'd suggest you start out with basic syntax highlighting at first, and
work up as your programs get larger.

Personally I'm a fan of [Geany](https://www.geany.org/Download/Releases) which is
one of the few editors with Rust support out-of-the-box; it's particularly easy
on Linux since it's available through the package manager, but it works fine on
other platforms.

The main thing is knowing how to edit, compile and run Rust programs.
You learn to program with your _fingers_; type in
the code yourself, and learn to rearrange things efficiently with your editor.

Zed Shaw's [advice](https://learnpythonthehardway.org/book/intro.html) about learning
to program in Python remains good, whatever the language. He says learning to program
is like learning a musical instrument - the secret is practice and persistence.
There's also good advice from Yoga and the soft martial arts like Tai Chi;
feel the strain, but don't over-strain. You are not building dumb muscle here.

I'd like to thank the many contributors who caught bad English or bad Rust for me,
and thanks to David Marino for his cool characterization
of Rust as a friendly-but-hardcore no-nonsense knight in shining armour.

Steve Donovan © 2017-2018 MIT license version 0.4.0

