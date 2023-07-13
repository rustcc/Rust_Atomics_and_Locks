# 第二章：Atomic

（<a href="https://marabos.nl/atomics/atomics.html" target="_blank">英文版本</a>）

*原子*（atomic）这个单词来自于希腊语 `ἄτομος`，意味着不可分割的，不能被切割成更小的块。在计算机科学中，它被用于描述一个不可分割的操作：它要么完全完成，要么还没发生。

正如在[第一章“借用和数据竞争”](./1_Basic_of_Rust_Concurrency.md#借用和数据竞争)中提及的，多线程并发地读取和修改相同的变量会导致未定义行为。然而，原子操作确实允许不同线程去安全地读取和修改相同的变量。因为该操作是不可分割的，它要么完全地在一个操作之前完成，要么在另一个操作之后完成，从而避免未定义行为。稍后，在[第七章](./7_Understanding_the_Processor.md)，我们将在硬件层面查看它们是如何工作的。

原子操作是任何涉及多线程的主要基石。所有其它的并发原语，例如互斥锁，条件变量都使用原子操作实现。

在 Rust 中，原子操作可以作为 `std::sync::atomic` 标准原子类型的方法使用。它们的名称都以 Atomic 开头，例如 AtomicI32 或 AtomicUsize。可用的原子类型取决于硬件架构和一些操作系统，但几乎所有的平台都提供了指针大小的所有原子类型。

与大多数类型不同，它们允许通过共享引用进行修改（例如，`&AtomicU8`）。正如[第一章“内部可变性”](./1_Basic_of_Rust_Concurrency.md#内部可变性)讨论的那样，这都要归功于它。

每一个可用的原子类型都有着相同的接口，包括存储（store）和加载（load）、原子“获取并修改（fetch-and-modify）”操作方法、和一些更高级的“比较并交换”（compare-and-exchange）[^4]方法。我们将在这章节的后半部分讨论它们。

但是，在我们研究不同原子操作之前，我们需要简要谈谈叫做*内存排序*[^1]的概念：

每一个原子操作都接收 `std::sync::atomic::Ordering` 类型的参数，这决定了我们对操作相对排序的保证。保证最少的简单变体是 `Relaxed`。`Relaxed` 只保证在单个原子变量中的一致性，但是在不同变量的相对操作顺序没有任何保证。

这意味着两个线程可能看到不同变量的操作以不同的顺序下发生。例如，如果一个线程首先写入一个变量，然后非常快速的写入第二个变量，另一个线程可能看见这以相反的顺序发生。

在这章节，我们将仅关注不会出现这种问题的使用情况，并且在所有地方都简单地使用 `Relaxed`，而不深入讨论更多细节。我们将在[第三章](./3_Memory_Ordering.md)讨论内存排序的所有细节以及其它可用内存排序。

## Atomic 的加载和存储操作

（<a href="https://marabos.nl/atomics/atomics.html#atomic-load-and-store-operations" target="_blank">英文版本</a>）

我们将查看的前两个原子操作是最基本的：load 和 store。它们的函数签名如下，使用 AtomicI32 作为示例：

```rust
impl AtomicI32 {
    pub fn load(&self, ordering: Ordering) -> i32;
    pub fn store(&self, value: i32, ordering: Ordering);
}
```

load 方法以原子方式加载存储在原子变量中的值，并且 store 方法原子方式存储新值。注意，store 方法采用共享引用（`&T`），而不是独占引用（`&mut T`），即使它修改了值。

让我们来看看这两种方式的使用示例。

### 示例：停止标识

（<a href="https://marabos.nl/atomics/atomics.html#example-stop-flag" target="_blank">英文版本</a>）

第一个示例使用 AtomicBool 作为*停止标识*。这个标识被用于告诉其它线程去停止运行：

```rust
use std::sync::atomic::AtomicBool;
use std::sync::atomic::Ordering::Relaxed;

fn main() {
    static STOP: AtomicBool = AtomicBool::new(false);

    // 产生一个线程，去做一些工作。
    let background_thread = thread::spawn(|| {
        while !STOP.load(Relaxed) {
            some_work();
        }
    });

    // 使用主线程监听用户的输入。
    for line in std::io::stdin().lines() {
        match line.unwrap().as_str() {
            "help" => println!("commands: help, stop"),
            "stop" => break,
            cmd => println!("unknown command: {cmd:?}"),
        }
    }

    // 告知后台线程需要停止。
    STOP.store(true, Relaxed);

    // 等待直到后台线程完成。
    background_thread.join().unwrap();
}
```

在本示例中，后台线程反复地运行 `some_work()`，而主线程允许用户输入一些命令与其它线程交互程序。在这个示例中，唯一有用的命令是 `stop`，来使程序停止。

为了使后台线程停止，原子 **STOP** 布尔值使用此条件与后台线程交互。当前台线程读取到 `stop` 命令，它设置标识到 true，在每次新迭代之前，后台线程都会检查。主线程等待直到后台线程使用 join 方法完成它当前的迭代。

只要后台线程定期检查标识，这个简单的工作方案就是好的。如果它在 `some_work()` 卡住很长时间，这可能在 stop 命令和程序退出之间出现不可接受的延迟。

### 示例：进度报道

（<a href="https://marabos.nl/atomics/atomics.html#example-progress-reporting" target="_blank">英文版本</a>）

在我们的下一个示例中，我们通过后台线程逐步地处理 100 个元素，而主线程为用户提供定期地更新：

```rust
use std::sync::atomic::AtomicUsize;

fn main() {
    let num_done = AtomicUsize::new(0);

    thread::scope(|s| {
        // 一个后台线程，去处理所有 100 个元素。
        s.spawn(|| {
            for i in 0..100 {
                process_item(i); // 假设该处理需要一些时间。
                num_done.store(i + 1, Relaxed);
            }
        });

        // 主线程没秒展示一次状态更新。
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

这次，我们使用一个[作用域内的线程](./1_Basic_of_Rust_Concurrency.md#作用域内的线程)，它将自动地为我们处理线程的 join，并且也允许我们借用局部变量。

每次后台线程完成处理元素时，它都会将处理的元素数量存储在 AtomicUsize 中。与此同时，主线程向用户显示该数字，告知该进度，大约每秒一次。一旦主线程看见所有 10 个元素已经被处理，它就会退出作用域，它会隐式地 join 后台线程，并且告知用户所有都完成。

#### 同步

（<a href="https://marabos.nl/atomics/atomics.html#synchronization" target="_blank">英文版本</a>）

一旦最后一个元素处理完成，主线程可能需要整整一秒才知道，这在最后引入了不必要的延迟。为了解决这个问题，我们在每当有新的消息有用时，使用线程阻塞（[第一章“线程阻塞”](./1_Basic_of_Rust_Concurrency.md#线程阻塞)）去唤醒睡眠中的主线程。

以下是相同的示例，但是现在使用 `thread::park_timeout` 而不是 `thread::sleep`:

```rust
fn main() {
    let num_done = AtomicUsize::new(0);

    let main_thread = thread::current();

    thread::scope(|s| {
        // // 一个后台线程，去处理所有 100 个元素。
        s.spawn(|| {
            for i in 0..100 {
                process_item(i); // 假设该处理需要一些时间。
                num_done.store(i + 1, Relaxed);
                main_thread.unpark(); // 唤醒主线程
            }
        });

        // 主线程展示的状态更新
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

没有什么变化，我们通过 `thread::current()` 获取主线程的句柄，该句柄现在被用于在每次后台线程状态更新后，阻塞主线程。主线程现在使用 park_timeout 而不是 sleep，这样它就可以被中断。

现在，任何状态更新都会立即告知用户，同时仍然每秒重复上一次更新，以展示程序仍然在运行。

### 示例：惰性初始化

（<a href="https://marabos.nl/atomics/atomics.html#example-lazy-init" target="_blank">英文版本</a>）

在移动到更高级的原子操作之前，我们来看最后一个关于*惰性初始化*的示例。

想象有一个值 x，我们可以从一个文件读取它、从操作系统获取或者以其他方式计算得到，我们期待去在程序运行期间它是一个常量。或许 x 是操作系统的版本、内存的总数或者 tau 的第 400 位。对于这个示例，这不重要。

因为我们不期待它去发生变化，我们仅在第一次请求或计算时需要它，并且记住它的结果。需要它的第一个线程必须计算值，但它可以存储它到一个 `static` 的原子中，使所有线程可用，包括稍后的自己。

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

然而，如果第二个线程调用 `get_x()`，而第一个线程仍然正在计算 x，第二个线程将也看见 0，并且也在并行的计算 x。其中一个线程将最后复写另一个线程的结果，这取决于哪一个线程先完成。这被叫做*竞争*。并不是数据竞争（这是一个未定义行为），在 Rust 中不使用 unsafe 的情况下是不可能发生的，但这仍然有一个不可预测的赢者的竞争。

因为我们期待 x 是常量，那么谁赢得比赛并不重要，因为无论如何结果都是一样。依赖于 `calculate_x()` 会花费多少时间，这可能非常好或者很糟糕。

如果 `calculate_x()` 预计花费很长时间，则最好在第一个线程仍在初始化 X 时等待，以避免不必要的浪费处理器时间。你可以使用一个条件变量或者线程阻塞（[第一章“等待-阻塞和条件变量”](./1_Basic_of_Rust_Concurrency.md#等待-阻塞park和条件变量)）来实现这个，但是对于一个小例子来说，这很快将变得复杂。Rust 标准库通过 `std::sync::Once` 和 `std::sync::OnceLock` 提供了此功能，所以通常这些不需要由你自己实现。

## 获取并修改操作

（<a href="https://marabos.nl/atomics/atomics.html#fetch-and-modify-operations" target="_blank">英文版本</a>）

注意，我们已经看见基础 load 和 store 操作的一些用例，让我们继续更有趣的操作：*获取并修改*（fetch-and-modify）操作。这些操作修改原子变量，但也加载（获取）原始值，作为一个单原子操作。

最常用的是 `fetch_add` 和 `fetch_sub`，它们分别执行加和减运算。一些其他可获得的操作是位操作 `fetch_or` 和 `fetch_and`，以及用于比较最大值和最小值的 `fetch_max` 和 `fetch_min`。

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

这里有一个快速的演示，展示了 `fetch_add` 如何在操作之前返回值：

```rust
use std::sync::atomic::AtomicI32;

let a = AtomicI32::new(100);
let b = a.fetch_add(23, Relaxed);
let c = a.load(Relaxed);

assert_eq!(b, 100);
assert_eq!(c, 123);
```

fetch_add 操作从 100 递增到 123，但是返回给我们还是旧值 100。任意下一个操作将看见 123。

来自这些操作的返回值并不总是相关的。如果你仅需要将操作用于原子值，但是值本身并没有用，那么你可以忽略该返回值。

需要记住的一个重要的事情是，fetch_add 和 fetch_sub 为溢出实现了环绕（wrapping）行为。将值递增超过最大可表示值将导致环绕到最小可表示值。这与常规整数上的递增和递减行为是不同的，后者在调试模式下的溢出将 panic。

在[“「比较并交换」操作”](#比较并交换操作)中，我们将看见如何使用溢出检查进行原子加运算。

但首先，让我们看看这些方法的现实使用示例。

### 示例：来自多线程的进度报道

（<a href="https://marabos.nl/atomics/atomics.html#example-progress-reporting-from-multiple-threads" target="_blank">英文版本</a>）

在[“示例：进度报道”](#示例进度报道)中，我们使用一个 AtomicUsize 去报道后台线程的进度。如果我们把工作分开，例如，四个线程，每个处理 25 个元素，我们将需要知道所有 4 个线程的进度。

我们可以为每个线程使用单独的 AtomicUsize 并且在主线程加载它们并进行汇总，但是更简单的解决方案是，使用单个 AtomicUsize 去跟踪所有线程处理元素的总数。

为了使其工作，我们不再使用 store 方法，因为这会覆盖其他线程的进度。相反，我们可以使用原子自增操作在每个处理元素之后递增计数器。

```rust
fn main() {
    let num_done = &AtomicUsize::new(0);

    thread::scope(|s| {
        // 4 个后台线程，去处理所有 100 个元素，每个 25 次。
        for t in 0..4 {
            s.spawn(move || {
                for i in 0..25 {
                    process_item(t * 25 + i); // 假设此处理需要花费一些时间。
                    num_done.fetch_add(1, Relaxed);
                }
            });
        }

        // 主线程每秒展示一次状态更新。
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

一些东西已经改变。更重要地是，我们现在产生了 4 个后台线程而不是 1 个，并且使用 fetch_add 而不是 store 去修改 `num_done` 原子变量。

更巧妙的是，我们现在对后台线程使用一个 move 闭包，并且 num_done 现在是一个引用。这与我们使用 fetch_add 无关，而是我们如何在循环中产生 4 个线程有关。此闭包捕获 t，以了解它是 4 个线程中的哪一个，从而确定是从元素 0、25、50 还是 75 开始。没有 move 关键字，闭包将尝试通过引用捕获 t。这是不允许的，因为它仅在循环期间短暂地存在。

由于 move 闭包，它移动（或复制）它的捕获而不是借用它们，这使它得到 t 的复制。因为它也捕获 num_done，我们已经改变该变量为一个引用，因为我们仍然想要借用相同的 AtomicUsize。注意，原子类型并没有实现 Copy trait，所以如果我们尝试移动一个原子类型的变量到多个线程，我们将得到错误。

撇开闭包的微妙不谈，在这里使用 fetch_add 更改是非常简单的。我们并不知道线程将以哪种顺序递增 num_done，但由于加运算是原子的，我们并不担心任何事情，并且当所有线程完成时，可以确信它将是 100。

### 示例：统计数据

（<a href="https://marabos.nl/atomics/atomics.html#example-statistics" target="_blank">英文版本</a>）

继续通过原子报道其他线程正在做什么的概念，让我们拓展我们的示例，也可以收集和报道一些关于处理元素所花费时间的统计数据。

在 num_done 旁边，我们递增了两个原子变量 `total_time` 和 `max_time`，以便跟踪处理元素所花费的时间。我们将使用这些报道平均和峰值处理时间。

```rust
fn main() {
    let num_done = &AtomicUsize::new(0);
    let total_time = &AtomicU64::new(0);
    let max_time = &AtomicU64::new(0);

    thread::scope(|s| {
        /// 4 个后台线程，去处理所有 100 个元素，每个 25 次。
        for t in 0..4 {
            s.spawn(move || {
                for i in 0..25 {
                    let start = Instant::now();
                    process_item(t * 25 + i); // 假设此处理需要花费一些时间。
                    let time_taken = start.elapsed().as_micros() as u64;
                    num_done.fetch_add(1, Relaxed);
                    total_time.fetch_add(time_taken, Relaxed);
                    max_time.fetch_max(time_taken, Relaxed);
                }
            });
        }

        // 主线程每秒展示一次状态更新。
        loop {
            let total_time = Duration::from_micros(total_time.load(Relaxed));
            let max_time = Duration::from_micros(max_time.load(Relaxed));
            let n = num_done.load(Relaxed);
            if n == 100 { break; }
            if n == 0 {
                println!("Working.. nothing done yet.");
            } else {
                println!(
                    "Working.. {n}/100 done, {:?} average, {:?} peak",
                    total_time / n as u32,
                    max_time,
                );
            }
            thread::sleep(Duration::from_secs(1));
        }
    });

    println!("Done!");
}
```

后台线程现在使用 `Install::now()` 和 `Install::elapsed()` 去衡量它们在 `process_item()` 所花费的时间。原子的递增操作用于将微秒数递增到 total_time，并且原子的 max 操作用于跟踪 max_time 中的最高测量值。

主线程将总时间除以处理器元素的数量以获取平均处理时间，然后将它与 max_time 的峰值时间一起报道。

由于三个原子变量是单独更新的，因此主线程可能在线程递增 num_done 后而在更新 total_time 之前加载值，导致低估了平均值。更微妙的是，因为 Relaxed 内存排序不能保证从另一个线程中看到操作的相对顺序，它甚至可能看到 total_time 新更新的值，同时仍然看到 num_done 的旧值，导致平均值的高估。

在我们的示例中，这两个都不是大问题。最糟糕的情况是向用户提交了不准确的平均值。

如果我们想要避免这个，我们可以把这三个统计数据放到一个 Mutex 中。然后，在更新三个数字时，我们短暂地锁定 mutex，这三个数字本身就不再是原子的。这有效地转变三次更新为单个原子操作，代价是锁定和解锁 mutex 的开销，并且可能临时地阻塞线程。

### 示例：ID 分配

（<a href="https://marabos.nl/atomics/atomics.html#example-id-allocation" target="_blank">英文版本</a>）

让我们转到一个用例，我们实际上需要 `fetch_add` 的返回值。

假设我们需要一些函数，`allocate_new_id()`，每次调用它时，都会给出新的唯一的数字。我们可能使用这些数字标识程序中的任务或其它事情；需要一个小而易于存储和在线程之间传递的东西来唯一标识事物，例如整数。

使用 `fetch_add` 实现此函数是轻松的：

```rust
use std::sync::atomic::AtomicU32;

fn allocate_new_id() -> u32 {
    static NEXT_ID: AtomicU32 = AtomicU32::new(0);
    NEXT_ID.fetch_add(1, Relaxed)
}
```

我们只是跟踪了*下一个*要给出的数字，并在每次加载时递增它。第一个调用者将得到 0，第二个调用者将得到 1，以此类推。

这里唯一的问题是溢出时包装行为。第 4,294,967,296 次调用将溢出 32 位整数，因此下一次调用将再次返回 0。

这是否是一个问题取决于用例：经常被这样调用的可能性是多大，如果数字不是唯一的，最糟糕的情况是什么？虽然这似乎是一个巨大的数字，现代计算机也可以在几秒内的轻松执行我们的函数。如果内存安全取决于这些数字的唯一性，那么我们上面的实现是不可接受的。

为了解决这个问题，如果调用次数太多，我们可以试图去使函数 panic，例如这样：

```rust
// 这个版本是有问题的。
fn allocate_new_id() -> u32 {
    static NEXT_ID: AtomicU32 = AtomicU32::new(0);
    let id = NEXT_ID.fetch_add(1, Relaxed);
    assert!(id < 1000, "too many IDs!");
    id
}
```

现在，assert 语句将在 1000 次调用后 panic。然而，这在原子加运算已经发生后发生，这意味着当我们 panic 时，`NEXT_ID` 已经递增到 1001。如果另一个线程在之后调用该函数，它将在 panic 之前递增到 1002，以此类推。虽然需要很长时间，我们将在 4,294,966,296 次 panic 过后，NEXT_ID 将再次溢出到 0 时，遇到相同的问题。

有三个通常的方式解决这个问题。第一种是在溢出时不使用 panic，而是完全终止进程。`std::process::abort` 函数将终止整个函数，排除任何继续调用我们函数的可能性。<a class="indexterm" id="index-abortingtheprocess"></a>尽管终止进程可能需要短暂的时间，但是函数仍然可能通过其它线程调用，但在程序真正的终止之前，发生数十亿次调用的可能性几乎可以忽略不计。

事实上，在标准库中的 `Arc::clone()` 溢出检查就是这么实现的，以防你在某种方式下克隆 `isize::MAX` 次。在 64 位计算机上，这需要上百年的时间，但如果 isize 只有 32 位，这仅需要几秒钟。

处理溢出的第二种方法是使用 fetch_sub 在 panic 之前再次递减计数器，就像这样：

```rust
fn allocate_new_id() -> u32 {
    static NEXT_ID: AtomicU32 = AtomicU32::new(0);
    let id = NEXT_ID.fetch_add(1, Relaxed);
    if id >= 1000 {
        NEXT_ID.fetch_sub(1, Relaxed);
        panic!("too many IDs!");
    }
    id
}
```

当多个线程在相同时间执行这个函数，计数器仍然有可能在非常短暂的时间超过 100，但这受到活动线程数量的限制。合理地假设是永远不会有数十亿个激活的线程，并非所有线程都在 fetch_add 和 fetch_sub 之间的短暂时间内同时执行相同的函数。

这就是标准库 `thread::scope` 实现中处理运行线程数量溢出的方式。

第三种处理溢出的方式可以说是唯一正确的方式，因为如果它溢出，它完全可以阻止加运算发生。然而，我们不能使用迄今为止看到的原子操作实现这一点。为此，我们需要「比较并交换」操作，接下来我们将探索。

## 比较并交换操作

（<a href="https://marabos.nl/atomics/atomics.html#cas" target="_blank">英文版本</a>）

更加高级和灵活的原子操作是「*比较并交换*」操作。这个操作检查是否原子值等同于给定的值，只有在这种情况下，它才以原子地方式使用新值替换它，作为单个操作完成。它会返回先前的值，并告诉我们是否进行了替换。

它的签名比我们到目前为止看到的要复杂一些。以 AtomicI32 为例，它看起来像这样：

```rust
impl AtomicI32 {
    pub fn compare_exchange(
        &self,
        expected: i32,
        new: i32,
        success_order: Ordering,
        failure_order: Ordering
    ) -> Result<i32, i32>;
}
```

暂时忽略内存排序，它基本上与以下实现相同，只是这一切都发生在单个不可分割原子操作上：

```rust
impl AtomicI32 {
    pub fn compare_exchange(&self, expected: i32, new: i32) -> Result<i32, i32> {
        // 在实际中，加载、比较以及存储操作，
        // 这些所有都是以单个原子操作发生的。
        let v = self.load();
        if v == expected {
            // 值符合预期
            // 替换它并报道成功。
            self.store(new);
            Ok(v)
        } else {
            // 值不符合预期。
            // 保持不变并报道失败。
            Err(v)
        }
    }
}
```

使用该函数，我们可以从原子变量中加载值，执行我们喜欢的任何计算，并且然后仅原子变量在此期间没有改变值的情况下，再存储新的计算值。如果我们把这个放在一个循环中重试，如果它确实发生了变化，我们可以使用它来实现所有其它原子操作，使它成为最通用的一个。

为了演示，让我们在不使用 `fetch_add` 的情况下将 AtomicU32 递增 1，仅是为了看看 compare_exchange 是如何使用的：

```rust
fn increment(a: &AtomicU32) {
    let mut current = a.load(Relaxed); // 1
    loop {
        let new = current + 1; // 2
        match a.compare_exchange(current, new, Relaxed, Relaxed) { // 3
            Ok(_) => return, // 4
            Err(v) => current = v, // 5
        }
    }
}
```

1. 首先，我们加载 a 的当前值。
2. 我们计算我们想要存储在 a 的新值，而不考虑其他线程的并发修改。
3. 我们使用 compare_exchange 去更新 a 的值，但*仅*当它的值仍然与我们之前加载的值相同时。
4. 如果 a 确实和之前一样，它现在被我们的新值所取代，我们就完成了。
5. 如果 a 并不和之前相同，那么自从我们加载它以来，它一定短时间被另一个线程改变了。`compare_exchange` 操作给我门提供了 a 的改变值，并且我们将再次尝试使用该值。加载和更新之间的时间非常短暂，它不可能循环超过几次迭代。

> 如果原子变量从某个值 A 更改到 B，但在 load 操作之后和 compare_exchange 操作之前又变回 A，即使原子变量在此期间发生了变化（并且回变），compare_exchange 操作也会成功。在很多示例中，就像在我们的递增示例中一样，这并不是问题。然而，有几种算法，通常涉及原子指针，这样的情况就会产生问题。这就是所谓的 <a id="index-ABAproblem"></a> ABA 问题。

在 `compare_exchange` 旁边，有一个名为 `compare_exchange_weak` 的类似方法。区别是 weak 版本有时可能仍然保留不改变值并且返回 Err，即使原子值匹配期待值。在某些平台，这个方法可以更有效地实现，并且对于虚假的「比较并交换」失败的后果不重要的情况下，比如上面的递增函数，应该优先使用它。在[第七章节](./7_Understanding_the_Processor.md)，我们将深入研究底层细节，以找出为什么 weak 版本会更有效。

### 示例：没有溢出的 ID 分配

（<a href="https://marabos.nl/atomics/atomics.html#example-handle-overflow" target="_blank">英文版本</a>）

现在，从[“示例：ID 分配”](#示例id-分配)中回到 `allocate_new_id()` 的溢出问题。

为了停止递增 NEXT_ID 超过某个限制以阻止溢出，我们可以使用 compare_exchange 去实现具有上限的原子操作加。使用这个想法，让我们制作一个始终正确处理溢出 allocate_new_id 的版本，即使在几乎不可能的情况下也是如此：

```rust
fn allocate_new_id() -> u32 {
    static NEXT_ID: AtomicU32 = AtomicU32::new(0);
    let mut id = NEXT_ID.load(Relaxed);
    loop {
        assert!(id < 1000, "too many IDs!");
        match NEXT_ID.compare_exchange_weak(id, id + 1, Relaxed, Relaxed) {
            Ok(_) => return id,
            Err(v) => id = v,
        }
    }
}
```

现在，我们在修改 NEXT_ID 之前，检查并 panic，保证它将从不递增超出 1000，使溢出变得不可能。如果我们愿意，我们现在可以将上限从 1000 提高到 `u32::MAX`，而不必担心它可能会超过极限的边缘情况。

<div class="box">
  <h2 style="text-align: center;">Fetch-Update</h2>
  <p>原子类型有一个名为 <code>fetch_update</code> 的简写方法，用于「比较并交换」循环模式。它相当于 load 操作，然后就是重复计算和 <code>compare_exchange_weak</code> 的循环，就像我们上面做的那样。</p>

  <p>使用它，我们可以使用一行实现我们的 allocate_new_id：</p>
  
  <pre>
  NEXT_ID.fetch_update(Relaxed, Relaxed,
      |n| n.checked_add(1)).expect("too many IDs!")</pre>
  
  <p>有关详细信息，请查看该方法的文档。</p>

  <p>我们不会在本书中使用 <code>fetch_update</code> 方法，因此我们可以专注于单个原子操作。</p>
</div>

### 示例：惰性一次性初始化

（<a href="https://marabos.nl/atomics/atomics.html#example-racy-init" target="_blank">英文版本</a>）

在[“示例：惰性初始化”](#示例惰性初始化)中，我们查看常量值的惰性初始化示例。我们做了一个函数，在第一次调用时惰性地初始化一个值，但在以后的调用中重用它。当多个线程并发地运行这个函数，多个线程可能执行初始化，并且它们将以不可预期的顺序覆盖彼此的结果。

对于我们期望值是常量，或者当我们不关心改变值时，这很好。然而，也有些用例，这些值每次都会初始化为不同的值，即便我们需要在程序的一次运行中返回相同的值。

例如，想象一个函数 `get_key()`，它返回一个随机生成的密钥，该密钥仅在程序每次运行时生成。它可能是用于与程序通信的加密密钥，该密钥每次运行程序时都需要是唯一的，但在进程中保持不变。

这意味着我们不能在生成密钥之后，简单地使用一个 store 操作，因为这可能仅在片刻之后由其他线程复写这个生成的密钥，导致两个线程使用不同的密钥。相反，我们可以使用 compare_exchange 去确保我们仅当没有其他线程完成该操作，去存储这个密钥，否则，扔掉我们的密钥，改用存储的密钥。

```rust
fn get_key() -> u64 {
    static KEY: AtomicU64 = AtomicU64::new(0);
    let key = KEY.load(Relaxed);
    if key == 0 {
        let new_key = generate_random_key(); // 1
        match KEY.compare_exchange(0, new_key, Relaxed, Relaxed) { // 2
            Ok(_) => new_key, // 3
            Err(k) => k, // 4
        }
    } else {
        key
    }
}
```

1. 我们仅在 KEY 仍然没有初始化时，生成一个新的密钥。
2. 我们用新生成的密钥替换 KEY，但前提是它仍然是 0。
3. 如果我们将 0 换成新密钥，我们将返回新生成的密钥。`get_key()` 的新调用将返回现在存储在 KEY 中的相同新密钥。
4. 如果我们输给了另一个初始化 KEY 的线程，我们忘记我们的新密钥，而是使用来自 KEY 的密钥。

这是一个很好的例子，在这里 `compare_exchange` 比 `weak` 变体更合适。我们不会在循环中运行「比较并交换」操作，如果操作虚假失败，我们不想返回 0。

正如[“示例：惰性初始化”](#示例惰性初始化)中提到的，如果 `generate_random_key()` 需要大量时间，那么在初始化期间阻塞线程可能更有意义，以避免可能花费时间生成不会使用的密钥。Rust 标准库通过 `std::sync::Once` 和 `std::sync::OnceLock` 提供此类功能。

## 总结

（<a href="https://marabos.nl/atomics/atomics.html#summary" target="_blank">英文版本</a>）

* 原子操作是不可分割的；它们要么完整的完成，要么它们还没有发生。
* 在 Rust 中的原子操作是通过 `std::sync::atomic` 原子类型完成的，例如 `AtomicI32`。
* 不是所有原子类型在所有平台都是可获得的。
* 当涉及多个变量时，原子操作的相对顺序是棘手的。更多细节，请看[第三章](./3_Memory_Ordering.md)。
* 简单的 load 和 store 操作非常适合非常简单的基本线程间通信，例如停止标识和状态报道。
* 我们可以使用竞争条件[^2]来惰性始化，而不会引发数据竞争[^3]。
* 「获取并修改」操作允许进行一小组基本的原子修改操作，当多个线程同时修改同一个原子变量时，非常有用。
* 原子加和减运算在溢出时会默默地进行环绕（wrap around）操作。
* 「比较并交换」操作是最灵活和通用的，并且是任意其它原子操作的基石。
* *weak* 版本「比较并交换」稍微更有效。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./3_Memory_Ordering.html">下一篇，第三章：内存排序</a>
</p>

[^1]: <https://zh.wikipedia.org/wiki/内存排序>
[^2]: 竞争条件是指多个线程并发访问和修改共享数据时，其最终结果依赖于线程执行的具体顺序。在某些情况下，我们可以利用竞争条件来实现延迟初始化。也就是说，多个线程可以同时尝试对共享资源进行初始化，但只有第一个成功的线程会完成初始化，而其他线程会放弃初始化操作。
[^3]: 数据竞争是指多个线程同时访问共享数据，并且至少有一个线程进行写操作，而没有适当的同步机制来保证正确的访问顺序。
[^4]: <https://zh.wikipedia.org/wiki/比较并交换>
