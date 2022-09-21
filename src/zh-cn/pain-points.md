## Pain Points 痛点

如实说Rust要比 _主流_ 语言难学。有些杰出的人不觉得它难，注意这里
_杰出_ 的严格定义 —— 他们是 _特例_。起先挣扎，然后成功。初始的困难
并不是后来能力的预言。

我们都来自某些地方，在编程语言情景中这意味着先前暴露在主流语言如
'动态'语言之一的Python或'静态'语言之一的C++。任何一条路上，Rust都
是充足的复杂而需要精神上的重新武装。聪明的人跳进来并失望于自己的
聪明没有立即被奖励；少有自我价值感的人认为他们还不够'聪明'。

对那些有动态语言经验的人(这里我包括Java)来说任何事都是引用，所有
的引用都是默认可修改的。然后垃圾回收让编写内存安全的程序容易点。
很多努力让JVM很快，以内存占用和预测为代价。通常认为这样的代价
是值得的 —— 旧的新观点认为程序员的生产力远比计算性能重要。

但世界上大多数计算机 —— 比如处理真正重要工作的汽车阀控制器 ——
并没有即便廉价笔记本拥有的大量资源，它们需要对事件 _实时_ 响应。
同样的，基本的基础设施软件需要是正确的、健壮的、快速的(旧的三位
一体工程)。很多这些东西是用不安全的C和C++编写的 —— 不安全的
总体代价就是这里要关注的。也许你很快把程序组装在一起，_然后_
真正的开发开始了。

系统语言不能承受垃圾回收，因为它是导致所有计算停滞的基础。
它们允许你对看到的资源浪费无计可施。

如果没有垃圾回收，内存需要通过其他方式管理。手动管理 —— 我申请内存，
使用它，然后显式释放它 —— 是很难做对。你可以用几周时间对C学习的足够
多以变得多产和危险 —— 但需要若干年成为好的安全的程序员，检查每一个
错误条件。

Rust管理内存像现代C++ —— 当对象销毁时，它们的内存被收回。你能用`Box`
在堆上分配内存，但在这个box在函数结束离开作用域时，内存被回收。所
以这里有点像`new`但不没有`delete`。你创建了一个`File`对象，在结束时文件
句柄资源被关闭。在Rust里这被称为 _废弃_。

你需要共享资源 —— 拷贝所有数据是很没效率的 —— 这是事情变得有趣的原因。
C++也有引用，尽管Rust的引用更像是C的指针 —— 你需要用`*r`来接引用值，
你需要用`&`来传递值的引用。

Rust的 _借用检查器_ 确保不让引用在原始数据被废弃的情况下存在。

## Type Inference 类型推导

静态和动态的区别不是全部情况。如在多数事情上，有更多维度来玩。
C是静态类型(每个变量在编译期有类型) 但是弱类型(如void*能指向任何
地方)；Python是动态类型(类型在值里而不在变量上)但是强类型。Java
是静态的/强类型(通过反射来方便的/危险的转换值)而Rust是静态的/强
类型，没有运行时反射。

Java以所有类型都需要令人麻木的_打_出来而出名，Rust喜欢 _推导_ 类型。
这是个好主意，但也表示你有时需要搞清楚实际类型是什么。你会看到
`let n = 100` 并好奇 —— 具体是什么整数类型？默认情况，它会是`i32` —— 
一个4字节的有符号整数。所有人都同意C里不指定整数类型(如`int`和`long`)
是不对的；最好显式指明。你可以总是拼出类型，就像`let n: u32 = 100`
或让常量强制类型，就像`let n = 100u32`。但类型推导比那更强！如果你
声明 `let n = 100` 然后 `rustc` 总知道`n`必须是 _某些_ 整数类型。如果
你把`n`传递给`u64`参数的函数那么将推导出`n`的类型就是它！

此后当你将`n`传给需要`u32`的函数时，`rustc`将不允许你这么做，因为`n`
已经被绑在`u64`上且它 _将不会_ 简单的为你转换这个整数。这就是
强类型行为 —— 这里没有那些虽导致你的过程平滑但有可能突然整数溢出
的小转换和类型提升。你需要显式的传递`n`为`n as u32` —— Rust中的typecast。
幸运的是，`rustc`很擅长在有行为的路径上突发坏消息 —— 即你能遵守编译器
的建议来修复问题。

所以Rust代码能不用显式类型:

```rust
let mut v = Vec::new();
// v is deduced to have type Vec<i32>
v.push(10);
v.push(20);
v.push("hello") <--- just can't do this, man!
```
不能将字符串放进整数vector是个特征，而不是问题。动态类型的
灵活性也是个祸根。

(如果你需要把整数和字符串放进同样的vector)，那么Rust的`enum`
是安全实现这个功能的方式。）

有时你需要至少给出类型 _线索_。`collect`是个令人惊叹的迭代器
方法，但它需要线索。比如我有个迭代器返回了`char`。那么`collect`
可以用两种方式转动:

```rust
// a vector of char ['h','e','l','l','o']
let v: Vec<_> = "hello".chars().collect();
// a string "doy"
let m: String = "dolly".chars().filter(|&c| c != 'l').collect();
```
当感觉对变量类型不确定时，也总有小方法，能强制`rustc`在错误消息
中揭示实际类型名称:

```rust
let x: () = var;
```

`rustc`可以挑选一个超类型。这里我们想将不同的类型引用作为`&Debug`
放到vector里但需要显式声明它。

```rust
use std::fmt::Debug;

let answer = 42;
let message = "hello";
let float = 2.7212;

let display: Vec<&Debug> = vec![&message, &answer, &float];

for d in display {
    println!("got {:?}", d);
}
```

## Mutable References 可变引用

规则是: 同一时刻只有一个可变引用。原因是当它可能在 _任何地方_
发生以致跟踪可变性是很难的。不像在小程序里那么明显，在大型
代码库里事情会变得更糟。

进一步的限制是在有了可变引用时不能获得不可变引用。否则，任何
持有那些引用的人没法保证值不变。C++也有不可变引用(如`const string&`)
但不能给你这种保证没有人不能持有`string&`引用并在背后修改它。

这会是个挑战如果你已经习惯了别的语言里每处引用都是可变的！不安全，
'松弛的'语言依赖人们理解他们的程序并高尚的决定不做坏事。但大型程序
被多个人编写且理解所有细节超越了个体的能力。

_令人生气的_ 是借用检查器不是那么聪明。

```rust
let mut m = HashMap::new();
m.insert("one", 1);
m.insert("two", 2);

if let Some(r) = m.get_mut("one") { // <-- mutable borrow of m
    *r = 10;
} else {
    m.insert("one", 1); // can't borrow mutably again!
}
```
很清楚这并不真违反了Rust规则因为如果我们得到了`None`那么
实际没有从map借用任何对象。

这是丑的处理方法:

```rust
let mut found = false;
if let Some(r) = m.get_mut("one") {
    *r = 10;
    found = true;
}
if ! found {
    m.insert("one", 1);
}
```

这令人讨厌，但它能工作因为吗发的借用被保持到第一个if语句结束。

更好的方法是使用 `HashMap`的 [entry API](https://doc.rust-lang.org/std/collections/hash_map/enum.Entry.html).

```rust
use std::collections::hash_map::Entry;

match m.entry("one") {
    Entry::Occupied(e) => {
        *e.into_mut() = 10;
    },
    Entry::Vacant(e) => {
        e.insert(1);
    }
};
```
借用检查器在 _非词法生命周期_ 在本年某些时候就绪后将少些沮丧。

借用检查器确实理解一些重要的情况，尽管如此。如果你有个结构体，
成员能被独立借用。组合是你的朋友；大结构体可以包含小结构体，
小的有自己的方法。定义所有可变方法在大结构体上将导致某些时候
你没法修改任何东西，甚至这些方法也许只引用了一个成员。

对可变数据，有特殊方法来单独对待每部分。例如，你有个可变切片，
那么 `split_at_mut`将切出两个可变切片。这完全安全，因Rust知道
两个切片不重叠。

## References and Lifetimes 引用和生命周期

Rust不允许一个引用超过其值的情况出现。否则我们将有个'悬垂引用'指向了
无效值 —— 段错误不可避免。
Rust cannot allow a situation where a reference outlives the value. Otherwise
we would have a 'dangling reference' where it refers to a dead value -
a segfault is inevitable.

`rustc`能常做关于函数内声明周期的有意义假设:

```rust
fn pair(s: &str, ch: char) -> (&str, &str) {
    if let Some(idx) = s.find(ch) {
        (&s[0..idx], &s[idx+1..])
    } else {
        (s, "")
    }
}
fn main() {
    let p = pair("hello:dolly", ':');
    println!("{:?}", p);
}
// ("hello", "dolly")
```

这很安全因为我们对付着分节符没出现的情况。`rustc`这里假设元组里的两个字符串
都借用与传入函数的参数字符串。

这函数显式的定义看起来这样:

```rust
fn pair<'a>(s: &'a str, ch: char) -> (&'a str, &'a str) {...}
```
这里符号的意思是输出字符串生命周期 _最长如_ 输入字符串。它的意思不是
生命周期相同，我们可以在任何时间废弃它们，那时不能超过`s`。
What the notation says is that the output strings live _at most as long_ as the
input string. It's not saying that the lifetimes are the same, we could drop them
at any time, just that they cannot outlive `s`.

所以，`rustc`以 _声明周期省略_ 美化常见情况。

现在，如果函数接收了2个字符串，那你可能需要显式的生命周期标注以告诉
Rust哪些输出字符串是借用于哪些输入字符串。

当结构体借用引用时你总是需要一个显式的生命周期:

```rust
struct Container<'a> {
    s: &'a str
}
```
标注再次约束结构体的生命周期不能超过引用。对结构体和函数来说，
声明周期需要被声明在`<>`像类型参数一样。

闭包是非常方便和强大的特征 —— Rust迭代器的许多能力来自闭包。但
如过你保存它们，你需要指明一个生命周期。这是因为基本上一个闭包
是个生成的能调用的结构体并不可变引用指向`m`和`c`。

```rust
let m = 2.0;
let c = 0.5;

let linear = |x| m*x + c;
let sc = |x| m*x.cos()
...
```

 `linear` 和 `sc`实现了 `Fn(x: f64)->f64` 但它们是不同物种 —— 它们有不同的
 类型和大小! 所以为保存它们你需要创建一个 `Box<Fn(x: f64)->f64 + 'a>`。

如果你习惯了Javascript或Lua里顺利的闭包，但C++与Rust做了类似的
工具`std::function`以保存不同的闭包，对虚拟调用有点惩罚。


## Strings 字符串

一开始对Rust字符串的烦恼是共同的。有不同的方式来创建它们，都是明显的:

```rust
let s1 = "hello".to_string();
let s2 = String::from("dolly");
```
难道"hello"本不是个字符串？好吧，这么说。`String`是自己拥有的字符串，
内存分配在堆上；而字符串常量"hello"是`&str`类型(即"string slice")且
也行被包在可执行文件里("static")或从`String`变量借用而来。系统语言需要
这种区分 —— 考虑一个微控制器，它有很小的RAM也没有更多ROM。常量字符
串被存在ROM(是"read-only")中，它既可以更便宜又可以消耗更少能源。 

但也许你会说这在C++里很简单:

```C
std::string s = "hello";
```
这是更短，但暗含了隐式的字符串对象创建。Rust喜欢显式分配内存，因此
用`to_string`。换句话说，从C++ 字符串上借用需要`c_str`，C字符串很笨。

幸运的是，Rust里事情更好 —— _一旦_ 你接受了那`String`和`&str`都是必要的。
`String`的方法大多数都是修改字符串的，如`push`是添加一个字符(在那边它
很像Vec<u8>)。`&str`的全部方法也可用。像解引用的机制，一个 `String`能够
当作`&str`传递一个函数 —— 这是为什么你很少看到函数定义里有`&String`。

把`&str`转换成`String`的方法很多，对应在各种各样的trait上。Rust需要这些
trait能与泛型工作。经验法则是，任何实现了`Display`的类型也有`to_string`方法，
像 `42.to_string()`。

一些操作符直觉性的不能用:

```rust
    let s1 = "hello".to_string();
    let s2 = s1.clone();
    assert!(s1 == s2);  // cool
    assert!(s1 == "hello"); // fine
    assert!(s1 == &s2); // WTF?
```

记住， `String` 和 `&String` 是不同的类型，`==` 不是为那组合定义的。
这也许会让习惯了引用与值可交换的C++开发者迷惑。更进一步，`&s2`
不会神奇的是`&str`，它是 _强制解引用_ 而只在赋给 `&str`变量或参数
时发生。(如同显式的调用`s2.as_str()` 。)

尽管如此，更真诚的值得一个WTF:
However, this more genuinely deserves a WTF:

```rust
let s3 = s1 + s2;  // <--- no can do
```
你不能连接两个 `String` 值，但你可以把一个`&str`连接一个 `String`上。
所以多数人不用操作符`+`而用宏`format!`，它很方便单不高效。

一些字符操作可用但不实现不同。例如语言字符常有`split`方法来打破
一个字符串成字符串数组。Rust字符串的这个方法返回一个 _迭代器_，
通过它你 _然后_ 你可以将子串收集到vector中。

```rust
let parts: Vec<_> = s.split(',').collect();
```
如果你着急获得一个vector这很笨拙。但你能对每部分做其他操作而
无须分配新vector！例如，找出长度最大的子串？

```rust
let max = s.split(',').map(|s| s.len()).max().unwrap();
```

(用 `unwrap` 是因为我们需要处理没有最啊长度的空迭代器情况。)

`collect`方法返回一个`Vec<&str>`，每个成员从原始字符串借用而来
—— 我们只需要为那些引用分配空间。在C++里没有类似方法，但直
到近来它不得不一个个的分配子串。(C++ 17有`std::string_view`行为
想Rust字符串切片。)

## 分号笔记

分号 _不是_ 可选的，但常放在与C语言里一样的位置，如在`{}`块后。
它们也不是需要在`enum`或`struct`后(这是C的特色。) 尽管如此，
如果块必须拥有一个 _值_，那么分号就放在那:

```rust
    let msg = if ok {"ok"} else {"error"};
```

注意在`let`语句后必须有个分号！
Note that there must be a semi-colon after this `let` statement!

如果分号在那些字符常量后那么返回值将是`()`(像`Nothing`或`void`)。
它是定义函数的常见错误:

```rust
fn sqr(x: f64) -> f64 {
    x * x;
}
```
这里`rustc`将给你清楚的错误信息。

## C++-specific Issues

### Rust value semantics are Different

在C++中，可以定义出新类型表现得像基础类型并拷贝自己。此外，
能定义move构造函数以指出一个值可以被move出临时上下文。

在Rust中，基础类型表现得如预期，但`Copy` trait只能被全不成员
是可拷贝的聚合类型(struct,tuple,enum)定义。任意类型可以有
`Clone`，但你需要调用该值的`clone`方法。Rust要求任何分配是
显式的且不能隐藏在copy构造器或赋值操作符中。

所以，拷贝和移动总是被定义就像只到处移动bits且不可覆盖。
So, copying and moving is always defined as just moving bits around and is
not overrideable.

如果`s1`不是可`Copy`的值类型，那么`s2 = s1;` 引起move发生，这 
_消耗_ 了`s1`! 所以你真正需要一个拷贝时，用`clone`。

借用常好于拷贝，但你必须在借用时遵守规则。幸运的是，借用 _是_
可覆盖的行为。例如，`String`能被借用为`&str`，共享`&str`的所有
不可变方法。_ 字符串切片_ 与C++的相似的'borrowing'操作比非常强大，
前者用`c_str`抽取一个`const char*`。 `&str`由一个指向自拥有的字节
(或字符串常量) 和一个 _大小_。这导致了一些非常有内存效率的模式。
你可以在所有字符串都被借用自一些底层字符串时得到`Vec<&str>`
—— 只有vector的空间需要被重分配:

例如，用空格切分字符串:

```rust
fn split_whitespace(s: &str) -> Vec<&str> {
    s.split_whitespace().collect()
}
```

同样的操作，调用 C++ `s.substr(0,2)` 将总是拷贝字符串，但Rust字符
常量只借用: `&s[0..2]`。

 `Vec<T>` 和 `&[T]` 是等价关系。

### Shared References 共享引用

Rust有像C++的 _智能指针_ —— 例如，等价于`std::unique_ptr` 的是 `Box`。
它不需要`delete`，因为任何内容和其他资源都将在box离开作用域时
被回收(Rust采纳了RAII)。

```rust
let mut answer = Box::new("hello".to_string());
*answer = "world".to_string();
answer.push('!');
println!("{} {}", answer, answer.len());
```
人们发现`to_string`首先令人烦恼，但是 _显式的_。

注意显式的接引用`*`，但智能指针上的方法不需要任何特殊符号
(不是 `(*answer).push('!')`)。

明显，借用只在原始内容有有清晰的owner时工作。在很多设计里
这不太可能。

在C++里，这是 `std::shared_ptr`的用途；拷贝只引起公共数据的
引用计数的被修改。这不是没代价的，尽管:

- 甚至如果数据是只读的，不断修改引用计数能引起缓存不一致
- `std::shared_ptr` 定义成是线程安全的且也带有锁开销

在Rust里，`std::rc::Rc`也表现得像采用引用计数的共享智能指针。
尽管如此，它只用于不可变引用！如果你需要一个线程安全的变量，
用`std::sync::Arc` (for 'Atomic Rc')。所以Rust这里提供2个变量有点
不方便，但你能为非多线程的操作避免锁开销。

那些必须是不可变引用因为这是Rust内存模型的基础。尽管如此，
也有跳出牌 `std::cell::RefCell`。如果你有个共享引用定义为
`Rc<RefCell<T>>`，那么你能通过它的`borrow_mut`方法进行可变的
借用。这 _动态的_ 应用了Rust的借用规则 —— 所以例如任何在借用
已经发生时尝试调用`borrow_mut` 将引起panic。

这仍是 _安全的_。因为Panic将发生在任何内存被非法触及 _之前_！
就像异常，它们回滚了调用栈。所以它对这样一个结构性的过程是个
不幸的词 —— 相比惊慌失措的撤退而言它是个有序的回收。

完整的`Rc<RefCell<T>>`是笨拙的，但应用程序比代码不高兴。这里
再次强调Rust喜欢显式的操作。

如果你想要线程安全的访问共享变量，那么`Arc<T>`唯一的 _安全_ 方法。
如果你需要可变的访问，那么`Arc<Mutex<T>>` 等价于 `Rc<RefCell<T>>`。
`Mutex` 与常规定义的动作有所区别: 它是个值的容器。你获取值的一个
_锁_ 然后能修改它。

```rust
let answer = Arc::new(Mutex::new(10));

// in another thread
..
{
  let mut answer_ref = answer.lock().unwrap();
  *answer_ref = 42;
}
```
为什么有`unwrap`? 如果前面的线程闪退了，那么这个`lock`失败了。
(这是文档中将`unwrap`看作合理事情的地方之一，因为某些事情已经
严重出错。Panics总是能在线程里发生。)

这很重要(总是带着互斥锁)以让排他锁被尽可能短的持有。所以
很普遍它们发生在有效的作用域 —— 那么锁在可变引用离开作用域
时失效。

对比C++中明显类似情景("用shared_ptr干")看起来尴尬。但现在任何对共享
状态的修改变得明显，且`Mutex`锁模式强迫线程安全性。

像其他，使用共享引用要 [小心](https://news.ycombinator.com/item?id=11698784)。.

### Iterators 迭代器

C++中迭代器被定义得完全不正式；他们引入智能指针，常开始于`c.begin()`
和结束于`c.end()`。迭代器上的操作被实现得像单独的模板函数，如`std::find_if`。

Rust迭代器被用 `Iterator` trait定义；`next`返回一个`Option`且当`Option`是`None`
时结束。

最常用的操作现在成了自己的方法。这里是 `find_if`的等价功能。它返回一个`Option`
(在没找到时返回`None`) 并且这里`if let`语句对抽取非`None`的值很方便:

```rust
let arr = [10, 2, 30, 5];
if let Some(res) = arr.find(|x| x == 2) {
    // res is 2
}
```

### Unsafety and Linked Lists 不安全性和链接表

Rust标准库部分实现时用了`unsafe`不是秘密。这并没有让借用检查器这样
保守的方法失效。记得"unsafe"有个特殊意思 —— Rust没法在编译期充分验证。
从Rust的角度，C++总是运行在unsafe模式！所以如果一个大型应用需要少量
行不安全代码，这没问题，因为少量几行能被人仔细检查。人类不擅长检查
100Kloc+的代码。

我提到，因为那里出现了模式:
一个有经验的C++程序员尝试实现一个链接表或树结构，会很沮丧。
好，一个双链接列表可能 _是_ 在Rust上安全。通过`Rc`引用向前移，
且`Weak`引用向后移。但标准库的实现比其他如用指针性能更好。
