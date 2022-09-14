# 模块与Cargo

## 模块

当程序变得更大时，有必要扩展功能代码到不同的文件，把函数和类型放到
不同的 _命名空间_。
Rust解决上述问题的方案是 _模块_。

C语言只处理了问题一而没处理问题二，所以你只好用一些可怕的名称如
`primitive_display_set_width`等。实际上文件名可以任意。

在Rust里完整的名称会是 `primitive::display::set_width`，在使用`use primitive::display`
后，你可以直接调用`display::set_width`。你甚至可以用 `use primitive::display::set_width` 
然后直接调用`set_width`，但展开到这个程度并不是好主意。`rustc`不会迷惑，但 _你_
后面可能会迷惑。为了这样，文件名需要遵守一些简单的规则。

一个新的关键字`mod`常被用来定义一个模块，Rust类型和函数可以写成这样:

```rust
mod foo {
    #[derive(Debug)]
    struct Foo {
        s: &'static str
    }
}

fn main() {
    let f = foo::Foo{s: "hello"};
    println!("{:?}", f);
}
```

但这还不对 —— 我们让'struct Foo是私有的'。为此，我们用`pub`关键字来导出`Foo`。
然后错误提示为'struct  foo::Foo的成员是私有的'，所以将`pub`也放在成员`s`前面以
导出`Foo::s`。然后正常了。

```rust
    pub struct Foo {
        pub s: &'static str
    }
```
需要显式的`pub`意味着你需要 _选择_ 让模块里的哪些成员是公开的。从模块里导出的
函数集合和类型被称为它的 _接口_。

它常更好隐藏结构体的内部信息，只允许通过方法来访问:

```rust
mod foo {
    #[derive(Debug)]
    pub struct Foo {
        s: &'static str
    }

    impl Foo {
        pub fn new(s: &'static str) -> Foo {
            Foo{s: s}
        }
    }
}

fn main() {
    let f = foo::Foo::new("hello");
    println!("{:?}", f);
}
```

为什么隐藏实现是个好事？因为这意味着你后面可以修改实现而不破坏接口，不让
一个模块的用户太依赖它的细节。一个大型程序的最大敌人是代码模块被纠缠在
一起，所以难以隔绝的理解某一片代码。

在理想情况下，一个模块只做一件事，做好，并隐藏着自己的秘密。

如果不隐藏呢？就像Bjarne Stroustrup说的，_接口_ 就是实现，如`struct Point{x: f32, y: f32}`。

在一个模块 _内_，每个成员都能相互看见其他成员。这是个舒适的区域在那里成员们
都可以相互交朋友并相互知道亲密的细节。

每个人都会在某点需要将程序修拆分一个个独立的文件，依赖个人的品味。我在程序
达到500行时开始不舒服，我们都同意超过2000行就一定拆分了。

如何拆分程序成多个独立文件？

我们把 `foo` 的代码放到 `foo.rs`里:

```rust
// foo.rs
#[derive(Debug)]
pub struct Foo {
    s: &'static str
}

impl Foo {
    pub fn new(s: &'static str) -> Foo {
        Foo{s: s}
    }
}
```
在主程序中使用 `mod foo` 语句 _不带_ 括号:

```rust
// mod3.rs
mod foo;

fn main() {
    let f = foo::Foo::new("hello");
    println!("{:?}", f);
}
```
现在执行 `rustc mod3.rs` 会让 `foo.rs` 也被编译上。这里不需要愚笨的makefile。

编译器也会检查`MODNAME/mod.rs`，这在我们创建了包含`mod.rs`的`boo` 目录时
发生:

```rust
// boo/mod.rs
pub fn answer()->u32 {
    42
}
```
现在主程序能使用两个模块的独立文件:

```
// mod3.rs
mod foo;
mod boo;

fn main() {
    let f = foo::Foo::new("hello");
    let res = boo::answer();
    println!("{:?} {}", f,res);
}
```
截至目前，这里`mod3.rs`，包含`main`函数，模块 `foo.rs`和包含了`mod.rs`的目录`boo`。
常规约定是包含了`main`函数的文件叫作`main.rs`。

为什么两个方法实现了同样的功能？因为`boo/mod.rs` 能引用其他`boo`模块，更新`boo/mod.rs` 并
加入新模块 —— 注意这里需要显示导出。(没有`pub`，`bar`只能在`boo`模块内可见。)

```rust
// boo/mod.rs
pub fn answer()->u32 {
	42
}

pub mod bar {
    pub fn question() -> &'static str {
        "the meaning of everything"
    }
}
```
那么我们问题的答案是(模块`bar`在模块`boo`内部):

```rust
let q = boo::bar::question();
```

这个模块能提出到 `boo/bar.rs` 文件:

```rust
// boo/bar.rs
pub fn question() -> &'static str {
    "the meaning of everything"
}
```
那么 `boo/mod.rs` 文件变成:

```rust
// boo/mod.rs
pub fn answer()->u32 {
	42
}

pub mod bar;
```
总之，模块是关于如何组织和可见性的，它们可以在一个
或多个文件里。

请注意`use`与导入没有关系，只简单指定了模块名称的可见性。例如:

```rust
{
    use boo::bar;
    let q = bar::question();
    ...
}
{
    use boo::bar::question();
    let q = question();
    ...
}

```
一个重要的点是，这里没有 _独立编译_。main程序和它的模块每次都需要重编译。大型程序
将花费很长时间编译，即便`rustc`在增量编译上越来越好。

## Crates 板块

Rust的'编译单元' 是 _crate_，它既是可执行文件也不是一个库。

为拆分从最后一节编译文件，首先编译`foo.rs`为一个Rust _静态库_ crate:

```
src$ rustc foo.rs --crate-type=lib
src$ ls -l libfoo.rlib
-rw-rw-r-- 1 steve steve 7888 Jan  5 13:35 libfoo.rlib
```
现在我们可以 _链接_ 它到main程序:

```
src$ rustc mod4.rs --extern foo=libfoo.rlib
```
但主程序必须这么看，`extern`名称需要与链接时的名称一样。这里暗示了
顶级模块`foo`与库crate的关系:

```
// mod4.rs
extern crate foo;

fn main() {
    let f = foo::Foo::new("hello");
    println!("{:?}", f);
}
```
在人们开始欢呼'Cargo! Cargo!'前我来解释下Rust底层构建过程。我是'Know Thy Toolchain'
的信徒，这将减少当我们用Cargo管理项目时你需要学习的新魔法。模块是基本的语言特征
且可以用在Cargo项目外。

是时候理解为什么Rust二进制那么大:

```
src$ ls -lh mod4
-rwxrwxr-x 1 steve steve 3,4M Jan  5 13:39 mod4
```
这太胖了！这里有太多debug信息放在执行文件里。这不是坏事，在程序pannic时如果你想调试
并实际上是需要有意义的调用栈。

所以让我们strip一下debug信息会看到:

```
src$ strip mod4
src$ ls -lh mod4
-rwxrwxr-x 1 steve steve 300K Jan  5 13:49 mod4
```
对如此简单的功能仍显得有点大，但这程序静态的链接了Rust标准库。这是好事，因为你可以把
这个可执行程序放到任何同样的操作系统上运行 —— 它们不需要'Rust 运行时'(甚至`rustup`也允许
你为别的操作系统和平台来交叉编译。)

我们可以动态链接不要Rust运行时以得到更小的可执行文件:

```
src$ rustc -C prefer-dynamic mod4.rs --extern foo=libfoo.rlib
src$ ls -lh mod4
-rwxrwxr-x 1 steve steve 14K Jan  5 13:53 mod4
src$ ldd mod4
	linux-vdso.so.1 =>  (0x00007fffa8746000)
	libstd-b4054fae3db32020.so => not found
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3cd47aa000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f3cd4d72000)
```
这里 'not found' 是因为 `rustup` 没有在系统上安装动态库。我们可以hack一下让它工作，
至少在Unix上可以（是的，最好的方法是用符号链接。）
can hack our way to happiness, at least on Unix (yes, I know the best solution is a symlink.)

```
src$ export LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib
src$ ./mod4
Foo { s: "hello" }
```
Rust没有动态链接的_是非_问题，与Go做法相同。只是每6周有个稳定发布不太方便重编译每个模块。
如果你有了稳定可用版本，很好。稳定的Rust版本得到越来越多的操作系统包管理器分发，动态
链接将变得更流行。

## Cargo

相比Java和Python而言Rust标准库不是很大；尽管它相比C或C++有很丰富的功能，后者对操作系统
提供的库依赖很重。

但它能通过 __Cargo__ 很直接的访问社区提供的库 [crates.io](https://crates.io)。Cargo将查询正确的
版本并为你下载源码，并确保其他依赖的库也被正确下载来。

我们来创建一个简单程序读取JSON。这数据格式广泛应用，但包含在标准库里也显得过于特化。我们
创建一个Cargo工程。

```
test$ cargo init --bin test-json
     Created binary (application) project
test$ cd test-json
test$ cat Cargo.toml
[package]
name = "test-json"
version = "0.1.0"
authors = ["Your Name <you@example.org>"]

[dependencies]
```

默认创建一个二进制 (应用) 工程，所以 `--bin` 可选的，用 `--lib` 来创建一个库工程。

为让这个项目依赖 [JSON crate](http://json.rs/doc/json/)，编辑 'Cargo.toml' 文件如下:

```
[dependencies]
json="0.11.4"
```

然后首次用 Cargo 编译:

```
test-json$ cargo build
    Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading json v0.11.4
   Compiling json v0.11.4
   Compiling test-json v0.1.0 (file:///home/steve/c/rust/test/test-json)
    Finished debug [unoptimized + debuginfo] target(s) in 1.75 secs
```
这个项目的主文件已经被创建 —— 就是'src'目录下的'main.rs'。它开始于就像'hello world'应用，
我们来把它编辑成一个合适的测试程序。

注意非常方便的'raw'字符串常量 —— 否则我们将需要丑陋的转移那些双引号:

```rust
// test-json/src/main.rs
extern crate json;

fn main() {
    let doc = json::parse(r#"
    {
        "code": 200,
        "success": true,
        "payload": {
            "features": [
                "awesome",
                "easyAPI",
                "lowLearningCurve"
            ]
        }
    }
    "#).expect("parse failed");

    println!("debug {:?}", doc);
    println!("display {}", doc);
}
```

现在可以编译和执行这个项目 —— 只有`main.rs`被修改过。

```
test-json$ cargo run
   Compiling test-json v0.1.0 (file:///home/steve/c/rust/test/test-json)
    Finished debug [unoptimized + debuginfo] target(s) in 0.21 secs
     Running `target/debug/test-json`
debug Object(Object { store: [("code", Number(Number { category: 1, exponent: 0, mantissa: 200 }),
 0, 1), ("success", Boolean(true), 0, 2), ("payload", Object(Object { store: [("features",
 Array([Short("awesome"), Short("easyAPI"), Short("lowLearningCurve")]), 0, 0)] }), 0, 0)] })
display {"code":200,"success":true,"payload":{"features":["awesome","easyAPI","lowLearningCurve"]}}
```

我们来探索JSON API。如果我们不能抽取值时它很有用。`as_TYPE` 方法返回`Option<TYPE>`因为
我们不确定成员存在或是正确的类型。看看  [docs for JsonValue](http://json.rs/doc/json/enum.JsonValue.html))

```rust
    let code = doc["code"].as_u32().unwrap_or(0);
    let success = doc["success"].as_bool().unwrap_or(false);

    assert_eq!(code, 200);
    assert_eq!(success, true);

    let features = &doc["payload"]["features"];
    for v in features.members() {
        println!("{}", v.as_str().unwrap()); // MIGHT explode
    }
    // awesome
    // easyAPI
    // lowLearningCurve
```
这里`features` 是指向 `JsonValue`的引用　——　它须是一个引用因为否则我们将尝试移动
一个 _值_ 到JSON文档外。这里我们知道它是个数组，所以`members()`将返回遍历`&JsonValue`
的非空迭代器。

如果'payload'对象没有'features'键值将发生什么？那么`features`将被设为`Null`。将不会爆。这个
便利表达了自由格式，任何JSON本质很好。这取决于你检查任何收到或创建的文档的结构，如果
不匹配返回自己定义的错误。

你可以修改那些结构体。如果我们用`let mut doc`那么这将执行如你期望的:

```rust
    let features = &mut doc["payload"]["features"];
    features.push("cargo!").expect("couldn't push");
```
`push` 将在  `features`不是个数组时失败，因此返回 `Result<()>`。

这里是个真正漂亮的宏以生成 _JSON常量_:

```rust
    let data = object!{
        "name"    => "John Doe",
        "age"     => 30,
        "numbers" => array![10,53,553]
    };
    assert_eq!(
        data.dump(),
        r#"{"name":"John Doe","age":30,"numbers":[10,53,553]}"#
    );
```
若需要这个，你应显式的从JSON crate里导入宏:

```rust
#[macro_use]
extern crate json;
```
用这个crate也有个缺点，因为无定形的、动态类型的JSON特性与结构化的、静态特性的Rust之间不匹配。
(说明文件中显式的'friction') 所以如果你 _确实_ 想映射JSON到Rust数据结构，你需要不再做很多检查，因为
你无法假设收到的结构体匹配你的！因此，更好的解决方案是 [serde_json](https://github.com/serde-rs/json),
这里你可以 _序列化_ Rust数据结构成JSON格式并反序列化回来。

由此，用 `cargo new test-serde-json` 创建用另一个Cargo二进制项目，到`test-serde-json`目录里编辑
`Cargo.toml`文件。如下；

```
[dependencies]
serde="0.9"
serde_derive="0.9"
serde_json="0.9"
```

然后编辑 `src/main.rs` 成这样:

```rust
#[macro_use]
extern crate serde_derive;
extern crate serde_json;

#[derive(Serialize, Deserialize, Debug)]
struct Person {
    name: String,
    age: u8,
    address: Address,
    phones: Vec<String>,
}

#[derive(Serialize, Deserialize, Debug)]
struct Address {
    street: String,
    city: String,
}

fn main() {
    let data = r#" {
     "name": "John Doe", "age": 43,
     "address": {"street": "main", "city":"Downtown"},
     "phones":["27726550023"]
    } "#;
    let p: Person = serde_json::from_str(data).expect("deserialize error");
    println!("Please call {} at the number {}", p.name, p.phones[0]);

    println!("{:#?}",p);
}
```
你在前面看到过`derive`属性，但`serde_derive` crate为特殊的`Serialize` 和 `Deserialize` trait
定义了 _custom derives_。结果显示了生成的Rust结构体:

```
Please call John Doe at the number 27726550023
Person {
    name: "John Doe",
    age: 43,
    address: Address {
        street: "main",
        city: "Downtown"
    },
    phones: [
        "27726550023"
    ]
}
```
现在，如果你采用`json` crate来实现这个，你需要几百行自己编写的代码，很多错误处理。太可怕，
容易搞得一团糟，这不是你想花费精力的地方。

这明显是你处理定义良好的外部JSON文件的最好方案(如果需要可以重映射字段名)并为Rust程序
提供了健壮的方式来与其他程序透过网络共享数据(因为大家都能理解JSON。) `serde`最酷的事
(SERialization DEserializatio)是也支持其他文件格式，如`toml`，它是Cargo用的友好配置文件。
所以你的程序能读取.toml文件到结构体，将结构体写到.json文件。

序列化是个重要的技术且与Java和Go中的存在解决方案相似 —— 但也有个大的区别。在那些语言
中结构体的数据是在 _ 运行时_ 通过 _反射_ 发现，但在序列化场景代码是在 _编译时_ 生成 —— 一起更有效!
Serialization is an important technique and similar solutions exist for Java and Go - but with a big
difference. In those languages the structure of the data is found at _run-time_ using _reflection_, but
in this case the serialization code is generated at _compile-time_ - altogether more efficient!

Cargo被认为是对Rust生态的最好的加强之一，因为它为我们做了很多工作。否则我们将不得不
从Github来下载那些库，编译成静态的crates，链接到程序中。在C++项目中做这些事很痛苦，
如果没有Cargo在Rust项目里做这些也是痛苦的。这里C++特别独特的痛苦是，我们需要比较
它与其他语言的包管理器。npm(Javascript的)和pip(Python的)管理依赖并为你下载库，但分发
比较难，你的程序用户需要安装NodeJS或Python。但Rust程序是静态链接它们的依赖库，它们
能被你的伙伴无外部依赖的分发使用。

## 更多风景

当处理任何不简单的文本时，正则表达式让你轻松点。这些是大多数语言中的共同需要，我将
假设基本熟悉regex符号。为使用[regex](https://github.com/rust-lang/regex) crate，把 'regex = "0.2.1"'
放到Cargo.toml文件的 "[dependencies]"节。

我们将使用'raw strings'以不需要转移反斜杠。用英语说，正则表达式是"匹配2个数字，字符':'，
然后任意数字。捕获这些数字":

```rust
extern crate regex;
use regex::Regex;

let re = Regex::new(r"(\d{2}):(\d+)").unwrap();
println!("{:?}", re.captures("  10:230"));
println!("{:?}", re.captures("[22:2]"));
println!("{:?}", re.captures("10:x23"));
// Some(Captures({0: Some("10:230"), 1: Some("10"), 2: Some("230")}))
// Some(Captures({0: Some("22:2"), 1: Some("22"), 2: Some("2")}))
// None
```
正确的输出包含了3个 _捕获内容_ —— 整体匹配，两个数字集合。正则表达式不是默认 _锚定的_，
所以 __regex__ 需要搜寻第一次发生的匹配，跳过任何不匹配的。(如果你丢掉'()'，它将返回整
个匹配。)

可能命名那些捕获内容，并扩展正则表达式到多行，甚至包含注释！编译正则也许失败
(第一个 _expect_)或匹配失败(第二个 _expect_)。这里我们能作为一个联合数组来使用结果并
用名称匹配。

```rust
let re = Regex::new(r"(?x)
(?P<year>\d{4})  # the year
-
(?P<month>\d{2}) # the month
-
(?P<day>\d{2})   # the day
").expect("bad regex");
let caps = re.captures("2010-03-14").expect("match failed");

assert_eq!("2010", &caps["year"]);
assert_eq!("03", &caps["month"]);
assert_eq!("14", &caps["day"]);
```

正则表达式将用模式打破一个字符串，但不会检查是否有意义。也就是说，你能指定和匹配ISO风格的
日期 _语法_，但在 _语义上_ 它们也许没意义，如 "2014-24-52"。

由此，你需要专用的日期时间处理，这由[chrono](https://github.com/lifthrasiir/rust-chrono)提供。
你在处理日期前需要决定时区:

```rust
extern crate chrono;
use chrono::*;

fn main() {
    let date = Local.ymd(2010,3,14);
    println!("date was {}", date);
}
// date was 2010-03-14+02:00
```
尽管如此，不推荐用这个因为给它错误日期时会报错panic! (尝试一个假的日期试试)
这里你需要的方法是`ymd_opt` 将返回 `LocalResult<Date>`.

```rust
    let date = Local.ymd_opt(2010,3,14);
    println!("date was {:?}", date);
    // date was Single(2010-03-14+02:00)

    let date = Local.ymd_opt(2014,24,52);
    println!("date was {:?}", date);
    // date was None
```
你也可以直接解析日期时间，或用标准UTC格式或用自定义[格式](https://lifthrasiir.github.io/rust-chrono/chrono/format/strftime/index.html#specifiers)。
那些自相似格式允许你用自己想要的格式打印出日期。

我特别标黄那些有用的crates因为在其他语言中它们可能会成为标准库的一部分。所以，实际上，
那些没成熟的crates曾是Rust标准库的一部分，但摆脱了影响。这是个小心翼翼的决定:
Rust团队严肃对待标准库的稳定性，所以功能只有稳定后它们才能通过用在不稳定的nightly版的
孵化期。那些需要实验和重构的库，它们最好保持独立并被Cargo跟踪。为实践目的，那两个
crates _是_ 标准的 —— 它们不会离开并将在某点合并进标准库。
