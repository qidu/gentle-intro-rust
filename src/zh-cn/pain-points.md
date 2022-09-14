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

The borrow checker will get less frustrating when _non-lexical lifetimes_
arrive sometime this year.

The borrow checker _does_ understand some important cases, however.
If you have a struct, fields can be independently borrowed. So
composition is your friend; a big struct should contain smaller
structs, which have their own methods. Defining all the mutable methods
on the big struct will lead to a situation where you can't modify
things, even though the methods might only refer to one field.

With mutable data, there are special methods for treating parts of the
data independently. For instance, if you have a mutable slice, then `split_at_mut`
will split this into two mutable slices. This is perfectly safe, since Rust
knows that the slices do not overlap.

## References and Lifetimes

Rust cannot allow a situation where a reference outlives the value. Otherwise
we would have a 'dangling reference' where it refers to a dead value -
a segfault is inevitable.

`rustc` can often make sensible assumptions about lifetimes in functions:

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

This is quite safe because we cope with the case where the delimiter isn't found.
`rustc` is here assuming that both strings in the tuple are borrowed from the
string passed as an argument to the function.

Explicitly, the function definition looks like this:

```rust
fn pair<'a>(s: &'a str, ch: char) -> (&'a str, &'a str) {...}
```
What the notation says is that the output strings live _at most as long_ as the
input string. It's not saying that the lifetimes are the same, we could drop them
at any time, just that they cannot outlive `s`.

So, `rustc` makes common cases prettier with _lifetime ellision_.

Now, if that function received _two_ strings, then you would need to
explicitly do lifetime annotation to tell Rust what output string is
borrowed from what input string.

You always need an explicit lifetime when a struct borrows a reference:

```rust
struct Container<'a> {
    s: &'a str
}
```

Which is again insisting that the struct cannot outlive the reference.
For both structs and functions, the lifetime needs to be declared in `<>`
like a type parameter.

Closures are very convenient and a powerful feature - a lot of the power
of Rust iterators comes from them. But if you store them, you have
to specify a lifetime. This is because basically a closure is a generated
struct that can be called, and that by default borrows its environment.
Here the `linear` closure has immutable references to `m` and `c`.

```rust
let m = 2.0;
let c = 0.5;

let linear = |x| m*x + c;
let sc = |x| m*x.cos()
...
```

Both `linear` and `sc` implement `Fn(x: f64)->f64` but they are _not_
the same animal - they have different types and sizes!  So to store
them you have to make a `Box<Fn(x: f64)->f64 + 'a>`.

Very irritating if you're used to how fluent closures are in Javascript
or Lua, but C++ does a similar thing to Rust and needs `std::function`
to store different closures, taking a little penalty for the virtual
call.


## Strings

It is common to feel irritated with Rust strings in the beginning. There are different
ways to create them, and they all feel verbose:

```rust
let s1 = "hello".to_string();
let s2 = String::from("dolly");
```
Isn't "hello" _already_ a string? Well, in a way. `String` is an _owned_ string,
allocated on the heap; a string literal "hello" is of type `&str` ("string slice")
and might be either baked into the executable ("static") or borrowed from a `String`.
System languages need this distinction - consider a tiny microcontroller, which has
a little bit of RAM and rather more ROM. Literal strings will get stored in ROM
("read-only") which is both cheaper and consumes much less power.

But (you may say) it's so simple in C++:

```C
std::string s = "hello";
```
Which is shorter yes, but hides the implicit creation of a string object.
Rust likes to be explicit about memory allocations, hence `to_string`.
On the other hand, to borrow from a C++ string requires `c_str`, and
C strings are stupid.

Fortunately, things are better in Rust - _once_ you accept that both `String` and `&str`
are necessary. The methods of `String` are mostly for changing the string,
like `push` adding a char (under the hood it's very much like a `Vec<u8>`).
But all the methods of `&str` are also available. By the same `Deref`
mechanism, a `String` can be passed as `&str` to a function - which is
why you rarely see `&String` in function definitions.

There are a number of ways to convert `&str` to `String`, corresponding
to various traits. Rust needs these traits to work with types generically.
As a rule of thumb, anything that implements `Display` also knows `to_string`,
like `42.to_string()`.

Some operators may not behave according to intuition:

```rust
    let s1 = "hello".to_string();
    let s2 = s1.clone();
    assert!(s1 == s2);  // cool
    assert!(s1 == "hello"); // fine
    assert!(s1 == &s2); // WTF?
```

Remember, `String` and `&String` are different types, and `==` isn't
defined for that combination. This might puzzle a C++ person who is
used to references being almost interchangeable with values.
Furthermore, `&s2` doesn't _magically_ become a `&str`, that's
a _deref coercion_ which only happens when assigning to a `&str`
variable or argument. (The explicit `s2.as_str()` would work.)

However, this more genuinely deserves a WTF:

```rust
let s3 = s1 + s2;  // <--- no can do
```
You cannot concatenate two `String` values, but you can concatenate
a `String` with a `&str`.  You furthermore cannot concatenate a
`&str` with a `String`. So mostly people don't use `+` and use
the `format!` macro, which is convenient but not so efficient.

Some string operations are available but work differently. For instance,
languages often have a `split` method for breaking up a string into an array
of strings. This method for Rust strings returns an _iterator_, which
you can _then_ collect into a vector.

```rust
let parts: Vec<_> = s.split(',').collect();
```

This is a bit clumsy if you are in a hurry to get a vector. But
you can do operations on the parts _without_ allocating a vector!
For instance, length of largest string in the split?

```rust
let max = s.split(',').map(|s| s.len()).max().unwrap();
```

(The `unwrap` is because an empty iterator has no maximum and we must
cover this case.)

The `collect` method returns a `Vec<&str>`, where the parts are
borrowed from the original string - we only need allocate space
for the references.  There is no method like this in C++, but until
recently it would have to individually allocate each substring. (C++ 17
has `std::string_view` which behaves like a Rust string slice.)

## A Note on Semicolons

Semicolons are _not_ optional, but usually left out in the same places as
in C, e.g. after `{}` blocks. They also aren't needed after `enum` or
`struct` (that's a C peculiarity.)  However, if the block must have a
_value_, then the semi-colons are dropped:

```rust
    let msg = if ok {"ok"} else {"error"};
```

Note that there must be a semi-colon after this `let` statement!

If there were semicolons after these string literals then the returned
value would be `()` (like `Nothing` or `void`). It's common error when
defining functions:

```rust
fn sqr(x: f64) -> f64 {
    x * x;
}
```

`rustc` will give you a clear error in this case.

## C++-specific Issues

### Rust value semantics are Different

In C++, it's possible to define types which behave exactly like primitives
and copy themselves. In addition, a move constructor can be defined to
specify how a value can be moved out of a temporary context.

In Rust, primitives behave as expected, but the `Copy` trait can only
be defined if the aggregate type (struct, tuple or enum) itself contains
only copyable types. Arbitrary types may have `Clone`, but you have
to call the `clone` method on values. Rust requires any allocation
to be explicit and not hide in copy constructors or assignment operators.

So, copying and moving is always defined as just moving bits around and is
not overrideable.

If `s1` is a non `Copy` value type, then `s2 = s1;` causes a move to happen,
and this _consumes_ `s1`!  So, when you really want a copy, use `clone`.

Borrowing is often better than copying, but then you must follow the
rules of borrowing. Fortunately, borrowing _is_ an overridable behaviour.
For instance, `String` can be borrowed as `&str`, and shares all the
immutable methods of `&str`. _String slices_ are very powerful compared
to the analogous C++ 'borrowing' operation, which is to extract a `const char*`
using `c_str`. `&str` consists of a pointer to some owned bytes (or a string
literal) and a _size_. This leads to some very memory-efficient patterns.
You can have a `Vec<&str>` where all the strings have been borrowed from
some underlying string - only space for the vector needs to be allocated:

For example, splitting by whitespace:

```rust
fn split_whitespace(s: &str) -> Vec<&str> {
    s.split_whitespace().collect()
}
```

Likewise, a C++ `s.substr(0,2)` call will always copy the string, but a slice
will just borrow: `&s[0..2]`.

There is an equivalent relationship between `Vec<T>` and `&[T]`.

### Shared References

Rust has _smart pointers_ like C++ - for instance, the equivalent of
`std::unique_ptr` is `Box`. There's no need for `delete`, since any
memory or other resources will be reclaimed when the box goes out of
scope (Rust very much embraces RAII).

```rust
let mut answer = Box::new("hello".to_string());
*answer = "world".to_string();
answer.push('!');
println!("{} {}", answer, answer.len());
```

People find `to_string` irritating at first, but it is _explicit_.

Note the explicit dererefence `*`, but methods on smart pointers
don't need any special notation (we do not say `(*answer).push('!')`)

Obviously, borrowing only works if there is a clearly defined owner of
the original content. In many designs this isn't possible.

In C++, this is where `std::shared_ptr` is used; copying just involves
modifying a reference count on the common data. This is not without
cost, however:

- even if the data is read-only, constantly modifying the reference
  count can cause cache invalidation
- `std::shared_ptr` is designed to be thread-safe and carries locking
  overhead as well

In Rust, `std::rc::Rc` also acts like a shared smart pointer using
reference-counting. However, it is for immutable references only! If you
want a thread-safe variant, use `std::sync::Arc` (for 'Atomic Rc').
So Rust is being a little awkward here in providing two variants, but you
get to avoid the locking overhead for non-threaded operations.

These must be immutable references because that is fundamental to Rust's
memory model. However, there's a get-out card: `std::cell::RefCell`.
If you have a shared reference defined as `Rc<RefCell<T>>` then you
can mutably borrow using its `borrow_mut` method. This applies the
Rust borrowing rules _dynamically_ - so e.g. any attempt to call
`borrow_mut` when a borrow was already happening will cause a panic.

This is still _safe_. Panics will happen
_before_ any memory has been touched inappropriately! Like exceptions,
they unroll the call stack. So it's an unfortunate word for such
a structured process - it's an ordered withdrawal rather than a
panicked retreat.

The full `Rc<RefCell<T>>` type is clumsy, but the application code isn't
unpleasant. Here Rust (again) is prefering to be explicit.

If you wanted thread-safe access to shared state, then `Arc<T>` is the
only _safe_ way to go. If you need mutable access, then `Arc<Mutex<T>>`
is the equivalent of `Rc<RefCell<T>>`. `Mutex` works a little differently
than how it's usually defined: it is a container for a value. You get
a _lock_ on the value and can then modify it.

```rust
let answer = Arc::new(Mutex::new(10));

// in another thread
..
{
  let mut answer_ref = answer.lock().unwrap();
  *answer_ref = 42;
}
```

Why the `unwrap`? If the previous holding thread panicked, then
this `lock` fails. (It's one place in the documentation where `unwrap`
is considered a reasonable thing to do, since clearly things have
gone seriously wrong. Panics can always be caught on threads.)

It's important (as always with mutexes) that this exclusive lock is
held for as little time as possible. So it's common for them to
happen in a limited scope - then the lock ends when the mutable
reference goes out of scope.

Compared with the apparently simpler situation in C++ ("use shared_ptr dude")
this seems awkward. But now any _modifications_ of shared state become obvious,
and the `Mutex` lock pattern forces thread safety.

Like everything, use shared references with [caution](https://news.ycombinator.com/item?id=11698784).

### Iterators

Iterators in C++ are defined fairly informally; they involve smart pointers,
usually starting with `c.begin()` and ending with `c.end()`. Operations on
iterators are then implemented as stand-alone template functions, like `std::find_if`.

Rust iterators are defined by the `Iterator` trait; `next` returns an `Option` and when
the `Option` is `None` we are finished.

The most common operations are now methods.
Here is the equivalent of `find_if`. It returns an `Option` (case
of not finding is `None`) and here the `if let` statement is convenient for
extracting the non-`None` case:

```rust
let arr = [10, 2, 30, 5];
if let Some(res) = arr.find(|x| x == 2) {
    // res is 2
}
```

### Unsafety and Linked Lists

It's no secret that parts of the Rust stdlib are implemented using `unsafe`. This
does not invalidate the conservative approach of the borrow checker. Remember that
"unsafe" has a particular meaning - operations which Rust cannot fully verify at
compile time. From Rust's perspective, C++ operates in unsafe mode all the time!
So if a large application needs a few dozen lines of unsafe code, then that's fine,
since these few lines can be carefully checked by a human. Humans are not good at
checking 100Kloc+ of code.

I mention this, because there appears to be a pattern:
an experienced C++ person tries to implement a linked list or a tree structure,
and gets frustrated. Well, a double-linked list _is_ possible in safe Rust,
with `Rc` references going forward, and `Weak` references going back. But the
standard library gets more performance out of using... pointers.
