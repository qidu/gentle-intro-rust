# Error Handling 错误处理

## Basic Error Handling 基本错误处理

在Rust中处理错误时不用问号操作符是笨拙的。为达到这个便利，我们需要返回一个
可以接受任何错误的 `Result`。所有的错误都实现了 trait `std::error::Error`，所以 _任何_
错误都可以被转换成 `Box<Error>`。

我们需要 _同时_ 处理i/o错误和从字符串转换成整数时的错误:

```rust
// box-error.rs
use std::fs::File;
use std::io::prelude::*;
use std::error::Error;

fn run(file: &str) -> Result<i32,Box<Error>> {
    let mut file = File::open(file)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents.trim().parse()?)
}
```
所以这里有2个问号操作符用于i/o错误(打开文件失败，或读出字符串失败) 和转换错误。
最后，我们包装结果到`Ok`中。Rust可以从`parse`的返回类型搞清楚它返回的是`i32`。

为`Result`类型创建别名方式也很容易:

```rust
type BoxResult<T> = Result<T,Box<Error>>;
```
尽管如此，我们这程序将有程序特定的错误条件，所以我们需要创建自己的错误类型。
基本的要求很直接:

  - May implement `Debug`
  - Must implement `Display`
  - Must implement `Error`

其他情况，你的错误可以如下这样:

```rust
// error1.rs
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct MyError {
    details: String
}

impl MyError {
    fn new(msg: &str) -> MyError {
        MyError{details: msg.to_string()}
    }
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f,"{}",self.details)
    }
}

impl Error for MyError {
    fn description(&self) -> &str {
        &self.details
    }
}

// a test function that returns our error result
fn raises_my_error(yes: bool) -> Result<(),MyError> {
    if yes {
        Err(MyError::new("borked"))
    } else {
        Ok(())
    }
}
```

打出 `Result<T,MyError>` 是很冗长的且很多Rust模块定义了自己的`Result` —— 例如`io::Result<T>`
是 `Result<T,io::Error>`的别名。

在下个例子中我们需要在字符串不能解析为浮点数时处理这样特定错误。

现在那`?`操作符的工作方式是查找一个从 _表达式_ 错误到 _返回值_ 错误的转换方式。
那么这个转换就是用 `From` trait来表达。`Box<Error>`因为为所有 实现`Error`类型的实现了
`From` trait 而能如预期工作。

因此你可以继续方便的使用`BoxResult`别名并如之前得到的；即从我们的错误类型到`Box<Error>`
的转换。这对小型程序是个很好的选择。但我想展示能与自定义错误显式合作的其他错误。

`ParseFloatError`实现了`Error`所以定义了`description()`方法。

```rust
use std::num::ParseFloatError;

impl From<ParseFloatError> for MyError {
    fn from(err: ParseFloatError) -> Self {
        MyError::new(err.description())
    }
}

// and test!
fn parse_f64(s: &str, yes: bool) -> Result<f64,MyError> {
    raises_my_error(yes)?;
    let x: f64 = s.parse()?;
    Ok(x)
}
```

第一个`?`可以(一个类型实现过`From`总是转换成自己)，而第二个`?`将`ParseFloatError`
转换成`MyError`。

结果如下:

```rust
fn main() {
    println!(" {:?}", parse_f64("42",false));
    println!(" {:?}", parse_f64("42",true));
    println!(" {:?}", parse_f64("?42",false));
}
//  Ok(42)
//  Err(MyError { details: "borked" })
//  Err(MyError { details: "invalid float literal" })
```

不太复杂，稍微有点绕。最让人烦的是需要为所有其他错误类型写 `From` 以使其能与 `MyError`
合用 —— 否则你简单依赖 `Box<Error>`。新人会对Rust中用不同方法实现同样功能迷惑；总是
有其他方式来到罗马。灵活性的代价是提供很多选项。对200行的程序来说能承担的错误处理
要比更大的程序简单。如果你曾想如Cargo crate一样打包你珍贵的模块，错误处理将是关键的。

目前，问号操作符只能用于 `Result`，而非`Option`，这是个功能选择而不是限制。 `Option` 有
 `ok_or_else`方法能转换自己成 `Result`。例如，我们有`HashMap`时必须在key不存在时get失败:

```rust
    let val = map.get("my_key").ok_or_else(|| MyError::new("my_key not defined"))?;
```
现在这里错误返回很清晰了！(这个形式使用了闭包，所以错误值只会在查找失败时创建。)

## simple-error 用于简单错误

[simple-error](https://docs.rs/simple-error/0.1.9/simple_error/) crate 为你提供了基于字符串的基本错误类型，
如同我们这里定义的，和一些方便的宏。如任何错误一样，它与`Box<Error>`一起用起来没问题:

```rust
#[macro_use]
extern crate simple_error;

use std::error::Error;

type BoxResult<T> = Result<T,Box<Error>>;

fn run(s: &str) -> BoxResult<i32> {
    if s.len() == 0 {
        bail!("empty string");
    }
    Ok(s.trim().parse()?)
}

fn main() {
    println!("{:?}", run("23"));
    println!("{:?}", run("2x"));
    println!("{:?}", run(""));
}
// Ok(23)
// Err(ParseIntError { kind: InvalidDigit })
// Err(StringError("empty string"))

```

`bail!(s)` 展开成  `return SimpleError::new(s).into();` —— 与 _到_ 收到类型的转换提前返回。

你需要用 `BoxResult` 把`SimpleError`与其他错误类型混合用，因为我们没法为它实现`From`，
因为trait和类型都来自其他crates。

## error-chain 用于严重错误

对大点的程序我们看看[error_chain](http://brson.github.io/2016/11/30/starting-with-error-chain) crate。
一个小小的宏魔法可以在Rust上经历很多。

用`cargo new --bin test-error-chain`创建一个二进制crate来改变目录。编辑 `Cargo.toml`加入 `error-chain="0.8.1"`。

__error-chain__ 为你所做的是创建所有需要手动实现错误类型的定义；创建一个结构体，实现必要的
traits: `Display`, `Debug` 和 `Error`。它也默认实现了`From` 所以字符串能转化成错误。

我们第一个`src/main.rs`文件看起来像这样。主程序所做的事就是调用 `run`，打印出任何错误，
并用非0退出程序。宏`error_chain`在`error` 模块内生成所有我们需要的类型 —— 在一个更大的程序
中你可以把这个放在单独的文件中。我们需要把 `error` 所有的东西带回全局范围因为需要在
生成的公共traits中可见。默认会有 `Error` struct和一个对应错误定义的`Result`。

这里我们要求实现`From`以让`std::io::Error` 用 `foreign_links`转换成我们的错误类型:

```rust
#[macro_use]
extern crate error_chain;

mod errors {
    error_chain!{
        foreign_links {
            Io(::std::io::Error);
        }
    }
}
use errors::*;

fn run() -> Result<()> {
    use std::fs::File;

    File::open("file")?;

    Ok(())
}


fn main() {
    if let Err(e) = run() {
        println!("error: {}", e);

        std::process::exit(1);
    }
}
// error: No such file or directory (os error 2)
```

'foreign_links' 让我们的工作简单点，因问号操作符知道如何转换 `std::io::Error`到 `error::Error`。
(在下面这个宏创建了一个`From<std::io::Error>`转换，如拼写的一样简单。)

所有的行为都在`run`里进行；我们让它打印出参数给定文件的头10行。有或没有传这个参数，
就不算是个错误。这里我们想把 `Option<String>` 转换成 `Result<String>`。有2个 `Option`
方法来进行转换，我选择最简单的那个。我们的`Error`类型为`&str`实现了`From`，所以很直观
的用简单文本来报个错。

```rust
fn run() -> Result<()> {
    use std::env::args;
    use std::fs::File;
    use std::io::BufReader;
    use std::io::prelude::*;

    let file = args().skip(1).next()
        .ok_or(Error::from("provide a file"))?;

    let f = File::open(&file)?;
    let mut l = 0;
    for line in BufReader::new(f).lines() {
        let line = line?;
        println!("{}", line);
        l += 1;
        if l == 10 {
            break;
        }
    }

    Ok(())
}
```
这里再有一个有用的宏`bail!`来'扔出'错误。这是`ok_or`方法的替代:

```rust
    let file = match args().skip(1).next() {
        Some(s) => s,
        None => bail!("provide a file")
    };
```

像`?`一样能 _早点返回_。

返回的错误包含一个枚举`ErrorKind`，它能让我们区分各类错误。总有变量`Msg`(当你调用 `Error::from(str)`)
和`foreign_links`声明的`Io`包装了I/O错误:

```rust
fn main() {
    if let Err(e) = run() {
        match e.kind() {
            &ErrorKind::Msg(ref s) => println!("msg {}",s),
            &ErrorKind::Io(ref s) => println!("io {}",s),
        }
        std::process::exit(1);
    }
}
// $ cargo run
// msg provide a file
// $ cargo run foo
// io No such file or directory (os error 2)
```

很直观添加新错误类型。加入`errors`节到`error_chain!`宏里:

```rust
    error_chain!{
        foreign_links {
            Io(::std::io::Error);
        }

        errors {
            NoArgument(t: String) {
                display("no argument provided: '{}'", t)
            }
        }

    }
```

这个定义了`Display`如何为新错误类型工作。现在我们能处理'no argument'特定错误了，
把`ErrorKind::NoArgument`看作为一个`String`值。

```rust
    let file = args().skip(1).next()
        .ok_or(ErrorKind::NoArgument("filename needed".to_string()))?;

```
这里有另一个`ErrorKind`变量你需要匹配:

```rust
fn main() {
    if let Err(e) = run() {
        println!("error {}",e);
        match e.kind() {
            &ErrorKind::Msg(ref s) => println!("msg {}", s),
            &ErrorKind::Io(ref s) => println!("io {}", s),
            &ErrorKind::NoArgument(ref s) => println!("no argument {:?}", s),
        }
        std::process::exit(1);
    }
}
// cargo run
// error no argument provided: 'filename needed'
// no argument "filename needed"
```

一般来说，让错误尽可能特化，_特别_ 是当在库函数中时！这个match-on-kind技术是传统
异常处理的特别好的等价实现，传统方式下你会在 `catch` 和 `except`块中匹配异常类型。

总而言之，__error-chain__ 创建了一个`Error`类型，并定义 `Result<T>`为`std::result::Result<T,Error>`。
`Error`包含枚举`ErrorKind`且默认有个变量 `Msg`保存由字符串创建而来的错误。你用
`foreign_links` 定义的额外错误做两件事。首先，它生成一个新的`ErrorKind`变量；其次，
它在额外的错误上定义了 `From`以让它们被转换成我们的自定义错误。新错误变量能容易添加。
一大堆恼人的锅碗瓢盆式代码可以避免。

## Chaining Errors 链式错误

这个crate提供的最酷的功能是 _错误链_。

作为一个 _库用户_，当一个方法简单到只'扔出'通用的I/O错误。OK，它不能打开一个文件，
没问题，但是什么文件出错？基本上，这些信息对我有什么用？

`error_chain` 进行 _错误链式化_ 能帮助解决这类超通用错误。当我们尝试打开文件，我们能
简单的依靠用`?`实现到`io::Error` 的转换，或 _链接_ 这个错误。

```rust
// non-specific error
let f = File::open(&file)?;

// a specific chained error
let f = File::open(&file).chain_err(|| "unable to read the damn file")?;
```

这里有个新版本，_不带_ 导入的'foreign'错误，只用默认:

```rust
#[macro_use]
extern crate error_chain;

mod errors {
    error_chain!{
    }
}
use errors::*;

fn run() -> Result<()> {
    use std::env::args;
    use std::fs::File;
    use std::io::BufReader;
    use std::io::prelude::*;

    let file = args().skip(1).next()
        .ok_or(Error::from("filename needed"))?;

    ///////// chain explicitly! ///////////
    let f = File::open(&file).chain_err(|| "unable to read the damn file")?;

    let mut l = 0;
    for line in BufReader::new(f).lines() {
        let line = line.chain_err(|| "cannot read a line")?;
        println!("{}", line);
        l += 1;
        if l == 10 {
            break;
        }
    }

    Ok(())
}


fn main() {
    if let Err(e) = run() {
        println!("error {}", e);

        /////// look at the chain of errors... ///////
        for e in e.iter().skip(1) {
            println!("caused by: {}", e);
        }

        std::process::exit(1);
    }
}
// $ cargo run foo
// error unable to read the damn file
// caused by: No such file or directory (os error 2)
```

那么`chain_err`方法接收了原始错误，并创建一个新错误能包含原始错误 —— 这能不确定的持续。
这个闭包期望返回任何能被 _转换_ 到错误的值。

Rust的宏能清晰的拯救你于很多输入。`error-chain`甚至提供一个别名能替换整个mian程序:

```rust
quick_main!(run);
```

(`run` 是所有动作发生的地方)

