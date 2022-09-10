# Basics

## Hello, World!

"hello world" 最初的目的，甚至在第一个C语言的版本编写时就有了,就是为测试
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
都很相似。感叹号符号暗示它是个_宏_调用。对C++程序员来说，这是无趣的事，
它们来自于严重愚蠢的C语言宏定义——但我可以向你保证这里的宏会更强更明智。

对另外些人，可能是“很好，现在我不得不面对惊喜！”尽管如此，编译器并不总是
有用；如果你不带上感叹号，会得到如下错误:

```
error[E0425]: unresolved name `println`
 --> hello2.rs:2:5
  |
2 |     println("Hello, World!");
  |     ^^^^^^^ did you mean the macro `println!`?

```

学一个语言就意味着要习惯它的错误。试着将编译器看作一个严格而友好的朋友而不是
一个_向你射击_的电脑，因为你在刚开始会看到很多红墨水注释。让编译器在编译时抓
住你的错误要比你的程序在真的用户面前爆发错误要好得多。

下一步是介绍 _变量_:

```rust
// let1.rs
fn main() {
    let answer = 42;
    println!("Hello {}", answer);
}

```

错误拼写是 _编译_ 错误，不是运行时错误如在其他动态语言Python或Javascript里。
这会降低你很多后期的压力！如果我把 'answer'写成 'answr'，编译器会_友好_的提示:

```
4 |     println!("Hello {}", answr);
  |                         ^^^^^ did you mean `answer`?

```

宏 `println!` 使用 [format string](https://doc.rust-lang.org/std/fmt/index.html)
和一些值；它非常像Python 3用的格式化。

另一个非常有用的宏是 `assert_eq!`。 它是Rust中的测试老黄牛；你 _断言_ 
两个值应相等，如不它们不相等, _报错_。

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
一个从0开始的数组_索引_一样。

有趣的事需要在_一定的条件下_进行:

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
_没有_括号，这有点像Go，但你_必须_对多语句块使用大括号。

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

在传统上，编程语言_语句_ (如 `if`) 和 _表达式_ (如 `1+i`)。在Rust中，几乎每个东西都
有对应值并能成为一个表达式。严重丑陋的C语言的 '三元操作符' `i % 2 == 0 ? "even" : "odd"`
不再被需要。

注意这里的代码块内并没有任何分号。

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
添加关键字 `mut` (_请让_ 这变量可以被修改) 实现这个小目的:

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

让声明的变量默认_只可读_是有理由的。在大型程序中，很难跟踪在哪修改了
变量。所以Rust让可修改属性('修改能力')成为显式的。这语言里有很多智能
的地方，但它尝试不让任何事隐藏着发生。

Rust即是静态类型也是强类型的语言 —— 这常看起来迷惑，但考虑下C语言
（它是静态类型但也是弱类型) 和Python语言(它动态类型但也是强类型)。
静态类型是指在编译期就知道类型了，动态类型是指仅在运行时才知道类型。

这时，尽管如此，看起来好像Rust在帮你_处理_那些类型。那 `i` 的类型是什么？
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
_Functions_ are one place where the compiler will not work out types for you.
And this in fact was a deliberate decision, since languages like Haskell have
such powerful type inference that there are hardly any explicit type names. It's
actually good Haskell style to put in explicit type signatures for functions.
Rust requires this always.

这是个用户定义函数:
Here is a simple user-defined function:

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
Rust goes back to an
older style of argument declaration, where the type follows the name. This is
how it was done in Algol-derived languages like Pascal.

再次说，没有整数到浮点数的转换 —— 如果你把`2.0`换成`2`将得到明确报错:
Again, no integer-to-float conversions - if you replace the `2.0` with `2` then we
get a clear error:

```
8 |     let res = sqr(2);
  |                   ^ expected f64, found integral variable
  |
```

你将很少看到函数里有`return`语句。更常见的是这样:
You will actually rarely see functions written using a `return` statement. More
often, it will look like this:

```rust
fn sqr(x: f64) -> f64 {
    x * x
}
```

这是因为函数体(`{}`里)的最后一条表达式有值，就像if-as-an-expression。
This is because the body of the function (inside `{}`) has the value of its
last expression, just like with if-as-an-expression.

因分号会被手指半自动的插入进来，你也许会添加它，将报以下错误:
Since semicolons are inserted semi-automatically by human fingers, you might add it
here and get the following error:

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
但有时就是nothing。编译器知道这是个常见错误，所以在 _帮_ 你。
(任何人如果花了很多时间在C++编译器上将知道 _damn unusual_ 是什么)
 Everything in Rust
has a value, but sometimes it's just nothing.  The compiler knows this is
a common mistake, and actually _helps_ you.  (Anybody who has spent time with a
C++ compiler will know how _damn unusual_ this is.)

更多没有return的表达式例子如下:
A few more examples of this no-return expression style:

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

写上`return`也不会错，但没它代码将更清晰。如果想 _早点返回_ 你将仍会用`return`。
It's not wrong to use `return`, but code is cleaner without it. You will still
use `return` for _returning early_ from a function.

有些操作能更优雅的 _递归_ 表达:
Some operations can be elegantly expressed _recursively_:

```rust
fn factorial(n: u64) -> u64 {
    if n == 0 {
        1
    } else {
        n * factorial(n-1)
    }
}
```
初看有点奇怪，最好拿起笔和纸来做几个例子。尽管做这样的操作常常不是最_有效率_的。
This can be a little strange at first, and the best thing is then to use pencil and paper
and work out some examples. It isn't usually the most _efficient_ way to do that
operation however.

也可以用引用来传参。引用通过`&`创建并用`*`来解引用。
Values can also be passed by _reference_. A reference is created by `&` and _dereferenced_
by `*`.

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
What if you want a function to modify one of its arguments?  Enter _mutable references_:

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
这是C做得比C++多的地方。你需要显式的传递引用符号(用`&`)并显式的 _解引用_
(用`*`)。然后再放入`mut`因为它不是默认可改的。(我感觉C++引用太简洁而不能
与C比较。)
This is more how C would do it than C++. You have to explicitly pass the
reference (with `&`) and explicitly _dereference_ with `*`. And then throw in `mut`
because it's not the default. (I've always felt that C++ references are
too easy to miss compared to C.)

基本上，Rust引入了一些 _分歧_ 到这里，不那么微妙的推动你直接从函数返回值。
幸运的是Rust有强大的方式来表达如"操作成功这是结果"所以`&mut`并不是常需要用到。
传引用在我们使用大对象不希望发生拷贝时是很有用的。
Basically, Rust is introducing some _friction_ here, and not-so-subtly pushing
you towards returning values from functions directly.  Fortunately, Rust has
powerful ways to express things like "operation succeeded and here's the result"
so `&mut` isn't needed that often. Passing by reference is important when we have a
large object and don't wish to copy it.

类型在变量后的风格也用在`let`语句上，当你真想弄清变量的类型时:
The type-after-variable style applies to `let` as well, when you really want to nail
down the type of a variable:

```rust
let bigint: i64 = 0;
```

## Learning Where to Find the Ropes

是时候开始用文档了。这将把文档装在你的机器上，可用`rustup doc --std`将打开
浏览器。
It's time to start using the documentation. This will be installed on your machine,
and you can use `rustup doc --std` to open it in a browser.

注意 _search_ 框在顶部，这是你好帮手；它可以在本地工作。
Note the _search_ field at the top, since this
is going to be your friend; it operates completely offline.

让我们看看数学函数在哪，搜搜'cos'。前两个提示显示它为单双精度浮点数
定义了两个函数。它用了 _值itself_ 以作为一个自身方法，如下:
Let's say we want to see where the mathematical
functions are, so search for 'cos'. The first two hits show it defined for both
single and double-precision floating point numbers.  It is defined on the
_value itself_ as a method, like so:

```rust
let pi: f64 = 3.1416;
let x = pi/2.0;
let cosine = x.cos();
```
结果会是0类型；我们明显需要一个pi-ness代码的授权。
And the result will be sort-of zero; we obviously need a more authoritative source
of pi-ness!

(为什么我需要显示的`f64`类型？没有它，常量可以是`f32`或`f64`，它们区别很大)
(Why do we need an explicit `f64` type? Because without it, the constant could
be either `f32` or `f64`, and these are very different.)

我们引用`cos`的例子，写得更完整一点(`assert!`是`assert_eq!`的表兄；表达式
返回true):
Let me quote the example given for `cos`, but written as a complete program
( `assert!` is a cousin of `assert_eq!`; the expression must be true):

```rust
fn main() {
    let x = 2.0 * std::f64::consts::PI;

    let abs_difference = (x.cos() - 1.0).abs();

    assert!(abs_difference < 1e-10);
}
```
`std::f64::consts::PI` is a mouthful! `::` 与在C++中的意思一样，means much the same as it does in C++,
(其他有些语言里会写成 '.') - 这是个 _有资格的名称_. We get
我们在搜`PI`时会得到完整的名称。
this full name from the second hit on searching for `PI`.

截至目前，我们的小Rust程序没有任何 `import` 和 `include` 等会拖慢探讨
'Hello World'程序。让我们加上`use`语句让程序的可读性更好:
Up to now, our little Rust programs have been free of all that `import` and
`include` stuff that tends to slow down the discussion of 'Hello World' programs.
Let's make this program more readable with a `use` statement:

```rust
use std::f64::consts;

fn main() {
    let x = 2.0 * consts::PI;

    let abs_difference = (x.cos() - 1.0).abs();

    assert!(abs_difference < 1e-10);
}
```
为什么我不需要这么做？因为Rust通过 _prelude_ 让一些基本可见的功能不必显式
用到`use`语句。
Why haven't we needed to do this up to now?
This is because Rust helpfully makes a lot of basic functionality visible without
explicit `use` statements through the Rust _prelude_.

## Arrays and Slices

所有的静态类型语言有 _数组_，它的值全部在内存中。数组的 _索引_ 从0开始:
All statically-typed languages have _arrays_, which are values packed nose to tail
in memory. Arrays are _indexed_ from zero:

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
And the output is:

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
In this case, Rust knows _exactly_ how big the array is and if you try to
access `arr[4]` it will be a _compile error_.

学习新语言时常从其他已知的语言中引入非学习的思想习惯；如果你是Python
开发者，那些组件即 `List`。我们将很快带来Rust的等价 `List`，但数组不是
替代品你要的；它们是 _固定大小_。他们可以被修改(合理调用)但你不能添加新元素。
Learning a new language often involves _unlearning_ mental habits from languages
you already know; if you are a Pythonista, then those brackets say `List`. We will
come to the Rust equivalent of `List` soon, but arrays are not the droids you're looking
for; they are _fixed in size_. They can be _mutable_ (if we ask nicely) but you
cannot add new elements.

在Rust里不常用数组，因为数组的类型包括了大小。这个数组的类型是`[i32; 4]`；
 `[10, 20]`的类型是`[i32; 2]`依次类推: 它们的类型不同。所以它们被用作函数参数是很奇怪。
Arrays are not used that often in Rust, because the type of an array includes its
size.  The type of the array in the example is
`[i32; 4]`; the type of `[10, 20]` would be `[i32; 2]` and so forth: they
have _different types_.  So they are bastards to pass around as
function arguments.

_实际_ 常用到的是 _切片_。你可以把它们看作是底层数组值的 _视图_。
它们在行为上很像数组，知道自己的 _大小_，不像C指针这种危险物种。
What _are_ used often are _slices_. You can think of these as _views_ into
an underlying array of values. They otherwise behave very much like an array, and
_know their size_, unlike those dangerous animals C pointers.

注意两个重要的点 —— 怎么写切片的类型，你需要用`&`来把它传递到函数里。
Note two important things here - how to write a slice's type, and that
you have to use `&` to pass it to the function.

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
先忽略一会儿`sum`的代码，看看`&[i32]`。Rust数组和切片的关系就像C语言中
数组与指针的关系一样，但还有2个重要区别——Rust的切片会跟踪数组长度(如果
你尝试访问超出数组长度的值它将报错)，且你需要显式的采用`&`操作符来把数组
作为切片传递。
Ignore the code of `sum` for a while, and look at `&[i32]`. The relationship between
Rust arrays and slices is similar to that between C arrays and pointers, except for
two important differences - Rust slices keep track of their size (and will
panic if you try to access outside that size) and you have to explicitly say that
you want to pass an array as a slice using the `&` operator.

一个C程序员对`&`读作'取地址‘；而Rust程序员则读作'借用’。这是学Rust时的关键词。
借用是编程语言中的普遍模式的名称；无论何时你传递引用时(动态语言里常发生)或
在C里传一个指针。任何被借用的变量仍是原ower持有。
A C programmer pronounces `&` as 'address of'; a Rust programmer pronounces it
'borrow'. This is going to be the key word when learning Rust. Borrowing is the name
given to a common pattern in programming; whenever you pass something by reference
(as nearly always happens in dynamic languages) or pass a pointer in C. Anything
borrowed remains owned by the original owner.

## Slicing and Dicing 切片

你不能使用普通的`{}`来打印出数组，但可以通过`{:?}`来打印调试信息。
You cannot print out an array in the usual way with `{}` but you can do a _debug_
print with `{:?}`.

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
So, arrays of arrays are no problem, but the important thing is that an array contains
values of _only one type_.  The values in an array are arranged next to each other
in memory so that they are _very_ efficient to access.

如果你对那些变量的实际类型好奇，这里有个有效的小方法。先声明一个你
知道是错误类型的变量:
If you are curious about the actual types of these variables, here is a useful trick.
Just declare a variable with an explicit type which you know will be wrong:

```rust
let var: () = [1.1, 1.2];
```
然后得到如下错误信息:
Here is the informative error:

```
3 |     let var: () = [1.1, 1.2];
  |                   ^^^^^^^^^^ expected (), found array of 2 elements
  |
  = note: expected type `()`
  = note:    found type `[{float}; 2]`
```
(`{float}` means 'some floating-point type which is not fully specified yet')

切片给你 _同一个_ 数组的不同 _视图_:
Slices give you different _views_ of the _same_ array:

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
这数据永远不能拷贝。那些切片全是从原数组  _借用_ 的数据。它们与数组有有非常紧密的
联系，Rust花费很大努力去保证联系还在。
This is a neat notation which looks similar to Python slices but with a big difference:
a copy of the data is never made.  These slices all _borrow_ their data from their
arrays. They have a very intimate relationship with that array, and Rust spends a lot
of effort to make sure that relationship does not break down.

## Optional Values 可选值

切片，像数组一样能被索引访问。Rust在编译时知道数组的大小，但切片的大小
只能在运行时知道。所以`s[i]`能引起越界错误，执行到这会 _pannic_。这肯定
不是你希望看到的——它就像安全发取消发射和非常昂贵的零星碎片遍布弗罗里达
区别一样大。那里 _没有异常_。
Slices, like arrays, can be _indexed_. Rust knows the size of an array at
compile-time, but the size of a slice is only known at run-time. So `s[i]` can
cause an out-of-bounds error when running and will _panic_.  This is really not
what you want to happen - it can be the difference between a safe launch abort and
scattering pieces of a very expensive satellite all over Florida. And there are
_no exceptions_.

领会那些，因为它的出现总是带来震惊。你不能包装可能会崩的代码在一些try块再
'抓住错误' —— 至少不能用你不想每天看见的方式。所以Rust如何保障安全？
Let that sink in, because it comes as a shock. You cannot wrap dodgy-may-panic
code in some try-block and 'catch the error' - at least not in a way you'd want to use
every day. So how can Rust be safe?

切片有个方法`get`不会崩。但它会返回什么呢？
There is a slice method `get` which does not panic. But what does it return?

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
`last`失败(假如我们忽略了数组是从0索引开始的)，但它返回了`None`。
`first`可以，但好像返回了一个包装在`Some`里的值。欢迎`Option`类型！
它可以是`Some`或者`None`。
`last` failed (we forgot about zero-based indexing), but returned something called `None`.
`first` is fine, but appears as a value wrapped in `Some`.  Welcome to the `Option`
type!  It may be _either_ `Some` or `None`.

类型`Option`有些有用的方法:
The `Option` type has some useful methods:

```rust
    println!("first {} {}", first.is_some(), first.is_none());
    println!("last {} {}", last.is_some(), last.is_none());
    println!("first value {}", first.unwrap());

// first true false
// last false true
// first value 1
```
如果将`last` _打开包装_，你会得到崩溃。但至少你可以先调用`is_none`来确认
—— 例如，你准备了一个别的默认值来返回:
If you were to _unwrap_ `last`, you would get a panic. But at least you can call
`is_some` first to make sure - for instance, if you had a distinct no-value default:

```rust
    let maybe_last = slice.get(5);
    let last = if maybe_last.is_some() {
        *maybe_last.unwrap()
    } else {
        -1
    };
```
注意`*`——在`Some`里的精确类型`&i32`是引用。我们需要解引用它来得到一个`i32`的值。
Note the `*` - the precise type inside the `Some` is `&i32`, which is a reference. We need
to dereference this to get back to a `i32` value.

这是冗长的，所以有快捷方式 —— `unwrap_or` 将返回值或`None`时的预设值。
注意类型需要匹配——`get`返回引用。你需要返回`&32`类的预设值`&-1`。最后，
注意使用`*`来解出`i32`类型的值。
Which is long-winded, so there's a shortcut - `unwrap_or` will return the value it
is given if the `Option` was `None`. The types must match up - `get` returns
a reference. so you have to make up a `&i32` with `&-1`. Finally, again use `*`
to get the value as `i32`.

```rust
    let last = *slice.get(5).unwrap_or(&-1);
```
很容易忽略`&`，但编译器会带会你到这。如果是`-1`，`rustc`会说
 'expected &{integer}, found integral variable' and then
'help: try with `&-1`'.

你可以把`Option`看作为一个包含了值的盒子，或者nothing(`None`)。
(在Haskell里叫`Maybe`)。它可以包含 _任何_ 类型的值，也就是它的
_类型参数_。在这例子，完整的类型是 `Option<&i32>`，为泛型用了
C++风格的符号。对盒子解包装将引起一个露出，但不像薛定谔的猫，
我们能提前知道它包含值。
You can think of `Option` as a box which may contain a value, or nothing (`None`).
(It is called `Maybe` in Haskell). It may contain _any_ kind of value, which is
its _type parameter_. In this case, the full type is `Option<&i32>`, using
C++-style notation for _generics_.  Unwrapping this box may cause an explosion,
but unlike Schroedinger's Cat, we can know in advance if it contains a value.

Rust函数/方法返回这样的maybe-boxes很常见，所以学学如何舒服的
[使用](https://doc.rust-lang.org/std/option/enum.Option.html) 

## Vectors 向量

我们将再返回到切片方法，但首先: 向量。它们是  _可变大小_ 的数组且行为很像
Python的`List`和C++的`std::vector`。Rust类型`Vec`(读作'vector')行为实际很像切片；
区别是你可以添加额外数据到vector中 —— 注意它需要被声明为mutable。
We'll return to slice methods again, but first: vectors. These are _re-sizeable_
arrays and behave much like Python `List` and C++ `std::vector`. The Rust type
`Vec` (pronounced 'vector') behaves very much like a slice in fact; the
difference is that you can append extra values to a vector - note that it must
be declared as mutable.

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
A common beginner mistake is to forget the `mut`; you will get a helpful error
message:

```
3 |     let v = Vec::new();
  |         - use `mut v` here to make mutable
4 |     v.push(10);
  |     ^ cannot borrow mutably
```
vectors和切片间有亲密的关系:
There is a very intimate relation between vectors and slices:

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
That little, so-important borrow operator `&` is _coercing_ the vector into a
slice. And it makes complete sense, because the vector is also looking after an array of
values, with the difference that the array is allocated _dynamically_.

如果你用过动态语言，是时候谈点了。在系统语言中，分配内存有两种类型：栈上内存和堆上内存。
在栈上分配内存是很快的，但栈有限制；典型的是内存大小限制在MB级。堆上的内存能到GB级，
但分配内存相对耗时，且内存不用后需要主动释放。在号称有'内存管理'的语言中（如Java，Go，
以及那些叫作'scripting'的语言）那些内存细节是用被称为垃圾回收的方便自治组件对你隐藏着的。
一旦系统确认一些数据不再被其他引用，将会把这些数据内存释放回可用的内存池子里。
If you come from a dynamic language, now is time for that little talk. In systems
languages, program memory comes in two kinds: the stack and the heap. It is very fast
to allocate data on the stack, but the stack is limited; typically of the order of
megabytes. The heap can be gigabytes, but allocating is relatively expensive, and
such memory must be _freed_ later. In so-called 'managed' languages (like Java, Go
and the so-called 'scripting' languages) these details are hidden from you by that
convenient municipal utility called the _garbage collector_. Once the system is sure
that data is no longer referenced by other data, it goes back into the pool
of available memory.

一般来说，这是值得付出的代价。与栈打交道是可怕的不安全，因为一旦你出错可能覆盖当前
程序的返回地址，那你的程序将会丢脸的崩溃或（更糟）被人恶意攻破。
Generally, this is a price worth paying. Playing with the stack is terribly unsafe,
because if you make one mistake you can override the return address of the current
function, and you die an ignominious death or (worse) got pwned by some guy living
in his Mom's basement in Minsk.

我写得第一个C程序（在DOS PC上）搞垮了整个电脑。Unix系统常表现得更好，程序进程崩溃
时只报_段错误_。为什么这比Rust（或Go）程序pannicking严重？因为panic发生在原始程序
中发生，而不是在程序变得毫无希望的迷惑消耗掉你全部精力时。Pannic是_内存安全的_因为
它们在任何内存非法访问前出现。这却是C语言的安全问题根源，因为所有的内存访问都是不
安全的且狡诈的攻击者可以利用这个薄弱环节。
The first C program I wrote (on a DOS PC)
took out the whole computer. Unix systems always behaved better, and only the process died
with a _segfault_. Why is this worse than a Rust (or Go) program panicking?
Because a panic happens when the original problem happens, not when the program
has become hopelessly confused and eaten all your homework. Panics are _memory safe_
because they happen before any illegal access to memory. This is a common cause of
security problems in C, because all memory accesses are unsafe and a cunning attacker
can exploit this weakness.

Panick听起来严重且意外，但Rust的panic是有结构的 —— 栈是_轻松的_就像普通异常。
所有分配的对象将被废弃，并将生成一个调用过程。
Panicking sounds desperate and unplanned, but Rust panics are structured - the stack is _unwound_
just as with exceptions. All allocated objects are dropped, and a backtrace is generated.

垃圾回收的缺点是什么？首先它会浪费内存，这在那些越来越主宰世界的小的嵌入式芯片会要紧。
其二它将决定在最差可能性时清理内存只能立刻开始。那些嵌入式系统需要实时响应('real-time')
且不能容忍非计划的内存清理打断任务。最优雅的动态语言Lua的首席设计者Roberto Ierusalimschy
曾说他不喜欢乘坐在依靠有垃圾回收的软件的飞机上。
The downsides of garbage collection? The first is that it is wasteful of memory, which
matters in those small embedded microchips which increasingly rule our world. The
second is that it will decide, at the worst possible time, that a clean up must happen
_now_. (The Mom analogy is that she wants to clean your room when you are at a
delicate stage with a new lover). Those embedded systems need to respond to things
_when they happen_ ('real-time') and can't tolerate unscheduled outbreaks of
cleaning. Roberto Ierusalimschy, the chief designer of Lua (one of the most elegant
dynamic languages ever) said that he would not like to fly on an airplane that
relied on garbage-collected software.

回到向量：当一个向量被修改或创建，在堆上分配了它并成为那片内存的_主人_。切片_借用_
了向量的内存。当向量被_废弃_时，它会释放内存。
Back to vectors: when a vector is modified or created, it allocates from the heap and becomes
 the _owner_ of that memory. The slice _borrows_ the memory from the vector.
When the vector dies or _drops_, it lets the memory go.

## Iterators 迭代器

截至目前我们探讨了这么多却没有提及Rust的一个关键谜语 —— 迭代器。
使用了范围的for-loop就用到了迭代器(`0..n`实际上类似于Python 3的`range`函数）。
We have got so far without mentioning a key part of the Rust puzzle - iterators.
The for-loop over a range was using an iterator (`0..n` is actually similar to the
Python 3 `range` function).

迭代器很容易被定义得不正式。它是一个有能返回`Option`值的`next`方法的'对象'。
当返回值还不是`None`时，我们可继续调用`next`:
An iterator is easy to define informally. It is an 'object' with a `next` method
which returns an `Option`. As long as that value is not `None`, we keep calling
`next`:

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

定义一个for-loop看起来不是个高效的方式，但`rustc`在release模式上做了疯狂的优化，
让它与`while`循环一样快。
This may seem an inefficient way to define a for-loop, but `rustc` does crazy-ass
optimizations in release mode and it will be just as fast as a `while` loop.

下面是第一次尝试迭代一个数组：
Here is the first attempt to iterate over an array:

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
which fails, but helpfully:
```
4 |     for i in arr {
  |     ^ the trait `std::iter::Iterator` is not implemented for `[{integer}; 3]`
  |
  = note: `[{integer}; 3]` is not an iterator; maybe try calling
   `.iter()` or a similar method
  = note: required by `std::iter::IntoIterator::into_iter`
```
遵守`rustc`的建议，用以下方式就可以了。
Following `rustc`'s advice, the following program works as expected.

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
实际上，用上面的方式去迭代一个数组或切片会比`for i in 0..slice.len() {}`用更有效，
因为Rust不需要着魔的检查每个索引操作。
In fact, it is more efficient to iterate over an array or slice this way
than to use `for i in 0..slice.len() {}` because Rust does not have to obsessively
check every index operation.

我们前面有个例子来加和一组整数。它引入了`mut`变量和一个循环。这里是符合语言习惯
的、专业的实现：
We had an example of summing up a range of integers earlier. It involved a `mut`
variable and a loop. Here's the _idiomatic_, pro-level way of doing the sum:

```rust
// sum1.rs
fn main() {
    let sum: i32  = (0..5).sum();
    println!("sum was {}", sum);

    let sum: i64 = [10, 20, 30].iter().sum();
    println!("sum was {}", sum);
}
```
注意，这是一个例子关于你需要显式使用变量_类型_，因除此之外Rust没有足够的信息。
这里我们用了不同整数类型来加和，没问题。（创建新变量用了已有名称。）
Note that this is one of those cases where you need to be explicit about
the _type_ of the variable, since otherwise Rust doesn't have enough information.
Here we do sums with two different integer sizes, no problem. (It is also no
problem to create a new variable of the same name if you run out of names to
give things.)

有了相关背景知识，了解更多 [slice methods](https://doc.rust-lang.org/std/primitive.slice.html)
会更有意义。

(Another documentation tip; on the right-hand side of every doc page there's a '[-]' which you can
click to collapse the method list. You can then expand the details of anything
that looks interesting. Anything that looks too weird, just ignore for now.)

The `windows` method gives you an iterator of slices - overlapping windows of
values!

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

## More about vectors...

There is a useful little macro `vec!` for initializing a vector. Note that you
can _remove_ values from the end of a vector using `pop`, and _extend_ a vector
using any compatible iterator.

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
Vectors compare with each other and with slices by value.

You can insert values into a vector at arbitrary positions with `insert`,
and remove with `remove`. This is not as efficient as pushing and popping since
the values will have to be moved to make room, so watch out for these operations on big
vectors.

Vectors have a size and a _capacity_. If you `clear` a vector, its size becomes zero,
but it still retains its old capacity. So refilling it with `push`, etc only requires
reallocation when the size gets larger than that capacity.

Vectors can be sorted, and then duplicates can be removed - these operate in-place
on the vector. (If you want to make a copy first, use `clone`.)

```rust
// vec4.rs
fn main() {
    let mut v1 = vec![1, 10, 5, 1, 2, 11, 2, 40];
    v1.sort();
    v1.dedup();
    assert_eq!(v1, &[1, 2, 5, 10, 11, 40]);
}
```

## Strings

Strings in Rust are a little more involved than in other languages; the `String` type,
like `Vec`, allocates dynamically and is resizeable. (So it's like C++'s `std::string`
but not like the immutable strings of Java and Python.) But a program may contain a lot
of _string literals_ (like "hello") and a system language should be able to store
these statically in the executable itself. In embedded micros, that could mean putting
them in cheap ROM rather than expensive RAM (for low-power devices, RAM is
also expensive in terms of power consumption.) A _system_ language has to have
two kinds of string, allocated or static.

So "hello" is not of type `String`. It is of type `&str` (pronounced 'string slice').
It's like the distinction between `const char*` and `std::string` in C++, except
`&str` is much more intelligent.  In fact, `&str` and `String` have a very
similar relationship to each other as do `&[T]` to `Vec<T>`.

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
Again, the borrow operator can coerce `String` into `&str`, just as `Vec<T>` could
be coerced into `&[T]`.

Under the hood, `String` is basically a `Vec<u8>` and `&str` is `&[u8]`, but
those bytes _must_ represent valid UTF-8 text.

Like a vector, you can `push` a character and `pop` one off the end of `String`:

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
You can convert many types to strings using `to_string`
(if you can display them with '{}' then they can be converted).
The `format!` macro is a very useful way to build
up more complicated strings using the same format strings as `println!`.

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
Note the `&` in front of `v.to_string()` - the `+=` operator is defined on a string
slice, not a `String` itself, so it needs a little persuasion to match.

The notation used for slices works with strings as well:

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

But, you cannot index strings!  This is because they use the One True Encoding,
UTF-8, where a 'character' may be a number of bytes.

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

Now, let that sink in - there are 25 bytes, but only 18 characters! However, if
you use a method like `find`, you will get a valid index (if found) and then
any slice will be fine.

(The Rust `char` type is a 4-byte Unicode code point. Strings are _not_ arrays
of chars!)

String slicing may explode like vector indexing, because it uses byte offsets. In this case,
the string consists of two bytes, so trying to pull out the first byte is a Unicode error. So be
careful to only slice strings using valid offsets that come from string methods.

```rust
    let s = "¡";
    println!("{}", &s[0..1]); <-- bad, first byte of a multibyte character
```

Breaking up strings is a popular and useful pastime. The string `split_whitespace`
method returns an _iterator_, and we then choose what to do with it. A common need
is to create a vector of the split substrings.

`collect` is very general and so needs some clues about _what_ it is collecting - hence
the explicit type.

```rust
    let text = "the red fox and the lazy dog";
    let words: Vec<&str> = text.split_whitespace().collect();
    // ["the", "red", "fox", "and", "the", "lazy", "dog"]
```
You could also say it like this, passing the iterator into the `extend` method:

```rust
    let mut words = Vec::new();
    words.extend(text.split_whitespace());
```
In most languages, we would have to make these _separately allocated strings_,
whereas here each slice in the vector is borrowing from the original string.
All we allocate is the space to keep the slices.

Have a look at this cute two-liner; we get an iterator over the chars,
and only take those characters which are not space. Again, `collect` needs
a clue (we may have wanted a vector of chars, say):

```rust
    let stripped: String = text.chars()
        .filter(|ch| ! ch.is_whitespace()).collect();
    // theredfoxandthelazydog
```
The `filter` method takes a _closure_, which is Rust-speak for
lambdas or anonymous functions.  Here the argument type is clear from the
context, so the explicit rule is relaxed.

Yes, you can do this as an explicit loop over chars, pushing the returned slices
into a mutable vector, but this is shorter, reads well (_when_ you are used to it,
of course) and just as fast. It is not a _sin_ to use a loop, however, and I encourage
you to write that version as well.

## Interlude: Getting Command Line Arguments

Up to now our programs have lived in blissful ignorance of the outside world; now
it's time to feed them data.

`std::env::args` is how you access command-line arguments; it returns an iterator
over the arguments as strings, including the program name.

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
Would it have been better to return a `Vec`? It's easy enough to use `collect` to
make that vector, using the iterator `skip` method to move past the program
name.

```rust
    let args: Vec<String> = std::env::args().skip(1).collect();
    if args.len() > 0 { // we have args!
        ...
    }
```
Which is fine; it's pretty much how you would do it in most languages.

A more Rust-y approach to reading a single argument (together with parsing an
integer value):

```rust
// args1.rs
use std::env;

fn main() {
    let first = env::args().nth(1).expect("please supply an argument");
    let n: i32 = first.parse().expect("not an integer!");
    // do your magic
}
```
`nth(1)` gives you the second value of the iterator, and `expect`
is like an `unwrap` with a readable message.

Converting a string into a number is straightforward, but you do need to specify
the type of the value - how else could `parse` know?

This program can panic, which is fine for dinky test programs. But don't get too
comfortable with this convenient habit.

## Matching

The code in `string3.rs` where we extract the Russian greeting is not how it would
be usually written. Enter _match_:

```rust
    match multilingual.find('п') {
        Some(idx) => {
            let hi = &multilingual[idx..];
            println!("Russian hi {}", hi);
        },
        None => println!("couldn't find the greeting, Товарищ")
    };
```
`match` consists of several _patterns_ with a matching value following the fat arrow,
separated by commas.  It has conveniently unwrapped the value from the `Option` and
bound it to `idx`.  You _must_ specify all the possibilities, so we have to handle
`None`.

Once you are used to it (and by that I mean, typed it out in full a few times) it
feels more natural than the explicit `is_some` check which needed an extra
variable to store the `Option`.

But if you're not interested in failure here, then `if let` is your friend:

```rust
    if let Some(idx) = multilingual.find('п') {
        println!("Russian hi {}", &multilingual[idx..]);
    }
```
This is convenient if you want to do a match and are _only_ interested in one possible
result.

`match` can also operate like a C `switch` statement, and like other Rust constructs
can return a value:

```rust
    let text = match n {
        0 => "zero",
        1 => "one",
        2 => "two",
        _ => "many",
    };
```
The `_` is like C `default` - it's a fall-back case. If you don't provide one then
`rustc` will consider it an error. (In C++ the best you can expect is a warning, which
says a lot about the respective languages).

Rust `match` statements can also match on ranges. Note that these ranges have
_three_ dots and are inclusive ranges, so that the first condition would match 3.

```rust
    let text = match n {
        0...3 => "small",
        4...6 => "medium",
        _ => "large",
     };
```
## Reading from Files

The next step to exposing our programs to the world is to _reading files_.

Recall that `expect` is like `unwrap` but gives a custom error message. We are
going to throw away a few errors here:

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
```
src$ file1 file1.rs
file had 366 bytes
src$ ./file1 frodo.txt
thread 'main' panicked at 'can't open the file: Error { repr: Os { code: 2, message: "No such file or directory" } }', ../src/libcore/result.rs:837
note: Run with `RUST_BACKTRACE=1` for a backtrace.
src$ file1 file1
thread 'main' panicked at 'can't read the file: Error { repr: Custom(Custom { kind: InvalidData, error: StringError("stream did not contain valid UTF-8") }) }', ../src/libcore/result.rs:837
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```
So `open` can fail because the file doesn't exist or we aren't allowed to read it,
and `read_to_string` can fail because the file doesn't contain valid UTF-8. (Which is
fair enough, you can use `read_to_end` and put the contents into a vector of bytes
instead.) For files that aren't too big, reading them in one gulp is useful and
straightforward.

If you know anything about file handling in other languages, you may wonder when
the file is _closed_. If we were writing to this file, then not closing it could
result in loss of data.
But the file here is closed when the function ends and the `file` variable is _dropped_.

This 'throwing away errors' thing is getting too much of a habit. You do not
want to put this code into a function, knowing that it could so easily crash
the whole program.  So now we have to talk about exactly what `File::open` returns.
If `Option` is a value that may contain something or nothing, then `Result` is a value
that may contain something or an error. They both understand `unwrap` (and its cousin
`expect`) but they are quite different. `Result` is defined by _two_ type parameters,
for the `Ok` value and the `Err` value.
The `Result` 'box' has two compartments, one labelled `Ok` and the other `Err`.

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
(The actual 'error' type is arbitrary - a lot of people use strings until
they are comfortable with Rust error types.) It's a convenient way to _either_
return one value _or_ another.

This version of the file reading function does not crash. It returns a `Result` and
it is the _caller_ who must decide how to handle the error.

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

The first match safely extracts the value from `Ok`, which
becomes the value of the match. If it's `Err` it returns the error,
rewrapped as an `Err`.

If successful, the second match returns the number of bytes which were
read and appended to `text`, wrapped up as an `Ok`, otherwise (again)
the error. The actual value in the `Ok` is unimportant, so we ignore
it with `_`.

This is not so pretty; when most of a function is error handling, then
the 'happy path' gets lost. Go tends to have this problem, with lots of
explicit early returns, or just _ignoring errors_.  (That is, by the way,
the closest thing to evil in the Rust universe.)

Fortunately, there is a shortcut.

The `std::io` module defines a type alias `io::Result<T>` which is exactly
the same as `Result<T,io::Error>` and easier to type.

```rust
fn read_to_string(filename: &str) -> io::Result<String> {
    let mut file = File::open(&filename)?;
    let mut text = String::new();
    file.read_to_string(&mut text)?;
    Ok(text)
}
```
That `?` operator does almost exactly what the match on `File::open` does;
if the result was an error, then it will immediately return that error.
Otherwise, it returns the `Ok` result.
At the end, we still need to wrap up the string as a result.

2017 was a good year for Rust, and `?` was one of the cool things that
became stable. You will still see the macro `try!` used in older code:

```rust
fn read_to_string(filename: &str) -> io::Result<String> {
    let mut file = try!(File::open(&filename));
    let mut text = String::new();
    try!(file.read_to_string(&mut text));
    Ok(text)
}
```

In summary, it's possible to write perfectly safe Rust that isn't ugly, without
needing exceptions.


