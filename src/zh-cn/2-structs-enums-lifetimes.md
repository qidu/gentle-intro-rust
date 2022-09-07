# Structs, Enums and Matching

## Rust 喜欢 Move It, Move It

我将往回看看，展示一些另人惊奇的事：

```rust
// move1.rs
fn main() {
    let s1 = "hello dolly".to_string();
    let s2 = s1;
    println!("s1 {}", s1);
}
```
我们会看到如下错误：

```
error[E0382]: use of moved value: `s1`
 --> move1.rs:5:22
  |
4 |     let s2 = s1;
  |         -- value moved here
5 |     println!("s1 {}", s1);
  |                      ^^ value used here after move
  |
  = note: move occurs because `s1` has type `std::string::String`,
  which does not implement the `Copy` trait
```
Rust相比其他语言有不同的行为。在一个变量总是引用的语言（如Java或Python），
`s2` 变成另一个引用也指向被 `s1` 指向的字符串对象。在C++中，`s1` 是值类型，
且它被_拷贝_给了 `s2`。

但Rust却move了这个值。它没有将string看作为可拷贝的对象("因没有实现Copy trait")。

我们不会将它看作仅仅是值的'基础'类型，如数字；基本类型能被允许拷贝是因为它们
的拷贝代价很低。但这里 `String` 分配了内存来保存 "Hello dolly"，拷贝将导致分配
更多内存来保存这些字符。Rust不会悄悄这么做。

考虑一个 `String` 对象包含 'Moby-Dick' 全部字符。这不是个很大的结构，只有文本的
内存地址，长度，分配的块大小。拷贝它的代价很大，因为内存被分配在堆上，拷贝将引起
新的内存块分配。

```
    String
    | addr | ---------> Call me Ishmael.....
    | size |                    |
    | cap  |                    |
                                |
    &str                        |
    | addr | -------------------|
    | size |

    f64
    | 8 bytes |
```

第二个值是个指向了相同字符串地址的string slice (`&str`)，和一个大小——只有名称，
拷贝代价低！

第三个值是一个 `f64` - 只有 8 字节大小。它不指向任何内存地址，所以拷贝它和移动它
的代价一样低。

可 `Copy` 的值都只定义在栈内存中，当Rust拷贝时，只将那些字节从栈的某个位置拷贝到另一个位置，
类似的，不可 `Copy` 的值就 _只能移动_。不像C++，这里拷贝或移动没有聪明与否的区别。

重写成一个函数调用揭示了完全相同的错误：

```rust
// move2.rs

fn dump(s: String) {
    println!("{}", s);
}

fn main() {
    let s1 = "hello dolly".to_string();
    dump(s1);
    println!("s1 {}", s1); // <---error: 'value used here after move'
}
```

这里你有其他选择。可以传递一个字符引用，或者显式调用它的 `clone` 方法。
一般来说，第一种方法更好。

```rust
fn dump(s: &String) {
    println!("{}", s);
}

fn main() {
    let s1 = "hello dolly".to_string();
    dump(&s1);
    println!("s1 {}", s1);
}
```

错误消失了。但你将很少看到一个简单 `String` 引用像这样传递，这种方式很丑_且_
引入了临时字符串变量。

```rust
    dump(&"hello world".to_string());
```

所以另个更好的函数声明方式如下的：

```rust
fn dump(s: &str) {
    println!("{}", s);
}
```

然后 `dump(&s1)` 和 `dump("hello world")` 都可以正常工作。(这里Rust的强制解引用 `Deref`
为你把 `&String` 转换成 `&str`。)

总结一下，分配non-Copy的值时它将被从一处移动到另一处。其他情况，Rust将强制_隐式_拷贝，
而不显式的分配内存。

## 变量范围

所以，经验法则是采用指向原始数据的引用——来'borrow'它。

但引用_不能_超出owner的生命周期。

首先，Rust是个 _block-scoped_ 语言。变量只在块范围中存活。

```rust
{
    let a = 10;
    let b = "hello";
    {
        let c = "hello".to_string();
        // a, b and c are visible
    }
    // the string c is dropped
    // a, b are visible
    for i in 0..a {
        let b = &b[1..];
        // original b is no longer visible - it is shadowed.
    }
    // the slice b is dropped, original b is visible again
    // i is _not_ visible!
}
```
Loop 变量(如e `i`) 有点不同，它们只在loop范围内有效。创建一个同名变量并不会出错(因为有'shadowing')
但容易造成迷惑。

当一个变量 'goes out of scope' 那么它将被 _dropped_。它使用的任何内存都会被回收，
且该变量拥有的任何其他 _resources_ 都会返还给系统。例如关闭一个文件句柄 `File` 。
不再使用的资源被立即回收。这是好事。

(一个更进一步的Rust特有错误是，变量还在该范围，但它的值已经被move走了。)

这里一个引用 `rs1` 执向了值 `tmp` ，但值仅在所在内层块中有效：

```rust
// ref1.rs
fn main() {
    let s1 = "hello dolly".to_string();
    let mut rs1 = &s1;
    {
        let tmp = "hello world".to_string();
        rs1 = &tmp;
    }
    println!("ref {}", rs1);
}
```

我们 borrow 了变量 `s1` 的值然后又 borrow 了变量 `tmp`的值。但 `tmp`的值在超出该块范围后
不再存在了！

```
error: `tmp` does not live long enough
  --> ref1.rs:8:5
   |
7  |         rs1 = &tmp;
   |                --- borrow occurs here
8  |     }
   |     ^ `tmp` dropped here while still borrowed
9  |     println!("ref {}", rs1);
10 | }
   | - borrowed value needs to live until here
```
`tmp`去哪了？离开，销毁，返回给内存堆栈空间: _dropped_。
Rust 这里将你从可怕的C语言 '悬空指针' 问题中拯救出来 —— 一个指针指向了脏数据。

## Tuples 元组

有时需要从函数中返回多个值。元组Tuple就是对应的简便方案：

```rust
// tuple1.rs

fn add_mul(x: f64, y: f64) -> (f64,f64) {
    (x + y, x * y)
}

fn main() {
    let t = add_mul(2.0,10.0);

    // 可以 debug 打印
    println!("t {:?}", t);

    // 可以用索引引用 'index' 各值
    println!("add {} mul {}", t.0,t.1);

    // 可以 _抽出_ 各值
    let (add,mul) = t;
    println!("add {} mul {}", add,mul);
}
// t (12, 20)
// add 12 mul 20
// add 12 mul 20
```

Tuples 可以包含  _不同_ 的类型，这是它与数组arrays的主要区别。

```rust
let tuple = ("hello", 5, 'c');

assert_eq!(tuple.0, "hello");
assert_eq!(tuple.1, 5);
assert_eq!(tuple.2, 'c');
```
它们出现在一些 `Iterator` 方法中。`enumerate` 很像Python的同名生成器:

```rust
    for t in ["zero","one","two"].iter().enumerate() {
        print!(" {} {};",t.0,t.1);
    }
    //  0 zero; 1 one; 2 two;
```
`zip` 把两个元组的iterators组合成一个iterator来枚举出两者的值:

```rust
    let names = ["ten","hundred","thousand"];
    let nums = [10,100,1000];
    for p in names.iter().zip(nums.iter()) {
        print!(" {} {};", p.0,p.1);
    }
    //  ten 10; hundred 100; thousand 1000;
```

## Structs 结构体

Tuples 很方便，但看着 `t.1` 并持续跟踪元组的每一部分不那么直观的值是很乏味的。

Rust的结构体 _structs_ 包含了 _fields_:

```rust
// struct1.rs

struct Person {
    first_name: String,
    last_name: String
}

fn main() {
    let p = Person {
        first_name: "John".to_string(),
        last_name: "Smith".to_string()
    };
    println!("person {} {}", p.first_name,p.last_name);
}
```

结构体的每个成员值在内存中会靠近放置，即便如此你应该别假设它的内存排列有任何特殊之处，
因为编译器为把内存组织的更有效而不太在意大小，可能填充对齐。

初始化结构体有点笨拙，所以我们把 `Person` 的构造函数变成它自己的。这个函数可以通过
`impl` 块变成`Person` 的  _associated function_ :

```rust
// struct2.rs

struct Person {
    first_name: String,
    last_name: String
}

impl Person {

    fn new(first: &str, name: &str) -> Person {
        Person {
            first_name: first.to_string(),
            last_name: name.to_string()
        }
    }

}

fn main() {
    let p = Person::new("John","Smith");
    println!("person {} {}", p.first_name,p.last_name);
}
```
这里命名为 `new` 没有神奇或保留之处。注意它采用了C++风格的访问方式双冒号`::`。

下面的 `Person` _method_ 使用了 _reference self_ 参数:

```rust
impl Person {
    ...

    fn full_name(&self) -> String {
        format!("{} {}", self.first_name, self.last_name)
    }

}
...
    println!("fullname {}", p.full_name());
// fullname John Smith
```
`self` 参数是显式使用，并作为一个引用传递。(你可以认为 `&self` 是 `self: &Person`的缩写)

关键字 `Self` 表示该结构体类型 —— 你也可以直接使用 `Person` 代替 `Self`:

```rust
    fn copy(&self) -> Self {
        Self::new(&self.first_name,&self.last_name)
    }
```

这些方法也允许修改结构体内部数据，通过引入 _mutable self_ 参数:

```rust
    fn set_first_name(&mut self, name: &str) {
        self.first_name = name.to_string();
    }
```
数据将被 _move_ 方法内当使用 self 对象作为参数时:

```rust
    fn to_tuple(self) -> (String,String) {
        (self.first_name, self.last_name)
    }
```
(尝试用 `&self` —— 结构体将不允许成员变量跑出去!)

注意在调用 `v.to_tuple()` 后，`v` 被move了而不再可用。

总结一下:
  - 没有 `self` 参数: 可以通过结构体来关联调用函数，就像 `new` "构造函数"一样。
  - 有 `&self` 参数: 可以用结构体的变量来引用函数，但不能修改结构体变量的成员值。
  - 有 `&mut self` 参数: 可以修改结构体变量的成员值。
  - 有 `self` 参数: 将消费该结构体变量，也就是 move 了它。

如果你尝试debug dump `Person` 变量，你将得到以下错误信息:

```
error[E0277]: the trait bound `Person: std::fmt::Debug` is not satisfied
  --> struct2.rs:23:21
   |
23 |     println!("{:?}", p);
   |                     ^ the trait `std::fmt::Debug` is not implemented for `Person`
   |
   = note: `Person` cannot be formatted using `:?`; if it is defined in your crate,
    add `#[derive(Debug)]` or manually implement it
   = note: required by `std::fmt::Debug::fmt`
```
编译器给了建议，所以我们将 `#[derive(Debug)]` 放在 `Person` 前，这样就有了有意义的输出:

```
Person { first_name: "John", last_name: "Smith" }
```

这个 _directive_ 让编译器生成 `Debug` trait的实现，非常有用。这对你的结构体是很好的实践，
所以可以打印出(或用 `format!` 写成字符串).  (如果 _全默认_ 这么做就非常不像Rust了)

最终这个小的程序是这样:

```rust
// struct4.rs
use std::fmt;

#[derive(Debug)]
struct Person {
    first_name: String,
    last_name: String
}

impl Person {

    fn new(first: &str, name: &str) -> Person {
        Person {
            first_name: first.to_string(),
            last_name: name.to_string()
        }
    }

    fn full_name(&self) -> String {
        format!("{} {}",self.first_name, self.last_name)
    }

    fn set_first_name(&mut self, name: &str) {
        self.first_name = name.to_string();
    }

    fn to_tuple(self) -> (String,String) {
        (self.first_name, self.last_name)
    }
}

fn main() {
    let mut p = Person::new("John","Smith");

    println!("{:?}", p);

    p.set_first_name("Jane");

    println!("{:?}", p);

    println!("{:?}", p.to_tuple());
    // p has now moved.

}
// Person { first_name: "John", last_name: "Smith" }
// Person { first_name: "Jane", last_name: "Smith" }
// ("Jane", "Smith")
```

## 紧叮生命周期

通常结构体包含值变量成员，但它们也常需要包含引用成员。
比如我们想放一个字符切片到结构体中，而不是字符串变量。

```rust
// life1.rs

#[derive(Debug)]
struct A {
    s: &str
}

fn main() {
    let a = A { s: "hello dammit" };

    println!("{:?}", a);
}
```

```
error[E0106]: missing lifetime specifier
 --> life1.rs:5:8
  |
5 |     s: &str
  |        ^ expected lifetime parameter
```
为理解上面的报错，你需要从Rust编译器的角度看问题。它将不允许一个引用
在不知道自己生命周期的情况下存在。所有的引用都是从一些值变量上借用而来，
所有的值变量都有生命周期。所以引用的生命周期不能比对应值变量的长。Rust
任何情况下都无法允许一个引用突然变得无效。

现在，字符串切片是用_字符常量_如"hello"或字符串变量 `String` 借用而来。
字符常量在整个程序的生命周期存在，它被称为是'static'的生命周期。

所以这就可以了——我们假设Rust字符串切片总是引用于静态字符串变量。

```rust
// life2.rs

#[derive(Debug)]
struct A {
    s: &'static str
}

fn main() {
    let a = A { s: "hello dammit" };

    println!("{:?}", a);
}
// A { s: "hello dammit" }
```

生命周期不是 _很好_ 的标注。但有时丑陋也是精确的必要代价。

以下可指定一个字符串切片用以作为函数的返回值：

```rust
fn how(i: u32) -> &'static str {
    match i {
    0 => "none",
    1 => "one",
    _ => "many"
    }
}
```
这能在静态字符串的特殊情况下可行，但有比较大的限制。

经管如此，我们可以指定一个引用的生命周期_最短_与结构体变量一样长。

```rust
// life3.rs

#[derive(Debug)]
struct A <'a> {
    s: &'a str
}

fn main() {
    let s = "I'm a little string".to_string();
    let a = A { s: &s };

    println!("{:?}", a);
}
```

生命周期常被称为 'a', 'b', 等等但你也可以把它称为 'me' 或其他。

在此，结构体 `a` 和字符串 `s` 都被严格契约限制为:
`a` 从字符串 `s` 借用，生命周期不能长于后者。

基于这个结构体定义，我们可以编写一个函数返回 `A` 结构体值:

```rust
fn makes_a() -> A {
    let string = "I'm a little string".to_string();
    A { s: &string }
}
```

但 `A` 也需要注明生命周期 - "expected lifetime parameter":

```
  = help: this function's return type contains a borrowed value,
   but there is no value for it to be borrowed from
  = help: consider giving it a 'static lifetime
```
`rustc` 给了些建议，我们遵从:

```rust
fn makes_a() -> A<'static> {
    let string = "I'm a little string".to_string();
    A { s: &string }
}
```
但现在错误变成：

```
8 |      A { s: &string }
  |              ^^^^^^ does not live long enough
9 | }
  | - borrowed value only lives until here
```

这就没办法让它安全又可行。因为 `string` 在该函数结束时将被丢弃，没有任何指向它的引用能
超过它的生命周期。

你可以实用的认为生命周期参数是变量类型的一部分。

有时让一个结构体包含一个引用_并_指向它的一个成员变量看起来是好主意。这基本是不可能的，
因为结构体必须是_可移动的_，任何移动操作会导致引用无效。没必要这么做，例如你的结构体
有一个字符串变量成员，在需要提供字符串切片时，可以使用一个位置变量，并提供一个方法函数
来确保返回的是特定字符串切片。

## Traits 特性

请注意Rust没有称 `struct` 为 _类_。关键字 `class` 在别的语言中被过度使而有效的屏蔽了
原有意思。

让我们这么看：Rust结构体不能_继承_其他结构体；它们每一个都是独特类型，没有_子类型_。
他们是哑数据（相互之间无法直接联系）。

那么_如何_建立类型之间的联系？这就是 _traits_ 要发挥的作用。

`rustc` 常提到应 `实现 X trait` ，所以该适当探讨下 traits 了。

这里有个小例子定义一个 trait 并为另一个类型 _实现_ 了它。

```rust
// trait1.rs

trait Show {
    fn show(&self) -> String;
}

impl Show for i32 {
    fn show(&self) -> String {
        format!("four-byte signed {}", self)
    }
}

impl Show for f64 {
    fn show(&self) -> String {
        format!("eight-byte float {}", self)
    }
}

fn main() {
    let answer = 42;
    let maybe_pi = 3.14;
    let s1 = answer.show();
    let s2 = maybe_pi.show();
    println!("show {}", s1);
    println!("show {}", s2);
}
// show four-byte signed 42
// show eight-byte float 3.14
```
这真酷，我们_添加了新方法_ 到 `i32` 和 `f64`!

要习惯Rust在标准库中引入了基本traits（它们将成群结队的出现）。

`Debug` 很常见。我们可以用 `#[derive(Debug)]` 给 `Person` 一个默认的实现，
但如果说我们希望 `Person` 只显示它的full name:

```rust
use std::fmt;

impl fmt::Debug for Person {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.full_name())
    }
}
...
    println!("{:?}", p);
    // John Smith
```
`write!` 是个很有用的宏——这里 `f` 可以是任何实现了 `Write` 特性的类型变量。
(可以是个 `File` - 或甚至是 `String`.)

`Display` 控制了值如何通过 "{}" 被打印出来，被实现的如 `Debug` 一样。
有个副作用是，`ToString` 也被任何实现了 `Display` 自动实现。所以如果
我们为 `Person` 实现了 `Display` ，那么 `p.to_string()` 也能用了。

`Clone` 定义了方法 `clone`，能简单的使用 `#[derive(Clone)]"` 来标注一个类型，
前提是它所有的成员变量类型也实现了 `Clone`。

## 例子: 迭代一个浮点数范围

我前面用过范围 (`0..n`) 但它们没法用浮点数。(你可以_强制_用浮点数但它会无趣的结束在1.0上）。

回忆迭代器interator的不正式定义；它是一个有 `next` 方法的结构体，这方法会返回 `Some` 或 `None`。
在这过程中，迭代器会修改自己，它会保持迭代的状态（如下一个序号等）。而迭代的数据本身一般不会被
修改，(但看看 `Vec::drain` 是个能修改自己数据的有趣的迭代器)

这里有迭代器的正式定义: [Iterator trait](https://doc.rust-lang.org/std/iter/trait.Iterator.html).

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    ...
}
```
这里我们遇到了 [associated type](https://doc.rust-lang.org/stable/book/associated-types.html) 是 `Iterator` trait
的关联类型。这个trait需要对任何类型都可用，所以需要指定一个返回类型。写 `next` 方法时就不用其他特殊类型——相反
用 `Self` 来引用自定义类型参数 `Item` 。

`f64` 的迭代器trait可以写成 `Iterator<Item=f64>`，它能被读作"一个迭代器的关联类型Item被设为f64".

`...` 表示`Iterator` _已提供的方法_。你只需要定义 `Item` 和 `next`，其他方法已为你的定义好了。

```rust
// trait3.rs

struct FRange {
    val: f64,
    end: f64,
    incr: f64
}

fn range(x1: f64, x2: f64, skip: f64) -> FRange {
    FRange {val: x1, end: x2, incr: skip}
}

impl Iterator for FRange {
    type Item = f64;

    fn next(&mut self) -> Option<Self::Item> {
        let res = self.val;
        if res >= self.end {
            None
        } else {
            self.val += self.incr;
            Some(res)
        }
    }
}


fn main() {
    for x in range(0.0, 1.0, 0.1) {
        println!("{} ", x);
    }
}
```
那么它们看起来如下：

```
0
0.1
0.2
0.30000000000000004
0.4
0.5
0.6
0.7
0.7999999999999999
0.8999999999999999
0.9999999999999999
```
这是因为 0.1 不是一个精确的浮点数表达，所以用点格式是必要的。修改 `println!` 为

```rust
println!("{:.1} ", x);
```
那我们会得到更清楚的输出 (这[format](https://doc.rust-lang.org/std/fmt/index.html)
 表示 '小数点后只保留一位数字'.)

迭代器的所有默认方法是可用的，所以我们能将那些值收集到一个vector, map, 或其他。

```rust
    let v: Vec<f64> = range(0.0, 1.0, 0.1).map(|x| x.sin()).collect();
```

## Generic Functions 泛型函数

我们需要一个能dump任何实现了`Debug`的类型值。这里第一次尝试泛型函数，能传递一个
_任意_类型值的引用。`T` 是个类型参数，需要紧挨着函数名来声明它:

```rust
fn dump<T> (value: &T) {
    println!("value is {:?}",value);
}

let n = 42;
dump(&n);
```
尽管如此，Rust完全不知道关于泛型 `T` 任何细节:


```
error[E0277]: the trait bound `T: std::fmt::Debug` is not satisfied
...
   = help: the trait `std::fmt::Debug` is not implemented for `T`
   = help: consider adding a `where T: std::fmt::Debug` bound
```
要这能行，Rust需要被告知 `T` 确实实现了 `Debug`!

```rust
fn dump<T> (value: &T)
where T: std::fmt::Debug {
    println!("value is {:?}",value);
}

let n = 42;
dump(&n);
// value is 42
```
Rust 泛型函数需要对类型参数作 _特性约束_ —— 我们这么形容"T是任何实现了Debug的类型"。
`rustc` 非常有用，能精确的建议到底需要什么样的约束。

现在Rust知道了约束 `T` 的特性，能给你有意义的编译信息:

```rust
struct Foo {
    name: String
}

let foo = Foo{name: "hello".to_string()};

dump(&foo)
```
这里的错误是 "还没有为`Foo`实现特性 `std::fmt::Debug` "。

在动态语言中由于变量带着类型一起，函数已是泛型的，在执行的时候检查类型——或失败得很惨。
在大型程序中，我们只想在编译时知道全部问题！相比冷静的坐着看编译错误，那些动态语言的
程序员只想费力处理那些在运行时才出现的错误。Murphy's Law 暗示那些问题只可能发生在困难
的/灾难的时刻。

平方数操作是泛型的： `x*x` 
对整数浮点数和所有知道乘法操作符 `*` 的类型都可以。
那这个类型的约束是什么？

```rust
// gen1.rs

fn sqr<T> (x: T) -> T {
    x * x
}

fn main() {
    let res = sqr(10.0);
    println!("res {}",res);
}
```
第一个问题是Rust不知道 `T` 可以被相乘:

```
error[E0369]: binary operation `*` cannot be applied to type `T`
 --> gen1.rs:4:5
  |
4 |     x * x
  |     ^
  |
note: an implementation of `std::ops::Mul` might be missing for `T`
 --> gen1.rs:4:5
  |
4 |     x * x
  |     ^
```
根据编译器的建议，我们来用 [这trait](https://doc.rust-lang.org/std/ops/trait.Mul.html) 约束下类型参数，
该trait被用于实现乘法操作符 `*`:

```rust
fn sqr<T> (x: T) -> T
where T: std::ops::Mul {
    x * x
}
```

还是不行:

```
rror[E0308]: mismatched types
 --> gen2.rs:6:5
  |
6 |     x * x
  |     ^^^ expected type parameter, found associated type
  |
  = note: expected type `T`
  = note:    found type `<T as std::ops::Mul>::Output`
```
`rustc` 实际上是在说 `x*x` 的类型是关联在 `T::Output`上而非 `T`。
没有理由认为 `x*x` 的类型与 `x` 一样，如两个向量的.乘是一个标量。

```rust
fn sqr<T> (x: T) -> T::Output
where T: std::ops::Mul {
    x * x
}
```

现在错误是:

```
error[E0382]: use of moved value: `x`
 --> gen2.rs:6:7
  |
6 |     x * x
  |     - ^ value used here after move
  |     |
  |     value moved here
  |
  = note: move occurs because `x` has type `T`, which does not implement the `Copy` trait
```

所以我们需要更多限制这个类型参数！

```rust
fn sqr<T> (x: T) -> T::Output
where T: std::ops::Mul + Copy {
    x * x
}
```
最终可行了。冷静的看看编译器错误常让你更接近魔法时刻，当所有的 ... 都编译干净.

这 _是_ 个C++的类似例子:

```cpp
template <typename T>
T sqr(x: T) {
    return x * x;
}
```
如实说这里 C++ 用了粗旷的牛仔战术。C++ 模板错误是出名的糟糕，因为编译器最终知道的
是有些函数或操作符没有定义。C++委员会知道这是个问题，所有他们正在研究[concepts](https://en.wikipedia.org/wiki/Concepts_(C%2B%2B))，
它很像Rust中对类型参数的trait约束。

Rust泛型函数开头看起来有点过头，但显式声明意味着你完全知道什么类型的值可以安全的喂给它，
只要看看定义。

那些被称为单态_monomorphic_ 的函数与被称为多态_polymorphic_ 的函数相反。前者的函数体
被编译器为每个类型编译成了独立的机器码。而后者以相同的机器代码来与任何匹配的类型一起执行，
动态的 _dispatching_ 执行方法。

单态能生成执行更快的代码，特别是为有些特殊类型常能生成 _inlined_ 函数。所以当看到 `sqr(x)`，
它被有效的替换成了 `x*x`。缺点是大的泛型函数将生成很多代码，每个类型都有对应的，造成 _code bloat_。
总是有个平衡取舍；一个有经验的程序员会学着为此做出正确的选择。

## Simple Enums

Enums are types which have a few definite values. For instance, a direction has
only four possible values.

```rust
enum Direction {
    Up,
    Down,
    Left,
    Right
}
...
    // `start` is type `Direction`
    let start = Direction::Left;
```
They can have methods defined on them, just like structs.
The  `match` expression is the basic way to handle `enum` values.

```rust
impl Direction {
    fn as_str(&self) -> &'static str {
        match *self { // *self has type Direction
            Direction::Up => "Up",
            Direction::Down => "Down",
            Direction::Left => "Left",
            Direction::Right => "Right"
        }
    }
}
```

Punctuation matters. Note that `*` before `self`. It's easy to forget, because often
Rust will assume it (we said `self.first_name`, not `(*self).first_name`). However,
matching is a more exact business. Leaving it out would give a whole spew of messages,
which boil down to this type mismatch:

```
   = note: expected type `&Direction`
   = note:    found type `Direction`
```
This is because `self` has type `&Direction`, so we have to throw in the `*` to
_deference_ the type.

Like structs, enums can implement traits, and our friend `#[derive(Debug)]` can
be added to `Direction`:

```rust
        println!("start {:?}",start);
        // start Left
```
So that `as_str` method isn't really necessary, since we can always get the name from `Debug`.
(But `as_str` does _not allocate_, which may be important.)

You should not assume any particular ordering here - there's no implied integer
'ordinal' value.

Here's a method which defines the 'successor' of each `Direction` value. The
very handy _wildcard use_ temporarily puts the enum names into the method context:

```rust
    fn next(&self) -> Direction {
        use Direction::*;
        match *self {
            Up => Right,
            Right => Down,
            Down => Left,
            Left => Up
        }
    }
    ...

    let mut d = start;
    for _ in 0..8 {
        println!("d {:?}", d);
        d = d.next();
    }
    // d Left
    // d Up
    // d Right
    // d Down
    // d Left
    // d Up
    // d Right
    // d Down
```
So this will cycle endlessly through the various directions in this particular, arbitrary,
order. It is (in fact) a very simple _state machine_.

These enum values can't be compared:

```
assert_eq!(start, Direction::Left);

error[E0369]: binary operation `==` cannot be applied to type `Direction`
  --> enum1.rs:42:5
   |
42 |     assert_eq!(start, Direction::Left);
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
note: an implementation of `std::cmp::PartialEq` might be missing for `Direction`
  --> enum1.rs:42:5
```

The solution is to say `#[derive(Debug,PartialEq)]` in front of `enum Direction`.

This is an important point - Rust user-defined types start out fresh and unadorned.
You give them sensible default behaviours by implementing the common traits. This
applies also to structs - if you ask for Rust to derive `PartialEq` for a struct it
will do the sensible thing, assume that all fields implement it and build up
a comparison. If this isn't so, or you want to redefine equality, then you are free
to define `PartialEq` explicitly.

Rust does 'C style enums' as well:

```rust
// enum2.rs

enum Speed {
    Slow = 10,
    Medium = 20,
    Fast = 50
}

fn main() {
    let s = Speed::Slow;
    let speed = s as u32;
    println!("speed {}", speed);
}
```
They are initialized with an integer value, and can be converted into that integer
with a type cast.

You only need to give the first name a value, and thereafter the
value goes up by one each time:

```rust
enum Difficulty {
    Easy = 1,
    Medium,  // is 2
    Hard   // is 3
}
```

By the way, 'name' is too vague, like saying 'thingy' all the time. The proper term here
is _variant_ - `Speed` has variants `Slow`,`Medium` and `Fast`.

These enums _do_ have a natural ordering, but you have to ask nicely.
After placing `#[derive(PartialEq,PartialOrd)]` in front of `enum Speed`, then it's indeed
true that `Speed::Fast > Speed::Slow` and `Speed::Medium != Speed::Slow`.

## Enums in their Full Glory

Rust enums in their full form are like C unions on steroids, like a Ferrari compared
to a Fiat Uno. Consider the problem of storing different values in a type-safe way.

```rust
// enum3.rs

#[derive(Debug)]
enum Value {
    Number(f64),
    Str(String),
    Bool(bool)
}

fn main() {
    use Value::*;
    let n = Number(2.3);
    let s = Str("hello".to_string());
    let b = Bool(true);

    println!("n {:?} s {:?} b {:?}", n,s,b);
}
// n Number(2.3) s Str("hello") b Bool(true)
```
Again, this enum can only contain _one_ of these values; its size will be the size of
the largest variant.

So far, not really a supercar, although it's cool that enums know how to print themselves
out. But they also know how _what kind_ of value they contain, and _that_ is the
superpower of `match`:

```rust
fn eat_and_dump(v: Value) {
    use Value::*;
    match v {
        Number(n) => println!("number is {}", n),
        Str(s) => println!("string is '{}'", s),
        Bool(b) => println!("boolean is {}", b)
    }
}
....
eat_and_dump(n);
eat_and_dump(s);
eat_and_dump(b);
//number is 2.3
//string is 'hello'
//boolean is true
```

(And that's what `Option` and `Result` are - enums.)

We like this `eat_and_dump` function, but we want to pass the value as a reference, because currently
a move takes place and the value is 'eaten':

```rust
fn dump(v: &Value) {
    use Value::*;
    match *v {  // type of *v is Value
        Number(n) => println!("number is {}", n),
        Str(s) => println!("string is '{}'", s),
        Bool(b) => println!("boolean is {}", b)
    }
}

error[E0507]: cannot move out of borrowed content
  --> enum3.rs:12:11
   |
12 |     match *v {
   |           ^^ cannot move out of borrowed content
13 |     Number(n) => println!("number is {}",n),
14 |     Str(s) => println!("string is '{}'",s),
   |         - hint: to prevent move, use `ref s` or `ref mut s`
```
There are things you cannot do with borrowed references. Rust is not letting
you _extract_ the string contained in the original value. It did not complain about `Number`
because it's happy to copy `f64`, but `String` does not implement `Copy`.

I mentioned earlier that `match` is picky about _exact_ types;
here we follow the hint and things will work; now we are just borrowing a reference
to that contained string.

```rust
fn dump(v: &Value) {
    use Value::*;
    match *v {
        Number(n) => println!("number is {}", n),
        Str(ref s) => println!("string is '{}'", s),
        Bool(b) => println!("boolean is {}", b)
    }
}
    ....

    dump(&s);
    // string is 'hello'
```
Before we move on, filled with the euphoria of a successful Rust compilation, let's
pause a little. `rustc` is unusually good at generating errors that have enough
context for a human to _fix_ the error without necessarily _understanding_ the error.

The issue is a combination of the exactness of matching, with the determination of the
borrow checker to foil any attempt to break the Rules.  One of those Rules is that
you cannot yank out a value which belongs to some owning type. Some knowledge of
C++ is a hindrance here, since C++ will copy its way out of the problem, whether that
copy even _makes sense_.  You will get exactly the same error if you try to pull out
a string from a vector, say with `*v.get(0).unwrap()` (`*` because indexing returns references.)
It will simply not let you do this. (Sometimes `clone` isn't such a bad solution to this.)

(By the way, `v[0]` does not work for non-copyable values like strings for precisely this reason.
You must either borrow with `&v[0]` or clone with `v[0].clone()`)

As for `match`, you can see `Str(s) =>` as short for `Str(s: String) =>`. A local variable
(often called a _binding_) is created.  Often that inferred type is cool, when you
eat up a value and extract its contents. But here we really needed is `s: &String`, and the
`ref` is a hint that ensures this: we just want to borrow that string.

Here we do want to extract that string, and don't care about
the enum value afterwards. `_` as usual will match anything.

```rust
impl Value {
    fn to_str(self) -> Option<String> {
        match self {
        Value::Str(s) => Some(s),
        _ => None
        }
    }
}
    ...
    println!("s? {:?}", s.to_str());
    // s? Some("hello")
    // println!("{:?}", s) // error! s has moved...
```
Naming matters - this is called `to_str`, not `as_str`. You can write a
method that just borrows that string as an `Option<&String>` (The reference will need
the same lifetime as the enum value.)  But you would not call it `to_str`.

You can write `to_str` like this - it is completely equivalent:

```rust
    fn to_str(self) -> Option<String> {
        if let Value::Str(s) = self {
            Some(s)
        } else {
            None
        }
    }
```

## More about Matching

Recall that the values of a tuple can be extracted with '()':

```rust
    let t = (10,"hello".to_string());
    ...
    let (n,s) = t;
    // t has been moved. It is No More
    // n is i32, s is String
```

This is a special case of _destructuring_; we have some
data and wish to either pull it apart (like here) or just borrow its values.
Either way, we get the parts of a structure.

The syntax is like that used in `match`. Here
we are explicitly borrowing the values.

```rust
    let (ref n,ref s) = t;
    // n and s are borrowed from t. It still lives!
    // n is &i32, s is &String
```

Destructuring works with structs as well:

```rust
    struct Point {
        x: f32,
        y: f32
    }

    let p = Point{x:1.0,y:2.0};
    ...
    let Point{x,y} = p;
    // p still lives, since x and y can and will be copied
    // both x and y are f32
```

Time to revisit `match` with some new patterns. The first two patterns are exactly like `let`
destructuring - it only matches tuples with first element zero, but _any_ string;
the second adds an `if` so that it only matches `(1,"hello")`.
Finally, just a variable matches _anything_. This is useful if the `match` applies
to an expression and you don't want to bind a variable to that expression. `_` works
like a variable but is ignored. It's a common
way to finish off a `match`.

```rust
fn match_tuple(t: (i32,String)) {
    let text = match t {
        (0, s) => format!("zero {}", s),
        (1, ref s) if s == "hello" => format!("hello one!"),
        tt => format!("no match {:?}", tt),
        // or say _ => format!("no match") if you're not interested in the value
     };
    println!("{}", text);
}
```

Why not just match against `(1,"hello")`? Matching is an exact business, and the compiler
will complain:

```
  = note: expected type `std::string::String`
  = note:    found type `&'static str`
```

Why do we need `ref s`? It's a slightly obscure gotcha (look up the E00008 error) where
if you have an _if guard_ you need to borrow, since the if guard happens in a different
context, a move will take place otherwise. It's a case of the implementation leaking
ever so slightly.

If the type _was_ `&str` then we match it directly:

```rust
    match (42,"answer") {
        (42,"answer") => println!("yes"),
        _ => println!("no")
    };
```

What applies to `match` applies to `if let`. This is a cool example, since if we
get a `Some`, we can match inside it and only extract the string from the tuple. So it
isn't necessary to have nested `if let` statements here. We use `_` because we aren't interested
in the first part of the tuple.

```rust
    let ot = Some((2,"hello".to_string());

    if let Some((_,ref s)) = ot {
        assert_eq!(s, "hello");
    }
    // we just borrowed the string, no 'destructive destructuring'
```

An interesting problem happens when using `parse` (or any function which needs to work
out its return type from context)

```rust
    if let Ok(n) = "42".parse() {
        ...
    }
```

So what's the type of `n`? You have to give a hint somehow - what kind of integer? Is it
even an integer?

```rust
    if let Ok(n) = "42".parse::<i32>() {
        ...
    }
```

This somewhat non-elegant syntax is called the 'turbofish operator'.

If you are in a function returning `Result`, then the question-mark operator provides a much
more elegant solution:

```rust
    let n: i32 = "42".parse()?;
```
However, the parse error needs to be convertible to the error type of the `Result`, which is a topic
we'll take up later when discussing [error handling](6-error-handling.html).

## Closures

A great deal of Rust's power comes from _closures_. In their simplest form, they
act like shortcut functions:

```rust
    let f = |x| x * x;

    let res = f(10);

    println!("res {}", res);
    // res 100
```

There are no explicit types in this example - everything is deduced, starting with the
integer literal 10.

We get an error if we call `f` on different types - Rust has already decided that
`f` must be called on an integer type:

```
    let res = f(10);

    let resf = f(1.2);
  |
8 |     let resf = f(1.2);
  |                  ^^^ expected integral variable, found floating-point variable
  |
  = note: expected type `{integer}`
  = note:    found type `{float}`

```

So, the first call fixes the type of the argument `x`. It's equivalent to this function:

```rust
    fn f (x: i32) -> i32 {
        x * x
    }
```

But there's a big difference between functions and closures, _apart_ from the need for explicit typing.
Here we evaluate a linear function:

```rust
    let m = 2.0;
    let c = 1.0;

    let lin = |x| m*x + c;

    println!("res {} {}", lin(1.0), lin(2.0));
    // res 3 5
```

You cannot do this with the explicit `fn` form - it does not know about variables
in the enclosing scope. The closure has _borrowed_ `m` and `c` from its context.

Now, what's the type of `lin`? Only `rustc` knows.
Under the hood, a closure is a _struct_ that is callable ('implements the call operator').
It behaves as if it was written out like this:

```rust
struct MyAnonymousClosure1<'a> {
    m: &'a f64,
    c: &'a f64
}

impl <'a>MyAnonymousClosure1<'a> {
    fn call(&self, x: f64) -> f64 {
        self.m * x  + self.c
    }
}
```
The compiler is certainly being helpful, turning simple closure syntax into all
that code! You do need to know that a closure is a _struct_ and it _borrows_ values
from its environment. And that therefore it has a _lifetime_.

All closures are unique types, but they have traits in common.
So even though we don't know the exact type, we know the generic constraint:

```rust
fn apply<F>(x: f64, f: F) -> f64
where F: Fn(f64)->f64  {
    f(x)
}
...
    let res1 = apply(3.0,lin);
    let res2 = apply(3.14, |x| x.sin());
```

In English: `apply` works for _any_ type `T` such that `T` implements `Fn(f64)->f64` - that
is, is a function which takes `f64` and returns `f64`.

After the call to `apply(3.0,lin)`, trying to access `lin` gives an interesting error:

```
    let l = lin;
error[E0382]: use of moved value: `lin`
  --> closure2.rs:22:9
   |
16 |     let res = apply(3.0,lin);
   |                         --- value moved here
...
22 |     let l = lin;
   |         ^ value used here after move
   |
   = note: move occurs because `lin` has type
    `[closure@closure2.rs:12:15: 12:26 m:&f64, c:&f64]`,
     which does not implement the `Copy` trait

```

That's it, `apply` ate our closure. And there's the actual type of the struct that
`rustc` made up to implement it. Always thinking of closures as structs is helpful.

Calling a closure is a _method call_:  the three kinds of function traits
correspond to the three kinds of methods:

  - `Fn` struct passed as `&self`
  - `FnMut` struct passed as `&mut self`
  - `FnOnce` struct passed as `self`

So it's possible for a closure to mutate its _captured_ references:

```rust
    fn mutate<F>(mut f: F)
    where F: FnMut() {
        f()
    }
    let mut s = "world";
    mutate(|| s = "hello");
    assert_eq!(s, "hello");
```

Note that `mut` - `f` needs to be mutable for this to work.

[#71: NLL makes this work]

However, you cannot escape the rules for borrowing. Consider this:

```rust
let mut s = "world";

// closure does a mutable borrow of s
let mut changer = || s = "world";

changer();
// does an immutable borrow of s
assert_eq!(s, "world");
```

Can't be done! The error is that we cannot borrow `s`
in the assert statement, because it has been previously borrowed by the
closure `changer` as mutable. As long as that closure lives, no other
code can access `s`, so the solution is to control that lifetime by
putting the closure in a limited scope:

```rust
let mut s = "world";
{
    let mut changer = || s = "world";
    changer();
}
assert_eq!(s, "world");
```

At this point, if you are used to languages like JavaScript or Lua, you may wonder at the
complexity of Rust closures compared with how straightforward they are in those languages.
This is the necessary cost of Rust's promise to not sneakily make any allocations. In JavaScript,
the equivalent `mutate(function() {s = "hello";})` will always result in a dynamically
allocated closure.

Sometimes you don't want a closure to borrow those variables, but instead _move_ them.

```rust
    let name = "dolly".to_string();
    let age = 42;

    let c = move || {
        println!("name {} age {}", name,age);
    };

    c();

    println!("name {}",name);
```

And the error at the last `println` is: "use of moved value: `name`". So one solution
here - if we _did_ want to keep `name` alive - is to move a cloned copy into the closure:

```rust
    let cname = name.to_string();
    let c = move || {
        println!("name {} age {}",cname,age);
    };
```
Why are moved closures needed? Because we might need to call them at a point where
the original context no longer exists.
A classic case is when creating a _thread_.
A moved closure does not borrow, so does not have a lifetime.

A major use of closures is within iterator methods. Recall the `range` iterator we
defined to go over a range of floating-point numbers. It's straightforward to operate
on this (or any other iterator) using closures:

```rust
    let sine: Vec<f64> = range(0.0,1.0,0.1).map(|x| x.sin()).collect();
```

`map` isn't defined on vectors (although it's easy enough to create a trait that does this),
because then _every_ map  will create a new vector.  This way, we have a choice. In this
sum, no temporary objects are created:

```rust
 let sum: f64 = range(0.0,1.0,0.1).map(|x| x.sin()).sum();
```

It will (in fact) be as fast as writing it out as an explicit loop! That performance
guarantee would be impossible if Rust closures were as 'frictionless'
as Javascript closures.

`filter` is another useful iterator method - it only lets through values that match
a condition:

```rust
    let tuples = [(10,"ten"),(20,"twenty"),(30,"thirty"),(40,"forty")];
    let iter = tuples.iter().filter(|t| t.0 > 20).map(|t| t.1);

    for name in iter {
        println!("{} ", name);
    }
    // thirty
    // forty
```

## The Three Kinds of Iterators

The three kinds correspond (again) to the three basic argument types. Assume we
have a vector of `String` values. Here are the iterator types explicitly, and
then _implicitly_, together with the actual type returned by the iterator.

```rust
for s in vec.iter() {...} // &String
for s in vec.iter_mut() {...} // &mut String
for s in vec.into_iter() {...} // String

// implicit!
for s in &vec {...} // &String
for s in &mut vec {...} // &mut String
for s in vec {...} // String
```
Personally I prefer being explicit, but it's important to understand both forms,
and their implications.

`into_iter` _consumes_ the vector and extracts its strings,
and so afterwards the vector is no longer available - it has been moved. It's
a definite gotcha for Pythonistas used to saying `for s in vec`!

 So the
implicit form `for s in &vec` is usually the one you want, just as `&T` is a good
default in passing arguments to functions.

It's important to understand how the three kinds works because Rust relies heavily
on type deduction - you won't often see explicit types in closure arguments. And this
is a Good Thing, because it would be noisy if all those types were explicitly
_typed out_. However, the price of this compact code is that you need to know
what the implicit types actually are!

`map` takes whatever value the iterator returns and converts it into something else,
but `filter` takes a _reference_ to that value. In this case, we're using `iter` so
the iterator item type is `&String`. Note that `filter` receives a reference to this type.

```rust
for n in vec.iter().map(|x: &String| x.len()) {...} // n is usize
....
}

for s in vec.iter().filter(|x: &&String| x.len() > 2) { // s is &String
...
}
```

When calling methods, Rust will derefence automatically, so the problem isn't obvious.
But `|x: &&String| x == "one"|` will _not_ work, because operators are more strict
about type matching. `rustc` will complain that there is no such operator that
compares `&&String` and `&str`. So you need an explicit deference to make that `&&String`
into a `&String` which _does_ match.

```rust
for s in vec.iter().filter(|x: &&String| *x == "one") {...}
// same as implicit form:
for s in vec.iter().filter(|x| *x == "one") {...}
```

If you leave out the explicit type, you can modify the argument so that the type of `s`
is now `&String`:

```rust
for s in vec.iter().filter(|&x| x == "one")
```

And that's usually how you will see it written.

## Structs with Dynamic Data

A most powerful technique is _a struct that contain references to itself_.

Here is the basic building block of a _binary tree_, expressed in C (everyone's
favourite old relative with a frightening fondness for using power tools without
protection.)

```rust
    struct Node {
        const char *payload;
        struct Node *left;
        struct Node *right;
    };
```

You can not do this by _directly_ including `Node` fields, because then the size of
`Node` depends on the size of `Node`... it just doesn't compute. So we use pointers
to `Node` structs, since the size of a pointer is always known.

If `left` isn't `NULL`, the `Node` will have a left pointing to another node, and so
moreorless indefinitely.

Rust does not do `NULL` (at least not _safely_) so it's clearly a job for `Option`.
But you cannot just put a `Node` in that `Option`, because we don't know the size
of `Node` (and so forth.)  This is a job for `Box`, since it contains an allocated
pointer to the data, and always has a fixed size.

So here's the Rust equivalent, using `type` to create an alias:

```rust
type NodeBox = Option<Box<Node>>;

#[derive(Debug)]
struct Node {
    payload: String,
    left: NodeBox,
    right: NodeBox
}
```
(Rust is forgiving in this way - no need for forward declarations.)

And a first test program:

```rust
impl Node {
    fn new(s: &str) -> Node {
        Node{payload: s.to_string(), left: None, right: None}
    }

    fn boxer(node: Node) -> NodeBox {
        Some(Box::new(node))
    }

    fn set_left(&mut self, node: Node) {
        self.left = Self::boxer(node);
    }

    fn set_right(&mut self, node: Node) {
        self.right = Self::boxer(node);
    }

}


fn main() {
    let mut root = Node::new("root");
    root.set_left(Node::new("left"));
    root.set_right(Node::new("right"));

    println!("arr {:#?}", root);
}
```
The output is surprisingly pretty, thanks to "{:#?}" ('#' means 'extended'.)

```
root Node {
    payload: "root",
    left: Some(
        Node {
            payload: "left",
            left: None,
            right: None
        }
    ),
    right: Some(
        Node {
            payload: "right",
            left: None,
            right: None
        }
    )
}
```
Now, what happens when `root` is dropped? All fields are dropped; if the 'branches' of
the tree are dropped, they drop _their_ fields and so on. `Box::new` may be the
closest you will get to a `new` keyword, but we have no need for `delete` or `free`.

We must now work out a use for this tree. Note that strings can be ordered:
'bar' < 'foo', 'abba' > 'aardvark'; so-called 'alphabetical order'. (Strictly speaking, this
is _lexical order_, since human languages are very diverse and have strange rules.)

Here is a method which inserts nodes in lexical order of the strings. We compare the new data
to the current node - if it's less, then we try to insert on the left, otherwise try to insert
on the right. There may be no node on the left, so then `set_left` and so forth.

```rust
    fn insert(&mut self, data: &str) {
        if data < &self.payload {
            match self.left {
                Some(ref mut n) => n.insert(data),
                None => self.set_left(Self::new(data)),
            }
        } else {
            match self.right {
                Some(ref mut n) => n.insert(data),
                None => self.set_right(Self::new(data)),
            }
        }
    }

    ...
    fn main() {
        let mut root = Node::new("root");
        root.insert("one");
        root.insert("two");
        root.insert("four");

        println!("root {:#?}", root);
    }
```

Note the `match` - we're pulling out a mutable reference to the box, if the `Option`
is `Some`, and applying the `insert` method. Otherwise, we need to create a new `Node`
for the left side and so forth. `Box` is a _smart_ pointer; note that no 'unboxing' was
needed to call `Node` methods on it!

And here's the output tree:

```
root Node {
    payload: "root",
    left: Some(
        Node {
            payload: "one",
            left: Some(
                Node {
                    payload: "four",
                    left: None,
                    right: None
                }
            ),
            right: None
        }
    ),
    right: Some(
        Node {
            payload: "two",
            left: None,
            right: None
        }
    )
}
```
The strings that are 'less' than other strings get put down the left side, otherwise
the right side.

Time for a visit. This is _in-order traversal_ - we visit the left, do something on
the node, and then visit the right.

```rust
    fn visit(&self) {
        if let Some(ref left) = self.left {
            left.visit();
        }
        println!("'{}'", self.payload);
        if let Some(ref right) = self.right {
            right.visit();
        }
    }
    ...
    ...
    root.visit();
    // 'four'
    // 'one'
    // 'root'
    // 'two'
```
So we're visiting the strings in order! Please note the reappearance of `ref` - `if let`
uses exactly the same rules as `match`.


## Generic Structs

Consider the previous example of a binary tree. It would be _seriously irritating_ to
have to rewrite it for all possible kinds of payload.
So here's our generic `Node` with its type parameter `T`.

```rust
type NodeBox<T> = Option<Box<Node<T>>>;

#[derive(Debug)]
struct Node<T> {
    payload: T,
    left: NodeBox<T>,
    right: NodeBox<T>
}
```

The implementation shows the difference between the languages. The fundamental operation
on the payload is comparison, so T must be comparable with `<`, i.e. implements `PartialOrd`.
The type parameter must be declared in the `impl` block with its constraints:


```rust
impl <T: PartialOrd> Node<T> {
    fn new(s: T) -> Node<T> {
        Node{payload: s, left: None, right: None}
    }

    fn boxer(node: Node<T>) -> NodeBox<T> {
        Some(Box::new(node))
    }

    fn set_left(&mut self, node: Node<T>) {
        self.left = Self::boxer(node);
    }

    fn set_right(&mut self, node: Node<T>) {
        self.right = Self::boxer(node);
    }

    fn insert(&mut self, data: T) {
        if data < self.payload {
            match self.left {
                Some(ref mut n) => n.insert(data),
                None => self.set_left(Self::new(data)),
            }
        } else {
            match self.right {
                Some(ref mut n) => n.insert(data),
                None => self.set_right(Self::new(data)),
            }
        }
    }
}


fn main() {
    let mut root = Node::new("root".to_string());
    root.insert("one".to_string());
    root.insert("two".to_string());
    root.insert("four".to_string());

    println!("root {:#?}", root);
}
```

So generic structs need their type parameter(s) specified
in angle brackets, like C++. Rust is usually smart enough to work out
that type parameter from context - it knows it has a `Node<T>`, and knows
that its `insert` method is passed `T`. The first call of `insert` nails
down `T` to be `String`. If any further calls are inconsistent it will complain.

But you do need to constrain that type appropriately!
