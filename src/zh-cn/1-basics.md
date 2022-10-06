# Basics

## Hello, World!

"hello world" 最初的目的，甚至在第一个C语言的版本编写时就有了，就是为测试
编译器和实际程序。

```rust
// hello.rs
fn main() {
    println!("Hello, World!");
}
```

```
$ rustc hello.rs
$ ./hello
Hello, World!
```

Rust是一个有大括号和分号的语言，C++风格的注释和 `main` 函数 —— 到目前为止
都很相似。感叹号符号暗示它是个 _宏_ 调用。对C++程序员来说，这是无趣的事，
它们来自于严重愚蠢的C语言宏定义——但我可以向你保证这里的宏会更强更明智。

对另外些人，可能是“很好，现在我不得不面对惊喜！” 尽管如此，编译器并不总是
有用；如果你不带上感叹号，会得到如下错误:

```
error[E0425]: unresolved name `println`
 --> hello2.rs:2:5
  |
2 |     println("Hello, World!");
  |     ^^^^^^^ did you mean the macro `println!`?

```

学一个语言就意味着要习惯它的错误。试着将编译器看作一个严格而友好的朋友而不是
一个 _向你射击_ 的电脑，因为你在刚开始会看到很多红线注释。让编译器在编译时抓
住你的错误要比你的程序在真的用户面前爆错要好得多。

下一步是介绍 _变量_:

```rust
// let1.rs
fn main() {
    let answer = 42;
    println!("Hello {}", answer);
}

```

错误拼写是 _编译_ 错误，不是如在其他动态语言Python或Javascript里的运行时错误。
这会降低你很多后期的压力！如果我把 'answer'写成 'answr'，编译器会 _友好_ 的提示:

```
4 |     println!("Hello {}", answr);
  |                         ^^^^^ did you mean `answer`?

```

宏 `println!` 使用 [format string](https://doc.rust-lang.org/std/fmt/index.html)
和一些值；它非常像Python 3用的格式化。

另一个非常有用的宏是 `assert_eq!`。 它是Rust中的测试老黄牛；你 _断言_ 
两个值应相等，如不它们不相等， _报错_。

```rust
// let2.rs
fn main() {
    let answer = 42;
    assert_eq!(answer,42);
}
```

它不是产生任何输出。如果将answer修改为 40 时:

```
thread 'main' panicked at
'assertion failed: `(left == right)` (left: `42`, right: `40`)',
let2.rs:4
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```
这是我们在Rust中的第一个 _运行时错误_ 。

## Looping and Ifing

任何有趣的事可以再做一次:

```rust
// for1.rs
fn main() {
    for i in 0..5 {
        println!("Hello {}", i);
    }
}
```

这里 _range_ 只包含一部分范围，所以 `i` 从 0 到 4 变化。这在语言里用起来很方便就像
一个从0开始的数组 _索引_ 一样。

有趣的事需要在 _一定的条件下_ 进行:

```rust
// for2.rs
fn main() {
    for i in 0..5 {
        if i % 2 == 0 {
            println!("even {}", i);
        } else {
            println!("odd {}", i);
        }
    }
}
```

```
even 0
odd 1
even 2
odd 3
even 4
```

`i % 2`  是0如果2 整除 `i` ；Rust使用了C风格的操作符。这里的条件语句上
_没有_ 小括号，这有点像Go，但你 _必须_ 对多语句块使用大括号。

下面用更有趣的方式编写了实现相同功能的语句:

```rust
// for3.rs
fn main() {
    for i in 0..5 {
        let even_odd = if i % 2 == 0 {"even"} else {"odd"};
        println!("{} {}", even_odd, i);
    }
}
```

在传统上，编程语言 _语句_  (如 `if`) 和 _表达式_ (如 `1+i`)。在Rust中，几乎每个东西都
有对应值并能成为一个表达式。严重丑陋的C语言的 '三元操作符' `i % 2 == 0 ? "even" : "odd"`
不再被需要。

注意这里的代码块内并没有任何分号 (因为该语句是返回值)。

## Adding Things Up

计算机很擅长数值计算。这里第一次尝试累加从0到4的所有数字:

```rust
// add1.rs
fn main() {
    let sum = 0;
    for i in 0..5 {
        sum += i;
    }
    println!("sum is {}", sum);
}
```

但编译失败了:

```
error[E0384]: re-assignment of immutable variable `sum`
 --> add1.rs:5:9
3 |     let sum = 0;
  |         --- first assignment to `sum`
4 |     for i in 0..5 {
5 |         sum += i;
  |         ^^^^^^^^ re-assignment of immutable variable

```

'Immutable'? 一个变量不能 _修改_?  `let` 定义变量默认只能在声明时被赋值。
添加关键字 `mut` (_请让_ 这变量可以被修改) 以实现这个小目的:

```rust
// add2.rs
fn main() {
    let mut sum = 0;
    for i in 0..5 {
        sum += i;
    }
    println!("sum is {}", sum);
}
```
习惯于其他语言的人会很迷惑，因为别的语言里变量值可修改是很默认行为。
'变量'的本质含义是它能被赋于一个运行时计算的值 —— 它不是个_常量_。这
也是它在数学里的函数，如我们说'让n是集合S中的一个最大数字'。

let声明的变量默认 _只可读_ 是有理由的。在大型程序中，很难跟踪在哪修改了
变量。所以Rust让可修改属性('修改能力')成为显式的。这语言里有很多智能
的地方，但它尝试不让任何事隐藏着发生。

Rust即是静态类型也是强类型的语言 —— 这常看起来迷惑，但考虑下C语言
(它是静态类型但也是弱类型) 和Python语言(它动态类型但也是强类型)。
静态类型是指在编译期就知道类型了，动态类型是指仅在运行时才知道类型。

这时，尽管如此，看起来好像Rust在帮你 _处理_ 那些类型。那 `i` 的类型是什么？
编译器能够搞清楚它，开始于0，通过 _类型推导_，得出是`i32`(4字节有符号整数。)

让我们稍微改改 —— 把 `0` 改成 `0.0`。那么得到如下错误:

```
error[E0277]: the trait bound `{float}: std::ops::AddAssign<{integer}>` is not satisfied
 --> add3.rs:5:9
  |
5 |         sum += i;
  |         ^^^^^^^^ the trait `std::ops::AddAssign<{integer}>` is not implemented for `{float}`
  |

```

好吧，蜜月结束: 这是什么意思？每个操作符(如`+=`)对应着一个 _trait_ ，就像一个
必须要具体类型实现的抽象接口。我们将在后面处理traits的细节，但这里你只需要
知道`AddAssign`是让整数类型实现`+=`操作符的trait，这里的错误是指浮点数没有
实现以整数作为该操作符的功能。(操作符的完整清单在[这里](https://doc.rust-lang.org/std/ops/index.html)。)


再次说，Rust喜欢显式 —— 它不会帮你悄悄的把整数转换成浮点数。

我们需要显式把那值 _转换_ 成浮点数。

```rust
// add3.rs
fn main() {
    let mut sum = 0.0;
    for i in 0..5 {
        sum += i as f64;
    }
    println!("sum is {}", sum);
}
```

## Function Types are Explicit

_函数_ 也是编译器不会帮你推导出类型的地方。这实际上是个小心翼翼的决定，
比如Haskell语言里几乎没有任何显式类型因它有类型推断能力。Haskell风格
的函数显式类型签名也很好。Rust需要这些。

这是个用户定义函数:

```rust
// fun1.rs

fn sqr(x: f64) -> f64 {
    return x * x;
}

fn main() {
    let res = sqr(2.0);
    println!("square is {}", res);
}
```

Rust回到一个更老的参数声明风格，类型跟在名称后。这是Algol驱动的语言如Pascal所
采用的方式。

再次说，没有整数到浮点数的转换 —— 如果你把`2.0`换成`2`将得到明确报错:

```
8 |     let res = sqr(2);
  |                   ^ expected f64, found integral variable
  |
```

你将很少看到函数里有`return`语句。更常见的是这样:

```rust
fn sqr(x: f64) -> f64 {
    x * x
}
```

这是因为函数体(`{}`里)的最后一条表达式有值，就像if-as-an-expression。

因分号会被手指半自动的插入进来，你也许会添加它，将报以下错误:

```
  |
3 | fn sqr(x: f64) -> f64 {
  |                       ^ expected f64, found ()
  |
  = note: expected type `f64`
  = note:    found type `()`
help: consider removing this semicolon:
 --> fun2.rs:4:8
  |
4 |     x * x;
  |       ^

```

`()` 类型是空类型，没什么， `void`，nothing。Rust里所有东西都有值，
但有时就是nothing。编译器知道这是个常见错误，所以在 _帮_ 你插入返回值。
(任何人如果花了很多时间在C++编译器上将知道 _damn unusual_ 是什么)

更多没有return的表达式例子如下:

```rust
// absolute value of a floating-point number
fn abs(x: f64) -> f64 {
    if x > 0.0 {
        x
    } else {
        -x
    }
}

// ensure the number always falls in the given range
fn clamp(x: f64, x1: f64, x2: f64) -> f64 {
    if x < x1 {
        x1
    } else if x > x2 {
        x2
    } else {
        x
    }
}
```

写上 `return` 也不会错，但没它代码将更清晰。如果想 _早点返回_ 你将仍会用到 `return`。

有些操作能更优雅的 _递归_ 表达:

```rust
fn factorial(n: u64) -> u64 {
    if n == 0 {
        1
    } else {
        n * factorial(n-1)
    }
}
```
初看有点奇怪，最好拿起笔和纸来做几个例子。尽管做这样的操作常常不是最 _有效率_的。

也可以用引用来传参。引用通过 `&` 创建并用 `*` 来解引用。

```rust
fn by_ref(x: &i32) -> i32{
    *x + 1
}

fn main() {
    let i = 10;
    let res1 = by_ref(&i);
    let res2 = by_ref(&41);
    println!("{} {}", res1,res2);
}
// 11 42
```
怎样能让一个函数修改它的参数呢？输入 _mutable引用_ :

```rust
// fun4.rs

fn modifies(x: &mut f64) {
    *x = 1.0;
}

fn main() {
    let mut res = 0.0;
    modifies(&mut res);
    println!("res is {}", res);
}
```
这是C做得比C++多的地方。你需要显式的传递引用符号(用 `&`)并显式的 _解引用_
(用 `*`)。然后再放入 `mut` 因为它默认不是可修改的。(我感觉C++的引用太简洁而不能
与C比较。)

基本上，Rust引入了一些 _分歧_ 到这里，不那么微妙的推动你直接从函数返回值。
幸运的是Rust有强大的方式来表达如"操作成功这是结果"所以 `&mut` 并不是常需要用到。
传引用在我们使用大对象不希望发生拷贝时是很有用的。

类型在变量后的风格也用在`let`语句上，当你真想弄清变量的类型时:

```rust
let bigint: i64 = 0;
```

## Learning Where to Find the Ropes

是时候开始用文档了。这将把文档装在你的机器上，可用`rustup doc --std`将打开
浏览器。

注意 _search_ 框在顶部，这是你好帮手；它可以在本地工作。

让我们看看数学函数在哪，搜搜 'cos'。前两个提示显示它为单双精度浮点数
定义了两个函数。它用了 _值itself_ 以作为一个自身方法，如下:

```rust
let pi: f64 = 3.1416;
let x = pi/2.0;
let cosine = x.cos();
```
结果会是0类型；我们明显需要一个pi-ness代码的授权。

(为什么我需要显式的 `f64` 类型？没有它，常量可以是 `f32` 或 `f64`，它们区别很大)

我们引用 `cos` 的例子，写得更完整一点( `assert!` 是 `assert_eq!` 的表兄；该表达式
返回true):

```rust
fn main() {
    let x = 2.0 * std::f64::consts::PI;

    let abs_difference = (x.cos() - 1.0).abs();

    assert!(abs_difference < 1e-10);
}
```
`std::f64::consts::PI` 很拗口! `::` 与其在C++中的意思一样，
(其他有些语言里会写成 '.') —— 这是个 _有资格的名称_。
我们在搜 `PI` 时会得到完整的名称。

截至目前，我们的小的Rust程序没有任何 `import` 和 `include` 等将拖慢探讨
'Hello World'程序。让我们加上 `use` 语句让程序的可读性更好:

```rust
use std::f64::consts;

fn main() {
    let x = 2.0 * consts::PI;

    let abs_difference = (x.cos() - 1.0).abs();

    assert!(abs_difference < 1e-10);
}
```
为什么我不需要这么做？因为Rust通过 _prelude_ 让一些基本可见的功能不必显式
用到 `use` 语句。

## Arrays and Slices

所有的静态类型语言有 _数组_，它的值全部在内存中。数组的 _索引_ 从0开始:

```rust
// array1.rs
fn main() {
    let arr = [10, 20, 30, 40];
    let first = arr[0];
    println!("first {}", first);

    for i in 0..4 {
        println!("[{}] = {}", i,arr[i]);
    }
    println!("length {}", arr.len());
}
```
输出如下:

```
first 10
[0] = 10
[1] = 20
[2] = 30
[3] = 40
length 4
```

在这例子中，Rust _清楚_ 知道数组有多大，如果你尝试访问 `arr[4]` ，
将出现 _编译错误_ 。

学习新语言时常从其他已知的语言中引入不再学的思维习惯；如果你是Python
开发者，那些组件即 `List`。我们将很快带来Rust的等价 `List`，但数组不是
你要的替代品；它们是 _固定大小_。他们可以被修改(合理调用)但你不能添加新元素。

在Rust里不常用数组，因为数组的类型包括了大小。这个数组的类型是 `[i32; 4]`；
 `[10, 20]`的类型是`[i32; 2]`依次类推: 它们的类型不同。所以它们被用作函数参数是很奇怪。

_实际_ 常用到的是 _切片_。你可以把它们看作是底层数组值的 _视图_。
它们在行为上很像数组，知道自己的 _大小_，不像C指针这种危险物种。

注意两个重要的点 —— 怎么写切片的类型，你需要用`&`来把它传递到函数里。

```rust
// array2.rs
// read as: slice of i32
fn sum(values: &[i32]) -> i32 {
    let mut res = 0;
    for i in 0..values.len() {
        res += values[i]
    }
    res
}

fn main() {
    let arr = [10,20,30,40];
    // look at that &
    let res = sum(&arr);
    println!("sum {}", res);
}
```
先忽略一会儿 `sum` 的代码，看看 `&[i32]`。Rust数组和切片的关系就像C语言中
数组与指针的关系一样，但还有2个重要区别 —— Rust的切片会跟踪数组长度(如果
你尝试访问超出数组长度的值它将报错)，且你需要显式的采用 `&` 操作符来把数组
作为切片传递。

一个C程序员对 `&` 读作'取地址‘；而Rust程序员则读作'借用’。这是学Rust时的关键词。
借用是编程语言中的普遍模式的名称；无论何时你传递引用时(动态语言里常发生)或
在C里传一个指针。任何被借用的变量仍是原ower持有。

## Slicing and Dicing 切片和切丁

你不能使用普通的 `{}` 来打印出数组，但可以通过 `{:?}` 来打印调试信息。

```rust
// array3.rs
fn main() {
    let ints = [1, 2, 3];
    let floats = [1.1, 2.1, 3.1];
    let strings = ["hello", "world"];
    let ints_ints = [[1, 2], [10, 20]];
    println!("ints {:?}", ints);
    println!("floats {:?}", floats);
    println!("strings {:?}", strings);
    println!("ints_ints {:?}", ints_ints);
}
```
它们输出:
Which gives:

```
ints [1, 2, 3]
floats [1.1, 2.1, 3.1]
strings ["hello", "world"]
ints_ints [[1, 2], [10, 20]]
```
所以用数组的数组也没问题，但要点是数组只能包含 _唯一_ 相同的元素类型。
数组里的值被安排在相邻的内存中所以访问起来 _非常_ 高效。

如果你对那些变量的实际类型好奇，这里有个有效的小技巧。先声明一个你
知道是错误类型的变量:

```rust
let var: () = [1.1, 1.2];
```
然后得到如下错误信息:

```
3 |     let var: () = [1.1, 1.2];
  |                   ^^^^^^^^^^ expected (), found array of 2 elements
  |
  = note: expected type `()`
  = note:    found type `[{float}; 2]`
```
(`{float}` means 'some floating-point type which is not fully specified yet')

切片给你 _同一个_ 数组的不同 _视图_:

```rust
// slice1.rs
fn main() {
    let ints = [1, 2, 3, 4, 5];
    let slice1 = &ints[0..2];
    let slice2 = &ints[1..];  // open range!

    println!("ints {:?}", ints);
    println!("slice1 {:?}", slice1);
    println!("slice2 {:?}", slice2);
}
```

```
ints [1, 2, 3, 4, 5]
slice1 [1, 2]
slice2 [2, 3, 4, 5]
```

这是像Python切片一样的整洁符号，但也有个大的区别:
这数据永远不能克隆。那些切片全是从原数组  _借用_ 的数据。它们与数组有有非常紧密的
联系，Rust花费很大努力去保证联系还在。

## Optional Values 可选值

切片，像数组一样能被索引访问。Rust在编译时知道数组的大小，但切片的大小
只能在运行时知道。所以 `s[i]` 能引起越界错误，执行到这会 _pannic_。这肯定
不是你希望看到的 —— 它就像安全发出取消发射指令和非常昂贵的零星碎片遍布弗罗里达
区别一样大。那里 _没有异常_。

领会那些，因为它的出现总是带来震惊。你不能包装可能会崩的代码在一些try块再
'抓住错误'  —— 至少不能用你不想每天看见的方式。所以Rust如何保障安全？

切片有个方法 `get` 不会崩。但它会返回什么呢？

```rust
// slice2.rs
fn main() {
    let ints = [1, 2, 3, 4, 5];
    let slice = &ints;
    let first = slice.get(0);
    let last = slice.get(5);

    println!("first {:?}", first);
    println!("last {:?}", last);
}
// first Some(1)
// last None
```
`last` 失败(假如我们忽略了数组是从0索引开始的)，但它返回了 `None`。
`first` 可以，但好像返回了一个包装在 `Some` 里的值。欢迎 `Option` 类型！
它可以是 `Some` 或者 `None`。

类型 `Option`有些有用的方法:

```rust
    println!("first {} {}", first.is_some(), first.is_none());
    println!("last {} {}", last.is_some(), last.is_none());
    println!("first value {}", first.unwrap());

// first true false
// last false true
// first value 1
```
如果将 `last` _打开包装_，你会得到崩溃。但至少你可以先调用 `is_none` 来确认
—— 例如，你准备了一个别的默认值来返回:

```rust
    let maybe_last = slice.get(5);
    let last = if maybe_last.is_some() {
        *maybe_last.unwrap()
    } else {
        -1
    };
```
注意 `*` ——在 `Some` 里的精确类型 `&i32` 是引用。我们需要解引用它来得到一个 `i32` 的值。

这是冗长的，所以有快捷方式 —— `unwrap_or` 将返回值或 `None` 时的预设值。
注意类型需要匹配—— `get` 返回引用。你需要返回 `&32` 类的预设值 `&-1`。最后，
注意使用 `*` 来解出 `i32` 类型的值。

```rust
    let last = *slice.get(5).unwrap_or(&-1);
```
很容易忽略 `&`，但编译器会带会你到这。如果是 `-1`，`rustc`会说
TODO

你可以把 `Option` 看作为一个包含了值的盒子，或者nothing(`None`)。
(在Haskell里叫`Maybe`)。它可以包含 _任何_ 类型的值，也就是它的
_类型参数_。在这例子，完整的类型是 `Option<&i32>`，为泛型用了
C++风格的符号。对盒子解包装将引起一个露出，但不像薛定谔的猫，
我们能提前知道它包含值。

Rust函数/方法返回这样的maybe-boxes很常见，所以学学如何舒服的
[使用](https://doc.rust-lang.org/std/option/enum.Option.html) 

## Vectors 向量

我们将再返回到切片方法，但首先: 向量。它们是  _可变大小_ 的数组且行为很像
Python的`List`和C++的`std::vector`。Rust类型`Vec`(读作'vector')行为实际很像切片；
区别是你可以添加额外数据到vector中 —— 注意它需要被声明为mutable。

```rust
// vec1.rs
fn main() {
    let mut v = Vec::new();
    v.push(10);
    v.push(20);
    v.push(30);

    let first = v[0];  // will panic if out-of-range
    let maybe_first = v.get(0);

    println!("v is {:?}", v);
    println!("first is {}", first);
    println!("maybe_first is {:?}", maybe_first);
}
// v is [10, 20, 30]
// first is 10
// maybe_first is Some(10)
```
一个常见的初学者问题是忘了`mut`关键字；你将得到有用的错误信息:

```
3 |     let v = Vec::new();
  |         - use `mut v` here to make mutable
4 |     v.push(10);
  |     ^ cannot borrow mutably
```
vectors和切片间有亲密的关系:

```rust
// vec2.rs
fn dump(arr: &[i32]) {
    println!("arr is {:?}", arr);
}

fn main() {
    let mut v = Vec::new();
    v.push(10);
    v.push(20);
    v.push(30);

    dump(&v);

    let slice = &v[1..];
    println!("slice is {:?}", slice);
}
```
那个小小的、非常重要的借用操作符 `&` 是把向量 _强制_ 转换成切片。
这完全可以，是因为向量也管理一个数组，区别是这个数组是_动态_分配的。

如果你用过动态语言，是时候谈点了。在系统语言中，分配内存有两种类型：栈上内存和堆上内存。
在栈上分配内存是很快的，但栈有限制；典型的是内存大小限制在MB级。堆上的内存能到GB级，
但分配内存相对耗时，且内存不用后需要主动释放。在号称有'内存管理'的语言中（如Java，Go，
以及那些叫作'scripting'的语言）那些内存细节是用被称为垃圾回收的方便自治组件对你隐藏着的。
一旦系统确认一些数据不再被其他引用，将会把这些数据内存释放回可用的内存池子里。

一般来说，这是值得付出的代价。与栈打交道是可怕的不安全，因为一旦你出错可能覆盖当前
程序的返回地址，那你的程序将会丢脸的崩溃或（更糟）被人恶意攻破。

我写得第一个C程序（在DOS PC上）搞垮了整个电脑。Unix系统常表现得更好，程序进程崩溃
时只报 _段错误_。为什么这比Rust（或Go）程序pannicking严重？因为panic在原始程序
中发生，而不是在程序变得毫无希望的迷惑消耗掉你全部精力时。Pannic是 _内存安全的_ 因为
它们在任何内存非法访问前出现。这却是C语言的安全问题根源，因为所有的内存访问都是不
安全的且狡诈的攻击者可以利用这个薄弱环节。

Panick听起来严重且意外，但Rust的panic是有结构的 —— 栈是 _轻松的_ 就像普通异常。
所有分配的对象将被废弃，并将生成一个调用过程。

垃圾回收的缺点是什么？首先它会浪费内存，这在那些越来越主宰世界的小的嵌入式芯片会要紧。
其二它将决定在最差可能性时清理内存只能立刻开始。那些嵌入式系统需要实时响应('real-time')
且不能容忍非计划的内存清理打断任务。最优雅的动态语言Lua的首席设计者Roberto Ierusalimschy
曾说他不喜欢乘坐在依靠有垃圾回收软件的飞机上。

回到向量：当一个向量被修改或创建，在堆上分配了它并成为那片内存的 _主人_。切片 _借用_
了向量的内存。当向量被 _废弃_ 时，它会释放内存。

## Iterators 迭代器

截至目前我们探讨了这么多却没有提及Rust的一个关键谜语 —— 迭代器。
使用了范围的for-loop就用到了迭代器(`0..n`实际上类似于Python 3的`range`函数）。

迭代器很容易被定义得不正式。它是一个有能返回 `Option` 值的 `next` 方法的'对象'。
当返回值还不是 `None` 时，我们可继续调用 `next`:

```rust
// iter1.rs
fn main() {
    let mut iter = 0..3;
    assert_eq!(iter.next(), Some(0));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), None);
}
```
这就是那 `for var in iter {}` 的作用。

定义一个for-loop看起来不是个高效的方式，但 `rustc` 在release模式上做了疯狂的优化，
让它与 `while` 循环一样快。

下面是第一次尝试迭代一个数组：

```rust
// iter2.rs
fn main() {
    let arr = [10, 20, 30];
    for i in arr {
        println!("{}", i);
    }
}
```
编译失败，但提示有用：
```
4 |     for i in arr {
  |     ^ the trait `std::iter::Iterator` is not implemented for `[{integer}; 3]`
  |
  = note: `[{integer}; 3]` is not an iterator; maybe try calling
   `.iter()` or a similar method
  = note: required by `std::iter::IntoIterator::into_iter`
```
遵守 `rustc` 的建议，用以下方式就可以了。

```rust
// iter3.rs
fn main() {
    let arr = [10, 20, 30];
    for i in arr.iter() {
        println!("{}", i);
    }

    // slices will be converted implicitly to iterators...
    let slice = &arr;
    for i in slice {
        println!("{}", i);
    }
}
```
实际上，用上面的方式去迭代一个数组或切片会比 `for i in 0..slice.len() {}` 有用且更有效，
因为Rust不需要着魔的检查每个索引操作。

我们前面有个例子来加和一组整数。它引入了 `mut` 变量和一个循环。这里是符合语言习惯
的、专业的实现：

```rust
// sum1.rs
fn main() {
    let sum: i32  = (0..5).sum();
    println!("sum was {}", sum);

    let sum: i64 = [10, 20, 30].iter().sum();
    println!("sum was {}", sum);
}
```
注意，这是一个例子关于你需要显式使用变量 _类型_，因除此之外Rust没有足够的信息。
这里我们用了不同整数类型来加和，没问题。（创建新变量用了已有名称。）

有了相关背景知识，了解更多 [slice methods](https://doc.rust-lang.org/std/primitive.slice.html)
会更有意义。

(另一个文档提示；在每页的右边有个'[-]'可以点击来收起方法列表。你可以展开看起来有趣的接口的细节。
任何看起来奇怪的可以先忽略掉)

`windows` 方法可以给你一个切片的迭代器 —— 重叠着多个值！

```rust
// slice4.rs
fn main() {
    let ints = [1, 2, 3, 4, 5];
    let slice = &ints;

    for s in slice.windows(2) {
        println!("window {:?}", s);
    }
}
// window [1, 2]
// window [2, 3]
// window [3, 4]
// window [4, 5]
```
Or `chunks`:

```rust
    for s in slice.chunks(2) {
        println!("chunks {:?}", s);
    }
// chunks [1, 2]
// chunks [3, 4]
// chunks [5]
```

## More about vectors... 更多关于vector的知识

有个有用的小宏定义 `vec!` 用来初始化一个vector。注意你可以用`pop`从vector的结尾 _删除_
元素,可以使用兼容迭代器来 _扩展_ 一个vector。

```rust
// vec3.rs
fn main() {
    let mut v1 = vec![10, 20, 30, 40];
    v1.pop();

    let mut v2 = Vec::new();
    v2.push(10);
    v2.push(20);
    v2.push(30);

    assert_eq!(v1, v2);

    v2.extend(0..2);
    assert_eq!(v2, &[10, 20, 30, 0, 1]);
}
```
Vectors可以用切片来相互比较。

你可以用 `insert` 来向vector任意位置插入新值，能用 `remove` 来删除值。这不太像
`push` 和 `pop` 一样高效因为它们会移动很多值来调整空间，当心那些操作发生在大的vector上。

Vector有大小和 _容量_。如果你 _清理_了一个vector，它的大小将变成0，但它仍然保留了旧的容量。
所以可以用 `push` 重新填充它，只会在填充大小超过容量时重新分配内存。

Vector可以被排序，且重复的元素会被删除 —— 那些操作在vector内执行。（如果你想要一个拷贝，
用`clone`。）

```rust
// vec4.rs
fn main() {
    let mut v1 = vec![1, 10, 5, 1, 2, 11, 2, 40];
    v1.sort();
    v1.dedup();
    assert_eq!(v1, &[1, 2, 5, 10, 11, 40]);
}
```

## Strings 字符串

Rust中字符串被引入得比其他语言更多；`String` 类型，像 `Vec`，动态分配并可
修改大小。(它很像C++的 `std::string` 但不像Java和Python中不可修改的字符串。)
程序中可以包含许多 _字符串常量_(如"hello")且系统语言能将它们保持在可执行
文件里。在嵌入式系统中，这意味着将字符串保存在便宜的ROOM而不是昂贵的
RAM(对低功耗设备而言RAM因能耗而显得昂贵。) 一个 _系统_ 语言需要有2种
字符串，动态分配的和静态的。

所以"hello"不是 `String` 类型。它是 `&str` 类型(读作'string slice')。它像C++中 `const char*`
与 `std::string` 的区别一样，除了这里 `&str` 更智能外。实际上，就像 `&[T]` 和 `Vec[T]` 一样
`&str` 和 `String` 有相似的关系。

```rust
// string1.rs
fn dump(s: &str) {
    println!("str '{}'", s);
}

fn main() {
    let text = "hello dolly";  // the string slice
    let s = text.to_string();  // it's now an allocated string

    dump(text);
    dump(&s);
}
```
再次说，借用操作符可以把 `String` 变成 `&str`，就像 `Vec[T]` 能被变成 `&[T]` 一样。

在内部，`String` 基本是一个 `Vec<u8>` 而 `&str` 是 `&[u8]`，但那些字节需要表达为
有效的UFT-8文本。

就像vector，你能 `push` 一个字符到从 `String` 结尾和 `pop` 一个出来。

```rust
// string5.rs
fn main() {
    let mut s = String::new();
    // initially empty!
    s.push('H');
    s.push_str("ello");
    s.push(' ');
    s += "World!"; // short for `push_str`
    // remove the last char
    s.pop();

    assert_eq!(s, "Hello World");
}
```
你可以用 `to_string()` 把很多类型转换成字符串(如果变量能用'{}'显示那它们能被转为字符串)。
宏 `format!` 是用与 `println!` 相同的格式来构建更复杂字符串的非常有用方式。

```rust
// string6.rs
fn array_to_str(arr: &[i32]) -> String {
    let mut res = '['.to_string();
    for v in arr {
        res += &v.to_string();
        res.push(',');
    }
    res.pop();
    res.push(']');
    res
}

fn main() {
    let arr = array_to_str(&[10, 20, 30]);
    let res = format!("hello {}", arr);

    assert_eq!(res, "hello [10,20,30]");
}
```
注意 `v.to_string()` 前的 `&` —— 操作符 `+=` 的参数被定义了用字符串切片，而不是`String`类型，
所以它需要这样的用法才匹配。

用在切片上的符号也能用在字符串上：

```rust
// string2.rs
fn main() {
    let text = "static";
    let string = "dynamic".to_string();

    let text_s = &text[1..];
    let string_s = &string[2..4];

    println!("slices {:?} {:?}", text_s, string_s);
}
// slices "tatic" "na"
```
但你不能用索引访问字符串！因为它们用了One True Encoding，UTF-8，里面的'字符'
可能是多个字节表达。

```rust
// string3.rs
fn main() {
    let multilingual = "Hi! ¡Hola! привет!";
    for ch in multilingual.chars() {
        print!("'{}' ", ch);
    }
    println!("");
    println!("len {}", multilingual.len());
    println!("count {}", multilingual.chars().count());

    let maybe = multilingual.find('п');
    if maybe.is_some() {
        let hi = &multilingual[maybe.unwrap()..];
        println!("Russian hi {}", hi);
    }
}
// 'H' 'i' '!' ' ' '¡' 'H' 'o' 'l' 'a' '!' ' ' 'п' 'р' 'и' 'в' 'е' 'т' '!'
// len 25
// count 18
// Russian hi привет!
```
现在，我们深入点 —— 这有25个字节，但只有18个字符！尽管如此，如果你用
`find` 这样的方法，你将得到一个有效的索引号（如果找到）且从这个返回的
任何切片都是正确的。

(Rust的 `char` 类型是4字节的Unicode代码。String并 _不是_ char的数组！)

字符串切片可以像vector索引一样访问，因为它使用字节便宜。在这种情况，
字符串由2个字节构成，所以尝试取第一个字节将得到Unicode报错。所以小心
使用字符串切片最好通过字符串函数返回的索引。

```rust
    let s = "¡";
    println!("{}", &s[0..1]); <-- bad, first byte of a multibyte character
```

分解字符串是个流行和有用的消遣。字符串的 `split_whitespace` 方法返回一个 _迭代器_，
我们可以选择怎么用它。一个常见的需求是创建一个子串的vector。

`collect` 非常通用所以需要一个线索来搞清楚在收集 _什么_　——　因此用显式类型。

```rust
    let text = "the red fox and the lazy dog";
    let words: Vec<&str> = text.split_whitespace().collect();
    // ["the", "red", "fox", "and", "the", "lazy", "dog"]
```
你也可以这么说，传递迭代器到 `extend` 方法：

```rust
    let mut words = Vec::new();
    words.extend(text.split_whitespace());
```
在大多数语言中，我们不得不构造那些 _独立分配的字符串_，但这里每个vector
中的切片是借用自原始字符串。我们分配的只是保存了切片的空间 `Vec<&str`。

看看很酷的两行；我们得到字符迭代器，且只取那些非空格的字符。再说，
`collect` 需要一个线索(我们需要字符vector):

```rust
    let stripped: String = text.chars()
        .filter(|ch| ! ch.is_whitespace() ).collect();
    // theredfoxandthelazydog
```
`filter` 方法采用了 _闭包_ 参数，是Rust的lambdas或匿名函数的说法。这里
参数类型是清楚的来自上下文，所以不用显式规则。

是的，你可以用迭代字符的显式loop来实现这个，将返回的切片放到一个可变的
vector，但这个更短，可读性好(当然在你习惯它时)，且一样快。使用loop也不是
大错误，尽管这样，我鼓励你使用更短的版本来写它。

## 插曲: 获取命令行参数

截至目前，我们的程序开心的忽略了外部世界而存在；现在是时候给它们更多数据。

`std::env::args` 是用来读取命令行参数的方法；它返回所有参数的 _字符串迭代器_，
包括程序名本身。

```rust
// args0.rs
fn main() {
    for arg in std::env::args() {
        println!("'{}'", arg);
    }
}
```
```
src$ rustc args0.rs
src$ ./args0 42 'hello dolly' frodo
'./args0'
'42'
'hello dolly'
'frodo'
```
如果它返回一个 `Vec` 是不是更好呢？用 `collect` 足够简单来操作那个vector，
用迭代器的 `skip` 方法可以跳过程序名。

```rust
    let args: Vec<String> = std::env::args().skip(1).collect();
    if args.len() > 0 { // we have args!
        ...
    }
```
这就可以了；就像你在大多数语言里做的一样。

下面是度一个整数参数的Rust方式:

```rust
// args1.rs
use std::env;

fn main() {
    let first = env::args().nth(1).expect("please supply an argument");
    let n: i32 = first.parse().expect("not an integer!");
    // do your magic
}
```
`nth(1)` 返回第二参数的迭代器给你，而 `expect` 就像 `unwrap` 一样在失败时返回可读信息。

把字符串转换成数字的方法是很直观的，但你需要指明值的类型 － 否则 `parse` 怎么知道？

这个程序可能panic，这对小的测试程序倒没事儿。但不要太习惯于这种方便。

## 匹配

在`string3.rs`代码里我们抽取了俄语问候但不知道如何写它。进入 _匹配_ :

```rust
    match multilingual.find('п') {
        Some(idx) => {
            let hi = &multilingual[idx..];
            println!("Russian hi {}", hi);
        },
        None => println!("couldn't find the greeting, Товарищ")
    };
```
`match` 由若干匹配到的值 _模式_ 跟随一个胖箭头组成，并用逗号分开。它很方法
的从 `Option` 中解构值来并绑在 `idx` 上。你 _必须_ 指明所有可能性，所以需要处理
`None`。

一旦你习惯它(我是指完整输入它很多次)，它自然就像显式的 `is_some` 检查并需要
一个额外的变量来保存在 `Option` 中。

但如果你在这对错误不感兴趣，那么用`if let`语句也很好:

```rust
    if let Some(idx) = multilingual.find('п') {
        println!("Russian hi {}", &multilingual[idx..]);
    }
```
这在你只希望匹配 _唯一_ 可能的值用if结构时更方便。

`match` 也能像C语言的 `switch` 语句一样运行，且像其他Rust构造函数一样返回一个值:

```rust
    let text = match n {
        0 => "zero",
        1 => "one",
        2 => "two",
        _ => "many",
    };
```
`_` 就像C语言的 `default`　——　一个缺省情况。如果你不提供它，`rustc` 将报个错。
(在C++中最好情况你得到一个很多信息的警告)

Rust的 `match` 语句能匹配一个范围。注意这些ranges有 _三_ 个点且有个范围，所以
第一个条件将匹配3。

```rust
    let text = match n {
        0...3 => "small",
        4...6 => "medium",
        _ => "large",
     };
```

## 从文件中读

展示我们程序的下一步是 _读文件_。

回忆下 `expect` 就像 `unwrap` 但给出自定义的错误信息。我们将扔出一些错误:

```rust
// file1.rs
use std::env;
use std::fs::File;
use std::io::Read;

fn main() {
    let first = env::args().nth(1).expect("please supply a filename");

    let mut file = File::open(&first).expect("can't open the file");

    let mut text = String::new();
    file.read_to_string(&mut text).expect("can't read the file");

    println!("file had {} bytes", text.len());
}
```
```rust
src$ file1 file1.rs
file had 366 bytes
src$ ./file1 frodo.txt
thread 'main' panicked at 'can't open the file: Error { repr: Os { code: 2, message: "No such file or directory" } }', ../src/libcore/result.rs:837
note: Run with `RUST_BACKTRACE=1` for a backtrace.
src$ file1 file1
thread 'main' panicked at 'can't read the file: Error { repr: Custom(Custom { kind: InvalidData, error: StringError("stream did not contain valid UTF-8") }) }', ../src/libcore/result.rs:837
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```
所以 `open` 可能会失败因为文件不存在或我们没权限读它，且 `read_to_string` 可能
失败因为文件不包含有效的UTF-8。(这没问题，你也可以用 `read_to_end` 来将内容
读到一个字节vector中。) 对不太大的文件，直接读进来是直观有用的。

如果你知道在其他语言里处理文件的任何知识，你也许会好奇什么时候 _关闭_文件。
如果我们往文件中写数据，不关闭它将会导致数据丢失。但这里文件会在函数结束
时关闭且 `file` 变量被废弃。

这里 `抛出任何错误` 是与习惯非常有关的。你不希望将这代码放在函数中，知道它很容易
搞崩整个程序。所以现在我们将探讨到底 `File::open` 返回了什么。如果 `Option` 是个将包含
了正确值和空值的值，那么 `Result` 则是个包含正确值或 `Err` 的值。它们都知道 `unwrap`
(和 `expect` 表兄)但还是很不一样。`Result` 被定义为 _2_ 类型参数，`Ok` 值和 `Err` 值。
`Result` 的'box'有两个部分，一个标为 `Ok` 另一个标为 `Err`。

```rust
fn good_or_bad(good: bool) -> Result<i32,String> {
    if good {
        Ok(42)
    } else {
        Err("bad".to_string())
    }
}

fn main() {
    println!("{:?}",good_or_bad(true));
    //Ok(42)
    println!("{:?}",good_or_bad(false));
    //Err("bad")

    match good_or_bad(true) {
        Ok(n) => println!("Cool, I got {}",n),
        Err(e) => println!("Huh, I just got {}",e)
    }
    // Cool, I got 42

}
```
(实际的'error'类型是任意的 —— 很多人一直使用字符串直到他们习惯于Rust错误类型。)
_或者_ 返回一个值 _或_ 另一个值是很方便的用法。

下面这个版本读文件时不会崩溃。它会返回`Result`那么 _调用者_ 需要决定如何处理
这个错误。

```rust
// file2.rs
use std::env;
use std::fs::File;
use std::io::Read;
use std::io;

fn read_to_string(filename: &str) -> Result<String,io::Error> {
    let mut file = match File::open(&filename) {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    let mut text = String::new();
    match file.read_to_string(&mut text) {
        Ok(_) => Ok(text),
        Err(e) => Err(e),
    }
}

fn main() {
    let file = env::args().nth(1).expect("please supply a filename");

    let text = read_to_string(&file).expect("bad file man!");

    println!("file had {} bytes", text.len());
}
```

第一个匹配安全的从 `Ok` 中抽取了值，它就是匹配的值。如果是 `Err` 它返回
了错误，再重包装为 `Err`。

如果成功，第二个匹配返回从文件里读取并添加到 `text` 的字节数，包装为
`Ok`，否则返回错误。在 `Ok` 里的真实值不重要，所以我们用 `_` 来忽略它。

这不太好看；当大多数函数是在处理错误，那么 `happy path` 没了。继续
看这个问题，很多显式的提前返回，或只 _忽略错误_ 。(也就是说，
在Rust空间总最接近罪恶的事。)

幸运的是，有快捷方式。

`std::io` 模块中有定义了类型 `io::Result<T>` 来准确表达 `Result<T,io::Error>` 并容易用。

```rust
fn read_to_string(filename: &str) -> io::Result<String> {
    let mut file = File::open(&filename)?;
    let mut text = String::new();
    file.read_to_string(&mut text)?;
    Ok(text)
}
```
那个 `?` 操作符作用是对 `File::open` 返回值进行匹配；如果返回值是错误的，
它将立即返回这个错误。否则，它返回 `Ok` 结果。
最后，我们仍需要包装个字符串作为结果。

2017年是Rust的好年份，`?` 操作符是其变得稳定的最酷的功能之一。
你将看到宏 `try!` 在更旧的代码中：

```rust
fn read_to_string(filename: &str) -> io::Result<String> {
    let mut file = try!(File::open(&filename));
    let mut text = String::new();
    try!(file.read_to_string(&mut text));
    Ok(text)
}
```

总之，可能写出即安全又美观的Rust代码，而不需要处理异常。


