# 第一章：Rust 并发基础

早在多核处理器司空见惯之前，操作系统就允许一台计算机运行多个程序。这是通过在进程之间快速切换来完成的，允许每个进程逐个地重重地取得一点进展。现在，几乎所有的电脑，甚至手机和手表都有着多核处理器，可以真正并行执行多个程序。

操作系统尽可能的将进程之间隔离，允许程序完全意识不到其他线程在做什么的情况下做自己的事情。例如，在不先询问操作系统内核的情况下，一个进程通常不能获取其他进程的内存，或者以任意方式与之通信。

然而，一个程序可以产生额外的*执行线程*作为*进程*的一部分。同一进程中的线程不会相互隔离。线程共享内存并且可以通过内存相互交互。

这一章节将阐述在 Rust 中如何产生线程，并且关于它们的所有基本概念，例如如何安全地在多个线程之间共享数据。本章中解释的概念是本书其余部分的基础。

> 如果你已经熟悉 Rust 中的这些部分，你可以随时跳过。然而，在你继续下一章节之前，请确保你对线程、内部可变性、Send 和 Sync 有一个好的理解，以及知道什么是互斥锁[^2]、条件变量[^1]以及线程阻塞（park）[^3]。

## Rust 中的线程

每个程序都从一个线程开始：主（main）线程。该线程将执行你的 main 函数，并且如果需要，它可以用于产生更多线程。

在 Rust 中，新线程使用来自标准库的 `std::thread::spawn` 函数产生。它接受一个参数：新线程执行的函数。一旦该函数停止，将立刻返回。

让我们看一个示例：

```rust
use std::thread;

fn main() {
    thread::spawn(f);
    thread::spawn(f);

    println!("Hello from the main thread.");
}

fn f() {
    println!("Hello from another thread!");

    let id = thread::current().id();
    println!("This is my thread id: {id:?}");
}
```

我们产生两个线程，它们都将执行 f 作为它们的主函数。这两个线程将输出一个信息并且展示它们的*线程 id*，主线程也将输出它自己的信息。

<div style="border:medium solid green; color:green;">
  <h2 style="text-align: center;">Thread ID</h2>
  Rust 标准库位每一个线程分配一个唯一的标识符。此标识符可以通过 Thread::id() 访问并且拥有 ThreadId 类型。除了复制 ThreadId 以及检查它们相等外，你也做不了什么。不能保证这些 ID 将会连续分配，只是每个线程都会有所不同。
</div>

如果你运行我们上面的示例几次，你可能注意到输出在运行之间有所不同。一次特定的运行在机器上的输出：

```txt
Hello from the main thread.
Hello from another thread!
This is my thread id:
```

惊讶地是，部分输出似乎失去了。

这里发生的情况是：新的线程结束执行它们的函数之前，主线程结束执行了主函数。

从主函数返回将退出整个程序，即使所有线程仍然在运行。

在这个特定的示例中，在程序被主线程关闭之前，其中一个新的线程有足够的消息到达第二条消息的一半。

如果我们想要在主函数返回之前，确保线程结束，我们可以通过 `join` 它们来等待。未来这样做，我们在 `spawn` 函数返回后使用 `JoinHandle`：

```rust
fn main() {
    let t1 = thread::spawn(f);
    let t2 = thread::spawn(f);

    println!("Hello from the main thread.");

    t1.join().unwrap();
    t2.join().unwrap();
}
```

`.join()` 方法等待直到线程结束执行并且返回 `std::thread::Result`。如果线程由于 panic 不能成功地完成它的函数，这将包含 panic 消息。我们试图去处理这种情况，或者在 join panic 线程仅调用 `.unwrap()` 去 panic。

运行我们程序的这个版本，将不再导致截断的输出：

```txt
Hello from the main thread.
Hello from another thread!
This is my thread id: ThreadId(3)
Hello from another thread!
This is my thread id: ThreadId(2)
```

唯一仍然改变的是消息的打印顺序：

```txt
Hello from the main thread.
Hello from another thread!
Hello from another thread!
This is my thread id: ThreadId(2)
This is my thread id: ThreadId(3)
```

<div style="border:medium solid green; color:green;">
  <h2 style="text-align: center;">输出锁定</h2>
  println 宏使用 std::io::Stdout::lock() 去确保输出没有被中断。println!() 将等待直到任意并发地运行完成后，在写入输出。如果不是这样，我们可以得到更多的交叉输出：

  <p style="background:rgb(165,255,144)">
  Hello fromHello from another thread!
  another This is my threthreadHello fromthread id: ThreadId!
  ( the main thread.
  2)This is my thread
  id: ThreadId(3)
  </p>
</div>

与其将函数的名称传递给 `std::thread::spawn`，不如像我们上面的示例那样，传递一个*闭包*。这允许我们捕获值移动到新的线程：

```rust
let numbers = vec![1, 2, 3];

thread::spawn(move || {
    for n in &numbers {
        println!("{n}");
    }
}).join().unwrap();
```

在这里，numbers 的所有权被转移到新产生的线程，因为我们使用了 `move` 闭包。如果我们没有使用 `move` 关键字，闭包将会通过引用补货 numbers。这将导致一个编译器错误，因为新的线程超出变量的生命周期。

由于线程可能运行直到程序执行结束，因此产生的线程在它的参数类型上有 `'static` 生命周期绑定。换句话说，它只接受永久保留的函数。闭包通过引用捕获局部变量不能够永久保留，因为当局部变量不存在时，引用将变得无效。

从线程中取回一个值，是从闭包中返回完成的。该返回值通过 `join` 方法返回的 `Result` 中获取：

```rust
let numbers = Vec::from_iter(0..=1000);

let t = thread::spawn(move || {
    let len = numbers.len();
    let sum = numbers.iter().sum::<usize>();
    sum / len  // 1
});

let average = t.join().unwrap(); // 2

println!("average: {average}");
```

在这里，线程闭包（1）返回的值通过 `join` 方法发送回主线程。

如果 numbers 是空的，当它尝试去除以 0 时（2），线程将发生 panic，而 `join` 将会发生 panic 消息，将由于 `unwarp` 导致主线程也 panic。

<div style="border:medium solid green; color:green;">
  <h2 style="text-align: center;">Thread Builder</h2>
  std::thread::spawn 函数事实上仅是 std::thread::Builder::new().spawn().unwrap() 的快捷缩写。

  std::thread::Builder 允许你在产生线程之前为新线程设置一些设置。你可以使用它为新线程配置栈大小并给新线程一个名字。线程的名字是可以通过 std::thread::current().name() 获得，这将在 panic 消息中可用，并在监控和大多数雕饰工具中可见。

  此外，Builder 的产生函数返回一个 std::io::Result，允许你处理新线程失败的情况。如果操作系统内存不足，或者资源限制已经应用于你对程序，这是可能发生的。如果 std::thread::spawn 函数不能去产生一个新线程，它只会 panic。
</div>

## 线程作用域

如果我们确信生成的线程不会比某个范围存活更久，那么线程可以安全地借用哪些不会一直存在的东西，例如局部变量，只要它们比该范围活得更久。

Rust 标准库提供了 `std::thread::scope` 去产生此类*线程作用域*。它允许我们产生不超过我们传递给该函数闭包的范围的线程，这使它可能安全地借用局部变量。

它的工作原理最好使用一个示例来展示：

```rust
let numbers = vec![1, 2, 3];

thread::scope(|s| { // 1
    s.spawn(|| { // 2
        println!("length: {}", numbers.len());
    });
    s.spawn(|| { // 2
        for n in &numbers {
            println!("{n}");
        }
    });
}); // 3
```

1. 我们使用闭包调用 `std::thread::scope` 函数。我们的闭包是直接执行，并得到一个参数，`s`，表示作用域。
2. 我们使用 `s` 去产生线程。该闭包可以借用本地变量，例如 numbers。
3. 当作用域结束，所有仍没有 join 的线程都会自动 join。

这种模式保证了，在作用域产生的线程没有会比作用域更长的生命周期。因此，作用域中的 `spawn` 方法在它的参数类型中没有一个 `'static` 约束，允许我们去引用任何东西，只要它比作用域有更长的生命周期，例如 numbers。

在以上示例中，这两个线程并发地获取 numbers。这是没问题的，因为它们其中的任何一个（或者主线程）都没有修改它。如果我们改变第一个线程去修改 numbers，正如下面展示的，编译器将不允许我们也产生另一个也使用数字的线程：

```rust
let mut numbers = vec![1, 2, 3];

thread::scope(|s| {
    s.spawn(|| {
        numbers.push(1);
    });
    s.spawn(|| {
        numbers.push(2); // Error!
    });
});
```

确切的错误信息以待遇 Rust 编译器版本，因为它经常被改进以提升更好的判断，但是试图去编译以上代码将导致一下问题：

```txt
error[E0499]: cannot borrow `numbers` as mutable more than once at a time
 --> example.rs:7:13
  |
4 |     s.spawn(|| {
  |             -- first mutable borrow occurs here
5 |         numbers.push(1);
  |         ------- first borrow occurs due to use of `numbers` in closure
  |
7 |     s.spawn(|| {
  |             ^^ second mutable borrow occurs here
8 |         numbers.push(2);
  |         ------- second borrow occurs due to use of `numbers` in closure
```

<div style="border:medium solid green; color:green;">
  <h2 style="text-align: center;">泄漏启示录</h2>
  在 Rust 1.0 之前，标准库有一个函数叫做 std::thread::scoped，它将直接产生一个线程，就像 std::thread::spawn。它允许无 'static 的捕获，因为它返回的不是 JoinGuard，而是当被 drop 时 join 到线程的 JoinGuard。任意的借用数据仅需要比这个 JoinGuard 活得更久。只要 JoinGuard 在某个时候被 drop，这似乎是安全的。

  就在 Rust 1.0 发布之前，人们慢慢发现它似乎不能保证某些东西被 drop。有很多种方式没有 drop 它，例如创建一个引用计数节点的循环，可以忘记某些东西或者*泄漏*它。

  最终，在一些人提及的“泄漏启示录”中得到结论，（安全）接口的设计不能依赖假设对象总是在它们的生命周期结束后 drop。泄漏一个对象可能会导致泄漏更多对象（例如，泄漏一个 Vec 将也导致泄漏它的元素），但它并不会导致未定义行为。因此，std::thread::scoped 将不再视为安全的并从标准库移除。此外，std::mem::forget 从一个不安全的函数升级到*安全*的函数，以强调忘记（或泄漏）总是一种可能性。

  直到后来，在 Rust 1.63 中，添加了一个新的 std::thread::scope 功能，其新设计不依赖 Drop 来获得正确性。
</div>

## 共享所有权以及引用计数

目前，我们已经使用了 `move` 闭包（[“Rust 中的线程”](#rust-中的线程)）将值的所有权转移到线程并从生命周期较长的父线程借用数据（[“线程作用域”](#线程作用域)）。当两个线程之间共享数据，它们之间的任何一个线程都不能保证比另一个线程的生命周期长，那么它们都不能称为该数据的所有者。它们之间共享的任何数据都需要与最长生命周期的线程一样长。

### Static

有几种方式去创建不属于单线程的东西。最简单的方式是**静态**值，它由整个程序“拥有”，而不是单个线程。在以下示例中，这两个线程都可以获取 X，但是它们并不能拥有它：

```rust
static X: [i32; 3] = [1, 2, 3];

thread::spawn(|| dbg!(&X));
thread::spawn(|| dbg!(&X));
```

静态项一半由一个常量初始化，它从不会被 drop，并且甚至在程序的主线程开始之前就已经存在。每个线程都可以借用它，因为它保证它总是存在。

### 泄漏（Leak）

另一种方式是通过*泄漏*分配的方式共享所有权。使用 `Box::leak`，人们可以释放 `Box` 的所有权，保证永远不会 drop 它。从那时起，`Box` 将永远存在，没有所有者，只要程序运行，任意线程都可以借用它。

```rust
let x: &'static [i32; 3] = Box::leak(Box::new([1, 2, 3]));

thread::spawn(move || dbg!(x));
thread::spawn(move || dbg!(x));
```

`move` 闭包可能会让它看起来像我们移动所有权进入线程，但仔细观察 x 的类型就会发现，我们只是给线程一个对数据的*引用*。

> 引用是 `Copy` 的，这意味着当你“move”它们的时候，原始内容仍然存在，这就像整数或者布尔内容一样。

注意，`'static` 生命周期并不意味着该值自程序开始时就存在，而只是意味着它一直存在到程序的结束。过去并不重要。

泄漏 `Box` 的缺点是我们正在泄漏内存。我们分配一些东西，但是从未 drop 和取消分配它。如果仅发生有限的次数，这就可以了。但是如果我们继续这样做，程序将慢慢地耗尽内存。

### 引用计数

为了确保共享数据能够 drop 和取消分配，我们不能完全放弃它的所有权。相反，我们可以*分享所有权*。通过跟踪所有者的数量，我们确保仅当没有所有者时，才会丢弃该值。

Rust 标准库通过 `std::rc::Rc` 类型提供了该功能，它是“引用计数”（reference counted）的缩写。它与 `Box` 非常类似，唯一的区别是克隆它将不会分配任何新内存，而是增加存储在包含值旁边的计数器。原始的 `Rc` 和克隆的 `Rc` 将引用相同的内存分配；它们*共享所有权*。

```rust
use std::rc::Rc;

let a = Rc::new([1, 2, 3]);
let b = a.clone();

assert_eq!(a.as_ptr(), b.as_ptr()); // Same allocation!
```

drop 一个 `Rc` 将减少计数。只有最后一个 `Rc`，计数器下降到 0，才会是 drop 和取消分配所包含的数据。

如果我们尝试去发送一个 Rc 到另一个线程，然而，我们将造成以下的编译错误：

```txt
error[E0277]: `Rc` cannot be sent between threads safely
    |
8   |     thread::spawn(move || dbg!(b));
    |                   ^^^^^^^^^^^^^^^
```

事实证明，`Rc` 不是*线程安全*的（详见，[线程安全：Send 和 Sync](#线程安全send-和-sync)）。如果多个线程有相同分配的 `Rc`，那么它们可能尝试同时修改引用计数，这可能产生不可预测的结果。

然而，我们可以使用 `std::sync::Arc`，它代表“原子引用计数”。它与 `Rc` 相同，只是它保证了对引用计数的修改时不可分割的*原子*操作，因此可以安全地与多个线程使用。（详见第二章。）

```rust
use std::sync::Arc;

let a = Arc::new([1, 2, 3]); // 1
let b = a.clone(); // 2

thread::spawn(move || dbg!(a)); // 3
thread::spawn(move || dbg!(b)); // 3
```

1. 我们在新的分配中放置了一个一个数组，以及从一开始的引用计数器。
2. 克隆 Arc 增加引用计数到两个，并为我们提供相同的分配到第二个 Arc。
3. 两个 Arc 都获取到自己的 Arc，通过 Arc 它们可以获取共享数组。当它们 drop 它们的 Arc，两者都会减少引用计数。最后一个 drop 它的 Arc 的线程将看见计数器减少到 0，并且将是 drop 和取消分配数组的线程。

<div style="border:medium solid green; color:green;">
  <h2 style="text-align: center;">命名克隆</h2>
  不得不给每个 Arc 的克隆取一个不同的名称，这可能使得代码变得混乱难以追踪。尽管每个 Arc 的克隆都是一个独立的对象，而通过每个克隆赋予不同的名称并不能很好地反映这一点。

  Rust 允许（并且鼓励）你通过定义有着新的名称的相同变量去*遮蔽*变量。如果你在相同作用域这么做，则无法再命名原始变量。但是通过打开一个新的作用域，可以使用类似 <code>let a = a.clone();</code> 的语句在该作用域内重用相同的名称，同时在作用于外保留原始变量的可用性。
  
  通过在新的作用域（使用 <code>{}</code>）中封装闭包，我们可以在将变量移动到闭包中之前，进行克隆，并不要重新命名它们。
  <div style="columns: 2;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
    <div style="break-inside: avoid">
      <pre>
         let a = Arc::new([1, 2, 3]);
         let b = a.clone();
         thread::spawn(move || {
            dbg!(b);
         });
         dbg!(a);
      </pre>
    Arc 克隆存活在相同作用域范围内。每个线程都有自己的克隆，只是名称不同。
    </div>
    <div style="break-inside: avoid">
      <pre>
         let a = Arc::new([1, 2, 3]);
         thread::spawn({
             let a = a.clone();
             move || {
                 dbg!(a);
             }
         });
         dbg!(a);
      </pre>
    Arc 的克隆存活在不同的生命周期范围内。我们可以在每个线程使用相同的名称。
    </div>
  </div>
</div>

因为所有权是共享的，引用计数指针（`Rc<T>` 和 `Arc<T>`）与共享引用（`&T`）有着相同的限制。它们并不能让你对它们包含的值进行可变访问，因为该值在同一时间，可能被其它代码借用。

例如，如果我们尝试去排序 `Arc<[i32]>` 中整数的切片，编译器将阻止我们这么做，告诉我们不允许改变数据：

```txt
error[E0596]: cannot borrow data in an `Arc` as mutable
  |
6 |     a.sort();
  |     ^^^^^^^^
```

## 借用和数据竞争

## 内部可变性

### Cell

### RefCell

### 互斥锁[^4]和读写锁[^5]

### Atomic

### UnsafeCell

## 线程安全：Send 和 Sync

## 锁：互斥锁和读写锁

### Rust 的互斥锁

### 锁毒化

### 读写锁

## 等待: 阻塞（Park）和条件变量

### 线程阻塞

### 条件变量

## 总结

[^1]: <https://zh.wikipedia.org/zh-cn/監視器_(程序同步化)>
[^2]: <https://zh.wikipedia.org/wiki/互斥锁>
[^3]: <https://rustwiki.org/zh-CN/std/thread/fn.park.html>
[^4]: <https://zh.wikipedia.org/wiki/互斥锁>
[^5]: <https://zh.wikipedia.org/wiki/读写锁>
