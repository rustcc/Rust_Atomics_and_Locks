# 第二章：Atomic

*原子*（atomic）这个单词来自于希腊语 `ἄτομος`，意味着不可分割的，不能被切割成更小的块。在计算机科学中，它被用于描述一个不可分割的操作：它要么完全完成，要么还没发生。

正如在[第一章“借用和数据竞争”](./1_Basic_of_Rust_Concurrency.md#借用和数据竞争)中提及的，多线程并发地读取和修改相同的变量会导致未定义行为。然而，原子操作确实允许不同线程去安全地读取和修改相同的变量。因为该操作是不可分割的，它要么完全地在一个操作之前完成，要么在另一个操作之后完成，从而避免未定义行为。稍后，[在第七章](./7_Understanding_the_Processor.md)，我们将在硬件层面查看它们是如何工作的。

原子操作是任何涉及多线程的主要基石。所有其它的并发原语，例如互斥锁，条件变量都使用原子操作实现。

在 Rust 中，原子操作可以作为 `std::sync::atomic` 标准原子类型的方法使用。它们的名称都以 Atomic 开头，例如 AtomicI32 或 AtomicUsize。可用的原子类型取决于硬件架构和一些操作系统，但几乎所有的平台都提供了指针大小的所有原子类型。

与大多数类型不同，它们允许通过共享引用进行修改（例如，`&AtomicU8`）。正如[第一章“内部可变性”](./1_Basic_of_Rust_Concurrency.md#内部可变性)讨论的那样，这都要归功于它。

每一个可用的原子类型都有着相同的接口，包括存储（store）和加载（load）、原子“获取和修改（fetch-and-modify）”操作方法、和一些更高级的“比较和交换（compare-and-exchange）”方法。我们将在这章节的后半部分讨论它们。

但是，在我们研究不同原子操作之前，我们需要简要谈谈叫做*内存排序*[^1]的概念：

每一个原子操作都采用了 `std::sync::atomic::Ordering` 类型的参数，这决定了我们对操作相对排序的保证。保证最少的简单变体是 `Relaxed`。`Relaxed` 只保证在单个原子变量中的一致性，但是在不同变量的相对操作顺序没有任何承诺。

这意味着两个线程可能看到不同变量的操作以不同的顺序下发生。例如，如果一个线程首先写入一个变量，然后非常快速的写入第二个变量，另一个线程可能看见这以相反的顺序发生。

在这章节，我们将仅关注不会出现问题的使用情况，并且在所有地方都简单地使用 `Relaxed`，而不深入讨论更多细节。我们将在[第三章](./3_Memory_Ordering.md)讨论内存排序的所有细节以及其它可用内存排序。

## Atomic 的加载和存储操作

我们将查看的前两个原子操作是最基本的：load 和 store。它们的函数签名如下，使用 AtomicI32 作为示例：

```rust
impl AtomicI32 {
    pub fn load(&self, ordering: Ordering) -> i32;
    pub fn store(&self, value: i32, ordering: Ordering);
}
```

load 方法以原子方式加载存储在原子变量中的值，并且 store 方法原子方式存储新值。注意，store 方法采用共享应用（`&T`），而不是独占引用（`&mut T`），即使它修改了值。

让我们来看看这两种方式的使用示例。

### 示例：停止标识

第一个示例使用 AtomicBool 作为*停止标识*。这个标识被用于告诉其它线程去停止运行：

```rust
use std::sync::atomic::AtomicBool;
use std::sync::atomic::Ordering::Relaxed;

fn main() {
    static STOP: AtomicBool = AtomicBool::new(false);

    // Spawn a thread to do the work.
    let background_thread = thread::spawn(|| {
        while !STOP.load(Relaxed) {
            some_work();
        }
    });

    // Use the main thread to listen for user input.
    for line in std::io::stdin().lines() {
        match line.unwrap().as_str() {
            "help" => println!("commands: help, stop"),
            "stop" => break,
            cmd => println!("unknown command: {cmd:?}"),
        }
    }

    // Inform the background thread it needs to stop.
    STOP.store(true, Relaxed);

    // Wait until the background thread finishes.
    background_thread.join().unwrap();
}
```

在本线程中，后台线程反复地运行 `some_work()`，而主线程允许用户输入一些命令与其它线程交互程序。在这个示例中，唯一有用的命令是 `stop`，以使程序停止。

为了使后台线程停止，原子 **STOP** 布尔值使用此条件与后台线程交互。当前台线程读取到 `stop` 命令，它设置标识到 true，在每次新迭代之前，后台线程都会检查。主线程等待直到后台线程使用 join 方法完成它当前的迭代。

只要后台线程定期检查标识，这个简单的工作方案就是好的。如果它在 `some_work()` 卡住很长时间，这可能在 stop 命令和程序退出之间出现不可退出的延迟。

### 示例：进度报道

在我们的下一个示例中，我们通过后台线程逐步地处理 100 项，而主线程为用户提供定期地更新：

```rust
use std::sync::atomic::AtomicUsize;

fn main() {
    let num_done = AtomicUsize::new(0);

    thread::scope(|s| {
        // A background thread to process all 100 items.
        s.spawn(|| {
            for i in 0..100 {
                process_item(i); // Assuming this takes some time.
                num_done.store(i + 1, Relaxed);
            }
        });

        // The main thread shows status updates, every second.
        loop {
            let n = num_done.load(Relaxed);
            if n == 100 { break; }
            println!("Working.. {n}/100 done");
            thread::sleep(Duration::from_secs(1));
        }
    });

    println!("Done!");
}
```

这次，我们使用一个[线程作用域](./1_Basic_of_Rust_Concurrency.md#线程作用域)，它将自动地为我们处理线程的 join，并且也允许我们借用局部变量。

每次后台线程完成处理项时，它都会将处理的项目数量存储在 AtomicUsize 中。与此同时，主线程向用户显示该数字，告知该进度，大约每秒一次。一旦主线程看见所有 10 项已经被处理，它就会退出作用域，它会隐式地 join 后台线程，并且告知用户所有都完成。

#### 同步

一旦最后一项处理完成，主线程可能需要整整一秒才知道，从而在最后引入不必要的延迟。为了解决这个，我们在每当有新的消息有用时，可以使用阻塞（[第一章“线程阻塞”](./1_Basic_of_Rust_Concurrency.md#线程阻塞)）去唤醒睡眠中的主线程。

以下是相同的示例，但是现在使用 `thread::park_timeout` 而不是 `thread::sleep`:

```rust
fn main() {
    let num_done = AtomicUsize::new(0);

    let main_thread = thread::current();

    thread::scope(|s| {
        // A background thread to process all 100 items.
        s.spawn(|| {
            for i in 0..100 {
                process_item(i); // Assuming this takes some time.
                num_done.store(i + 1, Relaxed);
                main_thread.unpark(); // Wake up the main thread.
            }
        });

        // The main thread shows status updates.
        loop {
            let n = num_done.load(Relaxed);
            if n == 100 { break; }
            println!("Working.. {n}/100 done");
            thread::park_timeout(Duration::from_secs(1));
        }
    });

    println!("Done!");
}
```

没有什么变化，我们通过 `thread::current()` 获取主线程的句柄，该句柄现在被用于在每次后台线程状态更新后，阻塞主线程。主线程现在使用 park_timeout，而不是 sleep，这样它可以被中断。

现在，任何状态更新都会立即告知用于，同时仍然报道每秒最后一次更新，以展示程序仍然在运行。

### 示例：惰性初始化

在移动到更高级的原子操作之前，最后一个示例是关于*惰性初始化*。

想象有一个值 x，我们从一个文件读取它，从操作系统获取，或者以其他方式计算，我们期待去在程序运行期间它是一个常量。获取 x 是操作系统的版本、内存的总数或者 tau 的第 400 位。对于这个示例，这真的不重要。

因为我们不期待它去发生变化，我们仅在第一次请求或计算时需要它，并且记住它的结果。需要它的第一个线程必须计算值，但它可以存储它到一个 `static` 的原子中，使所有线程可用，包括稍后它自己。

让我们看看这个示例。为了使这些简单，我们将假设 x 永远不会是 0，这样我们就可以在计算之前使用 0 作为占位符。

```rust
use std::sync::atomic::AtomicU64;

fn get_x() -> u64 {
    static X: AtomicU64 = AtomicU64::new(0);
    let mut x = X.load(Relaxed);
    if x == 0 {
        x = calculate_x();
        X.store(x, Relaxed);
    }
    x
}
```

第一个线程调用 `get_x()` 将检查 `static X` 并看见它仍然是 0，计算它的值并且存回结果到 static X 中，使它在未来可用。稍后，任意对 `get_x()` 的调用都将看见静态值不是 0，并立即返回，没有立即计算。

然而，如果第二个线程调用 `get_x()`，而第一个线程仍然正在计算 x，第二个线程将也看见 0，并且也在并行的计算 x。其中一个线程将最后复写另一个线程的结果，这取决于哪一个线程先完成。这被叫做*竞争*。不是数据竞争，这是一个未定义行为，在 Rust 中不使用 unsafe 的情况下是不可能发生的，但仍然有一个不可预测的赢者的竞争。

因为我们期待 x 是常量，那么谁赢得比赛并不重要，因为无论如何结果都是一样。依赖于 `calculate_x()` 会花费多少时间，这可能非常好或者很糟糕。

如果 `calculate_x()` 预计花费很长时间，则最好在第一个线程仍在初始化 X 时等待线程，以避免不必要的浪费处理器时间。你可以使用一个条件变量或者线程阻塞（[第一章“等待-阻塞和条件变量”](./1_Basic_of_Rust_Concurrency.md#等待-阻塞park和条件变量)）来实现这个，但是对于一个小例子来说，这很快将变得复杂。Rust 标准库通过 `std::sync::Once` 和 `std::sync::OnceLock` 正是此功能，所以通常这些不需要由你自己实现。

## 获取并修改操作

注意，我们已经看见基础 load 和 store 操作的一些用例，让我们继续更有趣的操作：*获取并修改*（fetch-and-modify）操作。这些操作修改原子变量，但也加载（获取）原始值，作为一个单原子操作。

最常用地是 `fetch_add` 和 `fetch_sub`，它们分别执行加法和减法。一些其他可获得的操作是位操作 `fetch_or` 和 `fetch_and`，以及用于比较最大值和最小值的 `fetch_max` 和 `fetch_min`。

它们的函数签名如下，使用 AtomicI32 作为示例：

```rust
impl AtomicI32 {
    pub fn fetch_add(&self, v: i32, ordering: Ordering) -> i32;
    pub fn fetch_sub(&self, v: i32, ordering: Ordering) -> i32;
    pub fn fetch_or(&self, v: i32, ordering: Ordering) -> i32;
    pub fn fetch_and(&self, v: i32, ordering: Ordering) -> i32;
    pub fn fetch_nand(&self, v: i32, ordering: Ordering) -> i32;
    pub fn fetch_xor(&self, v: i32, ordering: Ordering) -> i32;
    pub fn fetch_max(&self, v: i32, ordering: Ordering) -> i32;
    pub fn fetch_min(&self, v: i32, ordering: Ordering) -> i32;
    pub fn swap(&self, v: i32, ordering: Ordering) -> i32; // "fetch_store"
}
```

最后一个例外的操作是将一个新值存储到原子变量中，而不考虑原来的值。它不叫做 `fetch_store`，而是称为 `swap`。

这里有一个快速地演示，展示了 `fetch_add` 如何在操作之前返回值：

```rust
use std::sync::atomic::AtomicI32;

let a = AtomicI32::new(100);
let b = a.fetch_add(23, Relaxed);
let c = a.load(Relaxed);

assert_eq!(b, 100);
assert_eq!(c, 123);
```

fetch_add 操作从 100 增加到 123，但是返回给我们还是旧值 100。任意下一个操作将看见 123。

来自这些操作的返回值并不总是相关的。如果你仅需要将操作用于原子值，但是值本身并没有用，那么你可以忽略该返回值。

需要记住的一个重要的事情是，fetch_add 和 fetch_sub 为溢出实现了包装行为。将值增加超过最大可表示值将包裹起来，并导致最小可表示值。这与常规整数上的增加和减少行为是不同的，后者调试模式下的溢出将 panic。

在[“compare-and-exchange 操作”](#compare-and-exchange-操作)中，我们将看见如何使用溢出检查做原子加法。

但首先，让我们看看这些方法的现实使用示例。

### 示例：来自多线程的进度报道

### 示例：统计数据

### 示例：ID 分配

## compare-and-exchange 操作

### 示例：没有溢出的 ID 分配

### 示例：惰性一次性初始化

## 总结

* 原子操作是不可分割的；它们要么完整的完成，要么它们还没有发生。
* 在 Rust 中的原子操作是通过 `std::sync::atomic` 原子类型完成的，例如 `AtomicI32`。
* 不是所有原子类型在所有平台都是可获得的。
* 当涉及多个变量时，原子操作的相对顺序是棘手的。更多细节，请看[第三章](./3_Memory_Ordering.md)。
* 简单的 load 和 store 操作非常适合非常简单的基本线程间通信，例如停止标识和状态报道。
* 我们可以使用竞争条件[^2]来惰性始化，而不会引发数据竞争[^3]。
* 获取并修改操作允许进行一小组基本的原子修改操作，当多个线程同时修改同一个院子变量时，非常有用。
* 原子加法和减法在溢出时会默默地进行环绕（wrap around）操作。
* compare-and-exchange 操作是最灵活和通用的，并且是任意其它原子操作的基石。
* *weak* compare-and-exchange 稍微更有效。

[^1]: <https:·//zh.wikipedia.org/wiki/内存排序>
[^2]: 竞争条件是指多个线程并发访问和修改共享数据时，其最终结果依赖于线程执行的具体顺序。在某些情况下，我们可以利用竞争条件来实现延迟初始化。也就是说，多个线程可以同时尝试对共享资源进行初始化，但只有第一个成功的线程会完成初始化，而其他线程会放弃初始化操作。
[^3]: 数据竞争是指多个线程同时访问共享数据，并且至少有一个线程进行写操作，而没有适当的同步机制来保证正确的访问顺序。
