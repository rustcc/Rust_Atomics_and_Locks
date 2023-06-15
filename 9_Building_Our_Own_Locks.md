# 第九章：构建我们自己的「锁」

在该章节，我们将建造属于我们自己的 `mutex`、`condition variable` 以及 `reader-writer lock`。对于它们中的任何一个，我们都会从一个非常基础的版本开始，然后扩展它以使其更高效。

因为我们并不会使用来自标准库中的锁类型（因为这将是作弊行为），因此我们将不得不使用来自[第八章](./8_Operating_System_Primitives.md)的工具，才能够在不忙碌循环（`busy-looping`[^1]）的情况下使线程等待。然而，正如我们在该章节看到的，操作系统提供的可用工具因平台而异，因此很难去构建跨平台工作的东西。

幸运地是，更多现代化操作系统都支持类似 `futex` 的功能，或者至少支持唤醒（`wake`）和等待（`wait`）操作。正如我们在[第八章](./8_Operating_System_Primitives.md)看到的，Linux 自从 2003 年就一直支持 futex 系统调用，Winodws 自从 2012 年就支持 `WaitOnAddress` 系列功能，FreeBSD 自从 2016 年就将 `_umtx_op` 作为系统调用的一部分，等等。

最让人意外的是 macOS。尽管它的内核支持这些操作，但是它并没有暴露任意稳定、公共的 C 函数给我们使用。然而，macOS 附带了一个最新版本的 `libc++`（这是一个 C++ 标准库的实现）。该标准库包含对 C++20 的支持，该版本内置了非常基础对原子等待和唤醒操作（像 `std::atomic<T>::wait()`）。尽管由于各种原因，Rust 利用这些还非常的棘手，然而，这当然是可能的，这也可以让我们在 macOS 上访问基本的像 futex 的等待和唤醒功能。

我们将不再深入研究哪些复杂的细节，而是选择利用 `crates.io` 的 `atomic-wait` crate，为我们的「锁」原语提供基础的构建模块。该 crate 提供了三个函数：`wait()`、`wake_one()` 以及 `wake_all()`。它使用我们上面讨论的特定于平台规范的实现，为所有主要的平台实现了这些功能。这意味着我们只要坚持使用这三个函数，我们不再需要考虑任何平台的特定细节。

这些函数的行为就像我们在 Linux [第八章中的“Futex”](./8_Operating_System_Primitives.md#Futex)中实现的同名函数一样，不过让我们快速回顾一下如何工作的。

* *wait(&AtomicU32, u32)*

  该函数用于等待直到原子变量不再包含给定的值。如果原子变量中存储的值等于给定值，它将阻塞。当另一个线程修改了原子变量的值，该线程需要在统一原子变量上调用以下的任意一个唤醒函数，以将等待的线程从睡眠中唤醒。

  该函数可能没有对应的唤醒操作，从而虚假地返回。因此，请确保在原子变量返回后检查其值，并在必要时重复 `wait()`。

* *wake_one(&AtomicU32)*
  
  该函数将唤醒单个线程，其是当前在相同原子变量上通过 `wait()` 方法阻塞的线程。在修改原子变量后，立即使用它，以通知一个正在等待的线程该原子变量发生了变化。

* *wake_all(&AtomicU32)*

  该函数将唤醒所有线程，其是当前在相同原子变量上通过 `wait()` 方法阻塞的线程。在修改原子变量后，立即使用它，以通知正在等待的线程该原子变量发生了变化。

仅支持 32 位原子，因为在所有主要平台这是唯一受支持的大小。

> 在[第八章中的“Futex”](./8_Operating_System_Primitives.md#Futex)，我们讨论了一个最小示例，以展示这些函数在实践中是如何使用的。如果你已经忘记，请务必在继续之前查看该示例。

为了使用 atomic-wait crate，在你的 `Cargo.toml` 中增加 `atomic-wait="1"` 到 `[dependencies]`；或者运行 `cargo add atomic-wait@1`，这样也同样位你做到这点。这三个函数在 crate 的根中定义，你可以使用 `atomic_wait::{wait, wake_one, wake_all};` 导入它们。

> 当你阅读到这篇文章时，该 crate 可能有后续的可用版本，但该章节进使用主版本为 1 的构建。后续的版本可能有不兼容的接口。

现在，我们已经有基础的知识，让我们开始吧。

## Mutex

在构建 `Mutex<T>` 时，我们将采用来自[第四章](./4_Building_Our_Own_Spin_Lock.md)的 `SpinLock<T>` 类型。在不涉及阻塞的部分，例如守护类型的设计，将保持不变。

让我们从类型定义开始。与自旋锁相比，我们必须做一个更改：而不是将 `AtomicBool` 设置为 `false` 或者 `true`，我们将使用 `AtomicU32`，将其设为 0 或者 1，所以我们可以将其与原子等待和唤醒函数一起使用。

```rust
pub struct Mutex<T> {
    /// 0: unlocked
    /// 1: locked
    state: AtomicU32,
    value: UnsafeCell<T>,
}
```

就像是自旋锁一样，我们也需要承诺 `Mutex<T>` 也可以在线程之间共享，即使它包含一个可怕的 `UnsafeCell`：

我们将增加一个 `MutexGuard` 类型，该类型实现了 `Deref` trait，以提供一个完全安全的锁接口，就像我们在[第四章：使用锁守卫的安全接口](./4_Building_Our_Own_Spin_Lock.md#使用锁守卫的安全接口)：

```rust
pub struct MutexGuard<'a, T> {
    mutex: &'a Mutex<T>,
}

impl<T> Deref for MutexGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        unsafe { &*self.mutex.value.get() }
    }
}

impl<T> DerefMut for MutexGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.mutex.value.get() }
    }
}
```

> 对于锁守卫类型的设计和操作，参见[第四章：使用锁守卫的安全接口](./4_Building_Our_Own_Spin_Lock.md#使用锁守卫的安全接口)。

在我们进入有趣的部分之前，让我们也将 `Mutex::new` 函数拿出来。

```rust
impl<T> Mutex<T> {
    pub const fn new(value: T) -> Self {
        Self {
            state: AtomicU32::new(0), // unlocked state
            value: UnsafeCell::new(value),
        }
    }

    //…
}
```

现在，我们已经完成了，然而还有剩下的两块未完成：锁定（`Mutex::lock()`）和释放（为 `MutexGuard<T>` 的 `Drop`）。

我们为自旋锁实现的锁（lock）函数，使用了一个原子交换（`swap`）操作以试图去获取锁，如果它成功的将状态从“解锁”更改到“锁定”，则返回。如果未成功，它将立刻再次尝试。

为了锁住我们的 mutex，我们将做几乎相同的操作，除了在再次尝试之前，我们会使用 `wait()` 等待：

```rust
    pub fn lock(&self) -> MutexGuard<T> {
        // Set the state to 1: locked.
        while self.state.swap(1, Acquire) == 1 {
            // If it was already locked..
            // .. wait, unless the state is no longer 1.
            wait(&self.state, 1);
        }
        MutexGuard { mutex: self }
    }
```

> 对于内存排序，与我们的自旋锁相同。对于该细节，可以参考：[第四章](https://marabos.nl/atomics/building-spinlock.html)。

注意，仅有在我们调用它时，状态仍设置为 1（锁定）时，`wait()` 函数才会阻塞，这样我们就不必担心在交换和等待调用之间失去唤醒调用的可能性。

守卫类型的 Drop 实现是负责释放 mutex。释放我们的自旋锁是简单的：仅需要设置状态到 false（释放锁）。然而，对于我们的互斥体来说，这还不够。如果有一个线程等待锁定 mutex，除非我们使用唤醒操作通知它，否则它不会知道 mutex 已经被释放。如果我们不唤醒它，它将更可能永远保持睡眠。（也许它很幸运，在正确的时间被虚假地唤醒（spurious wake-up）[^2]，但是我们不要指望这一点。）

因此，我们不仅将状态设置回 0（释放），而且还会在之后立即调用 `wake_one()`：

```rust
impl<T> Drop for MutexGuard<'_, T> {
    fn drop(&mut self) {
        // Set the state back to 0: unlocked.
        self.mutex.state.store(0, Release);
        // Wake up one of the waiting threads, if any.
        wake_one(&self.mutex.state);
    }
}
```

唤醒一个线程就足够了，因为即使有多个线程在等待，也仅有其中的一个线程能够认领锁。锁定它的下一个线程将在锁定完成后唤醒另一个线程，以此类推。同时唤醒多个线程，只会让这些线程感到不满，浪费宝贵的处理器时间，因为其中除了一个幸运的线程能够获取锁，其它线程都会意识到自己失去机会后，从而再次进入休眠状态。

注意，不能保证我们唤醒的每一个线程都能抓住锁。另一个线程可能仍然在它有机会之前立刻抓住锁。

这里做出的一个重要的观察是，如果没有等待和唤醒功能，这个 mutex 的实现在技术上仍然是正确的（即内存安全）。因为 `wait()` 操作可以虚假地唤醒，我们无法对他何时返回作出任何假设。我们仍然得去管理我们锁定原语的状态。如果我们移除等待和唤醒函数调用，我们的 mutex 将与我们的自旋锁状态基本相同。

一般来说，从内存安全方面，原子等待和唤醒函数从不会影响正确性。它们仅是一个（非常重要的）优化，以避免忙碌循环。这并不意味着由任何实际标准都无法使用的低效锁将是“正确的”，但是尝试去推理关于不安全的 Rust 代码时，这种见解可能是有帮助的。

<div style="border:medium solid green; color:green;">
  <h2 style="text-align: center;">Lock API</h2>
  如果你正在计划将实现 Rust 锁当作一个新的爱好，那么你可能很快对涉及提供安全接口的样板代码感到厌烦。也就是说，UnsafeCell、Sync 实现、守护类型、Deref 实现等等。

  crate.io 上的 `lock_api` 可以自动地去处理这些事情。你仅需要制作一个锁定状态的类型，并通过（不安全）`lock_api::RawMutex` trait 提供（不安全）锁定和释放功能。`lock_api::Mutex` 类型将根据你的锁实现，提供一个完全安全的和符合人体工学的 mutex 类型作为返回，包括 mutex 守护。
</div>

### 避免系统调用

到目前为止，我们 mutex 中最慢的部分是等待和唤醒，因为这（可能）导致一个*系统调用*，即对操作系统内核的调用。像这样与内核交互是一个相当复杂的过程，往往相当缓慢，尤其是与原子操作比较。因此，对于一个高性能 mutex 的实现，我们应该尽可能尝试去避免等待和唤醒调用。

幸运地是，我们已经走了一半。因为在 `wait()` 调用之前，我们的锁定函数 `while` 循环才会检查状态，因此在 mutex 未锁定的情况下，等待操作在这种情况下将完全地跳过，我们并不需要等待。然而，我们在解锁时，会无条件地调用 `wake_one()` 函数。

如果我们知道没有其它线程在等待，我们可以跳过 `wake_one()`。为了知道是否有等待线程，我们需要自己跟踪这些信息。

我们可以通过将“锁定”状态分割成两个单独的状态来完成这一点：“没有等待者的锁定”和“有等待者的锁定”。我们将使用值 1 和 2 做这些，并更新我们文档中结构体定义中的状态字段的注释。

```rust
pub struct Mutex<T> {
    /// 0: unlocked
    /// 1: locked, no other threads waiting
    /// 2: locked, other threads waiting
    state: AtomicU32,
    value: UnsafeCell<T>,
}
```

现在，对于一个释放的 mutex，我们的 lock 函数仍然需要将状态设置为 1 才能锁定它。然而，如果它已经锁定，我们的 lock 函数需要在睡眠之前将状态设置为 2，以便释放（unlock）函数可以判断这有一个等待线程。

为了做到这一点，我们首先使用 `compare-and-exchange` 函数，试图改变状态从 0 到 1。如果成功，我们就已经锁定了 mutex，并且我们知道，这没有其它的等待者，因为 mutex 之前没有锁定。如果它失败了，那一定是因为 mutex 当前被锁定（在状态 1 或 2）。在这种情况下，我们将使用一个原子交换操作将其设置为 2。如果交换操作返回 1 或 2 的旧值，那这意味着 mutex 事实上仍然被锁定，并且仅有这样，我们才会使用 `wait()` 去阻塞，直到它改变。如果交换操作返回 0，这意味着我们通过改变它的状态从 0 到 2 已经成功锁定 mutex。

```rust
    pub fn lock(&self) -> MutexGuard<T> {
        if self.state.compare_exchange(0, 1, Acquire, Relaxed).is_err() {
            while self.state.swap(2, Acquire) != 0 {
                wait(&self.state, 2);
            }
        }
        MutexGuard { mutex: self }
    }
```

现在，我们释放功能可以在必要时通过跳过 `wake_one()` 调用来利用新信息。我们现在使用交换操作，而不是仅仅存储一个 0 去释放 mutex，这样我们就可以检查它之前的值。仅当值为 2 时，我们将继续唤醒一个线程：

```rust
impl<T> Drop for MutexGuard<'_, T> {
    fn drop(&mut self) {
        if self.mutex.state.swap(0, Release) == 2 {
            wake_one(&self.mutex.state);
        }
    }
}
```

注意，将状态设置回 0 后，它不再指示是否有任何其它的等待线程。唤醒的线程负责将状态设置为 2，以确保没有任何其它的线程忘记。这是为什么在我们的锁定功能中「比较和交换」（`compare-and-exchange`）操作不是我们 while 循环的一部分。

这确实以为这，每当线程在锁定时不得不 `wait()`，当释放时，它将也调用 `wake_one()`。然而，更重要的是，*在未考虑的情况下*，线程不试图获取锁的理想状况下，完全地避免了 `wait()` 和 `wake_one()` 调用。

图 9-1 展示了两个线程同时尝试锁定我们的 mutex 操作的情况下的 happens-before 关系。首先线程通过改变状态从 0 到 1 锁定 mutex。此时，第二个线程将无法获取锁，并且在改变状态从 1 到 2 后进入睡眠。当第一个线程释放 mutex 时，它会交换状态回 0。因为是 2，表示一个等待线程，它调用 `wake_one()` 来唤醒第二个线程。注意，我们不能依赖于唤醒和等待操作之间的任何 happens-before 关系。虽然唤醒操作可能是负责唤醒等待线程的操作，但 happens-before 关系是通过 `acquire` 交换操作建立的，观察 `release` 交换操作存储的值。

![ ](./picture/raal_0901.png)
*图 9-1。两个线程之间 happens-before 的关系同时试图锁定我们的 mutex。*

### 进一步优化

在这一点上，我们似乎没有什么可以进一步优化的了。在未考虑的情况下，我们执行零系统调用，并且剩下的只是两个非常简单的原子操作。

避免等待和唤醒操作的唯一方式是回到我们的旋转锁实现。尽管自旋通常是非常低效的，但它至少避免了系统调用的潜在开销。唯一能提高自旋效率的情况是等待时间很短的情况下。

对于锁定一个 mutex，这仅在当前持有锁的线程在不同的处理器内核上并行运行，并且非常短暂地保留锁定的情况下发生。然而，这是一个非常常见的案例。

我们可以尝试将两种方法的优点结合起来，在调用 `wait()` 之前进行非常短暂的自旋。这样，如果锁被很快释放，我们根本不需要调用 `wait()`，但我们仍然避免了消耗其它线程更好利用的不合理的处理器时间。

实现这个仅需要改变我们的锁定功能。

为了在未考虑的情况下尽可能保持性能，我们将在锁定功能开始时保留原始的 `compare-and-exchange` 操作。我们将使自旋等待作为一个单独的功能。

```rust
impl<T> Mutex<T> {
    //…

    pub fn lock(&self) -> MutexGuard<T> {
        if self.state.compare_exchange(0, 1, Acquire, Relaxed).is_err() {
            // The lock was already locked. :(
            lock_contended(&self.state);
        }
        MutexGuard { mutex: self }
    }
}

fn lock_contended(state: &AtomicU32) {
    //…
}
```

在 `lock_contended` 中，在继续等待循环之前，我们可能简单地重复相同的 `compare-and0-exchange` 操作几百次。然而，一个 `compare-and-exchange` 操作通常尝试去获取相关缓存行（cache line）的独占访问（[第七章 MESI 协议](./7_Understanding_the_Processor.md#mesi-协议)），当重复执行时，这可能比简单的加载操作更昂贵。

考虑到这一点，我们来到了以下 `lock_contented` 的实现：

```rust
fn lock_contended(state: &AtomicU32) {
    let mut spin_count = 0;

    while state.load(Relaxed) == 1 && spin_count < 100 {
        spin_count += 1;
        std::hint::spin_loop();
    }

    if state.compare_exchange(0, 1, Acquire, Relaxed).is_ok() {
        return;
    }

    while state.swap(2, Acquire) != 0 {
        wait(state, 2);
    }
}
```

首先，我们自旋到 100 次，这是我们像在[第四章](./4_Building_Our_Own_Spin_Lock.md)时使用 *`spin loop` 提示*。只要 mutex 没被锁定，并且没有等待者，我们就会自旋。如果另一个线程已经在等待，这意味着它放弃了自旋，因为它花费的时间太长，这可能表明自旋对这个线程也不是很有用。

> 一百个周期的自旋时间大多数是任意选择的。迭代锁花费的时间和系统调用期间（我们试图避免）很大程度上取决于平台。大范围的基础测试可以帮助我们选择一个正确的数字，但是不幸地是，没有一个准确的回答。
>
>Rust 标准库的 `std::sync::Mutex` 的 Linux 实现，至少是 Rust 1.66.0，使用至少是 100 次的自旋计数。

锁定状态改变后，我们再次尝试将其设定为 1 来锁定它，然后我们再放弃并且开始自旋等待。正如我们之前讨论的，我们调用 `wait()` 后，我们不能在通过设定它的状态到 1 来锁定 mutex，因为这可能导致其它的等待者被忘记。

<div style="border:medium solid green; color:green;">
  <h2 style="text-align: center;">Cold 和 Inline 属性</h2>
  你可以增加 `#[cold]` 属性到 `lock_contented` 函数定义，以帮助编译器理解在常见（未考虑）情况下不调用这个函数，这对 lock 方法的优化有帮助。

  额外地，你也可以增加 `#[inline]` 属性到 Mutex 和 MutexGuard 防区，以通知编译器将其内联可能是一个好主意：将生成的指令将其放置在调用方法的地方。一般来说，是否能提高性能很难说，但对于这些非常小的功能，通常如此。
</div>

### 基准测试

测试 mutex 实现的性能是很难的。写基准测试和得到一些数字很容易，但是很难去得到一些有意义的数字。

优化 mutex 的实现以在特定基准测试表现良好是相对容易的，但这并没有很有用。毕竟，关键是去做一些在真实世界良好的东西，而不仅是测试程序。

我们将试图去写两个简单的基础测试，表明我们的优化至少对一些用例产生了一些积极影响，但请注意，任何结论在不同场景都不一定成立。

在我们的第一次测试中，我们将创建一个 Mutex 并锁定和释放它几百万次，所有都在同一线程上，以测量它花费总的时间。这是对琐碎未讨论场景的测试，其中从来没有任何需要唤醒的线程。希望这将向我们展示 2-state 和 3-state 版本的差异。

```rust
fn main() {
    let m = Mutex::new(0);
    std::hint::black_box(&m);
    let start = Instant::now();
    for _ in 0..5_000_000 {
        *m.lock() += 1;
    }
    let duration = start.elapsed();
    println!("locked {} times in {:?}", *m.lock(), duration);
}
```

> 我们使用 `std::hint::black_box`（像我们在[第七章“对性能的影响”](./7_Understanding_the_Processor.md#对性能的影响)）去强制编译器假设有更多的代码去访问 mutex，阻止它优化循环或者锁定操作。

结果因硬件和操作系统不同而不同。在一台配备最新 AMD 处理器的特定 Linux 计算机上尝试，对于我们为优化的 2-state mutex 花费时间大约 400ms，对于我们优化过后的 3-state mutex 大约 40ms。一个因素获得十倍的性能提升！在另一个有着老式 Intel 处理器 Linux 计算机中，差异甚至更大：大约 1800ms 比上 60ms。这证实了，第三个状态的加入确实是一个非常大的优化。

然而，在 macOS 上运行，这回产生一个完全不同的结果：这两个版本大约都是 50ms，这展示了非常高的平台依赖。

事实证明，我们在 macOS 上使用的 libc++ 的 `std::atomic<T>::wake()` 实现，已经进行了自己的内部管理，独立于内核，以避免不必要的系统调用。Windows 上的 `WakeByAddressSingle()` 也是如此。

避免调用这些函数仍然可能带来稍微更好的性能，因为它们的实现并非是微不足道的，尤其因为它们并不能在原子变量本身中存储任何信息。然而，如果我们的目标仅针对这些操作系统，我们将质疑在我们的 mutex 中增加第三个状态是否是值得的。

为了看看我们的自旋优化是否有任何积极的影响，我们需要一个不同的基准测试：一个有着大量争用的测试，多个线程反复尝试去锁定一个已经上锁的 mutex。

让我们尝试一个场景，四个线程都尝试锁定和释放 mutex 上万次：

```rust
fn main() {
    let m = Mutex::new(0);
    std::hint::black_box(&m);
    let start = Instant::now();
    thread::scope(|s| {
        for _ in 0..4 {
            s.spawn(|| {
                for _ in 0..5_000_000 {
                    *m.lock() += 1;
                }
            });
        }
    });
    let duration = start.elapsed();
    println!("locked {} times in {:?}", *m.lock(), duration);
}
```

注意，这是一个极端和不切实际的场景。mutex 仅能保留一个极短的时间（进去增加一个整数），并且在释放后，线程将立刻试图再次锁定 mutex。不同的场景将导致非常不同的结果。

让我们像以前一样，在两台相同的 Linux 计算机上运行基准测试。在那台较旧的 Intel 处理器的计算机上，不使用自旋版本大约需要 900ms，而使用自旋版本大约需要 750ms。这是一个改进。而在那台新的 AMD 处理器计算机上，我们得到一个相反的结果：不使用自旋大约 650ms，而使用自旋大约需要 800ms。

总之，不幸的是，关于自旋是否真正提高性能的答案是“这取决于平台”，即使只看一个场景。

## Condition Variable

### 避免系统调用2

### 避免虚假唤醒

## Reader-Writer Lock

### 避免 writer 忙碌循环

### 避免 writer 陷入饥饿

## 总结

[^1]: <https://zh.wikipedia.org/wiki/忙碌等待>
[^2]: 虚假唤醒是一个线程在没有收到明确的信号的情况下，从等待状态中被唤醒
