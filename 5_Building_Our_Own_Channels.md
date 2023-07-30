# 第五章：构建我们自己的 Channel

（<a href="https://marabos.nl/atomics/building-channels.html" target="_blank">英文版本</a>）

*Channel* 可以被用于在线程之间发送数据，并且它有很多变体。一些 channel 仅能在一个发送者和一个接收者之间使用，而另一些可以在任意数量的线程之间发送，或者甚至允许多个接收者。一些 channel 是阻塞的，这意味着接收（有时也包括发送）是一个阻塞操作，这会使线程进入睡眠状态，直到你的操作完成。一些 channel 针对吞吐量进行优化，而另一些针对低延迟进行优化。

这些变体是无穷尽的，没有一种通用版本在所有场景都适合的。

在该章节，我们将实现一个相对简单的 channel，不仅可以探索更多的原子应用，同时也可以了解如何在 Rust 类型系统中捕获我们的需求和假设。

## 一个简单的以 mutex 为基础的 Channel

（<a href="https://marabos.nl/atomics/building-channels.html#a-simple-mutex-based-channel" target="_blank">英文版本</a>）

一个基础的 channel 实现并不需要任何关于原子的知识。我们可以接收 `VecDeque`，它根本上是一个 `Vec`，允许在两端高效地添加和移除元素，并使用 Mutex 保护它，以允许多个线程访问。然后，我们使用 `VecDeque` 作为已发送但尚未接受数据的消息队列。任何想要发送消息的线程只需要将其添加到队列的末尾，而任何想要接受消息的线程只需从队列的前端删除一个消息。

还有一点需要补充，用于将接收操作阻塞的 Condvar（参见[第一章“条件变量”](./1_Basic_of_Rust_Concurrency.md#条件变量)），当有新的消息，它会通知正在等待的接收者。

这样做的实现可能非常简短且相对直接，如下所示：

```rust
pub struct Channel<T> {
    queue: Mutex<VecDeque<T>>,
    item_ready: Condvar,
}

impl<T> Channel<T> {
    pub fn new() -> Self {
        Self {
            queue: Mutex::new(VecDeque::new()),
            item_ready: Condvar::new(),
        }
    }

    pub fn send(&self, message: T) {
        self.queue.lock().unwrap().push_back(message);
        self.item_ready.notify_one();
    }

    pub fn receive(&self) -> T {
        let mut b = self.queue.lock().unwrap();
        loop {
            if let Some(message) = b.pop_front() {
                return message;
            }
            b = self.item_ready.wait(b).unwrap();
        }
    }
}
```

注意，我们并没有使用任意的原子操作或者不安全代码，也不需要考虑 `Send` 或者 `Sync`。编译器理解 Mutex 的接口以及保证该提供什么类型，并且会隐式地理解，如果 `Mutex<T>` 和 Condvar 都可以在线程之间安全共享，那么我们的 `Channel<T>` 也可以这么做。

我们的 `send` 函数锁定 mutex，然后从队列的末尾推入消息，并且使用条件变量在解锁队列后直接通知可能等待的接收者。

`receive` 函数也锁定 mutex，然后从队列的首部弹出消息，但如果仍然没有可获得的消息，则会使用条件变量去等待。

> 记住，`Condvar::wait` 方法将在等待时解锁 Mutex，并在返回之前重新锁定它。因此，我们的 `receive` 函数将不会在等待时锁定 mutex。

尽管这个 channel 在使用上是非常灵活的，因为它允许任意数量的发送和接收线程，但在很多情况下，它的实现远非最佳。即使有大量的消息准备好被接收，任意的发送或者接收操作将短暂地阻塞任意其它的发送或者接收操作，因为它们必须都锁定相同的 mutex。如果 `VecDeque::push` 必须增加 VecDeque 的容量时，所有的发送和接收线程将不得不等待该线程完成重新分配容量，这在某些情况下是不可接受的。

另一个可能不可取的属性是，该 channel 的队列可能会无限制地增长。没有什么能阻止发送者以比接收者更高的速度持续发送新消息。

## 一个不安全的一次性 Channel

（<a href="https://marabos.nl/atomics/building-channels.html#an-unsafe-one-shot-channel" target="_blank">英文版本</a>）

channel 的各种用例几乎是无止尽的。然而，在本章的剩余部分，我们将专注于一种特定类型的用例：恰好从一个线程向另一个线程发送一条消息。为此类用例设计的 channel 通常被称为 *一次性*（one-shot）channel。

我们采用上述基于 `Mutex<VecDeque>` 的实现，并且将 `VecDeque` 替换为 `Option`，从而将队列的容量减小到恰好一个消息。这样可以避免内存浪费，但仍然会存在使用 Mutex 的一些缺点。我们可以通过使用原子操作从头构建我们自己的一次性 channel 来避免这个问题。

首先，让我们构建一个最小化的一次性 channel 实现，不需要考虑它的接口。在本章的稍后，我们将探索如何改进其接口以及如何与 Rust 类型相结合，为 channel 的用于提供愉快的体验。

我们需要开始的工具基本上与我们在[第四章](./4_Building_Our_Own_Spin_Lock.md)使用的 `SpinLock<T>` 基本相同：一个用于存储的 `UnsafeCell` 和用于指示状态的 `AtomicBool`。在该示例中，我们使用原子布尔值去指示消息是否准备好用于消费。

在发送消息之前，channel 是“空的”并且不包含任何类型为 T 的消息。我们可以在 cell 中使用 `Option<T>`，以允许 T 缺失。然而，这可能会浪费宝贵的内存空间，因为我们的原子布尔值已经告诉我们是否有消息。相反，我们可以使用 `std::mem::MaybeUninit<T>`，它本质上是裸露的 `Option<T>` 的不安全版本：它要求用户手动跟踪其是否已初始化，几乎整个接口都是不安全的，因为它不能执行自己的检查。

综合来看，我们从这个结构体定义开始我们的第一次尝试：

```rust
use std::mem::MaybeUninit;

pub struct Channel<T> {
    message: UnsafeCell<MaybeUninit<T>>,
    ready: AtomicBool,
}
```

就像我们的 `SpinLock<T>` 一样，我们需要告诉编译器，我们的 channel 在线程之间共享是安全的，或者至少只要 T 是 `Send` 的：

```rust
unsafe impl<T> Sync for Channel<T> where T: Send {}
```

一个新的 channel 是空的，将 `ready` 设置为 false，并且消息仍然没有初始化：

```rust
impl<T> Channel<T> {
    pub const fn new() -> Self {
        Self {
            message: UnsafeCell::new(MaybeUninit::uninit()),
            ready: AtomicBool::new(false),
        }
    }

    // …
}
```

要发送消息，它首先需要存储在 cell 中，之后我们可以通过将 ready 标识设置为 true 来将其释放给接收者。试图做这个超过一次是危险的，因为设置 ready 标识后，接收者可能在任意时刻读取消息，这可能会与第二次发送消息产生数据竞争。目前，我们通过使方法不安全并为它们留下备注，将此作为用户的责任：

```rust
    /// 安全性：仅能调用一次！
    pub unsafe fn send(&self, message: T) {
        (*self.message.get()).write(message);
        self.ready.store(true, Release);
    }
```

在上面这个片段中，我们使用 `UnsafeCell::get` 方法去获取指向 `MaybeUninit<T>` 的指针，并且通过不安全地解引用它来调用 `MaybeUninit::write` 进行初始化。当错误使用时，这可能导致未定义行为，但我们将这个责任转移到了调用方身上。

对于内存排序，我们需要使用 release 排序，因为原子的存储有效地将消息释放给接收者。这确保了如果接收线程从 `self.ready` 以 acquire 排序加载 true，则消息的初始化将从接受线程的角度完成。

对于接收，我们暂时不会提供阻塞的接口。相反，我们将提供两个方法：一个用于检查是否有可用消息，另一个用于接收消息。我们将让我们的 channel 用户决定是否使用[线程阻塞](./1_Basic_of_Rust_Concurrency.md#线程阻塞)的方法来阻塞。

以下是完成此版本我们 channel 的最后两种方法：

```rust
    pub fn is_ready(&self) -> bool {
        self.ready.load(Acquire)
    }

    /// 安全性：仅能调用一次，
    /// 并且仅在 is_ready() 返回 true 之后调用！
    pub unsafe fn receive(&self) -> T {
        (*self.message.get()).assume_init_read()
    }
```

虽然 `is_ready` 方法可以始终地安全调用，但是 `receive` 方法使用了 `MaybeUninit::assume_init_read()`，这不安全地假设它已经被初始化，且不会用于生成非 `Copy` 对象的多个副本。就像 `send` 方法一样，我们只需通过将函数本身标记为不安全来将这个问题交给用户解决。

结果是一个在技术上可用的 channel，但它用起来不便并且通常令人失望。如果正确使用，它会按预期进行操作，但有很多微妙的方式去错误地使用它。

多次调用 send 可能会导致数据竞争，因为第二个发送者在接收者尝试读取第一条消息时可能正在覆盖数据。即使接收操作得到了正确的同步，从多个线程调用 send 可能会导致两个线程尝试并发地写入 cell，再次导致数据竞争。此外，多次调用 `receive` 会导致获取两个消息的副本，即使 T 不实现 `Copy` 并且因此不能安全地进行复制。

更微妙的问题是我们的 Channel 缺乏 `Drop` 实现。`MaybeUninit` 类型不会跟踪它是否已经初始化，因此它在被丢弃时不会自动丢弃其内容。这意味着如果发送了一条消息但从未被接收，该消息将永远不会被释放。这并不是不正确的，但仍然是要避免。在 Rust 中，泄漏被普遍认为是安全的，但通常只有作为另一个泄漏的后果才是可接受的。例如，泄漏 Vec 的内存也会泄漏其内容，但正常使用 Vec 不会导致任何内存泄漏。

由于我们让用户对一切负责，不幸的事故只是时间问题。

## 通过运行时检查来达到安全

（<a href="https://marabos.nl/atomics/building-channels.html#safety-through-runtime-checks" target="_blank">英文版本</a>）

为了提供更安全的接口，我们可以增加一些检查，以确保误用会导致 panic 并显示清晰的错误信息，这比未定义行为要好得多。

让我们在消息准备好之前调用 `receive` 方法的问题开始处理。这个问题很容易解决，我们只需要在尝试读消息之前让 receive 方法验证 ready 标识即可：

```rust
    /// 如果仍然没有消息可获得，panic。
    ///
    /// 提示，首先使用 `is_ready` 检查。
    ///
    /// 安全地：仅能调用一次。
    pub unsafe fn receive(&self) -> T {
        if !self.ready.load(Acquire) {
            panic!("no message available!");
        }
        (*self.message.get()).assume_init_read()
    }
```

该函数仍然是不安全的，因为用户仍然需要确保只调用一次，但未能首先检查 `is_ready()` 不再导致未定义行为。

因为我们现在在 `receive` 方法里有一个 `ready` 标识的 acquire-load 操作，其提供了必要的同步，我们可以在 `is_ready` 中使用 Relaxed 内存排序，因为该操作现在仅用于指示目的：

```rust
    pub fn is_ready(&self) -> bool {
        self.ready.load(Relaxed)
    }
```

> 记住，ready 上的总修改顺序（参见[第三章的“Relaxed 排序”](./3_Memory_Ordering.md#relaxed-排序)）保证了从 `is_ready` 加载 true 之后，receive 也能看到 true。无论 is_ready 使用的内存排序如何，都不会出现 `is_ready` 返回 true，`receive()` 仍然出现 panic 的情况。

下一个要解决的问题是，当调用 receive 不止一次时会发生什么。通过在接收方法中将 `ready` 标识设置回 false，我们也可以很容易地导致 panic，例如：

```rust
    /// 如果仍然没有消息可获得，
    /// 或者消息已经被消费 panic。
    ///
    /// 提示，首先使用 `is_ready` 检查。
    pub fn receive(&self) -> T {
        if !self.ready.swap(false, Acquire) {
            panic!("no message available!");
        }
        // Safety: We've just checked (and reset) the ready flag.
        unsafe { (*self.message.get()).assume_init_read() }
    }
```

我们仅是将 load 操作更改为 swap 操作（交换的值为 `false`），突然之间，receive 方法在任何情况下都可以安全地调用。该函数不再标记为不安全。我们现在承担了不安全代码的责任，而不是让用户负责一切，从而减轻了用户的压力。

对于 send，事情稍微复杂一点。为了阻止多个 send 调用同时访问 cell，我们需要知道是否另一个 send 调用已经开始。ready 标识仅告诉我们是否另一个 send 调用已经完成，所以这还不够。

让我们增加第二个标识，命名为 `in_use`，以指示该 channel 是否已经在使用：

```rust
pub struct Channel<T> {
    message: UnsafeCell<MaybeUninit<T>>,
    in_use: AtomicBool, // 新增！
    ready: AtomicBool,
}

impl<T> Channel<T> {
    pub const fn new() -> Self {
        Self {
            message: UnsafeCell::new(MaybeUninit::uninit()),
            in_use: AtomicBool::new(false), // 新增！
            ready: AtomicBool::new(false),
        }
    }

    //…
}
```

现在我们需要做的就是在访问 cell 之前，在 send 方法中，将 `in_use` 设置为 true，如果它已经由另一个线程设置，则 panic：

```rust
    /// 当尝试发送不止一次消息时，Panic。
    pub fn send(&self, message: T) {
        if self.in_use.swap(true, Relaxed) {
            panic!("can't send more than one message!");
        }
        unsafe { (*self.message.get()).write(message) };
        self.ready.store(true, Release);
    }
```

我们可以为原子 swap 操作使用 relaxed 内存排序，因为 `in_use` 的*总修改顺序*（参见[第三章“Relaxed 排序”](./3_Memory_Ordering.md#relaxed-排序)）保证了在 in_use 上只会有一个 swap 操作返回的 false，而这是 send 方法尝试访问 cell 的唯一情况。

现在我们拥有了一个完全安全的接口，尽管还有一个问题未解决。最后一个问题出现在发送一个永远不会被接收的消息时：它将从不会被丢弃。虽然这不会导致未定义行为，并且在安全代码中是允许的，但确实应该避免这种情况。

由于我们在 receive 方法中重置了 ready 标识，修复这个问题很容易：ready 标识指示是否在 cell 中尚未接受的消息需要被丢弃。

在我们的 Channel 的 Drop 实现中，我们不需要使用一个原子操作去检查原子 ready 标识，因为只有对象完全被正在丢弃它的线程所拥有的时候，且没有任何未解除借用的情况下，才能丢弃一个对象。这意味着，我们可以使用 `AtomicBool::get_mut` 方法，它接受一个独占引用（`&mut self`），以证明原子访问是不必要的。对于 UnsafeCell 也是一样，通过 `UnsafeCell::get_mut` 方法来来获取独占引用。

使用它，这是我们完全安全且不泄漏的 channel 的最后一部分：

```rust
impl<T> Drop for Channel<T> {
    fn drop(&mut self) {
        if *self.ready.get_mut() {
            unsafe { self.message.get_mut().assume_init_drop() }
        }
    }
}
```

我们试试吧！

由于我们的 channel 仍没有提供一个阻塞的接口，我们将手动地使用线程阻塞去等待消息。只要没有消息准备好，接收线程将 `park()` 自身，并且发送线程将在发送东西后，立刻 `unpark()` 接收者。

这里是一个完整的测试程序，通过我们的 `Channel` 从第二个线程发送字符串字面量“hello world”到主线程：

```rust
fn main() {
    let channel = Channel::new();
    let t = thread::current();
    thread::scope(|s| {
        s.spawn(|| {
            channel.send("hello world!");
            t.unpark();
        });
        while !channel.is_ready() {
            thread::park();
        }
        assert_eq!(channel.receive(), "hello world!");
    });
}
```

该程序编译、运行和干净地退出，表明我们的 Channel 正常工作。

如果我们复制了 send 行，我们也可以在运行中看到我们的安全检查，当运行程序时，产生以下 panic：

```txt
thread '<unnamed>' panicked at 'can't send more than one message!', src/main.rs
```

尽管 panic 程序并不出色，但是程序可靠的 panic 比可能的未定义行为错误好太多。

<div class="box">
  <h2 style="text-align: center;">为 Channel 状态使用单原子</h2>
  
  <p>如果你对 channel 实现还不满意，这里有一个微妙的变体，可以节省一字节的内存。</p>

  <p>我们使用单个原子 <code>AtomicU8</code> 表示所有 4 个状态，而不是使用两个分开的布尔值去表示 channel 的状态。我们必须使用 <code>compare_exchange</code> 来原子地检查 channel 是否处于预期状态，并将其更改为另一个状态，而不是原子交换布尔值。</p>

  <pre>const EMPTY: u8 = 0;
const WRITING: u8 = 1;
const READY: u8 = 2;
const READING: u8 = 3;

pub struct Channel&lt;T&gt; {
    message: UnsafeCell&lt;MaybeUninit&lt;T&gt;&gt;,
    state: AtomicU8,
}

unsafe impl&lt;T: Send&gt; Sync for Channel&lt;T&gt; {}

impl&lt;T&gt; Channel&lt;T&gt; {
    pub const fn new() -> Self {
        Self {
            message: UnsafeCell::new(MaybeUninit::uninit()),
            state: AtomicU8::new(EMPTY),
        }
    }

    pub fn send(&self, message: T) {
        if self.state.compare_exchange(
            EMPTY, WRITING, Relaxed, Relaxed
        ).is_err() {
            panic!("can't send more than one message!");
        }
        unsafe { (*self.message.get()).write(message) };
        self.state.store(READY, Release);
    }

    pub fn is_ready(&self) -> bool {
        self.state.load(Relaxed) == READY
    }

    pub fn receive(&self) -> T {
        if self.state.compare_exchange(
            READY, READING, Acquire, Relaxed
        ).is_err() {
            panic!("no message available!");
        }
        unsafe { (*self.message.get()).assume_init_read() }
    }
}

impl&lt;T&gt; Drop for Channel&lt;T&gt; {
    fn drop(&mut self) {
        if *self.state.get_mut() == READY {
            unsafe { self.message.get_mut().assume_init_drop() }
        }
    }
}</pre>

</div>

## 通过类型来达到安全

（<a href="https://marabos.nl/atomics/building-channels.html#safety-through-types" target="_blank">英文版本</a>）

尽管我们已经成功地保护了我们 Channel 的用户免受未定义行为的问题，但是如果它们偶尔地不正确使用它，它们仍然有 panic 的风险。理想情况下，编译器将在程序运行之前检查正确的用法并指出滥用。

让我们来看看调用 send 或 receive 不止一次的问题。

为了防止函数被多次调用，我们可以让它*按值*接受参数，对于非 `Copy` 类型，这将消耗对象。对象被消耗或移动后，它会从调用者那里消失，防止它再次被使用。

通过将调用 send 或 receive 表示的能力作为单独的（非 `Copy`）类型，并在执行操作时消费对象，我们可以确保每个操作只能发生一次。

这给我们带来了以下接口设计，而不是单个 `Channel` 类型，一个 channel 由一对 `Sender` 和 `Receiver` 表示，它们各自都有以值接收 `self` 的方法：

```rust
pub fn channel<T>() -> (Sender<T>, Receiver<T>) { … }

pub struct Sender<T> { … }
pub struct Receiver<T> { … }

impl<T> Sender<T> {
    pub fn send(self, message: T) { … }
}

impl<T> Receiver<T> {
    pub fn is_ready(&self) -> bool { … }
    pub fn receive(self) -> T { … }
}
```

用户可以通过调用 `channel()` 创建一个 channel，这将给他们一个 Sender 和一个 Receiver。它们可以自由地传递每个对象，将它们移动到另一个线程，等等。然而，它们最终不能获得其中任何一个的多个副本，这保证了 send 和 receive 仅被调用一次。

为了实现这一点，我们需要为我们的 UnsafeCell 和 AtomicBool 找到一个位置。之前，我们仅有一个具有这些字段的结构体，但是现在我们有两个单独的结构体，每个结构体都可能存在更长的时间。

<a class="indexterm" id="index-Arc-usingforchannelstate"></a>
因为 sender 和 receiver 将需要共享这些变量的所有权，我们将使用 Arc（[第一章“引用计数”](./1_Basic_of_Rust_Concurrency.md#引用计数)）为我们提供引用计数共享内存分配，我们将在其中存储共享的 Channel 对象。正如以下展示的，Channel 类型不必是公共的，因为它的存在是与用户无关的细节。

```rust
pub struct Sender<T> {
    channel: Arc<Channel<T>>,
}

pub struct Receiver<T> {
    channel: Arc<Channel<T>>,
}

struct Channel<T> { // 不再 `pub`
    message: UnsafeCell<MaybeUninit<T>>,
    ready: AtomicBool,
}

unsafe impl<T> Sync for Channel<T> where T: Send {}
```

就像之前一样，我们在 T 是 Send 的情况下为 `Channel<T>` 实现了 `Sync`，以允许它跨线程使用。

注意，我们不再像我们之前 channel 实现中的那样，需要 `in_use` 原子布尔值。它仅通过 send 来检查它有没有被调用超过一次，现在通过类型系统静态地保证。

channel 函数去创建一个 channel 和一对发送者和接收者，它与我们之前的 `Channel::new` 函数类似，除了将 Channel 包装在 Arc 中，也将该 Arc 和其克隆包装在 Sender 和 Receiver 类型中：

```rust
pub fn channel<T>() -> (Sender<T>, Receiver<T>) {
    let a = Arc::new(Channel {
        message: UnsafeCell::new(MaybeUninit::uninit()),
        ready: AtomicBool::new(false),
    });
    (Sender { channel: a.clone() }, Receiver { channel: a })
}
```

`send`、`is_ready` 和 `receive` 方法与我们之前实现的方法基本相同，但有一些区别：

* 它们现在被移动到它们各自的类型中，因此只有（单个）发送者可以发送，并且只有（单个）接收者可以接收。
* 发送和接收现在通过值而不是引用来接收 `self`，以确保它们每个只能被调用一次。
* 发送不再 panic，因为它的先决条件（只被调用一次）现在被静态保证。

所以，他们现在看起来像这样：

```rust
impl<T> Sender<T> {
    /// 从不会 panic :)
    pub fn send(self, message: T) {
        unsafe { (*self.channel.message.get()).write(message) };
        self.channel.ready.store(true, Release);
    }
}

impl<T> Receiver<T> {
    pub fn is_ready(&self) -> bool {
        self.channel.ready.load(Relaxed)
    }

    pub fn receive(self) -> T {
        if !self.channel.ready.swap(false, Acquire) {
            panic!("no message available!");
        }
        unsafe { (*self.channel.message.get()).assume_init_read() }
    }
}
```

receive 函数仍然可以 panic，因为用户可能仍然会在 `is_ready()` 返回 `true` 之前调用它。它仍然使用 `swap` 将 ready 标识设置回 false（而不仅仅是 load 操作），以便 Channel 的 Drop 实现知道是否有需要删除的未读消息。

该 Drop 实现与我们之前实现的完全相同：

```rust
impl<T> Drop for Channel<T> {
    fn drop(&mut self) {
        if *self.ready.get_mut() {
            unsafe { self.message.get_mut().assume_init_drop() }
        }
    }
}
```

当 `Sender<T>` 或者 `Receiver<T>` 被丢弃时，`Arc<Channel<T>>` 的 Drop 实现将递减对共享内存分配的引用计数。当丢弃到第二个时，计数达到 0，并且 `Channel<T>` 自身被丢弃。这将调用我们上面的 Drop 实现，如果已发送但未收到消息，我们将丢弃该消息。

让我们尝试它：

```rust
fn main() {
    thread::scope(|s| {
        let (sender, receiver) = channel();
        let t = thread::current();
        s.spawn(move || {
            sender.send("hello world!");
            t.unpark();
        });
        while !receiver.is_ready() {
            thread::park();
        }
        assert_eq!(receiver.receive(), "hello world!");
    });
}
```

有一点不方便的是，我们仍然得手动地使用线程阻塞去等待一个消息，但是我们稍后将处理这个问题。

目前，我们的目标是在编译时使至少一种形式的滥用变得不可能。与过去不同，试图发送两次不会导致程序 Panic，相反，根本不会导致有效的程序。如果我们向上述工作程序增加另一个 send 调用，编译器现在捕捉问题并可能告知我们错误信息：

```txt
error[E0382]: use of moved value: `sender`
  --> src/main.rs
   |
   |             sender.send("hello world!");
   |                    --------------------
   |                     `sender` moved due to this method call
   |
   |             sender.send("second message");
   |             ^^^^^^ value used here after move
   |
note: this function takes ownership of the receiver `self`, which moves `sender`
  --> src/lib.rs
   |
   |     pub fn send(self, message: T) {
   |                 ^^^^
   = note: move occurs because `sender` has type `Sender<&str>`,
           which does not implement the `Copy` trait
```

根据情况，设计一个在编译时捕捉错误的接口可能非常棘手。如果这种情况确实适合这样的接口，它不仅可以为用户带来更多的便利，还可以**减少**运行时检查的数量，因为这些检查在静态上已经得到保证。例如，我们不再需要 `in_use` 标识，并从发送者法中移除了交换和检查步骤。

不幸的是，可能会出现新的问题，这可能导致运行时开销。在这种情况下，问题是拆分所有权，我们不得不使用 Arc 并承受 Arc 的代价。

不得不在安全性、便利性、灵活性、简单性和性能之间进行权衡是不幸的，但有时是不可避免的。Rust 通常致力于在这些方面取得最佳表现，但有时为了最大化某个方面的优势，我们需要在其中做出一些妥协。

## 借用以避免内存分配

（<a href="https://marabos.nl/atomics/building-channels.html#borrowing-to-avoid-allocation" target="_blank">英文版本</a>）

我们刚刚基于 Arc 的 channel 实现的设计可以非常方便的使用——代价是一些性能，因为它得内存分配。如果我们想要优化效率，我们可以通过用户对共享的 Channel 对象负责来获取一些性能。我们可以强制用户去创建一个通过可以由 Sender 和 Receiver 借用的 Channel，而不是在幕后处理 Channel 内存分配和所有权。这样，它们可以选择简单地放置 Channel 在局部变量中，从而避免内存分配的开销。

我们将也在一定程度上牺牲简洁性，因为我们现在不得不处理借用和生命周期。

因此，这三种类型现在看起来如下，Channel 再次公开，Sender 和 Receiver 借用它一段时间。

```rust
pub struct Channel<T> {
    message: UnsafeCell<MaybeUninit<T>>,
    ready: AtomicBool,
}

unsafe impl<T> Sync for Channel<T> where T: Send {}

pub struct Sender<'a, T> {
    channel: &'a Channel<T>,
}

pub struct Receiver<'a, T> {
    channel: &'a Channel<T>,
}
```

我们没有使用 `channel()` 函数来创建一对 Sender 和 Receiver，而是回到本章节使用的 `Channel::new`，这允许用户为此类对象创建局部变量。

此外，我们需要一种方法，让用户创建将借用 Channel 的 Sender 和 Receiver 对象。这将需要是一个独占借用（`&mut Channel`），以确保同一 channel 不能有多个发送者或接收者。通过同时提供 Sender 和 Receiver，我们可以将独占引用*分成*两个共享借用，这样发送者和接收者都可以引用 channel，同时防止其他任何东西接触 channel。

这导致我们实现以下内容：

```rust
impl<T> Channel<T> {
    pub const fn new() -> Self {
        Self {
            message: UnsafeCell::new(MaybeUninit::uninit()),
            ready: AtomicBool::new(false),
        }
    }

    pub fn split<'a>(&'a mut self) -> (Sender<'a, T>, Receiver<'a, T>) {
        *self = Self::new();
        (Sender { channel: self }, Receiver { channel: self })
    }
}
```

`split` 方法使用一个极其复杂的签名，值得好好观察。它通过一个独占引用独占地借用 `self`，但它分成了两个共享引用，包装在 Sender 和 Receiver 类型中。`'a` 生命周期清楚地表明，这两个对象借用了有限的生命周期的东西；在这种情况下，是 Channel 本身的生命周期。由于 Channel 是独占地借用，只要 Sender 或 Receiver 对象存在，调用者不能去借用或者移动它。

然而，一旦这些对象都不再存在，可变的借用就会过期，编译器会愉快地让 Channel 对象通过第二次调用 `split()` 再次被借用。尽管我们可以假设在 Sender 和 Receiver 存在时，不能再次调用 `split()`，我们不能阻止在这些对象被丢弃或者遗忘后再次调用 `split()`。我们需要确保我们不能偶然地在 channel 已经有它的 ready 标识设置的情况下创建新的 Sender 或 Receiver 对象，因为这将打包阻止未定义行为的假设。

通过在 `split()` 中用新的空 channel 覆盖 `*self`，我们确保它在创建 Sender 和 Receiver 状态时处于预期状态。这也会在旧的 `*self` 上调用 Drop 实现，它将负责丢弃之前发送但从未接收的消息。

> 由于 split 的签名的生命周期来自 `self`，它可以被省略。上面片段的 `split` 签名与这个不太冗长的版本相同
>
> ```rust
> pub fn split(&mut self) -> (Sender<T>, Receiver<T>) { … }
> ```
>
> 虽然此版本没有明确显示返回的对象借用了 self，但编译器仍然与更冗长的版本完全一样检查生命周期的正确使用情况。

其余的方法和 Drop 实现与我们基于 Arc 的实现相同，除了 Sender 和 Receiver 类型的额外 `'_` 生命周期参数。（如果你忘记了这些，编译器会建议添加它们。）

为了完全起效，以下是剩余的代码：

```rust
impl<T> Sender<'_, T> {
    pub fn send(self, message: T) {
        unsafe { (*self.channel.message.get()).write(message) };
        self.channel.ready.store(true, Release);
    }
}

impl<T> Receiver<'_, T> {
    pub fn is_ready(&self) -> bool {
        self.channel.ready.load(Relaxed)
    }

    pub fn receive(self) -> T {
        if !self.channel.ready.swap(false, Acquire) {
            panic!("no message available!");
        }
        unsafe { (*self.channel.message.get()).assume_init_read() }
    }
}

impl<T> Drop for Channel<T> {
    fn drop(&mut self) {
        if *self.ready.get_mut() {
            unsafe { self.message.get_mut().assume_init_drop() }
        }
    }
}
```

让我们来测试它！

```rust
fn main() {
    let mut channel = Channel::new();
    thread::scope(|s| {
        let (sender, receiver) = channel.split();
        let t = thread::current();
        s.spawn(move || {
            sender.send("hello world!");
            t.unpark();
        });
        while !receiver.is_ready() {
            thread::park();
        }
        assert_eq!(receiver.receive(), "hello world!");
    });
}
```

与基于 Arc 的版本相比，便利性的**减少**非常小：我们只需要多一行代码来手动创建一个 Channel 对象。然而，请注意，channel 必须在作用域之前创建，以向编译器证明其存在超过 Sender 和 Receiver 的时间。

要查看编译器的借用检查器的实际操作，请尝试在各个地方添加对 `channel.split()` 的第二次调用。你将看到，在线程作用域内第二次调用它会导致错误，而在作用域之后调用它是可以接受的。即使在作用域之前调用 `split()` 也没问题，只要你在作用域开始之前停止使用返回的 Sender 和 Receiver 。

## 阻塞

（<a href="https://marabos.nl/atomics/building-channels.html#blocking" target="_blank">英文版本</a>）

让我们最终处理一下我们 Channel 最后留下的最大不便，阻塞接口的缺乏。我们测试一个新的 channel 变体，每次都使用线程阻塞函数。将这种模式本身整合到 channel 应该不是太难。

为了能够释放接收者，发送者需要知道去释放哪个线程。`std::thread::Thread` 类型表示线程的句柄，正是我们调用 `unpark()` 所需要的。我们将把句柄存储到 Sender 对象内的接收线程，如下所示：

```rust
use std::thread::Thread;

pub struct Sender<'a, T> {
    channel: &'a Channel<T>,
    receiving_thread: Thread, // 新增！
}
```

然而，如果 Receiver 对象在线程之间发送，该句柄将引用错误的线程。Sender 将不会意识到这个，并且仍然会参考最初持有 Receiver 的线程。

我们可以通过使 Receiver 更具限制性，不再允许它在线程之间发送来处理这个问题。正如[第 1 章“线程安全：Send 和 Sync”](./1_Basic_of_Rust_Concurrency.md#线程安全send-和-sync)中所讨论的，我们可以使用特殊的 `PhantomData` 标记类型将此限制添加到我们的结构中。`PhantomData<*const ()>` 将完成这项工作，因为原始指针，如 `*const ()`，没有实现 Send：

```rust
pub struct Receiver<'a, T> {
    channel: &'a Channel<T>,
    _no_send: PhantomData<*const ()>, // 新增！
}
```

接下来，我们必须修改 `Channel::split` 方法来填充新字段，例如：

```rust
    pub fn split<'a>(&'a mut self) -> (Sender<'a, T>, Receiver<'a, T>) {
        *self = Self::new();
        (
            Sender {
                channel: self,
                receiving_thread: thread::current(), // 新增！
            },
            Receiver {
                channel: self,
                _no_send: PhantomData, // 新增！
            }
        )
    }
```

我们使用当前线程的句柄来填充 `receiving_thread` 字段，因为我们返回的 Receiver 对象将保留在当前线程上。

正如以下展示的，`send` 方法并不做改变。我们仅在 `receiving_thread` 字段上调用 `unpark()` 去唤醒接收者，以防止它正在等待：

```rust
impl<T> Sender<'_, T> {
    pub fn send(self, message: T) {
        unsafe { (*self.channel.message.get()).write(message) };
        self.channel.ready.store(true, Release);
        self.receiving_thread.unpark(); // 新增！
    }
}
```

receive 函数发生的变化稍大。如果它仍然没有消息，新版本不会 panic，而是使用 `thread::park()` 等待消息并再次尝试，并根据需要多次重试。

```rust
impl<T> Receiver<'_, T> {
    pub fn receive(self) -> T {
        while !self.channel.ready.swap(false, Acquire) {
            thread::park();
        }
        unsafe { (*self.channel.message.get()).assume_init_read() }
    }
}
```

> 请记住，`thread::park()` 可能会虚假返回。（或者因为除了我们的 send 方法以外的其它原因调用了 `unpark()`。）这意味着我们不能假设 `park()` 返回时已经设置了 ready 标识。因此，我们需要使用一个循环，在唤醒后再次检查 ready 标识。

`Channel<T>` 结构体、它的 Sync 实现、它的 new 函数以及它的 Drop 实现保持不变。

让我们尝试它！

```rust
fn main() {
    let mut channel = Channel::new();
    thread::scope(|s| {
        let (sender, receiver) = channel.split();
        s.spawn(move || {
            sender.send("hello world!");
        });
        assert_eq!(receiver.receive(), "hello world!");
    });
}
```

显然，这个 Channel 比上一个 Channel 更方便使用，至少在这个简单的测试程序中是这样。我们不得不牺牲一些灵活性来创造这种便利性：只有调用 `split()` 的线程才能调用 `receive()`。如果你交换 send 和 receive 行，此程序将不再编译。根据用例，这可能完全没问题、有用或非常不方便。

确实，有许多方法解决这个问题，其中有很多会增加一些额外的复杂度并影响一些性能。总的来说，我们可以继续探索的变种和权衡是无穷无尽的。

我们很容易花费大量的时间实现 20 个一次性 channel 不同的变体，每个变体都具有不同的属性，适用于每个可以想象到的用例甚至更多。尽管这听起来很有趣，但是我们应该避免陷入这个歧途，并在事情失控之前结束本章。

## 总结

（<a href="https://marabos.nl/atomics/building-channels.html#summary" target="_blank">英文版本</a>）

* *channel* 用于在线程之间发送*消息*。
* 一个简单、灵活但可能效率低下的 channel，只需一个 `Mutex` 和 `Condvar` 就很容易实现。
* *一次性*（one-shot）channel 是一个被设计仅发送一次信息的 channel。
* `MaybeUninit<T>` 类型可用于表示可能尚未初始化的 `T`。其接口大多不安全，使用户负责跟踪其是否已初始化，不要复制非 `Copy` 数据，并在必要时删除其内容。
* 不丢弃对象（也称为泄漏或者遗忘）是安全的，但如果没有充分理由而这样做，会被视为不良的做法。
* panic 是创建安全接口的重要工具。
* 按值获取一个非 Copy 对象可以用于阻止某个操作被重复执行。
* 独占借用和拆分借用是确保正确性的强大工具。
* 我们可以确保对象的类型不实现 `Send`，确保它在同一个线程，这可以通过 `PhantomData` 标记实现。
* 每个设计和实施决定都涉及权衡，最好在考虑特定用例的情况下做出。
* 在没有用例的情况下设计一些东西可能是有趣的和有教育意义的，但是这可能是一个无止境的任务。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./6_Building_Our_Own_Arc.html">下一篇，第六章：构建我们自己的“Arc”</a>
</p>
