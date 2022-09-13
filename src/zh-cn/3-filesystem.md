# 文件系统和进程

## 另一个角度看看读取文件 

在Part 1的结尾，我展示了如何读取整个文件内容到一个字符串。自然这并
不总是一个好的办法，所以这类展示如何一行一行的读文件。

`fs::File`实现了`io::Read`，它是任何可读类的trait。它定义了`read`方法能
用字节填充一个`u8`切片 —— 这是那个trait唯一 _要求_ 的方法，你会得到一些
_现成_ 的方法，就像`Iterator`一样。你能通过`read_to_end`用可读对象的字节
来填充一个字节vector，且通过`read_to_string`来填充一个字符串 —— 必须是
UTF-8编码的。

这是个'raw`读操作，没有缓冲。为了可缓冲的读可以用`io::BufRead` trait它能给
我们`read_line`和一个`lines`迭代器。`io::BufReader`为 _任何_ 可读对象实现了`io::BufRead`。

`fs::File` _也_ 实现了 `io::Write`.

最简单的方法是用`use std::io::prelude::*`来确认那些traits是可见的。

```rust
use std::fs::File;
use std::io;
use std::io::prelude::*;

fn read_all_lines(filename: &str) -> io::Result<()> {
    let file = File::open(&filename)?;

    let reader = io::BufReader::new(file);

    for line in reader.lines() {
        let line = line?;
        println!("{}", line);
    }
    Ok(())
}
```

语句 `let line = line?` 看起来有点奇怪。迭代器的返回值`line` 实际上是个 `io::Result<String>` 
它用操作符 `?`来解构值。因为迭代中 _可能_ 会遇到错误 —— 如I/O错误，读到非UTF-8的块
等等。

`lines` 是一个迭代器，在用`collect`读一个文件字符串到vector时它很直观，或者用`enumerate`
迭代器带这行号打印出每一个行内容。

读取所有的行并不是高效的方式，因为读每行时都分配了新字符串。使用`read_line`是更高效
的方式，更不方便。注意返回行包含了换行符，能用`trim_right`来去除。

```rust
    let mut reader = io::BufReader::new(file);
    let mut buf = String::new();
    while reader.read_line(&mut buf)? > 0 {
        {
            let line = buf.trim_right();
            println!("{}", line);
        }
        buf.clear();
    }
```
这就完全很少分配内存，因为 _清理_ 缓冲字符串而不是释放它的内存；一旦字符串有足够
容量，不会发生新的分配。

这也是我们用block来控制借用范围的例子。`line`是从`buf`里借用的，借用过程必须在我们
修改`buf`之前结束。再次强调，Rust在尝试阻止我们做些蠢事，比如在清理缓冲 _之后_
访问`line`。(借用检查器有时候会有限制。Rust负责得到'非词法的生命周期'，那么它将分析
代码看看`line`并没在`buf.clear()`之后使用。)

这并不特别完美。我不能给你一个合适的迭代器来返回一个缓冲的引用，但我可以给你一些
_看起来像_ 迭代器的东西。

第一次定义泛型结构体；参数`R`的类型是'任何实现了Read trait的类型'。它包含了reader和
buffer能让我们来借用。
First define a generic struct;
the type parameter `R` is 'any type that implements Read'. It contains the reader
and the buffer which we are going to borrow from.

```rust
// file5.rs
use std::fs::File;
use std::io;
use std::io::prelude::*;

struct Lines<R> {
    reader: io::BufReader<R>,
    buf: String
}

impl <R: Read> Lines<R> {
    fn new(r: R) -> Lines<R> {
        Lines{reader: io::BufReader::new(r), buf: String::new()}
    }
    ...
}
```
然后实现`next`方法。它返回一个`Option` —— 仅像一个迭代器，当返回`None`
时迭代器结束。因为`read_line`也许会失败所以返回类型是`Result`，我们
_从不扔出错误_。所以当它失败时，我们包装它的错误到`Some<Result>`中。
另外，它也许会读到0字节，这代表文件的自然结束 —— 不是个错误，只是`None`。

在这点，缓冲里添加有带着换行符('\n')的内容行。修剪掉它，然后打包字符切片返回。

```rust
    fn next<'a>(&'a mut self) -> Option<io::Result<&'a str>>{
        self.buf.clear();
        match self.reader.read_line(&mut self.buf) {
            Ok(nbytes) => if nbytes == 0 {
                None // no more lines!
            } else {
                let line = self.buf.trim_right();
                Some(Ok(line))
            },
            Err(e) => Some(Err(e))
        }
    }
```
现在，注意生命周期是怎么发挥作用的。我们需要一个显式的生命周期，因为Rust从
不允许我们在不知道生命周期的情况下传出字符切片。那么这里我们说这个借用的字符串
的生命周期不会长于`self`对象。

用这个签名，表示该生命周期，与`Iterator`的接口不兼容。但如果兼容的话很容易看出问题；
考虑`collect`尝试从那些字符串切片里生成一个vector。这不可能，因为它们都是
从同一个可修改的字符串中产生！(如果你读了 _全部_ 文件到一个字符串，那么字符串的
`lines`迭代器能返回字符串切片因为它们都是从原始字符串的 _不同部分_ 借用的。)

这会导致处理结果的循环特别容易，文件缓冲对用户不可见。

```rust
fn read_all_lines(filename: &str) -> io::Result<()> {
    let file = File::open(&filename)?;

    let mut lines = Lines::new(file);
    while let Some(line) = lines.next() {
        let line = line?;
        println!("{}", line);
    }

    Ok(())
}
```

你甚至可以这么写loop，因为显式匹配能取出字符切片:

```rust
    while let Some(Ok(line)) = lines.next() {
        println!("{}", line)?;
    }
```

虽然诱人，但这里你正抛出一个可能的错误；这个循环将静默的在遇到错误时结束。
特别是，它将在Rust第一次不能读来的行转换成UTF-8时结束。在不正式的代码里
这么做没问题，但在生产代码里这很糟！
It's tempting, but you are throwing away a possible error here; this loop will
silently stop whenever an error occurs. In particular, it will stop at the first place
where Rust can't convert a line to UTF-8.  Fine for casual code, bad for production code!

## 写入文件

我们在实现`Debug`时遇到过`write!`宏 —— 它对任何实现了`Write`的类也适用。所以
这里有另一个实现`println!`的方法:

```rust
    let mut stdout = io::stdout();
    ...
    write!(stdout,"answer is {}\n", 42).expect("write failed");
```

如果一个error是 _可能的_，你必须处理它。它也许 _不太可能_ 但也还是有可能发生。
这很正常，因为你在处理文件I/O你应该在操作符`?`能用的上下文中。

但这里有个区别：`println!` 在每次写出时锁住stdout。这常正是你希望的输出，
因为若不加锁多线程程序将会产生混合输出。但如果你要一次输出很多文本，
那么`write!`将更快。

对我们需要`write!`的任意文件，在`write_out`调用结束`out`被废弃时文件也会关闭，
这既方便又重要。

```rust
// file6.rs
use std::fs::File;
use std::io;
use std::io::prelude::*;

fn write_out(f: &str) -> io::Result<()> {
    let mut out = File::create(f)?;
    write!(out,"answer is {}\n", 42)?;
    Ok(())
}

fn main() {
  write_out("test.txt").expect("write failed");
}
```
如果你在意性能，你需要意识到Rust文件默认是不缓冲的。所以每写一行都直接
传给操作系统，这将非常慢。我提这个是因为这个默认方式与其他语言是不同的，
会导致震惊误解认为Rust远比不上脚本语言！
通过`Read`和`io::BufReader`，及用`io::BufWriter`来有缓冲的`Write`。

## 文件，路径和目录 

这里有个打印本机Cargo目录的小型程序。最简单的目录例子是'~/.cargo'。这是
Unix shell的展开，因为是跨平台的我们用`env::home_dir`。(也许访问失败，
但一台电脑若没有home目录就没法保存Rust的工具链。)

我们可以创建一个 [PathBuf](https://doc.rust-lang.org/std/path/struct.PathBuf.html)
并用其`push`方法和 _组件_ 来构建完整的文件路径。(这容易混淆在取决于系统的'/'和'\\'上。)

```rust
// file7.rs
use std::env;
use std::path::PathBuf;

fn main() {
    let home = env::home_dir().expect("no home!");
    let mut path = PathBuf::new();
    path.push(home);
    path.push(".cargo");

    if path.is_dir() {
        println!("{}", path.display());
    }
}
```
`Pathbuf`就像`String` —— 它有个可增长的字符集和，且有特定的方法来构建paths。
它大多数方法来自于借用的`Path`，就像`&str`，例如，`is_dir`就是`Path`的方法。

这也许听起来是可疑的就像某种形式的遗产，但这里的魔术[Deref](https://doc.rust-lang.org/book/deref-coercions.html)
trait 功能不同。它就像与`String/&str`一起发挥作用 —— `PathBuf` 的引用被 _强制_ 转换成`Path`的引用。
('强制'是个强大的词汇，但这确实是Rust帮你做转换的少数地方。)

```rust
fn foo(p: &Path) {...}
...
let path = PathBuf::from(home);
foo(&path);
```
`PathBuf` 与 `OsString` 有亲密的关系，它代表了我们能从系统直接得到的字符串。
(与`OsString/&OsStr`有对应关系。)

这些字符串并不 _确保_ 就是UTF-8编码的！
真实情况是 [complicated matter](https://news.ycombinator.com/item?id=10519932)，
特别看看'Why are they so hard?‘的回答。总之，首先有好多年的ASCII编码历史，
及多种其他语言的特定编码。其次，人类语言很复杂。例如有些是 _5_ 个Unicode
编码！

真实情况是大多数现代操作系统的文件名会是Unicode(Unix上的UTF-8和Windows上的
UTF-16),其他不是。但Rust必须严格处理那些极少情况。例如`Path`有个`as_os_str`
方法能返回一个`&OsStr`，但`to_str`方法只返回`Option<&str>`。不总是这样！

人们在这点会遇到麻烦因为他们太常依赖'string'和'character'作为必要的抽象。爱因斯坦
会说，一个编程语言应该尽可能的简单，但不用更简化。一门系统语言 _需要_ `String/&str`
差别(拥有和借用的差别: 也很方便) 且如果它希望标准化Unicode字符串那么也需要处理
其他非Unicode的文本格式 —— 因此有 `OsString/&OsStr`。注意这里操作系统相关的字符串
类型并有string类型的接口，准确说我们并不知道它们的编码方式。

但人们习惯于把文件名当作字符串处理，这就是为什么Rust用`PathBuf`及对应方法
来简化处理文件名的原因。

你可以接着`pop`来删除部分路径。这里我们从当前目录开始:

```rust
// file8.rs
use std::env;

fn main() {
    let mut path = env::current_dir().expect("can't access current dir");
    loop {
        println!("{}", path.display());
        if ! path.pop() {
            break;
        }
    }
}
// /home/steve/rust/gentle-intro/code
// /home/steve/rust/gentle-intro
// /home/steve/rust
// /home/steve
// /home
// /
```

这是个有用的变化。我有个程序来搜索配置文件，规则就是它们可能出现在当前目录
和任何子目录。
如我创建配置在`/home/steve/rust/config.txt` 而程序启动在`/home/steve/rust/gentle-intro/code`:

```rust
// file9.rs
use std::env;

fn main() {
    let mut path = env::current_dir().expect("can't access current dir");
    loop {
        path.push("config.txt");
        if path.is_file() {
            println!("gotcha {}", path.display());
            break;
        } else {
            path.pop();
        }
        if ! path.pop() {
            break;
        }
    }
}
// gotcha /home/steve/rust/config.txt
```

这正是当我们想知道当前仓库是什么时 __git__ 如何工作的。

一个文件的细节(如大小，类型等)被称为 _元数据_。就像常碰见的文件访问错误 
—— 并不仅是'not found' 也常因无权限读文件报错。

```rust
// file10.rs
use std::env;
use std::path::Path;

fn main() {
    let file = env::args().skip(1).next().unwrap_or("file10.rs".to_string());
    let path = Path::new(&file);
    match path.metadata() {
        Ok(data) => {
            println!("type {:?}", data.file_type());
            println!("len {}", data.len());
            println!("perm {:?}", data.permissions());
            println!("modified {:?}", data.modified());
        },
        Err(e) => println!("error {:?}", e)
    }
}
// type FileType(FileType { mode: 33204 })
// len 488
// perm Permissions(FilePermissions { mode: 436 })
// modified Ok(SystemTime { tv_sec: 1483866529, tv_nsec: 600495644 })
```

文件的长度(字节)和修改时间太直接不介绍。(注意也许我们获取不到!) 文件类型
有 `is_dir`，`is_file` 和 `is_symlink`方法。

`permissions`是个有趣的方法。Rust挣扎着要跨平台，这是'最小公分母'的例子。
一般来说，你能查询的只是文件是否只读 —— 'permissions'概念在Unix上包括
对不同用户/组/其他的读/写/可执行权限。

但如果你对Windows不感兴趣，那么通过一个平台有关的trait来桥接最少可以给
我们提供权限模式细节。(像往常，trait只对什么是可见的起作用。) 那么获取这个
程序字节的可执行权限如下:

```rust
use std::os::unix::fs::PermissionsExt;
...
println!("perm {:o}",data.permissions().mode());
// perm 755
```
(注意用 '{:o}' 来打印出 _8 进制_)

(一个文件是否可执行在Windows上取决于后缀名。可执行的后缀名在`PATHEXT` 环境
变量里 —— '.exe', '.bat'等)

`std::fs` 包含了一组有用的文件函数工作，如拷贝或移动，创建软链接和创建目录。

为找到目录内容，`std::fs::read_dir` 提供了一个迭代器。这里有读取所有'.rs'后缀名
和大于1024字节文件的例子:

```rust
fn dump_dir(dir: &str) -> io::Result<()> {
    for entry in fs::read_dir(dir)? {
        let entry = entry?;
        let data = entry.metadata()?;
        let path = entry.path();
        if data.is_file() {
            if let Some(ex) = path.extension() {
                if ex == "rs" && data.len() > 1024 {
                    println!("{} length {}", path.display(),data.len());
                }
            }
        }
    }
    Ok(())
}
// ./enum4.rs length 2401
// ./struct7.rs length 1151
// ./sexpr.rs length 7483
// ./struct6.rs length 1359
// ./new-sexpr.rs length 7719
```

明显`read_dir`也许失败(常如'not found' 和 'no permission')，但得到每个entry也可能
失败(就像读一个有缓冲Reader时的`lines`迭代器)。此外，我们也许获取不到
某个entry的metadata。一个文件也许没后缀名，所以都需要检查。

为什么不用迭代器便利目录？在Unix上就是`opendir`系统调用的工作方式，
但在Windows上你不能在得到metadata前迭代目录内容。所以这是个合理的优雅的
妥协以让跨平台代码尽可能有效工作。

你在这里可能忘记感受'错误疲劳'。但请注意 _错误无处不在_ —— 这不是Rust在
发明新东西。它只是努力来尽可能的让你不处理那些错误。任何操作系统调用
可能会失败的。

在Java和Python等语言会抛出异常；Go和Lua语言会返回2个值，第一个是结果
而第二个是错误: 就像Rust让函数库返回错误一样不是好方式。所以在函数中有
很多错误检查和提前返回。

Rust使用`Result`是因为它是或者：你不能同时得到一个结果和错误。
用问号操作符来处理错误就很干净。

## Processes 进程

程序的基本目的是为了运行自己，或 _启动进程_。你的程序可以生成很多子进程，
就像名称表示的它们与父进程有特殊关系。

运行程序可以直接使用`Command` struct，它能构建参数传递给程序：

```rust
use std::process::Command;

fn main() {
    let status = Command::new("rustc")
        .arg("-V")
        .status()
        .expect("no rustc?");

    println!("cool {} code {}", status.success(), status.code().unwrap());
}
// rustc 1.15.0-nightly (8f02c429a 2016-12-15)
// cool true code 0
```
所以`new`接收了程序名(如果不是绝对地址它将在`PATH`里找程序)，`arg`添加
一个新参数，而`status`启动该程序。这会返回一个`Result`，如果程序运行起来
将是`Ok`，包含了一个`ExitStatus`。这种情况下程序运行成功并以code 0退出。
(用`unwrap`是因为我们不一定总是得到code如程序被用信号杀掉时)。

如果我们修改`-V`为`-v`(是简单错误)那么`rustc`会失败:

```
error: no input filename given

cool false code 101
```
所以有三种可能性:

  - 程序不存在，坏文件，或不允许执行
  - 程序运行，但不成功 —— 退出时为非0状态
  - 程序运行，以0状态结束。执行成功！

默认情况，程序的标准输出和标准错误流都在终端窗口。

我常对捕获输出内容感兴趣，所以可用用`output`方法。

```rust
// process2.rs
use std::process::Command;

fn main() {
    let output = Command::new("rustc")
        .arg("-V")
        .output()
        .expect("no rustc?");

    if output.status.success() {
        println!("ok!");
    }
    println!("len stdout {} stderr {}", output.stdout.len(), output.stderr.len());
}
// ok!
// len stdout 44 stderr 0
```
由于用了`status`方法我们的程序将阻塞到子进程结束，然后我们得到三个东西
—— 执行状态，标准输出内容，标准错误内容。

捕获的输出是简单 `Vec<u8>`字节。回想前面我们说从操作系统接收的字符串不
保证是UTF-8编码的。_甚至_ 我们没法确保它是字符串 —— 程序也许输出任意二进制
数据。

如果我们很确定输出是UTF-8，那么用`String::from_utf8`将那些vector或字节转换
成字符串 —— 它将返回`Result` 因为转换也许会失败。
一个更草率的函数是 `String::from_utf8_lossy` ，它将充分尝试转换并插入Unicode
标记来替换失败的字节。

这里有个有用的程序将用shell运行一个程序。这是用了常规shell机制将stderr转发
到stdout。Windows上的shell名称不同，但其他功能如预期。

```rust
fn shell(cmd: &str) -> (String,bool) {
    let cmd = format!("{} 2>&1",cmd);
    let shell = if cfg!(windows) {"cmd.exe"} else {"/bin/sh"};
    let flag = if cfg!(windows) {"/c"} else {"-c"};
    let output = Command::new(shell)
        .arg(flag)
        .arg(&cmd)
        .output()
        .expect("no shell?");
    (
        String::from_utf8_lossy(&output.stdout).trim_right().to_string(),
        output.status.success()
    )
}

fn shell_success(cmd: &str) -> Option<String> {
    let (output,success) = shell(cmd);
    if success {Some(output)} else {None}
}
```
我剪掉了从右边开始任何空格所以如果你调用`shell("which rustc")`将得到一个
不含任何结束符的路径。

你可以通过`Process`并用 `current_dir`方法或环境变量`env`来指定目录控制一个程序的启动。

截至目前，你的程序简单的等着子进程结束。如果你用了`swawn`方法那主程序将立即结束，
所以必须显式的等待子进程 —— 或在此期间做点别的事！下面的例子展示了如何将标准输出
和错误压缩在一起:

```rust
// process5.rs
use std::process::{Command,Stdio};

fn main() {
    let mut child = Command::new("rustc")
        .stdout(Stdio::null())
        .stderr(Stdio::null())
        .spawn()
        .expect("no rustc?");

    let res = child.wait();
    println!("res {:?}", res);
}
```
默认情况，子进程'继承'了父进程的标准输入输出。这里，我们重定向子进程的输出
到'nowhere'。它等价于在Unix shell里用`> /dev/null 2> /dev/null`。

现在，也可能在Rust里用shell(`sh` 或 `cmd`) 来实现那些功能。但上面的方式你可以
得到创建进行的完整编程控制。

例如，如果我们只用 `.stdout(Stdio::piped())` 那么进程输出被重定向到一个pipe管道。
那么`child.stdout` 就可以被直接读取(如实现`Read`)。同样的，你可以用`.stdout(Stdio::piped())`
方法就能够写到 `child.stdin`。

但如果我们用`wait_with_output`代替`wait`那么它返回一个 `Result<Output>`且子进程
输出就像之前一样被以`Vec<u8>` 格式捕获到那个`Output`的`stdout`字段。

`Child` struct 也提供了显式的 `kill` 方法。

