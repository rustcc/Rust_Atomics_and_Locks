# 第五章：构建我们自己的 Channel

*Channel* 可以被用于在线程之间发送数据，并且它们有很多变体。一些 channel 仅能在一个发送方和一个接收方之间使用，而另一些可以在任意数量的线程之间发送，或者甚至允许多个接收放。一些 channel 是阻塞的，这意味着接收（有时也包括发送）是一个阻塞操作，这会使线程进入睡眠状态，知道你的操作完成。一些 channel 针对团兔粮进行优化，而另一些针对低延迟进行优化。

这些变体是无穷尽的，没有一种通用版本在所有场景都适合的。

在该章节，我们将实现一个相对简单的 channel，不仅可以探索更多的原子应用，同时也可以了解如何在 Rust 类型系统中捕获我们的需求和假设。

## 一个简单的以 mutex 为基础的 Channel

一个基础的 channel 实现并不需要任何关于原子的知识。我们可以采用 `VecDeque`，它根本上是一个 `Vec`，允许在两端高效的添加和移除元素，并使用 Mutex 保护它，以允许多个线程访问。然后，我们使用 `VecDeque` 作为已发送但尚未接受数据的消息队列。任何想要发送消息的线程只需要将其添加到队列的末尾，而任何想要接受消息的线程只需从队列的前端删除一个消息。

还有一件事需要补充，用于使接收操作阻塞：Condvar（参见[第一章“条件变量”](./1_Basic_of_Rust_Concurrency.md#条件变量)），以通知等待接收者新的消息。

这样做的实现可能非常简短且相对简单，如下所示：

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

注意，我们并没有使用任意的原子操作或者不安全代码，也不需要考虑 `Send` 或者 `Sync`。编译器理解 Mutex 的接口以及保证该提供什么类型，并且会隐式地理解如果 `Mutex<T>` 和 Condvar 都可以在线程之间安全共享，那么我们的 `Channel<T>` 也可以。

我们的 `send` 函数锁定 mutex，将新消息推入队列的末尾，并且使用条件变量在解锁队列后直接通知可能等待的接收者。

`receive` 函数也锁定 mutex，然后从队列的首部弹出消息，但如果仍然没有可获得的消息，则会使用条件变量去等待。

> 记住，`Condvar::wait` 方法将在等待时解锁 Mutex，并在返回之前重新锁定它。因此，我们的 `receive` 函数将不会在等待时锁定 mutex。

尽管这个 channel 在使用上是非常灵活的，因为它允许任意数量的发送和接收线程，它的实现在很多情况下远非最佳。即使有大量的消息准备好被接收，任意的发送或者接收操作将短暂地阻塞任意其它的发送或者接收操作，因为它们必须都锁定相同的 mutex。如果 `VecDeque::push` 不得不增加 VecDeque 的容量，所有的发送和接收线程将不得不等待该线程完成重新分配，这在某些情况下是不可接受的。

另一个可能不可取的属性是，该 channel 的队列可能会无限制地增长。没有什么能阻止发送者以比接收者更高的速度持续发送新消息。

## 一个不安全的一次性 Channel

channel 的各种用例几乎是无止尽的。然而，在本章的剩余部分，我们将专注于一种特定类型的用例：恰好从一个线程向另一个线程发送一条消息。为此类用例设计的 channel 通常被称为 *一次性*（one-shot）channel。

我们采用上述基于 `Mutex<VecDeque>` 的实现，并且将 `VecDeque` 替换为 `Option`，从而将队列的容量减小到恰好一个消息。这样可以避免内存分配，但是仍然会有使用 Mutex 的一些缺点。我们可以通过使用原子操作从头构建我们自己的一次性 channel 来避免这个问题。

首先，让我们构建一个最小化的一次性 channel 实现，不需要考虑它的接口。在本章的稍后，我们将探索如何改进其接口以及如何与 Rust 类型相结合，为 channel 的用于提供愉快的体验。

我们需要开始的工具基本上与我们在[第四章](./4_Building_Our_Own_Spin_Lock.md)使用的 `SpinLock<T>` 基本相同：一个用于存储的 `UnsafeCell` 和用于指示状态的 `AtomicBool`。在该示例中，我们使用原子布尔值去指示消息是否准备好用于消费。

在发送消息之前，channel 是“空的”并且不包含任意类型为 T 的消息。我们可以在 cell 中使用 `Option<T>`，以允许 T 缺失。然而，这可能会浪费宝贵的内存空间，因为我们的原子布尔值已经告诉我们是否有消息。相反，我们可以使用 `std::mem::MaybeUninit<T>`，它本质上是裸露的 `Option<T>` 的不安全版本：它要求用户手动跟踪其是否已初始化，几乎整个接口都是不安全的，因为它不能执行自己的检查。

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

要发送消息，它首先需要存储在 cell 中，之后我们可以通过将 ready 标志设置为 true 来将其释放给接收者。试图做这个超过一次是危险的，因为设置 ready 标志后，接收者可能在任意时刻读取消息，这可能会与第二次发送消息产生数据竞争。目前，我们通过使方法不安全并为它们留下备注，将此作为用户的责任：

```rust
    /// Safety: Only call this once!
    pub unsafe fn send(&self, message: T) {
        (*self.message.get()).write(message);
        self.ready.store(true, Release);
    }
```

在上面这个片段中，我们使用 `UnsafeCell::get` 方法去获取指向 `MaybeUninit<T>` 的指针，并且通过不安全地解引用它来调用 `MaybeUninit::write` 进行初始化。当错误使用时，这可能导致未定义行为，但我们将这个责任注意到了调用方身上。

对于内存排序，我们需要使用 release 排序，因为原子的存储有效地将消息释放给接收者。这确保了如果接收线程从 `self.ready` 以 acquire 排序加载 true，则消息的初始化将从接受线程的角度完成。

对于接收，我们暂时不会提供阻塞的接口。相反，我们将提供两个方法：一个用于检查是否有可用消息，另一个用于接收消息。我们将让我们的 channel 用户决定是否使用[线程阻塞](./1_Basic_of_Rust_Concurrency.md#线程阻塞)的方法来阻塞。

以下是完成此版本我们 channel 的最后两种方法：

```rust
    pub fn is_ready(&self) -> bool {
        self.ready.load(Acquire)
    }

    /// Safety: Only call this once,
    /// and only after is_ready() returns true!
    pub unsafe fn receive(&self) -> T {
        (*self.message.get()).assume_init_read()
    }
```

虽然 `is_ready` 方法可以始终地安全调用，但是 `receive` 方法使用了 `MaybeUninit::assume_init_read()`，这不安全地假设它已经被初始化，且不会用于生成非 `Copy` 对象的多个副本。就像 `send` 方法一样，我们只需通过将函数本身标记为不安全来将这个问题交给用户解决。

结果是一个在技术上可用的 channel，但它用起来不便并且通常令人失望。如果正确使用，它会按预期进行操作，但有很多微妙的方式去错误地使用它。

多次调用 send 可能会导致数据竞争，因为第二个发送者在接收者尝试读取第一条消息时可能正在覆盖数据。即使接收操作得到了正确的同步，从多个线程调用 send 可能会导致两个线程尝试同时写入 cell，再次导致数据竞争。此外，多次调用 `receive` 会导致获取两个消息的副本，即使 T 不实现 `Copy` 并且因此不能安全地进行复制。

更微妙的问题是我们的通道缺乏 `Drop` 实现。`MaybeUninit` 类型不会跟踪它是否已经初始化，因此在被 drop 时不会自动 drop 其内容。这意味着如果发送了一条消息但从未被接收，该消息将永远不会被 drop。这不是不正确的，但仍然是要避免。在 Rust 中，泄漏被普遍认为是安全的，但通常只有作为另一个泄漏的结果才是可接受的。例如，泄漏 Vec 也会泄漏其内容，但正常使用 Vec 不会导致任何泄漏。

由于我们让用户对一切负责，不幸的事故只是时间问题。

## 通过运行时检查来达到安全

## 通过类型来达到安全

## 借用以避免分配

## 阻塞

## 总结

* *channel* 用于在线程之间发送*消息*。
* 一个简单、灵活但可能效率低下的 channel，只需一个 `Mutex` 和 `Condvar` 就很容易实现。
* *一次性*（one-shot）channel 是一个被设计仅发送一次信息的 channel。
* `MaybeUninit<T>` 类型可用于表示可能尚未初始化的 `T`。其接口大多不安全，使用户负责跟踪其是否已初始化，不要复制非 `Copy` 数据，并在必要时删除其内容。
* 不 drop 对象（也称为泄漏或者遗忘）是安全的，但如果没有充分理由而这样做，会被视为不良的做法。
* panic 是创建安全接口的重要工具。
* 按值获取一个非 Copy 对象可以用于阻止某个操作被重复执行。
* 独占借用和拆分借用是确保正确性的强大工具。
* 我们可以确保对象的类型不实现 `Send`，确保它在同一个线程，这可以通过 `PhantomData` 标记实现。
* 每个设计和实施决定都涉及权衡，最好在考虑特定用例的情况下做出。
* 在没有用例的情况下设计一些东西可能是有趣的和有教育意义的，但是这可能是一个无止境的任务。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./6_Building_Our_Own_Arc.html">下一篇，第六章：构建我们自己的“Arc”</a>
</p>
