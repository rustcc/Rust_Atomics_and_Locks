# 第一章：Rust 并发基础

（<a href="https://marabos.nl/atomics/basics.html" target="_blank">英文版本</a>）

早在多核处理器司空见惯之前，操作系统就允许一台计算机运行多个程序。这是通过在进程之间快速切换来完成的，允许每个进程逐个地逐次取得一点进展。现在，几乎所有的电脑，甚至手机和手表都有着多核处理器，可以真正并行执行多个程序。

操作系统尽可能的将进程之间隔离，允许程序完全意识不到其他线程在做什么的情况下做自己的事情。例如，在不先询问操作系统内核的情况下，一个进程通常不能获取其他进程的内存，或者以任意方式与之通信。

然而，一个程序可以产生额外的*执行线程*作为*进程*的一部分。同一进程中的线程不会相互隔离。线程共享内存并且可以通过内存相互交互。

这一章节将阐述在 Rust 中如何产生线程，并且关于它们的所有基本概念，例如如何安全地在多个线程之间共享数据。本章中解释的概念是本书其余部分的基础。

> 如果你已经熟悉 Rust 中的这些部分，你可以随时跳过。然而，在你继续下一章节之前，请确保你对线程、内部可变性、Send 和 Sync 有一个好的理解，以及知道什么是互斥锁[^2]、条件变量[^1]以及线程阻塞（park）[^3]。

## Rust 中的线程

（<a href="https://marabos.nl/atomics/basics.html#threads" target="_blank">英文版本</a>）

每个程序都从一个线程开始：主（main）线程。该线程将执行你的 main 函数，并且如果你需要，可以用它产生更多线程。

在 Rust 中，新线程使用来自标准库的 `std::thread::spawn` 函数产生。它接受一个参数：新线程执行的函数。一旦该函数返回，线程就会停止。

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

<div class="box">
  <h2 style="text-align: center;">Thread ID</h2>
  Rust 标准库为每个线程分配一个唯一的标识符。此标识符可以通过 <code>Thread::id()</code> 访问并且拥有 ThreadId 类型。除了复制 ThreadId 以及检查它们是否相等外，你什么也做不了。不能保证这些 ID 将会连续分配，而每个线程都会有所不同。
</div>

如果你运行我们上面的示例几次，你可能注意到输出在运行之间有所不同。一次特定的运行在机器上的输出：

```txt
Hello from the main thread.
Hello from another thread!
This is my thread id:
```

惊讶的是，部分输出似乎丢失了。

这里发生的情况是：新的线程完成其函数的执行之前，主线程完成了主函数的执行。

从主函数返回将退出整个程序，即使其它线程仍然在运行。

在这个特定的示例中，在程序被主线程关闭之前，其中一个新的线程只有够到达第二条消息一半的时间。

如果我们想要线程在主函数返回之前完成执行，我们可以通过 `join` 它们来等待。为此，我们使用 `spawn` 函数返回的 `JoinHandle`：

```rust
fn main() {
    let t1 = thread::spawn(f);
    let t2 = thread::spawn(f);

    println!("Hello from the main thread.");

    t1.join().unwrap();
    t2.join().unwrap();
}
```

`.join()` 方法等待直到线程结束执行并且返回 `std::thread::Result`。如果线程由于 panic 不能成功地完成它的函数，这将包含 panic 消息。我们试图去处理这种情况，或者为 join panic 的线程调用 `.unwrap()` 去 panic。

运行我们程序的这个版本，将不再导致输出被截断：

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

<div class="box">
  <h2 style="text-align: center;">输出锁定</h2>
  println 宏使用 <code>std::io::Stdout::lock()</code> 去确保输出没有被中断。<code>println!()</code> 表达式将等待直到任意并发的表达式运行完成后，再写入输出。如果不是这样，我们可能得到更多的交错输出：

  <pre>
  Hello fromHello from another thread!
  another This is my threthreadHello fromthread id: ThreadId!
  ( the main thread.
  2)This is my thread
  id: ThreadId(3)</pre>
</div>

与其将函数的名称传递给 `std::thread::spawn`（像我们上面的示例那样），不如传递一个*闭包*。这允许我们捕获值并移动到新的线程：

```rust
let numbers = vec![1, 2, 3];

thread::spawn(move || {
    for n in &numbers {
        println!("{n}");
    }
}).join().unwrap();
```

在这里，因为我们使用了 `move` 闭包，numbers 的所有权被转移到新产生的线程。如果我们没有使用 `move` 关键字，闭包将会通过引用捕获 numbers。这将导致一个编译错误，因为新的线程可能比变量的生命周期更长。

由于线程可能运行直到程序执行结束，因此产生的线程在它的参数类型上有 `'static` 生命周期绑定。换句话说，它只接受永久保留的函数。闭包通过引用捕获局部变量不能够永久保留，因为当局部变量不存在时，引用将变得无效。

从线程中取回一个值，是从闭包中返回值来完成的。该返回值可以通过 `join` 方法返回的 `Result` 中获取：

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

<div class="box">
  <h2 style="text-align: center;">Thread Builder</h2>
  <p><code>std::thread::spawn</code> 函数事实上仅是 <code>std::thread::Builder::new().spawn().unwrap()</code> 的简写。</p>

  <p><code>std::thread::Builder</code> 允许你在产生线程之前为新线程做一些配置。你可以使用它为新线程配置栈大小并给新线程一个名字。线程的名字是可以通过 <code>std::thread::current().name()</code> 获得，这将在 panic 消息中可用，并在监控和大多数调试工具中可见。</p>

  <p>此外，Builder 的产生函数返回一个 <code>std::io::Result</code>，允许你处理产生新线程失败的情况。如果操作系统内存不足，或者资源限制已经应用于你的程序，这是可能发生的。如果 <code>std::thread::spawn</code> 函数不能去产生一个新线程，它就会 panic。</p>
</div>

## 作用域内的线程

（<a href="https://marabos.nl/atomics/basics.html#scoped-threads" target="_blank">英文版本</a>）

如果我们确信生成的线程不会比某个作用域存活更久，那么线程可以安全地借用那些不会一直存在的东西，例如局部变量，只要它们比该范围活得更久。

Rust 标准库提供了 `std::thread::scope` 去产生此类*作用域内的线程*。它允许我们产生不超过我们传递给该函数闭包的范围的线程，这使它可能安全地借用局部变量。

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

这种模式保证了，在作用域产生的线程没有会比作用域更长的生命周期。因此，作用域中的 `spawn` 方法在它的参数类型中没有 `'static` 约束，允许我们去引用任何东西，只要它比作用域有更长的生命周期，例如 numbers。

在以上示例中，这两个线程并发地获取 numbers。这是没问题的，因为它们其中的任何一个（或者主线程）都没有修改它。如果我们改变第一个线程去修改 numbers，正如下面展示的，编译器将不允许我们也产生另一个也使用数字的线程：

```rust
let mut numbers = vec![1, 2, 3];

thread::scope(|s| {
    s.spawn(|| {
        numbers.push(1);
    });
    s.spawn(|| {
        numbers.push(2); // 报错！
    });
});
```

确切的错误信息取决于 Rust 编译器版本，因为它会在不断改进中以产生更好的诊断，但是试图去编译以上代码将导致以下问题：

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

<div class="box">
  <h2 style="text-align: center;">泄漏启示录</h2>
  <p>在 Rust 1.0 之前，标准库有一个函数叫做 <code>std::thread::scoped</code>，它将直接产生一个线程，就像 <code>std::thread::spawn</code>。它允许无 <code>'static</code> 的捕获，因为它返回的不是 JoinHandle，而是当被丢弃时 join 到线程的 JoinGuard。任意的借用数据仅需要比这个 JoinGuard 活得更久。只要 JoinGuard 在某个时候被丢弃，这似乎是安全的。</p>

  <p>就在 Rust 1.0 发布之前，人们慢慢发现它似乎不能保证某些东西一定被丢弃。有很多种方式不能丢弃它，例如创建一个引用计数节点的循环，可以忘记某些东西或者<i>泄漏</i>它。</p>

  <p>最终，在一些人提及的“泄漏启示录”中得到结论，（安全）接口的设计不能依赖假设对象总是在它们的生命周期结束后丢弃。泄漏一个对象可能会导致泄漏更多对象（例如，泄漏一个 Vec 将也导致泄漏它的元素），但它并不会导致未定义行为（undefind behavior）。因此，<code>std::thread::scoped</code> 将不再视为安全的并从标准库移除。此外，<code>std::mem::forget</code> 从一个不安全的函数升级到<i>安全</i>的函数，以强调忘记（或泄漏）总是一种可能性。</p>

  <p>直到后来，在 Rust 1.63 中，添加了一个新的 <code>std::thread::scope</code> 功能，其新设计不依赖 Drop 来获得正确性。</p>
</div>

## 共享所有权以及引用计数

（<a href="https://marabos.nl/atomics/basics.html#shared-ownership-and-reference-counting" target="_blank">英文版本</a>）

目前，我们已经使用了 `move` 闭包（[“Rust 中的线程”](#rust-中的线程)）将值的所有权转移到线程并从生命周期较长的父线程借用数据（[作用域内的线程](#作用域内的线程)）。当两个线程之间共享数据，它们之间的任何一个线程都不能保证比另一个线程的生命周期长，那么它们都不能称为该数据的所有者。它们之间共享的任何数据都需要与最长生命周期的线程一样长。

### 静态值（static）

（<a href="https://marabos.nl/atomics/basics.html#statics" target="_blank">英文版本</a>）

有几种方式去创建不属于单线程的东西。最简单的方式是**静态**值，它由整个程序“拥有”，而不是单个线程。在以下示例中，这两个线程都可以获取 X，但是它们并不拥有它：

```rust
static X: [i32; 3] = [1, 2, 3];

thread::spawn(|| dbg!(&X));
thread::spawn(|| dbg!(&X));
```

静态值一般由一个常量初始化，它从不会被丢弃，并且甚至在程序的主线程开始之前就已经存在。每个线程都可以借用它，因为可以保证它它总是存在。

### 泄漏（Leak）

（<a href="https://marabos.nl/atomics/basics.html#leaking" target="_blank">英文版本</a>）

另一种方式是通过*泄漏*内存分配的方式共享所有权。使用 `Box::leak`，人们可以释放 `Box` 的所有权，保证永远不会丢弃它。从那时起，`Box` 将永远存在，没有所有者，只要程序运行，任意线程都可以借用它。

```rust
let x: &'static [i32; 3] = Box::leak(Box::new([1, 2, 3]));

thread::spawn(move || dbg!(x));
thread::spawn(move || dbg!(x));
```

`move` 闭包可能会让它看起来像我们移动所有权进入线程，但仔细观察 x 的类型就会发现，我们只是给线程一个对数据的*引用*。

> 引用是 `Copy` 的，这意味着当你“移动”（move）它们的时候，原始内容仍然存在，这就像整数或者布尔内容一样。

注意，`'static` 生命周期并不意味着该值自程序开始时就存在，而只是意味着它一直存在到程序的结束。过去并不重要。

泄漏 `Box` 的缺点是我们正在泄漏内存。我们获取一些内存，但是从未丢弃和释放它。如果仅发生有限的次数，这就可以了。但是如果我们继续这样做，程序将慢慢地耗尽内存。

### 引用计数

（<a href="https://marabos.nl/atomics/basics.html#arc" target="_blank">英文版本</a>）

为了确保共享数据能够丢弃和释放内存，我们不能完全放弃它的所有权。相反，我们可以*分享所有权*。通过跟踪所有者的数量，我们确保仅当没有所有者时，才会丢弃该值。

Rust 标准库通过 `std::rc::Rc` 类型提供了该功能，它是“引用计数”（reference counted）的缩写。它与 `Box` 非常类似，唯一的区别是克隆它将不会分配任何新内存，而是递增存储在包含值旁边的计数器。原始的 `Rc` 和克隆的 `Rc` 将引用相同的内存分配；它们*共享所有权*。

```rust
use std::rc::Rc;

let a = Rc::new([1, 2, 3]);
let b = a.clone();

assert_eq!(a.as_ptr(), b.as_ptr()); // 相同内存分配！
```

丢弃一个 `Rc` 将递减计数。只有最后一个 `Rc`，计数器下降到 0，才会丢弃且释放内存分配中所包含的数据。

如果我们尝试去发送一个 Rc 到另一个线程，然而，我们将造成以下的编译错误：

```txt
error[E0277]: `Rc` cannot be sent between threads safely
    |
8   |     thread::spawn(move || dbg!(b));
    |                   ^^^^^^^^^^^^^^^
```

事实证明，`Rc` 不是*线程安全*的（详见，[线程安全：Send 和 Sync](#线程安全send-和-sync)）。如果多个线程有相同内存分配的 `Rc`，那么它们可能尝试并发修改引用计数，这可能产生不可预测的结果。

然而，我们可以使用 `std::sync::Arc`，它代表“原子引用计数”。它与 `Rc` 相同，只是它保证了对引用计数的修改是不可分割的*原子*操作，因此可以安全地与多个线程使用。（详见[第二章](./2_Atomics.md)。）

```rust
use std::sync::Arc;

let a = Arc::new([1, 2, 3]); // 1
let b = a.clone(); // 2

thread::spawn(move || dbg!(a)); // 3
thread::spawn(move || dbg!(b)); // 3
```

1. 我们在新的内存分配中放置了一个数组，以及从一开始的引用计数器。
2. 克隆 Arc 递增引用计数到 2，并为我们提供第二个指向相同内存分配的 Arc。
3. 两个线程通过各自的 Arc 访问共享的数组。当它们丢弃 Arc 时，两者都会递减引用计数。最后一个丢弃 Arc 的线程将看见计数器递减到 0，并且将丢弃和回收数组的内存。

<div class="box">
  <h2 style="text-align: center;">命名克隆</h2>
  <p>如果给每个 Arc 的克隆取一个不同的名称，这可能使得代码变得混乱难以追踪。尽管每个 Arc 的克隆都是一个独立的对象，而给每个克隆赋予不同的名称也并不能很好地反映这一点。</p>

  <p>Rust 允许（并且鼓励）你通过定义有着新的名称的相同变量去<i>遮蔽</i>变量。如果你在同一作用域这么做，则无法再命名原始变量。但是通过打开一个新的作用域，可以使用类似 <code>let a = a.clone();</code> 的语句在该作用域内重用相同的名称，同时在作用于外保留原始变量的可用性。</p>
  
  <p>通过在新的作用域（使用 <code>{}</code>）中封装闭包，我们可以在将变量移动到闭包中之前，进行克隆，而不重新命名它们。</p>
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
    Arc 克隆存活在同一作用域内。每个线程都有自己的克隆，只是名称不同。
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
    Arc 的克隆存活在不同的作用域内。我们可以在每个线程使用相同的名称。
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

（<a href="https://marabos.nl/atomics/basics.html#borrowing-and-races" target="_blank">英文版本</a>）

在 Rust 中，可以使用两种方式借用值。

* *不可变借用*
  * 使用 `&` 借用会得到一个*不可变借用*。这样的引用可以被复制。对于它引用访问的数据在所有引用副本之间是共享的。顾名思义，编译器通常不允许你通过这样的引用改变数据，因为那可能会影响当前引用相同数据的其它代码。
* *可变借用*
  * 使用 `&mut` 借用会得到一个*可变引用*。可变借用保证了它是该数据的唯一激活的借用。这确保了可变的数据将不会改变任何其它代码正在查看的数据。

这两个概念一起，完全阻止了*数据竞争*：一个线程正在改变数据，而另一个线程正在并发地访问数据的情况。数据竞争通常是*未定义行为*[^6]，这意味着编译器不需要考虑这些情况。它只是假设它们并不会发生。

为了清晰地表达这个意思，让我们来看一看编译器可以使用借用规则作出有用假设的示例：

```rust
fn f(a: &i32, b: &mut i32) {
    let before = *a;
    *b += 1;
    let after = *a;
    if before != after {
        x(); // 从不发生
    }
}
```

这里，我们得到一个整数的不可变引用，并在递增 b 所引用的整数之前和之后存储整数的值。编译器可以自由地假设关于借用和数据竞争的基本规则得到了遵守，这意味着 b 不可能引用与 a 相同的整数。实际上，在对 a 进行引用时，整个程序中没有任何地方对 a 借用的整数进行可变借用。因此，编译器可以轻松地推断 `*a` 不会发生变化，并且 `if` 语句将永远不是 true，并且可以作为优化完全地删除 x 调用。

除了使用不安全的块（`unsafe`）禁止一些编译器的安全检查之外，不可能写出打破编译器假设的 Rust 程序。

<div class="box">
  <h2 style="text-align: center;">未定义行为</h2>
  <p>类似 C、C++ 和 Rust 都有一套需要遵守的规则，以避免未定义行为。例如，Rust 的规则之一是，对任何对象的可变引用永远不可能超过一个。</p>

  <p>在 Rust 中，仅当使用 unsafe 代码块才能打破这些规则。“unsafe”并不意味着代码是错误的或者不安全的，而是编译器并没有为你验证你的代码是安全的。如果代码确实违法了这些规则，则称为不健全的（unsound）。</p>

  <p>允许编译器在不检查的情况下假设这些规则从未破坏。当破坏是，这将导致叫做未定义行为的问题，我们需要不惜一切代价去避免。如果我们允许编译器作出与实际不符的假设，那么它可能很容易导致关于代码不同部分更错误的结论，影响你整个程序。</p>

  <p>作为一个具体的例子，让我们看看在切片上使用 <code>get_unchecked</code> 方法的小片段：</p>

  <pre>let a = [123, 456, 789];
let b = unsafe { a.get_unchecked(index) };</pre>

  <p><code>get_unchecked</code> 方法给我们一个给定索引的切片元素，就像 <code>a[index]</code>，但是允许编译器假设索引总是在边界，没有任何检查。</p>

  <p>这意味着，在代码片段中，由于 a 的长度是 3，编译器可能假设索引小于 3。这使我们确保其假设成立。</p>

  <p>如果我们破坏了这个假设，例如，我们以等于 3 的索引运行，任何事情都可能发生。它可能导致读取 a 之后存储的任何内存内容。这可能导致程序崩溃。它可能会执行程序中完全无关的部分。它可能会引起各种糟糕的情况。</p>

  <p>或许令人惊讶的是，<a href="https://en.wikipedia.org/wiki/Speculative_execution">未定义行为甚至可以“时间回溯”，导致之前的代码出问题</a>。要理解这种情况是如何发生的，想象我们上面的片段有一个 match 语句，如下：</p>

  <pre>match index {
   0 => x(),
   1 => y(),
   _ => z(index),
}

let a = [123, 456, 789];
let b = unsafe { a.get_unchecked(index) };</pre>

  <p>由于不安全的代码，编译器被允许假设 index 只有 0、1 或 2。它可能会逻辑的得出结论，我们的 match 语句的最后分支仅会匹配到 2，因此 z 仅会调用为 <code>z(2)</code>。这个结论不仅可以优化匹配，还可以优化 z 本身。这可能包括丢弃代码中未使用的部分。</p>

  <p>如果我们以 3 为 index 执行此设置，我们的程序可能会尝试执行被优化的部分，导致在我们到达最后一行的 unsafe 块之前就出现不可预测的行为。就像这样，未定义行为通过整个程序向后或者向前传播，而这往往是以非常出乎意料的方式发生。</p>

  <p>当调用任何的不安全函数时，仔细阅读其文档，确保你完全理解它的<i>安全要求</i>：作为调用者，你需要维持的假设，以避免未定义行为。</p>
</div>

## 内部可变性

（<a href="https://marabos.nl/atomics/basics.html#interior-mutability" target="_blank">英文版本</a>）

上一节介绍的借用规则可能非常有限——尤其涉及多个线程时。遵循这些规则在线程之间通信极其有限，并且是不可能的，因为多个线程访问的数据都无法改变。

幸运的是，有一个逃生方式：<i>内部可变性</i>。有着内部可变性的数据类型略微改变了借用规则。在某些情况下，这些类型可以使用“不可变”的引用进行可变。

在[“引用计数”](#引用计数)中，我们已经看到一个设计内部可变性的微妙示例。在 `Rc` 和 `Arc` 都变为引用计数器，即使可能有多个克隆都使用相同的引用计数器。

一旦设计内部可变性类型，称“不可变”和“可变”将变得混乱和不准确，因为一些类型可以通过两者变得可变。更准确的称呼是“共享”和“独占”：共享引用（`&T`）可以被复制以及与其它引用共享，然而*独占引用*（`&mut T`）保证了仅有一个对 T 的独占借用。对于大多数类型，共享引用并不允许可变，但有一些例外。由于本书我们将主要处理这些例外情况，我们将在这本书的剩余内容中使用更准确的术语。

> 请记住，内部可变性仅会影响共享借用的规则，以便在共享时允许可变。它不能改变任意关于独占借用的规则。独占借用仍然保证没有任意激活的借用。导致超过一个活动的独占引用的不安全代码总是涉及未定义行为，不管内部可变性如何。

让我们看一看有着内部可变性的一些示例，以及如何通过共享引用允许可变性而不导致未定义行为。

### Cell

（<a href="https://marabos.nl/atomics/basics.html#cell" target="_blank">英文版本</a>）

`std::cell::Cell<T>` 仅是包装了 T，但允许通过共享引用进行可变。为避免未定义行为，它仅允许你将值复制出来（如果 T 实现 Copy）或者将其替换为另一个整体值。此外，它仅用于单个线程。

让我们看一看与上一节相似的示例，但是这一次使用 `Cell<i32>` 而不是 `i32`：

```rust
use std::cell::Cell;

fn f(a: &Cell<i32>, b: &Cell<i32>) {
    let before = a.get();
    b.set(b.get() + 1);
    let after = a.get();
    if before != after {
        x(); // 可能发生
    }
}
```

与上次不同，现在 if 条件有可能为真。因为 `Cell<i32>` 是内部可变的，只要我们有对它的共享引用，编译器不再假设它的值不再改变。a 和 b 可能引用相同的值，通过 b 也可能影响 a。然而，它可能假设没有其它线程并发获取 cell。

对 Cell 的限制并不总是容易处理的。因为它不能直接让我们借用它所持有的值，我们需要将值移动出去（让一些东西替换它的位置），修改它，然后将它放回去，以改变它的内容：

```rust
fn f(v: &Cell<Vec<i32>>) {
    let mut v2 = v.take(); // 使用空的 Vec 替换 Cell 中的内容
    v2.push(1);
    v.set(v2); // 将修改的 Vec 返回
}
```

### RefCell

（<a href="https://marabos.nl/atomics/basics.html#refcell" target="_blank">英文版本</a>）

与常规的 Cell 不同的是，`std::cell::RefCell` 允许你以很小的运行时花费，去借用它的内容。`RefCell<T>` 不仅持有 T，同时也持跟踪任何未解除的借用。如果你尝试在已可变借用时尝试借用它（或反之亦然），它会引发 panic，以避免出现未定义行为。就像 Cell，RefCell 只能在单个线程中使用。

借用 RefCell 的内容通过调用 `borrow` 或者 `borrow_mut` 完成：

```rust
use std::cell::RefCell;

fn f(v: &RefCell<Vec<i32>>) {
    v.borrow_mut().push(1); // 我们可以直接修改 `Vec`。
}
```

尽管 Cell 和 RefCell 有时是非常有用的，但是当我们使用多线程的时候，它们会变得无用。所以让我们继续讨论与并发相关的类型。

### 互斥锁和读写锁

（<a href="https://marabos.nl/atomics/basics.html#mutex-and-rwlock" target="_blank">英文版本</a>）

*读写锁*（RwLock）[^5]是 `RefCell` 的并发版本。`RwLock<T>` 持有 T 并且跟踪任意未解除的借用。然而，与 RefCell 不同，它在冲突的借用中不会 panic。相反，它会阻塞当前线程——使它进入睡眠——直到冲突的借用消失才会唤醒。在其它线程完成后，我们仅需要耐心的等待轮到我们处理数据。

借用 RwLock 的内容称为*锁*。通过锁定它，我们临时阻塞并发的冲突借用，这允许我们没有导致数据竞争的借用它。

`Mutex`[^4] 与其是非常相似的，但是概念上相对简单的。它不像 RwLock 跟踪共享借用和独占借用的数量，它仅允许独占借用。

我们将在[“锁：互斥锁和读写锁”](#锁互斥锁和读写锁)更详细地介绍这些类型。

### Atomic

（<a href="https://marabos.nl/atomics/basics.html#atomics" target="_blank">英文版本</a>）

原子类型表示 Cell 的并发版本，是第 [2](./2_Atomics.md) 章和第 [3](./3_Memory_Ordering.md) 章的主题。与 Cell 相同，它们通过将整个值进行复制来避免未定义行为，而不直接让我们借用内容。

与 Cell 不同的是，它们不能是任意大小的。因此，任何 T 都没有通用的 `Atomic<T>` 类型，但仅有特定的原子类型，例如 `AtomicU32` 和 `AtomicPtr<T>`。哪些可用取决于平台，因为它们需要处理器的支持来避免数据竞争。（我们将在[第七章](./7_Understanding_the_Processor.md)研究这个问题。）

因为它们的大小非常有限，原子类型通常不直接在线程之间共享所需的信息。相反，它们通常用作工具，是线程之间共享其它（通常是更大的）东西作为可能。当原子用于表示其它数据时，情况可能变得令人意外地复杂。

### UnsafeCell

（<a href="https://marabos.nl/atomics/basics.html#unsafecell" target="_blank">英文版本</a>）

`UnsafeCell` 是内部可变性的原始基石。

`UnsafeCell<T>` 包装 T，但是没有附带任何条件和限制来避免未定义行为。相反，它的 `get()` 方法仅是给出了它包装值的原始指针，该值仅可以在 `unsafe` 块中使用。它以用户不会导致任何未定义行为的方式使用它。

更常见的是，不会直接使用 UnsafeCell，而是将它包装在另一个类型，通过限制接口提供安全，例如 `Cell` 和 `Mutex`。所有有着内部可变性的类型——包括所有以上讨论的类型都建立在 UnsafeCell 之上。

## 线程安全：Send 和 Sync

（<a href="https://marabos.nl/atomics/basics.html#thread-safety" target="_blank">英文版本</a>）

在这一章节中，我们已经看见一个不是*线程安全*的类型，这些类型仅用于一个单线程，例如 `Rc`、`Cell` 以及其它。由于需要这些限制来避免未定义行为，所以编译器需要理解并为你检查这个限制，这样你就可以在不使用 unsafe 块的情况下使用这些类型。

该语言使用两种特殊的 trait 以跟踪这些类型可以安全地用作交叉线程：

* *Send*
  * 如果可以发送到另一个线程，则是 `Send` 类型。换句话说，是否该类型值的所有权可以转移到另一个线程。例如，`Arc<i32>` 是 `Send`，而 `Rc<i32>` 不是。
* *Sync*
  * 如果可以共享到另一个线程，则是 `Sync` 类型。换句话说，当且仅当对该类型的共享引用 `&T` 是 `Send` 的，类型 T 是 `Sync`。例如，i32 是 `Sync`，而 `Cell<i32>` 不是。（然而 `Cell<i32>` 是 `Send` 的）

原始类型，例如 i32、bool 以及 str 都是 `Send` 和 `Sync`。

这两个 trait 会自动地为你实现 trait，这意味着他们会基于它们的字段为你的类型自动地实现。带有全部 `Send` 和 `Sync` 的结构体字段，本身也是 `Send` 和 `Sync`。

选择退出其中任何一种的方式是去**增加**没有实现该 trait 的字段到你的类型。为此，特殊的 `std::marker::PhantomData<T>` 类型经常派上用场。实际上它在运行时并不存在，它会被被编译器视为 T。它是零开销类型，不占用任何空间。

让我们来看看以下的结构体：

```rust
use std::marker::PhantomData;

struct X {
    handle: i32,
    _not_sync: PhantomData<Cell<()>>,
}
```

在这个示例中，如果 `handle` 是它唯一的字段，`X` 将是 `Send` 和 `Sync`。然而，我们增加一个零开销的 `PhantomData<Cell<()>>` 字段，该字段被视为 `Cell<()>`。因为 `Cell<()>` 字段不是 Sync，X 也将不是。但它仍然是 Send，因为所有字段都实现了 Send。

原始指针（`*const T` 和 `*mut T`）既不是 Send 也不是 Sync，因为编译器不了解他们表示什么。

选择任意 trait 的方式和使用任意其它 trait 相同；使用一个 impl 为你的类型实现 trait：

```rust
struct X {
    p: *mut i32,
}

unsafe impl Send for X {}
unsafe impl Sync for X {}
```

注意，实现这些 trait 需要 `unsafe` 关键字，因为编译器不能为你检查它是否正确。这是你对编译器作出的承诺，你不得不信任它。

如果你尝试去移动一些未实现 Send 的值进入另一个线程，编译器将阻止你这样做。用一个小的示例去演示：

```rust
fn main() {
    let a = Rc::new(123);
    thread::spawn(move || { // 报错！
        dbg!(a);
    });
}
```

这里，我们尝试去发送 `Rc<i32>` 到一个新线程，但是 `Rc<i32>` 与 `Arc<i32>` 不同，因为它没有实现 Send。

如果我们尝试去编译以上示例，我们将面临一个类似这样的错误：

```txt
error[E0277]: `Rc<i32>` cannot be sent between threads safely
   --> src/main.rs:3:5
    |
3   |     thread::spawn(move || {
    |     ^^^^^^^^^^^^^ `Rc<i32>` cannot be sent between threads safely
    |
    = help: within `[closure]`, the trait `Send` is not implemented for `Rc<i32>`
note: required because it's used within this closure
   --> src/main.rs:3:19
    |
3   |     thread::spawn(move || {
    |                   ^^^^^^^
note: required by a bound in `spawn`
```

`thread::spawn` 函数需要它的参数实现 Send，并且只有当其所有的捕获都是 Send，闭包才是 Send。如果我们尝试捕获未实现 Send，就会捕捉我们的错误，保护我们避免未定义行为的影响。

## 锁：互斥锁和读写锁

（<a href="https://marabos.nl/atomics/basics.html#mutexes" target="_blank">英文版本</a>）

在线程之间共享（可变）数据更常规的有用工具是 `mutex`，它是“互斥”（mutual exclusion）的缩写。mutex 的工作是通过暂时阻塞其它试图同时访问某些数据的线程，来确保线程对某些数据进行独占访问。

概念上，mutex 仅有两个状态：锁定和解锁。当线程锁定一个未解锁的 mutex，mutex 被标记为锁定，线程可以立即继续。当线程尝试锁定一个已上锁的 mutex，操作将*阻塞*。当线程等待 mutex 解锁时，其会置入睡眠状态。解锁仅能在锁定的 mutex 上进行，并且应当由锁定它的同一线程完成。如果其它线程正在等待锁定 mutex，解锁将导致唤醒其中一个线程，因此它可以尝试再次锁定 mutex 并且继续它的过程。

使用 mutex 保护数据仅是所有线程之间的约定，当它们持有 mutex 锁定时，它们才能获取数据。这种方式，没有两个线程可以并发地获取数据和导致数据竞争。

### Rust 的互斥锁

（<a href="https://marabos.nl/atomics/basics.html#rusts-mutex" target="_blank">英文版本</a>）

Rust 的标准库通过 `std::sync::Mutex<T>` 提供这个功能。它对类型 T 进行范型化，该类型 T 是 mutex 所保护的数据类型。通过将 T 作为 mutex 的一部分，该数据仅可以通过 mutex 获取，从而提供一个安全的接口，以保证所有线程都遵守这个约定。

为确保锁定的 mutex 仅通过锁定它的线程解锁，所以它没有 `unlock()` 方法。然而，它的 `lock()` 方法返回一个称为 `MutexGuard` 的特殊类型。该 guard 表示保证我们已经锁定 mutex。它通过 `DerefMut` trait 行为表现像一个独占引用，使我们能够独占访问互斥体保护的数据。解锁 mutex 通过丢弃 guard 完成。当我们丢弃 guard 时，我们我们放弃了获取数据的能力，并且 guard 的 `Drop` 实现将解锁 mutex。

让我们看一个示例，实践中的 mutex：

```rust
use std::sync::Mutex;

fn main() {
    let n = Mutex::new(0);
    thread::scope(|s| {
        for _ in 0..10 {
            s.spawn(|| {
                let mut guard = n.lock().unwrap();
                for _ in 0..100 {
                    *guard += 1;
                }
            });
        }
    });
    assert_eq!(n.into_inner().unwrap(), 1000);
}
```

在这里，我们有一个 `Mutex<i32>`，一个保护整数的 mutex，并且我们启动了十个线程，每个线程会递增这个整数 100 次。每个线程将首先锁定 mutex 去获取 MutexGuard，并且然后使用 guard 去获取整数并修改它。当该变量超出作用域后，guard 会立即隐式丢弃。

线程完成后，我们可以通过 `into_inner()` 安全地从整数中移除保护。`into_inner` 方法获取 mutex 的所有权，这确保了没有其它东西可以引用 mutex，从而使 mutex 变得不再必要。

尽管递增是逐步地的，但是线程仅能够看见 100 的倍数，因为它只能在 mutex 解锁时查看整数。实际上，由于 mutex 的存在，这一百次递增称为了一个单一不可分割的原子操作。

为了清晰地看见 mutex 的效果，我们可以让每个线程在解锁 mutex 之前等待一秒：

```rust
use std::time::Duration;

fn main() {
    let n = Mutex::new(0);
    thread::scope(|s| {
        for _ in 0..10 {
            s.spawn(|| {
                let mut guard = n.lock().unwrap();
                for _ in 0..100 {
                    *guard += 1;
                }
                thread::sleep(Duration::from_secs(1)); // 新增！
            });
        }
    });
    assert_eq!(n.into_inner().unwrap(), 1000);
}
```

当你现在运行程序，你将看见大约需要花费 10s 才能完成。每个线程仅等待 1s，但是 mutex 确保一次仅有一个线程这么做。

如果我们在睡眠 1s 之前丢弃 guard，并且因此解锁 mutex，我们将看到并行发生：

```rust
fn main() {
    let n = Mutex::new(0);
    thread::scope(|s| {
        for _ in 0..10 {
            s.spawn(|| {
                let mut guard = n.lock().unwrap();
                for _ in 0..100 {
                    *guard += 1;
                }
                drop(guard); // 新增：在睡眠之前丢弃 guard！
                thread::sleep(Duration::from_secs(1));
            });
        }
    });
    assert_eq!(n.into_inner().unwrap(), 1000);
}
```

有了这些变化，这个程序大约仅需要 1s，因为 10 个线程现在可以同时执行 1s 的睡眠。这表明了 mutex 锁定时间保持尽可能短的重要性。将 mutex 锁定时间超过必要时间可能会完全抵消并行带来的好处，实际上会强制所有操作按顺序执行。

### 锁中毒（posion）

（<a href="https://marabos.nl/atomics/basics.html#lock-poisoning" target="_blank">英文版本</a>）

上述示例中 `unwarp()` 调用和*锁中毒*有关。

当线程在持有锁时 panic，Rust 中的 mutex 将被标记为*中毒*。当这种情况发生时，Mutex 将不再被锁定，但调用它的 `lock` 方法将导致 `Err`，以表明它已经中毒。

这是一个防止由 mutex 保护的数据处于不一致状态的机制。在我们上面的示例中，如果一个线程将在整数递增不到 100 之后崩溃，mutex 将解锁并且整数将处于一个意外的状态，它不再是 100 的倍数，这可能打破其它线程的假设。在这种情况下，自动标记 mutex 中毒，强制用户处理这种可能。

在中毒的 mutex 上调用 `lock()` 仍然可能锁定 mutex。由 `lock()` 返回的 Err 包含 `MutexGuard`，允许我们在必要时纠正不一致的状态。

虽然锁中毒是一种强大的机制，在实践中，从潜在的不一致状态恢复并不常见。如果锁中毒，大多数代码要么忽略了中毒或者使用 `unwrap()` 去 panic，这有效地将 panic 传递给使用 mutex 的所有用户。

<div class="box">
  <h2 style="text-align: center;">MutexGuard 的生命周期</h2>
  <p>尽管隐式丢弃 guard 解锁 mutex 很方便，但是它有时会导致微妙的意外。如果我们使用 let 语句授任 guard 一个名字（正如我们上面的示例），看它什么时候会被丢弃相对简单，因为局部变量定义在它们作用域的末尾。然而，正如上述示例所示，不明确地丢弃 guard 可能导致 mutex 锁定的时间超过所需时间。</p>

  <p>在不给它指定名称的情况下使用 guard 也是可能的，并且有时非常方便。因为 MutexGuard 保护数据的行为像独占引用，我们可以直接使用它，而无需首先为他授任一个名称。例如，你有一个 <code>Mutex&lt;Vec&lt;i32&gt;&gt;</code>，你可以在单个语句中锁定 mutex，将项推入 Vec，并且再次锁定 mutex：</p>
  
  <pre>list.lock().unwrap().push(1);</pre>

  <p>任何更大表达式产生的临时值，例如通过 <code>lock()</code> 返回的 guard，将在语句结束后被丢弃。尽管这似乎显而易见，但它导致了一个常见的问题，这通常涉及 <code>match</code>、<code>if let</code> 以及 <code>while let</code> 语句。以下是遇到该陷阱的示例：</p>

  <pre>if let Some(item) = list.lock().unwrap().pop() {
    process_item(item);
}</pre>

  <p>如果我们的旨意就是锁定 list、弹出 item、解锁 list 然后在解锁 list 后处理 item，我们在这里犯了一个微妙而严重的错误。临时的 guard 直到完整的 <code>if let</code> 语句结束后才能被丢弃，这意味着我们在处理 item 时不必要地持有锁。</p>

  或许，意外地是，对于类似地 <code>if</code> 语句，这并不会发生，例如以下示例：

  <pre>if list.lock().unwrap().pop() == Some(1) {
    do_something();
}</pre>
  
  <p>在这里，临时的 guard 在 <code>if</code> 语句的主体执行之前就已经丢弃了。该原因是，通常 if 语句的条件总是一个布尔值，它并不能借用任何东西。没有理由将临时的生命周期从条件开始延长到语句的结尾。对于 <code>if let</code> 语句，情况可能并非如此。例如，如果我们使用 <code>front()</code>，而不是 <code>pop()</code>，项将会从 list 中借用，因此有必要保持 guard 存在。因为借用检查实际上只是一种检查，它并不会影响何时以及什么顺序丢弃，所以即使我们使用了 <code>pop()</code>，情况仍然是相同的，尽管那并不是必须的。</p>

  <p>我们可以通过将弹出操作移动到单独的 let 语句来避免这种情况。然后在该语句的末尾放下 guard，在 <code>if let</code> 之前：</p>

  <pre>let item = list.lock().unwrap().pop();
if let Some(item) = item {
    process_item(item);
}</pre>
</div>

### 读写锁

（<a href="https://marabos.nl/atomics/basics.html#reader-writer-lock" target="_blank">英文版本</a>）

互斥锁仅涉及独占访问。MutexGuard 将提供受保护数据的一个独占引用（`&mut T`），即使我们仅想要查看数据，并且共享引用（`&T`）就足够了。

读写锁是一个略微更复杂的 mutex 版本，它能够区分独占访问和共享访问的区别，并且可以提供两种访问方式。它有三种状态：解锁、由单个 *writer* 锁定（用于独占访问）以及由任意数量的 reader 锁定（用于共享访问）。它通常用于通常由多个线程读取的数据，但只是偶尔一次。

Rust 标准库通过 `std::sync::RwLock<T>` 类型提供该锁。它与标准库的 Mutex 工作类似，只是它的接口大多是分成两个部分。然而，单个 `lock()` 方法，它有 `read()` 和 `write()` 方法，用于为 reader 或 writer 进行锁定。它还附带了两种守卫类型，一种用于 reader，一种用于 writer：RwLockReadGuard 和 RwLockWriteGuard。前者只实现了 Deref，其行为像受保护数据共享引用，后者还实现了 DerefMut，其行为像独占引用。

它实际上是 `RefCell` 的多线程版本，动态地跟踪引用的数量以确保借用规则得到维护。

`Mutex<T>` 和 `RwLock<T>` 都需要 T 是 Send，因为它们可能发送 T 到另一个线程。除此之外，`RwLock<T>` 也需要 T 实现 Sync，因为它允许多个线程对受保护的数据持有共享引用（`&T`）。（严格地说，你可以创建一个并没有实现这些需求 T 的锁定，但是你不能在线程之间共享它，因为锁本身并没有实现 Sync）。

Rust 标准库仅提供一种通用的 `RwLock` 类型，但它的实现依赖于操作系统。读写锁之间有很多细微差别。当有 writer 等待时，即使当锁已经读取锁定时，很多实现将阻塞新的 reader。这样做是为了防止 *writer 挨饿*，在这种情况下，很多 reader 将共同持有锁而导致锁从不解锁，从而不允许任何 writer 更新数据。

<div class="box">
  <h2 style="text-align: center;">在其他语言中的互斥锁</h2>
  <p>Rust 标准的 Mutex 和 RwLock 类型与你在其它语言（例如 C、C++）发现的看起来有一点不同。</p>

  <p>最大的区别是，Rust 的 <code>Mutex&lt;T&gt;</code> 数据<i>包含</i>它正在保护的数据。例如，在 C++ 中，<code>std::mutex</code> 并不包含着它保护的数据，甚至不知道它在保护什么。这意味着，用户有指责记住哪些数据由 mutex 保护，并且确保每次访问“受保护”的数据都锁定正确的 mutex。注意，当读取其它语言涉及到 mutex 的代码，或者与不熟悉 Rust 程序员沟通时，非常有用。Rust 程序员可能讨论关于“数据在 mutex 之中”，或者说“mutex 中包装数据”这类话，这可能让只熟悉其它语言 mutex 的程序员感到困惑。</p>

  <p>如果你真的需要一个不包含任何内容的独立 mutex，例如，保护一些外部硬件，你可以使用 <code>Mutex&lt;()&gt;</code>。但即使是这种情况，你最好定义一个（可能 0 大小开销）的类型来与该硬件对接，并将其包装在 Mutex 之中。这样，在与硬件交互之前，你仍然可以强制锁定 mutex。</p>
</div>

## 等待: 阻塞（Park）和条件变量

（<a href="https://marabos.nl/atomics/basics.html#waiting" target="_blank">英文版本</a>）

当数据由多个线程更改时，在许多情况下，它们需要等待一些事件，以便管有数据的某些条件变为真。例如，如果我们有一个保护 Vec 的 mutex，我们可能想要等待直到它包含任何东西。

尽管 mutex 允许线程等待直到它解锁，但它不提供等待任何其它条件的功能。如果我们只拥有一个 mutex，我们不得不持有锁定的 mutex，以反复检查 Vec 中是否有任意东西。

### 线程阻塞

（<a href="https://marabos.nl/atomics/basics.html#parking" target="_blank">英文版本</a>）

一种方式是去等待来自另一个线程的通知，其被称为*线程阻塞*。一个线程可以阻塞它自己，将它置入睡眠状态，阻止它消耗任意 CPU 周期。然后，另一个线程可以解锁阻塞的线程，将其从睡眠中唤醒。

线程阻塞可以通过 `std::thread::park()` 函数获得。对于解锁，你可以在 `Thread` 对象中调用 `unpark()` 函数表示你想要解锁该线程。这样的对象可以通过 spawn 返回的 join 句柄获得，或者也可以通过 `std::thread::current()` 从线程本身中获得。

让我们深入研究在线程之间使用 mutex 共享队列的示例。在以下示例中，一个新产生的线程将消费来自队列的项，尽管主线程将每秒插入新的项到队列。线程阻塞被用于在队列为空时使消费线程等待。

```rust
use std::collections::VecDeque;

fn main() {
    let queue = Mutex::new(VecDeque::new());

    thread::scope(|s| {
        // 消费线程
        let t = s.spawn(|| loop {
            let item = queue.lock().unwrap().pop_front();
            if let Some(item) = item {
                dbg!(item);
            } else {
                thread::park();
            }
        });

        // 产生线程
        for i in 0.. {
            queue.lock().unwrap().push_back(i);
            t.thread().unpark();
            thread::sleep(Duration::from_secs(1));
        }
    });
}
```

消费线程运行一个无限循环，它将项弹出队列，使用 `dbg` 宏展示它们。当队列为空的时候，它停止并且使用 `park()` 函数进行睡眠。如果它得到解锁，`park()` 调用将返回，循环继续，再次从队列中弹出项，直到它是空的。等等。

生产线程将其推入队列，每秒产生一个新的数字。每次递增一个项时，它都会在 Thread 对象上使用 `unpark()` 方法，该方法引用消费线程来解锁它。这样，消费线程就会被唤醒处理新的元素。

需要注意的一点是，即使我们移除**阻塞**，这个程序在理论上仍然是正确的，尽管效率低下。这是重要的，因为 `park()` 不能保证它将由于匹配 `unpark()` 而返回。尽管有些罕见，但它很可能会有*虚假唤醒*。我们的示例处理得很好，因为消费线程会锁定队列，可以看到它是空的，然后直接解锁它并再次阻塞。

线程阻塞的一个重要属性是，在线程自己进入阻塞之前，对 `unpark()` 的调用不会丢失。对 unpark 的请求仍然被记录下来，并且下次线程尝试挂起自己的时候，它会清除该请求并且直接继续执行，实际上并不会进入睡眠状态。为了理解这对于正确操作的关键性，让我们来看一下程序可能执行步骤的顺序：

1. 消费线程（让我们称之为 C）锁定队列。
2. C 尝试去从队列中弹出项，但是它是空的，导致 None。
3. C 解锁队列。
4. 生产线程（我们将称为 P）锁定队列。
5. P 推入一个新的项进入队列。
6. P 再次解锁队列。
7. P 调用 `unpark()` 去通知 C，有一些新的项。
8. C 调用 `park()` 去睡眠，以等待更多的项。

虽然在步骤 3 解锁队列和在步骤 8 阻塞之间很可能仅有一个很短的时间，但第 4 步和第 7 步可能在线程阻塞自己之前发生。如果 `unpark()` 在线程没有挂起时不执行任何操作，那么通知将会丢失。即使队列中有项，消费线程仍然在等待。由于 `unpark()` 请求被保存，以供将来调用 `park()` 时使用，我们不必担心这个问题。

然而，unpark 请求并不会堆起来。先调用两次 `unpark()`，然后再调用两次 `park()`，线程仍然会进入睡眠状态。第一次 `park()` 清除请求并直接返回，但第二次调用通常让它进入睡眠。

这意味着，在我们上面的示例中，重要的是我们看见队列为空的时候，我们仅会阻塞线程，而不是在处理每个项之后将其阻塞。然而由于巨长的（1s）睡眠，这种情况在本示例中几乎不可能发生，但多个 `unpark()` 调用仅能唤醒单个 `park()` 调用。

不幸的是，这确实意味着，如果在 `park()` 返回后，立即调用 `unpark()`，但是在队列得到锁定并清空之前，`unpark()` 调用是不必要的，但仍然会导致下一个 `park()` 调用立即返回。这导致（空的）队列多次被锁定并解锁。虽然这不会影响程序的正确性，但这确实会影响它的效率和性能。

这种机制在简单的情况下是好的，比如我们的示例，但是当东西变得复杂，情况可能会很糟糕。例如，如果我们有多个消费线程从相同的队列获取项时，生产线程将不会知道有哪些消费者实际上在等待以及应该被唤醒。生产者将必须知道消费者正在等待的时间以及正在等待的条件。

### 条件变量

（<a href="https://marabos.nl/atomics/basics.html#condvar" target="_blank">英文版本</a>）

条件变量是一个更通用的选项，用于等待受 mutex 保护的数据发生变化。它有两种基本操作：等待和通知。线程可以在条件变量上等待，然后在另一个线程通知相同条件变量时被唤醒。多个线程可以在同样的条件变量上等待，通知可以发送给一个等待线程或者所有等待线程。

这意味着我们可以为我们感兴趣的事件或条件创建一个条件变量，例如，队列是非空的，并且在该条件下等待。任意导致事件或条件发生的线程都会通知条件变量，无需知道哪个或有多个线程对该通知感兴趣。

为了避免在解锁 mutex 和等待条件变量的短暂时间失去通知的问题，条件变量提供了一种*原子地*解锁 mutex 和开始等待的方式。这意味着根本没有通知丢失的时刻。

Rust 标准库提供了 `std::sync::Condvar` 作为条件变量。它的等待方法接收 `MutexGuard`，以保证我们已经锁定 mutex。它首先解锁 mutex 并进入睡眠。稍后，当唤醒时，它重新锁定 mutex 并且返回一个新的 MutexGuard（这证明了 mutex 再次被锁定）。

它有两个通知方法：`notify_one` 仅唤醒一个线程（如果有），和 `notify_all` 去唤醒所有线程。

让我们改用 Condvar 修改我们用于线程阻塞的示例：

```rust
use std::sync::Condvar;

let queue = Mutex::new(VecDeque::new());
let not_empty = Condvar::new();

thread::scope(|s| {
    s.spawn(|| {
        loop {
            let mut q = queue.lock().unwrap();
            let item = loop {
                if let Some(item) = q.pop_front() {
                    break item;
                } else {
                    q = not_empty.wait(q).unwrap();
                }
            };
            drop(q);
            dbg!(item);
        }
    });

    for i in 0.. {
        queue.lock().unwrap().push_back(i);
        not_empty.notify_one();
        thread::sleep(Duration::from_secs(1));
    }
});
```

* 我们必须改变一些事情：
  * 我们现在不仅有一个包含队列的 Mutex，同时有一个 Condvar 去通信“不为空”的条件。
  * 我们不再需要知道要唤醒哪个线程，因此我们不再存储 spawn 的返回值。而是，我们通过使用 `notify_one` 方法的条件变量通知消费者。
  * 解锁、等待以及重新锁定都是通过 `wait` 方法完成的。我们不得不稍微重组控制流，以便传递 guard 到 wait 方法，同时在处理项之前仍然丢弃它。

现在，我们可以根据自己的需求生成尽可能多的消费线程，甚至稍后生成更多线程，而无需更改任何东西。条件变量会负责将通知传递给任何感兴趣的线程。

如果我们有个更加复杂的系统，其线程对不同条件感兴趣，我们可以为每个条件定义一个 `Condvar`。例如，我们能定义一个来指示队列是非空的条件，并且另一个指示队列是空的条件。然后，每个线程将等待与它们正在做的事情相关的条件。

通常，Condvar 仅能与单个 Mutex 一起使用。如果两个线程尝试使用两个不同的 mutex 去并发地等待条件变量，它可能导致 panic。

Condvar 的缺点是，它仅能与 Mutex 一起工作，对于大多数用例是没问题的，因为已经在保护数据时使用了 mutex。

`thread::park()` 和 `Condvar::wait()` 也都有一个有时间限制的变体：`thread::park_timeout()` 和 `Condvar::wait_timeout()`。它们接受一个额外的参数 Duration，表示在多长时间后放弃等待通知并无条件地唤醒。

## 总结

（<a href="https://marabos.nl/atomics/basics.html#summary" target="_blank">英文版本</a>）

* 多线程可以并发地运行在相同程序并且可以在任意时间生成。
* 当主线程结束，主程序结束。
* 数据竞争是未定义行为，它会由 Rust 的类型系统完全地阻止（在安全的代码中）。
* 常规的线程可以像程序运行一样长时间，并且因此只能借用 `'static` 数据。例如静态值和泄漏内存分配。
* 引用计数（Arc）可以用于共享所有权，以确保只要有一个线程使用它，数据就会存在。
* 作用域线程用于限制线程的生命周期，以允许其借用非 `'static` 数据，例如作用域变量。
* `&T` 是*共享引用*。`&mut T` 是*独占引用*。常规类型不允许通过共享引用可变。
* 一些类型有着内部可变性，这要归功于 `UnsafeCell`，它允许通过共享引用改变。
* Cell 和 RefCell 是单线程内部可变性的标准类型。Atomic、Mutex 以及 RwLock 是它们多线程等价物。
* Cell 和原子类型仅允许作为整体替换值，而 RefCell、Mutex 和 RwLock 允许你通过动态执行访问规则直接替换值。
* 线程阻塞可以是等待某种条件的便捷方式。
* 当条件是关于由 Mutex 保护的数据时，使用 `Condvar` 时更方便，并且比线程阻塞更有效。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./2_Atomics.html">下一篇，第二章：Atomic</a>
</p>

[^1]: <https://zh.wikipedia.org/zh-cn/監視器_(程序同步化)>
[^2]: <https://zh.wikipedia.org/wiki/互斥锁>
[^3]: <https://rustwiki.org/zh-CN/std/thread/fn.park.html>
[^4]: <https://zh.wikipedia.org/wiki/互斥锁>
[^5]: <https://zh.wikipedia.org/wiki/读写锁>
[^6]: <https://zh.wikipedia.org/wiki/未定义行为>
