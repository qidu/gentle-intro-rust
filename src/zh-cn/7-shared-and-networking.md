# Threads, Networking and Sharing

## Changing the Unchangeable

如果你正体会着头大而想知道是否 _可能_ 绕过借用检查器的限制。

参考如下程序，它能正常编译和运行。

```rust
// cell.rs
use std::cell::Cell;

fn main() {
    let answer = Cell::new(42);

    assert_eq!(answer.get(), 42);

    answer.set(77);

    assert_eq!(answer.get(), 77);
}
```

结果变了 —— 而 _变量_ `answer` 还是不可变的！

这明显完全安全，因为值在cell内只能通过`set`和`get`来访问。这通过气派的
_内部可变性_ 来实现。一般的称为 _继承可变性_ ：如果有结构体`v`，那么只能
在`v`是可写时能写到一个成员`v.a`。`Cell`放松了这个规则，因我们能通过`set`
修改它们包含的值甚至在cell自身不是可变的情况下。

尽管如此，`Cell`只能用于`Copy`类型上(例如，基本类型和用户自定义的实现了
`Copy` trait的类型)。

对其他类型，我们得获得一个引用然后就能这么用了，可变的或不可变的都行。
这正是 `RefCell` 提供的功能 —— 你显式的要求得到一个包含值的引用:

```rust
// refcell.rs
use std::cell::RefCell;

fn main() {
    let greeting = RefCell::new("hello".to_string());

    assert_eq!(*greeting.borrow(), "hello");
    assert_eq!(greeting.borrow().len(), 5);

    *greeting.borrow_mut() = "hola".to_string();

    assert_eq!(*greeting.borrow(), "hola");
}
```

再次说明，`greeting`没被声明为可变的!

在Rust中显式解引用操作符`*`会有点费解，因为你常不需要它 —— 例如
调用 `greeting.borrow().len()` 就没问题是因为方法调用时将隐式的解引用。
但你 _确实_ 需要`*`来从`greeting.borrow()`和`greeting.borrow_mut()`分别
提取出底层`&String`和 `&mut String`。

使用`RefCell` 并不总是安全，因为从那些方法返回的任何引用必须遵守一般
规则。

```rust
    let mut gr = greeting.borrow_mut(); // gr is a mutable borrow
    *gr = "hola".to_string();

    assert_eq!(*greeting.borrow(), "hola"); // <== we blow up here!
....
thread 'main' panicked at 'already mutably borrowed: BorrowError'
```

你在已可变借用后将不能再进行不可变借用！此外 —— 很重要 —— 违法规则发生在
_运行时_。解决方案(常用的) 是将可变借用的范围控制得越小越好 —— 这里你可以
放一个括号在前两行以让可变引用`gr`被及时废弃才能在后续再借用它。

所以，这不是个你可以无条件使用的功能，因为你将不会在编译时得到报错。
那些类型能提供 _动态借用_ 的条件是一般规则限制了功能实现。

## Shared References

截至目前，一个值和其借用引用之间的关系在编译时是清晰知道的。值是拥有者，引用
生命周期不能超出它。但很多情况下不能简单套用这个有序模式。例如，如果我们有
个`Player`结构体和`Role`结构体，`Player`持有一个`Role`引用的vector。那么在
那些值之间就没有清晰的1对1关系，想让`rustc`正常编译就变得不那么愉快。
Up to now, the relationship between a value and its borrowed references has been clear
and known at compile time.  The value is the owner, and the references cannot outlive it.
But many cases simply don't fit into this neat pattern. For example, say we have
a `Player` struct and a `Role` struct. A `Player` keeps a vector of references to `Role`
objects. There isn't a neat one-to-one relationship between these values, and persuading
`rustc` to cooperate becomes nasty.

`Rc`就像`Box`一样 —— 分配堆内存且值被移动过去。如果你克隆了`Box`，它会分配一个
值的完整克隆拷贝。而克隆一个`Rc`的代价更低，因为每次你克隆它只是简单的更新
_同一数据_ 的 _引用计数_。这是古老而流行的内存管理策略。例如，在iOS/MacOS的
OC运行时中使用了它(在现代C++中，通过 `std::shared_ptr`来实现了)。

当一个`Rc`被废弃，引用计数会减小。当计数变成0时对应的值将被废弃且内存被释放。

```rust
// rc1.rs
use std::rc::Rc;

fn main() {
    let s = "hello dolly".to_string();
    let rs1 = Rc::new(s); // s moves to heap; ref count 1
    let rs2 = rs1.clone(); // ref count 2

    println!("len {}, {}", rs1.len(), rs2.len());
} // both rs1 and rs2 drop, string dies.
```
你可以创建尽可能多到原始值的引用 —— 这是 _动态借用_ 。你不需奥小心跟踪
值`T`和它的引用`&T`之间的关系。这里有些运行时代价，所以这不是 _首选_ 方案，
但它提供了共享模式与借用检查器相冲突。注意，`Rc`给你不可变的共享引用，
否则会破坏一个非常基本的借用规则。一只豹子不能改变它的斑点除非它不是豹子。

在`Player`的例子中，它现在可以作为 `Vec<Rc<Role>>` 那么就可以了 —— 我们能
添加或删除roles而不在创建后改变它们。

尽管如此，如果每个`Player`需要保留到一组`Player`引用的vector作为 _team_ 
会发生什么？那么每个对象都是不可变的，因为所有`Player`值需要保存为`Rc`!
这正是`RefCell`变得必要的地方。这个team可以定义为 `Vec<Rc<RefCell<Player>>>`。
现在可以通过使用`borrow_mut`改变一个`Player`的值，确保没有谁同时提取出
了相应的player引用。例如，我们有个规则让特殊事情发生在player上时让整个team
变得强壮:

```rust
    for p in &self.team {
        p.borrow_mut().make_stronger();
    }
```
所以这段代码不算太差，但类型签名变得吓人。但你总是可以通过`type`别名来简化
它们:

```rust
type PlayerRef = Rc<RefCell<Player>>;
```

## Multithreading

在过去20多年里，有个从纯粹CPU处理速度到多核的转变。所以唯一的让现代电脑
变快的方式是让所有的核忙碌起来。确实可能生成一个后台子进程就像我们`Command`
一样但也有同步问题：我们不精确的知道什么时候那些子进程执行完。

当然也有其他因素例如需要分开 _线程执行_。例如你不能承担锁住整个进程来等待阻塞的
I/O。

Rust中生成线程很直观——你可以喂给`spawn`一个能在后台运行的闭包。

```rust
// thread1.rs
use std::thread;
use std::time;

fn main() {
    thread::spawn(|| println!("hello"));
    thread::spawn(|| println!("dolly"));

    println!("so fine");
    // wait a little bit
    thread::sleep(time::Duration::from_millis(100));
}
// so fine
// hello
// dolly
```

很明显只是'等待一阵'不是个很严谨的方案！最好调用线程返回对象的`join` ——
那么主线程等待到生成现场执行完。

```rust
// thread2.rs
use std::thread;

fn main() {
    let t = thread::spawn(|| {
        println!("hello");
    });
    println!("wait {:?}", t.join());
}
// hello
// wait Ok(())
```
这里有个有趣的变化：强制新线程panic。

```rust
    let t = thread::spawn(|| {
        println!("hello");
        panic!("I give up!");
    });
    println!("wait {:?}", t.join());
```
我们如预期得到一个panic，但只是panic的子线程退出！我们仍然可控的从`join`上打印出了错误信息。
所以，pannics并不总是致命，但线程相对代价高，所以这不应被视作处理panics的例行方案。

```
hello
thread '<unnamed>' panicked at 'I give up!', thread2.rs:7
note: Run with `RUST_BACKTRACE=1` for a backtrace.
wait Err(Any)
```

返回对象可以被用作保持对多个线程的跟踪:

```rust
// thread4.rs
use std::thread;

fn main() {
    let mut threads = Vec::new();

    for i in 0..5 {
        let t = thread::spawn(move || {
            println!("hello {}", i);
        });
        threads.push(t);
    }

    for t in threads {
        t.join().expect("thread failed");
    }
}
// hello 0
// hello 2
// hello 4
// hello 3
// hello 1

```
Rust坚持让我们处理join失败 —— 例如线程pannicked。(你一般在发生这种情况时将不会从
主程序跳出，而只是注意到这个错误，并重试等。)

线程的执行没有特别的顺序(这个程序每次运行时线程的执行顺序都不一样)，且这很关键 —— 
它们真是 _独立线程的执行_。生成多线程很简单；复杂的是 _并发_ —— 管理和同步多线程的执行。

## Threads Don't Borrow

线程的闭包可以抓住变量，但要通过 _moving_ 方式，而不是通过 _borrowing_！

```rust
// thread3.rs
use std::thread;

fn main() {
    let name = "dolly".to_string();
    let t = thread::spawn(|| {
        println!("hello {}", name);
    });
    println!("wait {:?}", t.join());
}
```

这里是有帮助的错误信息：

```
error[E0373]: closure may outlive the current function, but it borrows `name`, which is owned by the current function
 --> thread3.rs:6:27
  |
6 |     let t = thread::spawn(|| {
  |                           ^^ may outlive borrowed value `name`
7 |         println!("hello {}", name);
  |                             ---- `name` is borrowed here
  |
help: to force the closure to take ownership of `name` (and any other referenced variables), use the `move` keyword, as shown:
  |     let t = thread::spawn(move || {
```
这很公平！想象从一个函数中生成的现场 —— 函数调用后就退出那么`name`就被废弃了。所以
添加`move`解决这个问题。
That's fair enough! Imagine spawning this thread from a function - it will exist
after the function call has finished and `name` gets dropped.  So adding `move` solves our
problem.

但这是个 _move_，所以'name'可以只出现在一个线程中！这个例子中我想强调的 _是_ 共享
引用的可能性，但它们得是`static`的生命周期:

```rust
let name = "dolly";
let t1 = thread::spawn(move || {
    println!("hello {}", name);
});
let t2 = thread::spawn(move || {
    println!("goodbye {}", name);
});
```
`name`在整个程序(`static`)存活期间存在，所以`rustc`对闭包没有超出`name`的生命周期满意。
尽管这样，大多数我们使用的引用不会是`static`生命周期的！

线程间不能共享上下文环境 —— Rust _设计_ 。特别是，它们不能共享常规引用因为闭包会
move它们捕获的变量。

尽管如此 _共享引用_ 也是可以有的，因为它们的生命周期要 '尽可能的长' —— 但你不能使用
`Rc`让它变长。因为 `Rc`不是 _线程安全的_ —— 它被优化速度用于非线程场景。幸运的是，
使用`Rc`在这里会报编译错误；编译器总是监视着你。 

对线程间共享，你需要 `std::sync::Arc` —— 'Arc' 表示  'Atomic Reference Counting'。
也就是说，它保证引用计数只在一个原子逻辑操作下修改。为确保这个，它必须确认操作
是加锁的只有当前线程能访问。`clone` 代价也比实际拷贝要更低点。

```rust
// thread5.rs
use std::thread;
use std::sync::Arc;

struct MyString(String);

impl MyString {
    fn new(s: &str) -> MyString {
        MyString(s.to_string())
    }
}

fn main() {
    let mut threads = Vec::new();
    let name = Arc::new(MyString::new("dolly"));

    for i in 0..5 {
        let tname = name.clone();
        let t = thread::spawn(move || {
            println!("hello {} count {}", tname.0, i);
        });
        threads.push(t);
    }

    for t in threads {
        t.join().expect("thread failed");
    }
}
```
这里我故意创建了一个`String`的包装类型(新的)因为我们的`MyString`没有实现`Clone`
trait。但 _共享引用_ 可以被克隆！
I"ve deliberately created a wrapper type for `String` here (a 'newtype') since
our `MyString` does not implement `Clone`. But the _shared reference_ can be cloned!

共享引用`name`被通过`clone`生成新引用后传给每个新线程move到闭包内。这有点明显，
但这是安全的模式。安全在并发中是重要的因为由此产生的问题是不可预期的。程序或许
在你的机器上正常运行，但偶尔会在服务器上崩溃，还通常是周末。更糟的是，问题症状
不容易分析诊断。

## Channels 管道

在线程间发送数据的方法有多种。Rust使用 _管道_ 来实现这个。 `std::sync::mpsc::channel()` 
返回一个由 _receiver_ 管道和 _sender_ 管道构成的元组。每个线程被通过`clone`传入一个
sender的拷贝，成为`send`。同时主线程在receiver上调用`recv`。

'MPSC' 表示 'Multiple Producer Single Consumer'。我们创建多个线程它们尝试发送数据到
管道，而主线程'消费'管道数据。

```rust
// thread9.rs
use std::thread;
use std::sync::mpsc;

fn main() {
    let nthreads = 5;
    let (tx, rx) = mpsc::channel();

    for i in 0..nthreads {
        let tx = tx.clone();
        thread::spawn(move || {
            let response = format!("hello {}", i);
            tx.send(response).unwrap();
        });
    }

    for _ in 0..nthreads {
        println!("got {:?}", rx.recv());
    }
}
// got Ok("hello 0")
// got Ok("hello 1")
// got Ok("hello 3")
// got Ok("hello 4")
// got Ok("hello 2")
```

这里没有必要对线程join因为线程会在结束前发送响应，但明显线程的执行时间不是确定的。
`recv`会阻塞，并在sender管道关闭时将返回错。 `recv_timeout` 只会阻塞特定时间，并可
返回一个超时错误。

`send`从不阻塞，它很有用因为线程能发送出数据而无需等待receiver端处理。还有，管道
是由buffer的所以多个发送操作能够同时发送，消息会被顺序处理。

尽管如此，不阻塞意味着这是`Ok`的并不自动意味`成功发送了消息`！

`sync_channel` 在发送时阻塞。使用参数0，发送将阻塞到recv发生。线程必须相遇或 _会合_ 。

```rust
    let (tx, rx) = mpsc::sync_channel(0);

    let t1 = thread::spawn(move || {
        for i in 0..5 {
            tx.send(i).unwrap();
        }
    });

    for _ in 0..5 {
        let res = rx.recv().unwrap();
        println!("{}",res);
    }
    t1.join().unwrap();
```
在没有对应的`send`发生时我们可以容易地在调用`recv`得到一个错误，例如在循环 `for i in 0..4`时。
现场结束后`tx`废弃，那么`recv`将失败。这也将在线程panic引起栈回收废弃所有栈变量时发生。

如果 `sync_channel`创建时传了非0的参数`n`，那么它表现得像一个最大长度为`n`的对列 —— `send`
将只在队列消息达到`n`时被阻塞。

管道是强类型的 —— 这里的消息类型是 `i32` —— 但类型推导是隐式的。如果你需要传递不同类型的
消息，那么使用枚举enums是个好的表达方式。

## Synchronization 同步

我们看看 _同步_ 。`join`是很基础的，仅仅等待到一个特定的线程执行结束。一个`sync_channel` 
能同步两个线程 —— 在上个例子中，生成的线程和主线程完全被锁到一起。

Barrier同步是个检查点当线程必须等待到它们 _全部_ 都到达这个点。然后它们可以继续往前执行。
barrier与一组线程一起创建。像之前一样我们使用`Arc`来将barrier共享给所有线程。

```rust
// thread7.rs
use std::thread;
use std::sync::Arc;
use std::sync::Barrier;

fn main() {
    let nthreads = 5;
    let mut threads = Vec::new();
    let barrier = Arc::new(Barrier::new(nthreads));

    for i in 0..nthreads {
        let barrier = barrier.clone();
        let t = thread::spawn(move || {
            println!("before wait {}", i);
            barrier.wait();
            println!("after wait {}", i);
        });
        threads.push(t);
    }

    for t in threads {
        t.join().unwrap();
    }
}
// before wait 2
// before wait 0
// before wait 1
// before wait 3
// before wait 4
// after wait 4
// after wait 2
// after wait 3
// after wait 0
// after wait 1
```
线程半随机的执行，会合，然后继续。它像一种可继续的`join`且当你需要分配一片工作到
不同的线程且希望在所有分片工作后做些动作时会有用。

## Shared State 共享状态

线程如何 _修改_ 共享数据？
回想下 `Rc<RefCell<T>>`策略来 _动态地_ 在共享引用上进行可变借用。线程中的等价对象
是`Mutex` —— 你可以通过调用`lock`来获得可变引用。当这个引用存在时，没有其他线程能
访问它。`mutex`表示 '互斥' —— 我们锁住一段代码以只让一个线程能访问它，然后解锁。
你通过`lock`方法来加锁，在引用被废弃时解锁。 

```rust
// thread9.rs
use std::thread;
use std::sync::Arc;
use std::sync::Mutex;

fn main() {
    let answer = Arc::new(Mutex::new(42));

    let answer_ref = answer.clone();
    let t = thread::spawn(move || {
        let mut answer = answer_ref.lock().unwrap();
        *answer = 55;
    });

    t.join().unwrap();

    let ar = answer.lock().unwrap();
    assert_eq!(*ar, 55);

}
```
使用`RefCell`不是很直接因为要求锁在互斥锁上可能失败，如果另一个线程在持有锁时panic了。
(这种情况下，文档实际上推荐只要退出线程且调用`unwrap`因为情况变得严峻！)

保留这个可变借用尽可能短是很重要的，因为一旦获得互斥锁，其他线程也就被 _阻塞_ 住了。
这里不适宜进行昂贵的计算！所以典型代码像如下:

```rust
// ... do something in the thread
// get a locked reference and use it briefly!
{
    let mut data = data_ref.lock().unwrap();
    // modify data
}
//... continue with the thread
```
## Higher-Level Operations

最好找到 higher-level 的方式来进行多线程操作，比管理自己管理同步更好。一个例子是当你需要
并行执行且收集结果时。一个非常酷的crate是 [pipeliner](https://docs.rs/pipeliner/0.1.1/pipeliner/) 
它有个非常直观的API。这里是 'Hello, World!'  —— 一个迭代器喂我们输入而我们在这个值上并行的
执行到`n`个操作。

```rust
extern crate pipeliner;
use pipeliner::Pipeline;

fn main() {
    for result in (0..10).with_threads(4).map(|x| x + 1) {
        println!("result: {}", result);
    }
}
// result: 1
// result: 2
// result: 5
// result: 3
// result: 6
// result: 7
// result: 8
// result: 9
// result: 10
// result: 4
```
这是个傻傻的例子，因为操作太轻量计算，只是展示并行执行代码多简单。

下面是个更有用的例子。并行的进行网络操作很有用，因为它们要占用时间，且你不想等待它们 _全部_ 
完成才做其他工作。

这个例子非常粗糙(相信我，有更好的方式做这个) 但这里我们想聚焦于原理上。我们复用了在第4节定义
的`shell`函数来调用`ping`在一组IPv4地址上。

```rust
extern crate pipeliner;
use pipeliner::Pipeline;

use std::process::Command;

fn shell(cmd: &str) -> (String,bool) {
    let cmd = format!("{} 2>&1",cmd);
    let output = Command::new("/bin/sh")
        .arg("-c")
        .arg(&cmd)
        .output()
        .expect("no shell?");
    (
        String::from_utf8_lossy(&output.stdout).trim_right().to_string(),
        output.status.success()
    )
}

fn main() {
    let addresses: Vec<_> = (1..40).map(|n| format!("ping -c1 192.168.0.{}",n)).collect();
    let n = addresses.len();

    for result in addresses.with_threads(n).map(|s| shell(&s)) {
        if result.1 {
            println!("got: {}", result.0);
        }
    }

}
```

在我家的网络上看起来如下:

```
got: PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
64 bytes from 192.168.0.1: icmp_seq=1 ttl=64 time=43.2 ms

--- 192.168.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 43.284/43.284/43.284/0.000 ms
got: PING 192.168.0.18 (192.168.0.18) 56(84) bytes of data.
64 bytes from 192.168.0.18: icmp_seq=1 ttl=64 time=0.029 ms

--- 192.168.0.18 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.029/0.029/0.029/0.000 ms
got: PING 192.168.0.3 (192.168.0.3) 56(84) bytes of data.
64 bytes from 192.168.0.3: icmp_seq=1 ttl=64 time=110 ms

--- 192.168.0.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 110.008/110.008/110.008/0.000 ms
got: PING 192.168.0.5 (192.168.0.5) 56(84) bytes of data.
64 bytes from 192.168.0.5: icmp_seq=1 ttl=64 time=207 ms
...
```

活跃的地址会在半秒内快速返回，然后我们等待负面的结果。此外，我们将等待大约
半分钟！我们现在可以处理摩擦性事情如返回结果中的ping耗时。不过这只在Linux上可以。
`ping`是通用的，但实际的输出格式在不同平台上是不同的。为了做得更好我们需要使用
跨平台的Rust网络API，所以我们继续到网络部分。

## A Better Way to Resolve Addresses

如果你 _只_ 想得到可用性而不是ping的细节统计，`std::net::ToSocketAddrs` trait 将能为你进行
任何DNS解析:

```rust
use std::net::*;

fn main() {
    for res in "google.com:80".to_socket_addrs().expect("bad") {
        println!("got {:?}", res);
    }
}
// got V4(216.58.223.14:80)
// got V6([2c0f:fb50:4002:803::200e]:80)
```
这里返回是个迭代器因为通常有多个IP网卡绑定在同一个域名上 —— 这里Google有IPv６和IPv６的地址。

所以，我们天真的使用这个方法来重写前面的例子。大多数网络协议同时使用地址和端口:

```rust
extern crate pipeliner;
use pipeliner::Pipeline;

use std::net::*;

fn main() {
    let addresses: Vec<_> = (1..40).map(|n| format!("192.168.0.{}:0",n)).collect();
    let n = addresses.len();

    for result in addresses.with_threads(n).map(|s| s.to_socket_addrs()) {
        println!("got: {:?}", result);
    }
}
// got: Ok(IntoIter([V4(192.168.0.1:0)]))
// got: Ok(IntoIter([V4(192.168.0.39:0)]))
// got: Ok(IntoIter([V4(192.168.0.2:0)]))
// got: Ok(IntoIter([V4(192.168.0.3:0)]))
// got: Ok(IntoIter([V4(192.168.0.5:0)]))
// ....
```

这个比ping的例子要快多了因为它只检查IP地址是否合法 —— 如果我们喂一个真实域名的列表DNS查找
将花费更长时间，因为并行执行是重要的。

惊奇的是，它只是可以用。事实上标准库中的每个对象都实现了`Debug`而很方便探索和调试它们。
迭代器返回`Result` (因此 `Ok`) 和`Result`中的`IntoIter`转换成`SocketAddr` ，它是个enum包含了
IPv4和IPv6地址。
为何是 `IntoIter`? 因为一个socket可以有多个地址(包括IPv4和IPv6)。

```rust
    for result in addresses.with_threads(n)
        .map(|s| s.to_socket_addrs().unwrap().next().unwrap())
    {
        println!("got: {:?}", result);
    }
// got: V4(192.168.0.1:0)
// got: V4(192.168.0.39:0)
// got: V4(192.168.0.3:0)
```
这样也能工作，足够惊奇，至少对我们的简单例子来说。第一个 `unwrap` 负责`Result`，然后我们
显式的从迭代器中提取出值来。`Result`在我们输入无意义的地址(如有域名无端口)时通常会报错。

## TCP Client Server

Rust向大多数常见网络协议提供了直观的接口，如TCP。它非常健壮且是我们的网络世界构建的
基础 —— 数据的 _packets_ 被有确认的发送和接收。相反，UDP不用确认的发送包到网络上 ——
有个笑话说"我可以告诉你一个关于UDP的笑话但也许你get不到它"。
(关于网络的笑话只是'funny'的特定含义)

尽管如此，错误处理对网络来说是非常重要的，因为任何情况可发生，并将最终发生。

TCP以client/server模式工作；server监听在一个地址和特定网络端口上，而客户端连接到那个
server。一个连接建立且之后client和server能通过socket通讯。

`TcpStream::connect`接受任何可以被转换成`SocketAddr`的参数，特别是我们用的普通字符串。
both vs usual,
在Rust中一个简单TCP客户端是容易的 —— 一个 `TcpStream` struct 是两者都与平常相比，
我们需要带入`Read`, `Write` 和其他 `std::io` traits到这个范围。

```rust
// client.rs
use std::net::TcpStream;
use std::io::prelude::*;

fn main() {
    let mut stream = TcpStream::connect("127.0.0.1:8000").expect("connection failed");

    write!(stream,"hello from the client!\n").expect("write failed");
 }
```

server也不是特别复杂；我们设置一个listener并等待连接。当一个客户端连接时，
我们在server端得到一个 `TcpStream` 。在这种情况，我们读取客户端写入的所有东西。

```rust
// server.rs
use std::net::TcpListener;
use std::io::prelude::*;

fn main() {

    let listener = TcpListener::bind("127.0.0.1:8000").expect("could not start server");

    // accept connections and get a TcpStream
    for connection in listener.incoming() {
        match connection {
            Ok(mut stream) => {
                let mut text = String::new();
                stream.read_to_string(&mut text).expect("read failed");
                println!("got '{}'", text);
            }
            Err(e) => { println!("connection failed {}", e); }
        }
    }
}
```

这里我们选择一个端口号或多或少是随机的，但[很多端口](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers)
已经被赋予了特定含义。

注意两端都要采用相同协议 —— 客户端期望它能写字符到流中，而server端期望从流中读出文本。
如果它们不是相同游戏规则，一起情况是当其中一方阻塞了，等待着不可能收到的字节。

错误检查很重要 —— 网络I/O会因很多原因而失败，且也许在本地文件系统上蓝屏一次的错误
在一般基础上也能出现。
有人可能拔掉网线，另外一端可能崩溃，等等。这个小的server不是非常健壮，因为它将
在第一次读取错误时退出。

这里是个更健壮的server能不退出的处理错误。它也特别能从流里读取一行，通过 `io::BufReader`
来创建一个 `io::BufRead` 能让我们调用`read_line`。

```rust
// server2.rs
use std::net::{TcpListener, TcpStream};
use std::io::prelude::*;
use std::io;

fn handle_connection(stream: TcpStream) -> io::Result<()>{
    let mut rdr = io::BufReader::new(stream);
    let mut text = String::new();
    rdr.read_line(&mut text)?;
    println!("got '{}'", text.trim_right());
    Ok(())
}

fn main() {

    let listener = TcpListener::bind("127.0.0.1:8000").expect("could not start server");

    // accept connections and get a TcpStream
    for connection in listener.incoming() {
        match connection {
            Ok(stream) => {
                if let Err(e) = handle_connection(stream) {
                    println!("error {:?}", e);
                }
            }
            Err(e) => { print!("connection failed {}\n", e); }
        }
    }
}
```
`handle_connection` 的 `read_line` 也许会失败，但错误结果被安全的处理了。

像这样的单向通讯确实有用 —— 例如。一组跨网络的服务想收集它们的状态报告到一个中心化
地方。但期望一个好的回应是合理的，即便只是回复'ok'！

一个简单的例子是这个基本的'echo' server。客户端写出一些文本到server端并以换行结束，
收到相同的回复文本并换行 —— 流是可读的和可写的。

```rust
// client_echo.rs
use std::io::prelude::*;
use std::net::TcpStream;

fn main() {
    let mut stream = TcpStream::connect("127.0.0.1:8000").expect("connection failed");
    let msg = "hello from the client!";

    write!(stream,"{}\n", msg).expect("write failed");

    let mut resp = String::new();
    stream.read_to_string(&mut resp).expect("read failed");
    let text = resp.trim_right();
    assert_eq!(msg,text);
}
```
server有个有趣的扭曲。只有`handle_connection` 有改变:

```rust
fn handle_connection(stream: TcpStream) -> io::Result<()>{
    let mut ostream = stream.try_clone()?;
    let mut rdr = io::BufReader::new(stream);
    let mut text = String::new();
    rdr.read_line(&mut text)?;
    ostream.write_all(text.as_bytes())?;
    Ok(())
}
```

这是个简单双向socket通讯的普通gotcha；我们想读一行，所以需要喂指向 `BufReader` 的可读流
—— 但它消费了流！所以我们不得不clone这个流，创建一个新的结构体能引用到相同的底层socket。
然后我们拥有欢乐。

