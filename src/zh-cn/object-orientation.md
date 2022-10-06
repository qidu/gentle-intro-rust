## Object-Orientation in Rust

每个人来自某些地方，机会很好，比如你之前的编程语言通过特定方式实现了OOP：

  * 'classes' 表现得像个工厂一样来生成 _objects_ (常叫作 _instances_) 定义了独特的类型。
  * Classes 可以从其他classes(它们的 _父类_) _继承_ , 包括数据 (_fields_) 和行为 (_methods_)
  * 如果B继承了A，那么一个B实例可以当作A实例(_子类型_)传递给某些地方 
  * 对象应隐藏它的数据 (_封装_)，只能通过方法来操作它们

面向对象的 _设计_ 是关于识别类('名词')和方法('动词')并建立它们之间的关系，_is-a_ 和 _has-a_。

这正是在旧星际迷航系列中医生可对舰长说的一个关键点，"这是生活，Jim，只是不是我们
像知道的生活"。那么这很像Rust调味的面向对象：它另人震惊，因为Rust数据聚合(结构体，
枚举和元组)都是哑巴。你可以在它们上定义方法，并让数据是私有的，所有一般封装策略，
但它们全是 _不相关的类型_。
这里没有子类型，也没有数据继承(解引用`Deref`强制转换的特殊情况不算)。

Rust中各种变量数据类型之间的关系通过 _traits_ 来建立。学习Rust的更大部分是理解标准库中
traits如何操作的，因为那是黏合所有数据类型到一起的网。

Traits是有趣的因为在它们和主流语言中没有1v1的对应。它依靠你是否动态的或静态的思考。
在动态情况，它们像Java或Go接口。


### Trait Objects

看看先用来介绍traits的例子:

```rust
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
```
这里带着大的暗示的小程序:

```rust
# trait Show {
#     fn show(&self) -> String;
# }
#
# impl Show for i32 {
#     fn show(&self) -> String {
#         format!("four-byte signed {}", self)
#     }
# }
#
# impl Show for f64 {
#     fn show(&self) -> String {
#         format!("eight-byte float {}", self)
#     }
# }

fn main() {
    let answer = 42;
    let maybe_pi = 3.14;
    let v: Vec<&Show> = vec![&answer,&maybe_pi];
    for d in v.iter() {
        println!("show {}",d.show());
    }
}
// show four-byte signed 42
// show eight-byte float 3.14
```
这是Rust需要类型指蓝的例子 —— 我特别想要一个指向任何实现了`Show`的引用的vector。
现在注意 `i32` 和 `f64` 之间没有联系，但它们都理解`show`方法，因为它们都是些了相同
的trait。这方法是 _虚拟的_，因为实现的方法实现对不同的类型都是不同的，而且正确的方法
基于 _运行时_ 的信息来调用。那些引用被称为[trait objects](https://doc.rust-lang.org/stable/book/trait-objects.html)。

而且 _那_ 是你如何能将不同类型的对象放到相同vector的方法。如果你有Java或Go背景，你能理解
`Show`表现的像个接口。

对这个例子稍作提炼 —— 我们 _box_ 这些值。一个box包含了指向堆上数据的引用，表现非常像
引用 —— 但它是 _智能指针_。当boxes超过作用域时会发生`Drop`，然后内存被释放。

```rust
# trait Show {
#     fn show(&self) -> String;
# }
#
# impl Show for i32 {
#     fn show(&self) -> String {
#         format!("four-byte signed {}", self)
#     }
# }
#
# impl Show for f64 {
#     fn show(&self) -> String {
#         format!("eight-byte float {}", self)
#     }
# }

let answer = Box::new(42);
let maybe_pi = Box::new(3.14);

let show_list: Vec<Box<Show>> = vec![answer,maybe_pi];
for d in &show_list {
    println!("show {}",d.show());
}
// show four-byte signed 42
// show eight-byte float 3.14
```
区别是现在你可以拿着这个vector，作为一个引用传递或移走它而不需要跟踪
任何借用的引用。当vector被废弃时，那些boxes也将被废弃，所有内存都被回收。

## Animals

因为一些原因，任何对OOP和继承的讨论都好像结束于讨论动物。它创造了一个好故事：
"瞧，一只猫是肉食动物。而一个肉食动物是动物"。但我将从一个来自Ruby的经典slogan
开始："如果它嘎嘎叫，那它是只鸭子"。你所有的对象需要做的就是定义`嘎嘎叫`然后就可
被视作是鸭子，尽管在非常窄的路上。

```rust

trait Quack {
    fn quack(&self);
}

struct Duck ();

impl Quack for Duck {
    fn quack(&self) {
        println!("quack!");
    }
}

struct RandomBird {
    is_a_parrot: bool
}

impl Quack for RandomBird {
    fn quack(&self) {
        if ! self.is_a_parrot {
            println!("quack!");
        } else {
            println!("squawk!");
        }
    }
}

let duck1 = Duck();
let duck2 = RandomBird{is_a_parrot: false};
let parrot = RandomBird{is_a_parrot: true};

let ducks: Vec<&Quack> = vec![&duck1,&duck2,&parrot];

for d in &ducks {
    d.quack();
}
// quack!
// quack!
// squawk!
```

这里我们有两个完全不同的类型(一个是哑巴甚至没有任何数据)，是的，它们都能 `quack()`。
其中一个表现得稍古怪（对鸭子来说）但它们共用同样的方法名称而Rust能用类型安全的方式
维持一个这样对象的集合。

类型安全是很奇妙的事儿。没有静态类型，你能插入一只 _猫_ 到那个嘎嘎叫的集合，导致运行
时的混乱。

这里是个有趣的例子:

```rust
// and why the hell not!
impl Quack for i32 {
    fn quack(&self) {
        for i in 0..*self {
            print!("quack {} ",i);
        }
        println!("");
    }
}

let int = 4;

let ducks: Vec<&Quack> = vec![&duck1,&duck2,&parrot,&int];
...
// quack!
// quack!
// squawk!
// quack 0 quack 1 quack 2 quack 3
```
怎么说？它嘎嘎叫了，它必须是只鸭子。有趣的哦是你能应用你的traits到任何Rust值，
不仅仅是'对象'。(因为`嘎嘎叫`是通过引用传递，没有一个显式的接引用来获得整数值)。

尽管如此，你能只做这个如果你定义了trait或类型在相同的crate中，也就是说，你能
为外部的类型`i32`来实现我们的trait  `Quack`，但你不能为`i32`重复实现`ToString`(外部
的trait) 。所以标准库不能被'猴子补丁'。

截至到这，trait  `Quack` 行为非常像Java的接口，并像现代Java接口一样如果你实现了
_必要_ 的方法你能为其提供默认实现。(`Iterator` trait 是个好例子)

但注意，traits不是类型定义的一部分而你能在任何类型上定义和实现新traits，符合同crate限制。

也可能传递一个引用到任何 `Quack`：

```rust
fn quack_ref (q: &Quack) {
    q.quack();
}

quack_ref(&d);
```

这就是Rust风格的子类型。

因为我们在这里比较语言特色，我将提及Go有个有趣的呱呱功能 —— 如果有个Go接口 `Quack`，
一个有 `quack` 方法的类型，那么那个类型满足 `Quack`而不需要任何显式定义。这也
打破了baked-into-definition的Java模型，允许了编译时的duck类型，代价是查找和类型安全。

但duck-typing也有问题。坏的OOP标志是太多方法都有相同的通用名称如`run`。"如果
它有run()，它必须是Runnable" 听起来不那么容易记住！所以一个Go接口可能是 _偶然的_
有效。在Rust中， `Debug` 和 `Display` traits 都定义了`fmt`方法，但它们表达了不同的事。

所以Rust traits允许传统的多态OOP。但继承呢？人们通常认为是 _实现上的继承_ 的地方Rust认为
是 _接口上的继承_。这好像一个Java程序员从不使用`extend`而是用`implements`。这实际
上是Alan Holu的 [推荐实践](http://www.javaworld.com/article/2073649/core-java/why-extends-is-evil.html)
他说：

> 我曾加入一个Java用户群组会议James Gosling(Java发明人)是有号召力的发言人。在难忘的Q&A
> 环节，有人问他："如果你重新发明Java，你会改变什么？" "'我将抛弃classes"，他答复。在
> 大家笑倒之后他解释道，真正的本质问题不是classes，而是实现上的集成(extends relationship)。
> 接口上的集成(implements relaitonship)更好。你应该尽可能的避免实现上的集成。

所以即便在Java中，你很可能滥用了classes!

实现上的集成有些严重问题。但它让人感觉很 _方便_。有这种肥胖的基类叫作 `Animal` 而
它有有用的函数负担(它甚至可以展示自己的内脏！) 我们派生类`Cat`能用。也即，它是种
代码复用的形式。但代码复用是不同的关注点。

知道实现上和接口上继承之间的区别在理解Rust时很重要。

注意traits可以 _提供_ 实现方法。考虑 `Iterator` —— 你 _只需要_ 重载`next`，但免费得到一整簇其他
方法。这与Java接口的`default`方法类似。这类我们只定义`name` 而 `upper_case`已为我们
定义好了。我们也 _可以_ 重载 `upper_case`，但它不是 _必须的_。

```rust
trait Named {
    fn name(&self) -> String;

    fn upper_case(&self) -> String {
        self.name().to_uppercase()
    }
}

struct Boo();

impl Named for Boo {
    fn name(&self) -> String {
        "boo".to_string()
    }
}

let f = Boo();

assert_eq!(f.name(),"boo".to_string());
assert_eq!(f.upper_case(),"BOO".to_string());
```

这是一类代码复用，真的，但注意它没应用在数据上，只是借口！

## Ducks and Generics

在Rust中一个关于泛型友好的鸭子函数例子可以非常微小:

```rust
fn quack<Q> (q: &Q)
where Q: Quack {
    q.quack();
}

let d = Duck();
quack(&d);
```

类型参数可以是任何实现了`Quack`的类型。这里在`quack`和上节的`quack_ref`之间有个
重要的区别。这个函数体为每个调用类型编译一遍而没有虚函数；这种函数可以完全inlined。
它以不同的方式使用trait `Quack`，作为一个对泛型的 _限制_。

这是泛型`quack`的C++等价实现(注意`const`)：

```cpp
template <class Q>
void quack(const Q& q) {
    q.quack();
}
```
注意类型参数没有任何限制。

这是编译时的duck-typing —— 如果我们传递一个到non-quackable的引用，那么编译器
将报错它没有`quack`方法。至少这类错误在编译时发现，要是突然一个类型Quackable了
只会更糟，就像Go里发生的一样。越多引入模板函数和classes导致可怕的错误信息，
因为那里 _没有_ 泛型上的限制约束。

你可以定义个函数来处理一个遍历Quacker指针的迭代器:

```cpp
template <class It>
void quack_everyone (It start, It finish) {
    for (It i = start; i != finish; i++) {
        (*i)->quack();
    }
}
```

这将为每个迭代类型`It`实现。
Rust的等价实现稍微有点挑战:

```rust
fn quack_everyone <I> (iter: I)
where I: Iterator<Item=Box<Quack>> {
    for d in iter {
        d.quack();
    }
}

let ducks: Vec<Box<Quack>> = vec![Box::new(duck1),Box::new(duck2),Box::new(parrot),Box::new(int)];

quack_everyone(ducks.into_iter());
```

Rust中的迭代器不是duck-typed但是必须实现`Iterator`的类型，而这个例子中迭代器提供了`Quack`
的box。这里引入的类型没有任何模糊歧义，且值必须满足`Quack`。函数签名常是Rust
泛型函数上最有挑战性的部分，这正是我推荐读标准库源码的原因 —— 实现常比声明要简单很多！
这里仅有的类型参数实际是迭代器类型，意味着这将与任何能传递`Box<Duck>`,序列的对象一起
work，不只是个vector迭代器。

## Inheritance

面向对象设计的一个共同问题是尝试强制对象间变成 _is-a_ 关系，而忽视 _has-a_ 关系。
[GoF](https://en.wikipedia.org/wiki/Design_Patterns) 20年前在设计模式数里说"相比继承宁愿用组合"。

这里有个例子：当你对公司雇员建模时，`Employee`看起来是好的类名。那么Manager是Employee
(确实)所以我们开始将`Manager`作为`Employee`的子类来构建层级。这看起来并不明智。也许
我们对识别重要名词忘乎所以，也许我们(无意中)认为managers与employees是不同动物？最好
让`Employee`有个 `Roles` 集合，然后一个manager也就是个有更多能力和职责的`Employee`。 

或考虑机动车 —— 范围包括摩托车到300t的卡车。有多种方式来看待机动车，道路价值(全地形，
城市道，轨道铁路等)，动力源(电力，汽油，油电混合等)，货物或人，等等。你基于一个侧面创建的
任何固定结构的类都会忽略所有其他侧面。也就是说，有多种分类机动车的可能性！

在Rust中组合非常重要的原因很明显，你不能偷懒的从一个基类继承功能。

组合也因借用检查器足够聪明而重要，因为它知道借用不同结构体成员是独立的借用。
你可以让一个字段可变借用而让另一个字段不可变借用，等等。Rust不能说一个方法
只访问一个字段，字段应该是结构体自己拥有的方法来实现便利。
(结构体的 _外部接口_ 可以是任何你喜欢用的合适traits。)

一个 'split borrowing'的具体例子会让这更清晰。我们有个结构体拥有一些东西，
用一个方法来可变的借用第一个物品。

```rust
struct Foo {
    one: String,
    two: String
}

impl Foo {
    fn borrow_one_mut(&mut self) -> &mut String {
        &mut self.one
    }
    ....
}
```
(这是个Rust命名规范例子 —— 这样方法应该结束于`_mut`)

现在，另一个方法来同时借用两个东西，复用了第一个方法：

```rust
    fn borrow_both(&self) -> (&str,&str) {
        (self.borrow_one_mut(), &self.two)
    }
```

这不能work！我们从`self`作可变借用又从`self`作不可变借用。如果Rust允许这样的情况，那么
不可变引用就不能被确保不被修改了。

正确的方案是：

```rust
    fn borrow_both(&self) -> (&str,&str) {
        (&self.one, &self.two)
    }
```

那这很好，因为借用检查器看待那些是独立的借用。所以想象下成员是任何类型，那么你可以看到方法
被调用访问到那些成员时将不引起借用问题。

通过 [Deref](https://rust-lang.github.io/book/second-edition/ch15-02-deref.html) 的'继承'很重要
但有个限制，就是解引用操作符`*`的trait。
`String` 实现了 `Deref<Target=str>` 而定义在 `&str` 上的所有方法自动也都对 `String` 是可用的！
同样的， `Foo` 的方法也可以被`Box<Foo>`调用。一些人发现这个...很魔幻，但它极其方便。
比现代Rust有更简单的语言，但它用起来不会有一半快乐。它应该被用在有个自定义的和可变的
类型和一个跟简单的借用类型。

一半在Rust里有 _trait继承_ ：

```rust
trait Show {
    fn show(&self) -> String;
}

trait Location {
    fn location(&self) -> String;
}

trait ShowTell: Show + Location {}
```

最后一个trait简单的组合了前面两个不同trait，尽管它还可以指定其他方法。

像之前处理的一样：

```rust
#[derive(Debug)]
struct Foo {
    name: String,
    location: String
}

impl Foo {
    fn new(name: &str, location: &str) -> Foo {
        Foo{
            name: name.to_string(),
            location: location.to_string()
        }
    }
}

impl Show for Foo {
    fn show(&self) -> String {
        self.name.clone()
    }
}

impl Location for Foo {
    fn location(&self) -> String {
        self.location.clone()
    }
}

impl ShowTell for Foo {}

```

现在，如果我有个`Foo`类型的`foo`值，那么它的一个引用将满足 `&Show`, `&Location` 
或 `&ShowTell` (二者都实现)。

这里是个有用的宏:

```rust
macro_rules! dbg {
    ($x:expr) => {
        println!("{} = {:?}",stringify!($x),$x);
    }
}
```
它接收一个参数(用`$x`表达) 须是一个'表达式'。我们打印出它的值，和其版本的 _字符串格式_。
C程序员在这里 _稍有点_ 沾沾自喜，但这意味着如果我传递`1+2`(表达式)那么 `stringify!(1+2)` 
是字面意义的字符串"1+2"。这将节省我们输入代码的时间：

```rust
let foo = Foo::new("Pete","bathroom");
dbg!(foo.show());
dbg!(foo.location());

let st: &ShowTell = &foo;

dbg!(st.show());
dbg!(st.location());

fn show_it_all(r: &ShowTell) {
    dbg!(r.show());
    dbg!(r.location());
}

let boo = Foo::new("Alice","cupboard");
show_it_all(&boo);

fn show(s: &Show) {
    dbg!(s.show());
}

show(&boo);

// foo.show() = "Pete"
// foo.location() = "bathroom"
// st.show() = "Pete"
// st.location() = "bathroom"
// r.show() = "Alice"
// r.location() = "cupboard"
// s.show() = "Alice"
```

这 _是_ 面向对象，只是不是你习惯的那类。

请注意传给`show`的`Show`引用不能被 _动态的_ 升级成`ShowTell`！有更多动态class
系统的语言允许你检查一个传递对象是否是一个类的实例二进行动态类型转换。一般来
说这并不是个好主意，在Rust特别不行的原因是`Show`引用已'忘记'它原本是个`ShowTell`
引用。

你总是有其他选择：多态，通过trait对象，或单态，通过trait泛型约束。现代C++和Rust
标准库倾向于泛型路线，但也没隔绝多态路线。你确实需要理解不同的取舍 —— 泛型生成
更快的代码，可以inlined嵌入。这也会导致代码膨胀。但不是每个程序需要 _尽可能快_
—— 它在一个典型程序运行的生命周期中只出现少量次。

所以，总结如下：

  - `class`的角色是共享数据和traits
  - 结构体和枚举是哑的(不能继承)，尽管你可以定义方法和进行数据隐藏
  - 一个  _受限的_ 子类型形式是在数据上用`Deref` trait
  - traits不包含任何数据，但可以为任何类型所实现(不仅是结构体)
  - traits可以从其他traits继承
  - traits可以有方法的默认实现，允许接口代码复用
  - traits同时给你虚拟方法(多态) 和泛型限制(单态)

## Example: Windows API

传统OOP被广泛使用的地方之一是GUI工具库。一个`EditControl` 或 `ListWindow` 是个(is-a)
`Window`，以此类推。这让Rust更难绑到GUI库。

Rust里的Win32编程可以[直接用这个] (https://www.codeproject.com/Tips/1053658/Win-GUI-Programming-In-Rust-Language)，
它不相比原始的C语言不那么尴尬。我从C转到C++开始就想要一些更干净的OOP包装。

一个典型的Win32 API函数是 [ShowWindow](https://docs.rs/user32-sys/0.0.9/i686-pc-windows-gnu/user32_sys/fn.ShowWindow.html) 它用来控制窗口的可见性。现在，一个`EditControl` 有一些特殊功能，
但它全通过设置`HWND`(句柄)的透明度值来实现。你可能想要 `EditControl` 也有个`show`
方法，它传统的通过继承来实现。你不想为每个类去型输入所有那些继承方法！而Rust traits
提供了一个方案。这里有个`Window` trait:

```rust
trait Window {
    // you need to define this!
    fn get_hwnd(&self) -> HWND;

    // and all these will be provided
    fn show(&self, visible: bool) {
        unsafe {
         user32_sys::ShowWindow(self.get_hwnd(), if visible {1} else {0})
        }
    }

    // ..... oodles of methods operating on Windows

}
```

所以实现`EditControl`的结构体只需要包含 `HWND` 成员并实现`Window` trait定义的方法。
`EditControl` 是个继承自 `Window` 的trait且定义了扩展接口。像 `ComboxBox` 这样的 ——
行为像 `EditControl` _和_ `ListWindow`的窗口可以简单的通过trait继承来实现。

Win32 API('32'不再表示32位)实际上是面向对象的，但用了旧的方式，被Alan Kay的定义
所影响：对象包含了隐藏的数据，能通过 _消息_ 来操作。所以在任何Windows应用的里面
有个消息loop，而各种类型的窗口('window classes')通过自己的swtch语句实现了那些函数。
有个消息叫作 `WM_SETTEXT` 但有不同的实现方式：一个显示框的文本变化后，对应顶级
窗口标题跟着变化。


[这里](https://gabdube.github.io/native-windows-gui/book_20.html) 是个个最小Windows GUI
框架。但对我来说，太多 `unwrap`实例存在 —— 有些甚至不是错误。因为NWG在探索消息的
松弛动态特性。在合适的类型安全接口中，更多错误在编译时被捕获。


Rust编程语言书的[下一版](https://rust-lang.github.io/book/second-edition/ch17-00-oop.html)
对'面向对象'在Rust中的含义有个很好的讨论：


## [BOOK](https://doc.rust-lang.org/book/ch17-00-oop.html)

OOP是建模程序的一个方法。对象是Simula编程语言在1960s引入的编程概念。那些对象影响
了Alan Kay的对象相互传递消息的编程架构。为了描述这个架构，他在1967年创造了OOP这个
术语。有很多竞争性的定义描述了OOP是什么，根据其中有些定义，Rust是面向对象的，而对
另一些定义则不是。在本节(Charpter 17)，我们将探索被认为是OOP公共的确定性特征，以及
那些特征如何对应为Rust习惯。然后我们将向你展示如何在Rust中实现OOP设计模式，并讨论
与实现这个相对的Rust增强替代之间的取舍。

### 面向对象语言的特征
在编程社区中没有关于一个语言应具备什么特征才能被看作是面向对象的共识。Rust被很多编
程范式所影响，包括OOP；例如，我们在Ch13中探索了来自函数式语言的特征；按道理，OOP
语言共用确定的共同特征，也就是对象，封装和继承。我们来看看每个特征的含义和Rust是如何
支持它的。

#### 对象包含数据和行为
Erich Gamma, Richard Helm, Ralph Johnson, 和 John Vlissides的书《Design Patterns: 
Elements of Reusable Object-Oriented Software》(Addison-Wesley Professional, 1994)，
通俗的引用为GoF书，是个OO设计模式分类。它这么定义OOP：
> OOP由对象构成，对象打包了数据和操作这些数据的过程。传统上过程也被称为方法
> 和操作。
采用这个定义，Rust是面向对象的：结构体和枚举拥有数据，而 `impl` 块在结构体和枚举上
提供了方法。即便如此，有了方法的结构体和枚举也不被称为对象，虽然它们提供了GoF
书的对象定义的功能。

#### 封装隐藏了实现细节
普遍联系在OOP的另一个层面是封装，它表示使用对象的代码不能访问实现对象的细节。
因此，与对象打交道的唯一方式是通过它的公共API；外面的代码不应该接触到对象的内
部直接修改数据或行为。这让程序员能够在不修改调用代码的前提下修改和重构一个对象
的内部。

我们在Ch7中讨论了如何控制封装：我们可以用 `pub` 关键字来决定我们代码中的哪个modules，
types，functions，和methods是public的，默认其他的都是private的。例如，我们定义一个
结构体 `AveragedCollection` 带着 i32的vector成员。这结构体也有个成员代表了vector值的
平均值，意味着在它被任何人调用时也不需要计算平均值。换句话说， `AveragedCollection` 
将为我们缓存平均值。17-1列举了 `AveragedCollection` 结构体的定义：
```rust
pub struct AveragedCollection {
    list: Vec<i32>,
    average: f64,
}
```
这结构体被标记为 `pub` 的让其他代码能使用它，结构体内的成员是private的。这种情况下
这很重要因为我们想要确保无论何时添加和删除一个vector的值，average也能被更新。我们
通过实现结构体的 `add` ， `remove` ，和 `average` 方法来实现，在17-2中如下：
```rust
pub struct AveragedCollection {
    list: Vec<i32>,
    average: f64,
}

impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        match result {
            Some(value) => {
                self.update_average();
                Some(value)
            }
            None => None,
        }
    }

    pub fn average(&self) -> f64 {
        self.average
    }

    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
```
公共方法 `add` ， `remove` ，和 `average` 是访问和修改 `AveragedCollection` 实例数据
的唯一方式。当一个新值被通过 `add` 方法加到 list 中，或通过 `remove` 方法删除一个值，
它们都调用了private的方法 `update_average` 来处理更新 `avarage` 字段。
我们让 `list` 和 `average` 是private的字段，让外部代码没法直接添加和删除 `list` 成员，
否则 `average` 可能在 `list` 变化时不同步了。`average` 方法返回 `average`字段的值，允
许外部代码读到 `average` 但不能修改它。
因为我们封装了结构体 `AveragedCollection` 的实现细节，我们在后面可以容易的修改一些
部分，如数据结构。举例，我们可以用一个 `HashSet<i32>` 代替 `list` 前的 `Vec<i32>`。
因为 `add` ， `remove` ，和 `average` 公共方法的签名是不变的，使用 `AveragedCollection`
的代码也不用修改。如果我们让 `list` 是public的，这种情况将不太必要： `HashSet<i32>` 和
`Vec<i32>` 有不同的添加和删除方法，所以外部调用代码很可能不得不跟着我们直接修改 `list`
一起修改。
如果封装是面向对象语言的必要层面，那么Rust符合需要。在代码不同部分使用 `pub` 与否的
选择能封装实现细节。

#### 继承用作了类型系统和代码复用
'继承'是一个对象可以继承另一个对象定义元素的机制，因此获得父对象的数据和行为而不需要
再定义一遍。
如果一个语言必须有继承才是面向对象语言，那么Rust不是这类。这里不用宏没有办法定义一个
结构体去继承父结构体的成员和方法实现。
尽管如此，如果你习惯于在自己的编程工具箱里拥有继承，那么你在Rust中可以使用其他方案，
取决于你一开始使用继承的目的。

你可以为两个主要原因选择继承。其一是代码复用：你可以为一个类型实现特殊行为，继承让你
能为另一个类型复用这个实现。你可以在Rust代码中通过使用trait默认方法的有限方式实现这个，
就是你在ch10-14中看到的我们添加了 `Summary` trait 的 `summarize` 的默认方法。任何实现了
`Summary` 的类型拥有可用的  `summarize` 方法而不需要更多代码。这类似于基类有个方法实现
也为派生类所拥有，我们也可以在实现 `Summary` trait时重载  `summarize` 方法的默认实现，这
类似于子类重载父类的方法。

另一个原因是使用继承来关联类型系统：让子类可以被用在父类相同的位置。这也被称为是多态，
它意味着你可以在每次运行时替换多个共享相同特定特征的对象。

> #### 多态
> 对很多人来说，多态等同于继承。但实际上一个更通用的概念是关系到代码可以与多个类型的
> 数据一起使用。例如，那些类似是一般的子类。
> Rust代替使用泛型来抽象不同可能性的类型且用trait约束来添加约束到那些类型须提供什么方法。
> 这有时被称为限定参数的多态。
>
继承最近不再被很多编程语言作为喜爱的设计方案是因为它常有共享更多非必要代码的风险。子类
不应该总共享所有父类的特征但用继承却如此。这让程序设计缺少弹性。它也引入了调用子类方法
无意义的可能性或因为方法不能应用在子类上而引起错误。额外的，一些语言只允许单个继承(意味
着一个子类只能继承自一个父类)，更进一步限制了程序设计的弹性。

由于这些理由，Rust采用了trait对象这样不同方法来代替继承。我们来看看trait对象如何在Rust使多
态可用。


### 为不同类型的值采用Trait对象
在Ch8我们提到vectors的一个限制是它们只能存储同一种类型的元素。我们在列表8-9创建了一个变通
方法，通过定义 `SpreadsheetCell` 枚举类让它有变量来保存整数，浮点和文本。这意味这我们可以
保存不同的类型在每个单元格里而仍有一个vector来代表一行单元格值。当我们可交换的条目是一个
在编译时我们就已知类型的固定集合，这是个完美的好方案。

尽管如此，有时我们希望自己库的用户在一个特殊的情况下能够扩展类型集合。为展示我们如何达成
这个，我们将创建一个例子GUI工具遍历成员列表时在调用它们每一个 `draw` 方法来画到屏幕上 ——
一个公共的GUI工具技术。我们将创建一个库crate称为 `gui` 包含一个GUI库的结构体。这个crate也许
包含一些用户使用的类型，比如 `Button` 或 `TextField` 。此外，`gui` 用户也希望创建他们自己的类型
且能画出：例如一个程序员可以创建 `Image` 而另外一个将添加 `SelectBox`。

我们不将在这个例子中实现完整的GUI库但将展示这些组件如何适配在一起。在编写这个库时，我们
不能知道和定义所有其他程序员可能想要创建的类型。但我们知道 `gui` 需要保持跟踪很多类型的值，
并也需要调用每个不同类型值的 `draw` 方法。在我们调用 `draw` 时它不需要知道实际将发生什么，
只是这类型值将有这个方法可以让我们调用。

在一个有继承的语言中做这个，我们可以定义一个名叫 `Component` 的类并有一个叫作 `draw` 的方法。
而其他类型，如 `Button`, `Image`, 和 `SelectBox`，可以继承自 `Component` 类和它的 `draw` 方法。
它们每个可以重载 `draw` 方法来定义它们自己的行为，但框架可以把所有这些类型看作 `Component` 
实例对待并调用它们的 `draw` 方法。但因为Rust没有继承，我们需要另一个方式来结构化 `gui` 库以
允许用户扩展新类型。

#### 定义一个公共行为的Trait
为实现我们希望 `gui` 库拥有的行为，我们将定义个trait名为 `Draw` 并将有一个方法叫作 `draw` 。然后
我们可以定义个vector来保存trait对象。一个trait对象同时指向一个实现了我们特定trait的类型实例和
一个用来在运行时查找trait类型方法的列表。我们通过指定一些指针来创建一个trait对象，例如一个 `&`
引用或一个 `Box<T>` 智能指针，然后 `dyn` 关键字，然后指定相关trait。(我们将在Ch19探讨trait对象
必须使用一个指针的原因  "Dynamically Sized Types and the Sized Trait" )。我们在泛型或具体类型的地方
可以使用trait对象。任何我们使用trait对象的地方，Rust类型系统将确保在编译时用在那个上下文的任何值
将实现了对应的trait。相应的，我们在编译时不需要知道所有可能的类型。

如我们提到的，在Rust中我们节制的不称结构体和枚举为"对象" 来区分于其他语言的对象。在结构体和枚举中，
结构体成员的数据和在 `impl` 代码块的行为是分开的，而在其他语言里，数据和行为合并到一个常称为
对象的概念中。尽管如此，trait对象因合并了数据和行为而更像其他语言中的对象。Trait对象一般不像其他
语言中对象一样有用：它们特定的目的是让允许公共行为的抽象。

列表17-3展示了如何定义个名为 `Draw` 的trait并有一个名为 `draw` 的方法：
```rust
pub trait Draw {
    fn draw(&self);
}
```
这个语法看起来像我们Ch10讨论过的如何定义trait。接下来有个新语法：一个
名为 `Screen` 的结构体持有一个名为 `components` 的vector。这个vector成员
类型常是 `Box<dyn Draw>`，它是个trait对象；它是个为任何实现了 `Draw` trait
并放在 `Box` 时的替身。
```rust
pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}
```
在 `Screen` 结构体上，我们将定义一个名为 `run` 的方法，它将调用每个 `component` 的 `draw` 方法
如下：
```rust
impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```
这与定义结构体时用trait约束泛型参数不同。一个泛型参数一次只能适用于具体类型，而trait对象允许
多个具体类型在运行时来充当trait对象。例如，我们可以使用泛型和trait约束来定义 `Screen` 结构体：
```rust
pub struct Screen<T: Draw> {
    pub components: Vec<T>,
}

impl<T> Screen<T>
where
    T: Draw,
{
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```
这限制了一个 `Screen` 实例有所有 `Button` 类型或所有 `TextField` 类型的列表。如果你将只用
到同类集合，使用泛型和trait约束更好因为定义在编译时被用具体类型来单态化了。

#### 实现Trait
现在我们添加一些实现了 `Draw` trait的类型。我们将提供 `Button` 类型。再次说明实现一个GUI
库超出了本书的范围，所以 `draw` 方法体将不会有任何有用实现。想象实现像这样，一个 `Buttom`
结构体可以有 `width`，`height` 和 `label` 成员：
```rust
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // code to actually draw a button
    }
}
```
在 `Button` 的成员 `width`，`height` 和 `label` 成员与其他component的成员不同；
例如，一个 `TextField` 类型可以有签名这些字段外加一个 `placeholder` 字段。
每个我们想画在屏幕上的类型将实现 `Draw` trait但将在 `draw` 方法里使用不同的
代码来定义如何画这个特殊类型，就像 `Button` 这里有的(如之前提的，没有实际
GUI代码)。`Button` 类型，例如，可以有一个额外的 `impl` 块包含了关于在用户
点击button时发生什么的方法。那些方法不会用在其他类型如 `TextField` 上。

如果有人使用我们的库并决定实现一个有 `width`, `height`, 和 `options` 字段的
`SelectBox` 结构体，它们在 `SelectBox` 上也实现了 `Draw` trait。如下：
```rust
use gui::Draw;

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // code to actually draw a select box
    }
}
```
我们的库用户现在可以写出他们的 `main` 函数以创建出一个 `Screen` 实例。对 `Screen` 实例，
它们可以通过放到 `Box<T>` 中变成trait对象来添加一个 `SelectBox` 和一个 `Button` 。它们
然后在 `Screen` 实例上调用 `run` 方法，然后调用每个component的 `draw`。如下：
```rust
use gui::{Button, Screen};

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No"),
                ],
            }),
            Box::new(Button {
                width: 50,
                height: 10,
                label: String::from("OK"),
            }),
        ],
    };

    screen.run();
}
```
在我们写这个GUI库时，并不知道有人也许添加 `SelectBox` 类型，但我们 `Screen` ，但
我们的 `Screen` 实现能操作在新类型上并画出它是因为 `SelectBox` 实现了 `Draw` trait，
这意味这它实现了 `draw` 方法。

这个概念 —— 只考虑这消息是个响应值而不管值的具体类型 —— 与动态语言中的 'duck typing' 
相似：如果它走路像Duck并嘎嘎叫，那么它必然是个Duck！在 `Screen` 的 `run` 实现中，
`run` 不需要知道每个component的具体类型。它不检查一个component是否是个 `Button`
或  `SelectBox` ，它只是在这component上调用 `draw` 方法。通过指定 `Box<dyn Draw>`
作为components vector中的值类型，我们完成定义 `Screen` 以需要一些我们能在它上调用
`draw` 的变量值。

使用trait对象和Rust类型系统写出相似于duck typing代码的好处是，我们从不需要检查
一个变量值在运行时是否实现过特定方法，或担心如果没实现这个必要的trait而得到错误。

例如，以下展示了如果我们尝试创建一个 `Screen` 和一个 `String` 作为component时会发生
什么：
```rust
use gui::Screen;

fn main() {
    let screen = Screen {
        components: vec![Box::new(String::from("Hi"))],
    };

    screen.run();
}
```
我们得到如下错误因为 `String` 没实现 `Draw` trait：
```rust
$ cargo run
   Compiling gui v0.1.0 (file:///projects/gui)
error[E0277]: the trait bound `String: Draw` is not satisfied
 --> src/main.rs:5:26
  |
5 |         components: vec![Box::new(String::from("Hi"))],
  |                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Draw` is not implemented for `String`
  |
  = help: the trait `Draw` is implemented for `Button`
  = note: required for the cast to the object type `dyn Draw`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `gui` due to previous error
```
这个错误让我们知道要么我们传递给 `Screen` 的不是真想传的值而需要传一个不同类型，或
我们应该在 `String` 上实现 `Draw` 所以 `Screen` 能够调用它的 `draw` 。

#### Trait 对象执行动态派发
回想Ch10的 "Performance of Code Using Generics" 节里我们讨论了当我们在泛型上用trait约束
编译器执行的单态化处理：编译器为每个我们用了泛型参数的具体类型生成了非泛型的函数和方法
实现。来自单态化的代码在进行静态派发，它是当编译器知道在编译时你调用了什么方法。这与
动态派发相反，后者是在编译时编码器没法告诉你要调用哪个方法。在动态派发情况里，编译器
产生一些能在运行时搞清楚调用哪个方法的代码。

当我们使用trait对象时，Rust必须使用动态派发。编译器不知道所有也许被与trait对象一起使用的
类型，所以它不知道哪个方法实现在哪个类型上需要调用。作为代替，在运行时，Rust使用trait
对象内的指针以弄清要调用哪个方法。这个查找引起一个在静态派发时不存在的运行时代价。
动态派发也阻止了编译器去inline一个方法的代码，它也相应的阻止了一些优化。
尽管如此，我们在代码里得到额外的弹性如在列表17-5中写的和支持了列表17-9，所以是个取舍。


### 实现一个面向对象的设计模式
