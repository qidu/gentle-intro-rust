# Standard Library Containers 标准库容器

## Reading the Documentation

这节我们简单的介绍下标准库的公共部分。文档很好但一点上下文和少量例子总是
有帮助的。

刚开始读Rust文档很有挑战，所以我将直接通过`Vec`作为一个例子。一个有用的提示
是点击'[-]'盒子来展开文档。(如果你用 `rustup component add rust-src` 下载了标准库代码
后一个'[src]'链接将靠近这儿。) 这给你一个所有方法的鹰眼视图。

第一点要注意的是，_不是所有可能方法_ 被定义在`Vec`自身上。他们大多数是能修改vector的
可变方法，如`push`。有些方法只是实现了让vector的成员类型匹配一些约束。例如，你能只调用
`dedup`(去重)如果成员类型确实是一些能够相互比较的。那里有很多`impl`代码块定义了`Vec`的
不同成员类型约束。

`Vec<T>` 和 `&[T]`的关系很特殊。任何能在切片上工作的方法也直接能在vector上工作，不
显式的话需要用`as_slice`方法。关系被`Deref<Target=[T]>`描述。这也发生在当你传递一个
vector引用到一些期望切片的地方 —— 这是少量类型转换自动发生的地方之一。所以切片的
方法如`first`，也许会返回一个引用到第一个元素，或`last`也能用于vector。很多方法与对应的
字符串方法很类似，所以有`split_at`来得到一对切片拆分了索引，`start_with`来检查vector是否
开始于序列值，而`contains`方法来检查vector是否包含特殊值。

没有`search`方法来找到特定值的索引，但经验法则是: 如果你在容器上找不到一个方法，
那么看看迭代器上有没有:

```rust
    let v = vec![10,20,30,40,50];
    assert_eq!(v.iter().position(|&i| i == 30).unwrap(), 2);
```

(用 `&` 是因为这是个 _引用_ 的迭代器 —— 反过来你可以用 `*i == 30` 来比较也行。)

同样的，这里vector上没有`map`方法，因为 `iter().map(...).collect()`将提供这个功能。
Rust不喜欢不必要的分配 —— 你常不需要那`map`的结果作为一个实际要分配内存的vector。

我建议你对所有迭代器的方法都熟悉起来，因为它们对写好的不依赖loop的Rust代码总是
至关重要。总是可以写小的程序来探索迭代器方法，而不是在更复杂程序的上下午中奋力
解决它们问题。

`Vec<T>` 和 `&[T]`的方法被共同的trait跟随: vector知道如何进行debug显示自己
(但只能在成员类型实现了`Debug `时)。同样的，它们能在成员类型能克隆时支持克隆。
它们实现了`Drop`，在vector结束时调用；释放了内存，和所有成员对象。

`Extent` trait 意思是从迭代器里得到的值能不依赖loop被添加到vector:

```rust
v.extend([60,70,80].iter());
let mut strings = vec!["hello".to_string(), "dolly".to_string()];
strings.extend(["you","are","fine"].iter().map(|s| s.to_string()));
```

也有`FromIterator` trait，它让vector从迭代器上构建自己。(迭代器的`collect`方法依靠这个。)

任何容器需要是可迭代的。回忆那里有[3种迭代器](2-structs-enums-lifetimes.html#the-three-kinds-of-iterators)

```rust
for x in v {...} // returns T, consumes v
for x in &v {...} // returns &T
for x in &mut v {...} // returns &mut T
```
`for`语句依赖 `IntoIterator` trait，确实有3个实现。
The `for` statement relies on the `IntoIterator` trait, and there's indeed three implementations.

也有索引功能，被`Index`(读vetor时)和`IndexMut`(修改vector时)。有很多可能性，因为那些
切片也索引，如`v[0..2]`，返回那些索引，像返回单个元素引用`v[0]`一样。

也有些实现了`From` trait。例如，`Vec::from("hello".to_string())`将返回一个底层字节vector 
`Vec<u8>`。现在 `String`上已经有 `into_bytes`方法，为什么有上面那个冗余方法？
看起来用不同方式实现了相同功能。但这是必要的因为显式的traits让泛型方法能实现。

Rust类型系统的有些限制让事情看起来笨拙。一个例子是 `PartialEq`怎么单独定义了数组大小
不超过32字节？(这将更好。) 它允许直接用限定大小的数组方便地比较vectors。 

这有[Hidden Gems](http://xion.io/post/code/rust-iter-patterns.html) 深藏在文档中。像
Karol Kuczmarski所说"诚实的说：没有人会翻看到那么远"。
如何处理迭代器上的错误？比如你在一些操作上map时也许会失败且返回`Result`，然后
收集结果：

```rust
fn main() {
    let nums = ["5","52","65"];
    let iter = nums.iter().map(|s| s.parse::<i32>());
    let converted: Vec<_> = iter.collect();
    println!("{:?}",converted);
}
//[Ok(5), Ok(52), Ok(65)]
```
有道理，但你不得不unwrap那些错误 —— 小心的！
但Rust已经知道如何做正确的事，如果你让vector被包含在一个`Result`中
—— 就是说，或返回一个vector或一个错误:

```rust
    let converted: Result<Vec<_>,_> = iter.collect();
//Ok([5, 52, 65])
```
如果遇到错误的转换怎么办？那么你将在第一次遇到错误时得到`Err`。这是个
`collect`极为灵活的好例子。(这里的符号令人恐惧 —— `Vec<_>`意思是"这是个
vector，帮我搞清楚实际类型而" `Result<Vec<_>,_>`更进一步要求Rust也搞清
楚error类型。)

所以这里文档里有很多细节。但它确实比C++里的`std::vector`要更清晰

> 对成员类型的要求依靠实际在容器上执行的操作。一般来说，要求元素类型
> 是完整类型以满足可擦除的需要，但很多成员函数推行了更严的要求。 

在C++中，你自己定。Rust的显式在开始让人气馁，当你学习读读约束你将准确
知道`Vec`需要什么特殊方法。

我建议你用`rustup component add rust-src`得到源码，因标准库有非常好的可读性，
且方法实现常没有方法声明那么恐怖。

## Maps

_Maps_  (有时被称为 _associative arrays_ 或 _dicts_) 让你用  _key_ 查找关联的值。
它不是一个花哨的概念，可以用元组数组实现:

```rust
    let entries = [("one","eins"),("two","zwei"),("three","drei")];

    if let Some(val) = entries.iter().find(|t| t.0 == "two") {
        assert_eq!(val.1,"zwei");
    }
```

这对小的map没问题且只要求对keys的相等性，它查找花费线性时间 —— 与map的
大小成比例。

`HashMap`在有更多key/value对需要被搜索时做得更好:

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
map.insert("one","eins");
map.insert("two","zwei");
map.insert("three","drei");

assert_eq! (map.contains_key("two"), true);
assert_eq! (map.get("two"), Some(&"zwei"));
```

这里用`&"zwei"`? 这是因为`get`返回的是值的 _引用_，而不是值本身。这里值
的类型是`&str`，所以我们get的结果是`&&str`。一般它 _必须_ 是个引用，因为我们
不能去 _move_ 一个值到它的类型外。

`get_mut`就像`get`但返回可能可修改的引用。这里我们有个从字符串到整数的
map，想为'two'更新值:

```rust
let mut map = HashMap::new();
map.insert("one",1);
map.insert("two",2);
map.insert("three",3);

println!("before {}", map.get("two").unwrap());

{
    let mut mref = map.get_mut("two").unwrap();
    *mref = 20;
}

println!("after {}", map.get("two").unwrap());
// before 2
// after 20
```
注意可写的引用发生在原数据块上 —— 否则，我们需要一个可写的借用持续到结束，
那时Rust不允许你从`map`再靠 `map.get("two")`借用 ；在作用域内已有可写引用时
它不能允许再有任何可读引用。(如果允许了，它就没法保证那些可读引用是有效的。)
所以方案是确保可变借用持续得很短。

它不是可能的最佳的API，但我们不能抛出任何错误。Python会用个异常摆脱，而C++
会生成一个默认值。(这很方便但有点鬼鬼祟祟的；容易忘记 `a_map["two"]`的代价
总是返回整数的结果是我们不能区分0和'not found'，_加上_ 创建了一个额外的元素!)

而且没人只调用`unwrap`，只是在例子中。尽管如此，大多数Rust code你看到组成
小的独立的例子！多数很可能有个匹配发生:

```rust
if let Some(v) = map.get("two") {
    let res = v + 1;
    assert_eq!(res, 3);
}
...
match map.get_mut("two") {
    Some(mref) => *mref = 20,
    None => panic!("_now_ we can panic!")
}
```
我们可以迭代所有key/value对，但不会是任何特殊顺序。

```rust
for (k,v) in map.iter() {
    println!("key {} value {}", k,v);
}
// key one value eins
// key three value drei
// key two value zwei
```

这里也有`keys`和`values`方法能返回分别遍历keys和values的迭代器，能让
创建值的vectors很容易。

## Example: Counting Words

处理text时一个有趣的事是统计词的频率。很直接的用`split_whitespace`将text
打破成词，但我们也真得考虑标点符号。词被定义为只由字符组成的。词之间
用小写字符形式比较。

在map上进行可变查找很直接，但也需要处理查找失败的尴尬例子。幸运的是
有个优雅的方式来更新map的值:

```rust
let mut map = HashMap::new();

for s in text.split(|c: char| ! c.is_alphabetic()) {
    let word = s.to_lowercase();
    let mut count = map.entry(word).or_insert(0);
    *count += 1;
}
```

如果某个词还没有被统计，那么我们创建一个值为0的新entry并 _插入_ 到map。
这正是C++ map的做法，除了这里做的显式并非悄悄的。

这小段完全是显式的类型，那是需要的`char`因为`split`用了奇怪的字符串
`Pattern` trait。我们能推导出key类型是`String`且值是`i32`。

通过Gutenberg项目的 [The Adventures of Sherlock Holmes](http://www.gutenberg.org/cache/epub/1661/pg1661.txt)
我们能仔细的测出这个。全部单词的数量是8071 (`map.len`)。

如果找到最常见的20个词？首先，将map转换成一个(key,value)元组的vector。
(这样消耗了map，因为我们用了`into_iter`.)

```rust
let mut entries: Vec<_> = map.into_iter().collect();
```
接着我们降序排序。 `sort_by`期望来自 `Ord` trait的`cmp` 方法的结果，它是
用整数类型实现的:

```rust
    entries.sort_by(|a,b| b.1.cmp(&a.1));
```

最好打印出头20个键值对:

```rust
    for e in entries.iter().take(20) {
        println!("{} {}", e.0, e.1);
    }
```

(好吧，你可以只循环`0..20`并索引vector —— 它没错，只是有点不习惯 —— 且大迭代
器潜在的代价更大。)

```
 38765
the 5810
and 3088
i 3038
to 2823
of 2778
a 2701
in 1823
that 1767
it 1749
you 1572
he 1486
was 1411
his 1159
is 1150
my 1007
have 929
with 877
as 863
had 830
```
一点惊喜 —— 那个空白词是什么？它是因为`split`在单个分割符上，所以任何符号或
额外的空格导致新切分。

## Sets

Sets是只保存keys不含values的maps。所以`insert`只取一个参数，你用`contains`
来测试set里是否有某个值。

如同其他容器，你可以从迭代器创建`HashSet`。这正是`collect`的功能，一旦你
能给它必要的类型线索。

```rust
// set1.rs
use std::collections::HashSet;

fn make_set(words: &str) -> HashSet<&str> {
    words.split_whitespace().collect()
}

fn main() {
    let fruit = make_set("apple orange pear orange");

    println!("{:?}", fruit);
}
// {"orange", "pear", "apple"}
```
注意(如预期) 重复插入同样的key没有效果，在set中值的顺序不重要。

没有常见操作算不上是sets:

```rust
let fruit = make_set("apple orange pear");
let colours = make_set("brown purple orange yellow");

for c in fruit.intersection(&colours) {
    println!("{:?}",c);
}
// "orange"
```
它们都能生成迭代器，你可以使用`collect`来生成sets。

这里是快捷方式，就如我们为vectors定义的一样:

```rust
use std::hash::Hash;

trait ToSet<T> {
    fn to_set(self) -> HashSet<T>;
}

impl <T,I> ToSet<T> for I
where T: Eq + Hash, I: Iterator<Item=T> {

    fn to_set(self) -> HashSet<T> {
       self.collect()
    }
}

...

let intersect = fruit.intersection(&colours).to_set();
```
伴随所有Rust泛型，你不再需要包含类型 —— 只对理解相等性 (`Eq`)和存在哈希函数 (`Hash`)
的类型重要。记住没有 _类型_ 叫作`Iterator`，所以`I`代表任何 _实现_ 了`Iterator`的类型。

这个技术在实现我们自己在基础库类型上的方法时会显得特别强大，但也还是有规则的。
我们只能在自己的traits上这么做。如果struct和trait来自相同的crate(特别是stdlib)那么
这种实现就不被允许。这样你就免于产生迷惑了。

在祝贺我们自己如此聪明前，方便的快捷方式，你也需要意识到其后果。如果这么写了
`make_set`，所以那些是自己类型的sets，那么 `intersect` 的实际类型会比较意外:

```rust
fn make_set(words: &str) -> HashSet<String> {
    words.split_whitespace().map(|s| s.to_string()).collect()
}
...
// intersect is HashSet<&String>!
let intersect = fruit.intersection(&colours).to_set();
```
这也不是例外，因为Rust将不突然开始进行我们自己拥有的字符串拷贝。 `intersect` 
包含了借用于`fruit`的字符串引用 `&String`。我保证后面当你开始修补生命周期时，
这会让你疑惑！一个更好的方案是使用迭代器的 `cloned` 方法来为交集创建自己的
字符串拷贝。

```rust
// intersect is HashSet<String> - much better
let intersect = fruit.intersection(&colours).cloned().to_set();
```
一个更健壮的 `to_set`定义是 `self.cloned().collect()`，建议你最好试试。

## Example:  交互命令处理

在程序中进行交互任务常是有用的。读取每行然后切分成单词；第一个词是作为命令
查找，后面的词被传为该命令的参数。

一个自然的实现是用一个map来保存命令名称和闭包映射。但我们如何保存闭包，因给出的
闭包都有不同的类型大小？用Box来把它们保存在堆上即可:

这是第一次尝试:

```rust
    let mut v = Vec::new();
    v.push(Box::new(|x| x * x));
    v.push(Box::new(|x| x / 2.0));

    for f in v.iter() {
        let res = f(1.0);
        println!("res {}", res);
    }
```

我们在第二个push处得到确定的错误:

```
  = note: expected type `[closure@closure4.rs:4:21: 4:28]`
  = note:    found type `[closure@closure4.rs:5:21: 5:28]`
note: no two closures, even if identical, have the same type
```

`rustc` 在太确凿时会推导出类型，所以有必要确保那vector是 _boxed trait type_
才行:

```rust
    let mut v: Vec<Box<Fn(f64)->f64>> = Vec::new();
```
我们现在使用相同的小技巧来保持那些boxed闭包到 `HashMap`中。我们仍需要小心生命周期，
因为闭包能从上下文中借用变量。

在首次创建为 `FnMut` 类型时是吸引人的 —— 这是因为它们可以修改任何被捕获的变量。但我们
将有多个命令，每个有自己的闭包，而你不能可变借用相同的变量。

所以闭包被传入一个 _可变引用_ 作为参数，加上一个字符串切片的切片(`&[&str]`) 作为命令参数。
它们将返回一些 `Result` —— 我们将先使用`String`错误。

`D`是个数据类型，它可以是任何有大小的类型。

```rust
type CliResult = Result<String,String>;

struct Cli<'a,D> {
    data: D,
    callbacks: HashMap<String, Box<Fn(&mut D,&[&str])->CliResult + 'a>>
}

impl <'a,D: Sized> Cli<'a,D> {
    fn new(data: D) -> Cli<'a,D> {
        Cli{data: data, callbacks: HashMap::new()}
    }

    fn cmd<F>(&mut self, name: &str, callback: F)
    where F: Fn(&mut D, &[&str])->CliResult + 'a {
        self.callbacks.insert(name.to_string(),Box::new(callback));
    }
```
给`cmd`传入一个名字和任何匹配类型签名的闭包，它是boxed的并放入map。`Fn`意味着
我们的闭包可借用它们的上下文变量但不能修改它。它是那些声明比实现吓人的泛型方法
之一！忘记显式的生命周期是这里的常见错误 —— Rust不让我们忘记那些闭包需要一个上下文
限制的生命周期！

现在可以读取和运行命令:

```rust
    fn process(&mut self,line: &str) -> CliResult {
        let parts: Vec<_> = line.split_whitespace().collect();
        if parts.len() == 0 {
            return Ok("".to_string());
        }
        match self.callbacks.get(parts[0]) {
            Some(callback) => callback(&mut self.data,&parts[1..]),
            None => Err("no such command".to_string())
        }
    }

    fn go(&mut self) {
        let mut buff = String::new();
        while io::stdin().read_line(&mut buff).expect("error") > 0 {
            {
                let line = buff.trim_left();
                let res = self.process(line);
                println!("{:?}", res);

            }
            buff.clear();
        }
    }

```
这个合理又直观 —— 切片行成单词保存在vector中，从map里查找第一个词对应的命令
闭包并带着保存的数据参数来调用它。一个空行被忽略并不认为是错误。

接着，定义一个帮助函数来让闭包容易返回正确和不正确的结果。有个衡量聪明程度的
_little_ 前进标记；它们是对任何可转换成`String`的类型都起作用的泛型函数。

```rust
fn ok<T: ToString>(s: T) -> CliResult {
    Ok(s.to_string())
}

fn err<T: ToString>(s: T) -> CliResult {
    Err(s.to_string())
}
```
所以最后的主程序。看看 `ok(answer)`如何工作 —— 因为整数知道如何转换成字符串。

```rust
use std::error::Error;

fn main() {
    println!("Welcome to the Interactive Prompt! ");

    struct Data {
        answer: i32
    }

    let mut cli = Cli::new(Data{answer: 42});

    cli.cmd("go",|data,args| {
        if args.len() == 0 { return err("need 1 argument"); }
        data.answer = match args[0].parse::<i32>() {
            Ok(n) => n,
            Err(e) => return err(e.description())
        };
        println!("got {:?}", args);
        ok(data.answer)
    });

    cli.cmd("show",|data,_| {
        ok(data.answer)
    });

    cli.go();
}
```
这里的错误处理是有点笨重，稍后我们将看到如何在这种情况下使用问号操作符。
基本上，错误类`std::num::ParseIntError`实现了trait `std::error::Error`，我们必须
带它到能使用`description`的范围 —— Rust不允许traits在不可见下运作。

执行如下:

```
Welcome to the Interactive Prompt!
go 32
got ["32"]
Ok("32")
show
Ok("32")
goop one two three
Err("no such command")
go 42 one two three
got ["42", "one", "two", "three"]
Ok("42")
go boo!
Err("invalid digit found in string")
```

这里是一些你可以尝试的明显改进。首先，如果我们给`cmd`提供三个参数并带上帮助行，
那么我们可以保存那些帮助行并实现`help`命令。其次，保留一些命令编辑和历史会很方便，
所以可以用Cargo中的 [rustyline](https://crates.io/crates/rustyline) crate。
