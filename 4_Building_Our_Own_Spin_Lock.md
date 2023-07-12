# 第四章：构建我们自己的自旋锁

（<a href="https://marabos.nl/atomics/building-spinlock.html" target="_blank">英文版本</a>）

对普通互斥锁（参见[第一章中的“锁：互斥锁和读写锁”](./1_Basic_of_Rust_Concurrency.md#锁互斥锁和读写锁)）进行加锁时，如果互斥锁已经被锁定，线程将被置于睡眠状态。这避免在等待锁被释放时浪费资源。如果一个锁只会被短暂地持有，并且锁定它的线程可以在不同的处理器核心并发地运行，那么线程最好反复尝试锁定它而不实际进入睡眠态。

自旋锁是能够做到这一点的 mutex。试图锁定一个已经锁定的 mutex 将导致*忙碌循环*或者*自旋*：一遍又一遍的尝试。直到它成功。这可能浪费处理器周期，但有时会导致锁定时的延迟更低。

> 在某些平台上，许多现实世界中的 mutex 实现，包括 `std::sync::Mutex`，在告诉操作系统将线程置于睡眠状态之前，短暂地表现得像一个自旋锁。这是为了将两者的优点结合起来，尽管具体使用情况是否有益，这取决于特定的用例。

在该章节总，我们将建造我们自己的 `SpinLock` 类型，应用我们已经在第 [2](./2_Atomics.md) 章和第 [3](./3_Memory_Ordering.md) 章学习的，并且了解如何使用 Rust 的类型系统为我们的 SpinLock 用户提供安全且有用的接口。

## 一个最小实现

（<a href="https://marabos.nl/atomics/building-spinlock.html#a-minimal-implementation" target="_blank">英文版本</a>）

让我们从头实现这样的自旋锁。

最小的版本非常简单，如下：

```rust
pub struct SpinLock {
    locked: AtomicBool,
}
```

我们需要的只是一个布尔值，指示自旋锁是否已锁定。我们使用一个*原子*布尔值，因为我们希望多个线程能够同时与它交互。

然后，我们只需要一个构造函数，以及锁定和解锁的方法：

```rust
impl SpinLock {
    pub const fn new() -> Self {
        Self { locked: AtomicBool::new(false) }
    }

    pub fn lock(&self) {
        while self.locked.swap(true, Acquire) {
            std::hint::spin_loop();
        }
    }

    pub fn unlock(&self) {
        self.locked.store(false, Release);
    }
}
```

`locked` 的布尔值是从 false 开始的，`lock` 方法会将其替换为 true，如果它已经是 true，那么它将继续尝试，并且 `unlock` 方法仅将它设回 false。

> 与其使用 `swap` 操作，我们也可以使用「比较并交换」操作去原子地检查布尔值是否是 false，如果是这种情况，将它设置为 true：
>
> ```rust
> while self.locked.compare_exchange_weak(
>            false, true, Acquire, Relaxed).is_err()
> ```
>
> 这更细节一点，但是根据你的思维，这可能会更容易理解，因为它更容易地表述了可能失败和可能成功的情况。然而，它也导致了稍微不同的指令，正如我们在[第七章](./7_Understanding_the_Processor.md)所看到的那样。

在 while 循环中，我们使用一个自旋循环提示，它告诉处理器我们正在自旋等待某些变化。在大多数平台上，该自旋导致处理器核心采取优化行为以应对这种情况。例如，它可能暂时地降低速度或优先处理其它有用的任务。然而，与 `thread::sleep` 或者 `thread::park` 等阻塞操作不同，自旋循环提示并不会调用操作系统调用，将你的线程置于睡眠状态以便执行其它线程。

> 总的来说，在自旋循环中包含这样的提示是一个好的主意。根据情况，在尝试再次访问原子变量之前，最好多次执行此提示。如果你关心最后几纳秒的性能并且想要找到最优的策略，你将不得不为你特定用例编写基准测试。不幸地是，正如我们将在[第7章](./7_Understanding_the_Processor.md)中看到的那样，此类基准测试的结果可能在很大程度上取决于硬件。

我们可以使用 acquire 和 release 内存排序去确保每个 `unlock()` 调用和随后的 `lock()` 调用都建立了一个 happens-before 关系。换句话说，为了确保锁定它后，我们可以安全地假设上次锁定期间的任何事情已经发生。这是 acquire 和 release 最经典的使用案列：获取和释放一个锁。

图 4-1 展示了使用 `SpinLock` 来保护对一些共享数据的访问情况，其中两个线程同时尝试获取锁。请注意，第一个线程上的解锁操作与第二个线程上的锁定操作形成 happens-before 关系，这确保了线程不能并发地访问数据。

![ ](https://github.com/rustcc/Rust_Atomics_and_Locks/raw/main/picture/raal_0401.png)
图 4-1。在使用 `SpinLock` 保护对某些共享数据访问的两个线程之间的 happens-before 关系。

## 一个不安全的自旋锁

（<a href="https://marabos.nl/atomics/building-spinlock.html#an-unsafe-spin-lock" target="_blank">英文版本</a>）

我们上面实现的 SpinLock 类型有一个完全安全地接口，它并不会引起任何未定义行为。然而，在大多数的使用案列中，它将被用于保护共享变量的可变性，这意味着用于将仍然使用一个不安全的、未检查的代码。

为了提供一个简单的接口，我们可以改变 `lock` 方法为直接提供受锁保护数据的独占的引用（`&mut T`），因为在大多数情况下，是锁操作保证了可以安全地假设具有独占访问权限。

为了能够做到这一点，我们必须将类型更改为更加通用，而不是受保护的数据类型，并且添加一个字段持有数据。因为即使自旋锁是共享的，数据也是可变的（或者独占访问），我们需要去使用内部可变性（参见[第 1 章中的“内部可变性”](./1_Basic_of_Rust_Concurrency.md#内部可变性)），为此我们将使用 `UnsafeCell`：

```rust
use std::cell::UnsafeCell;

pub struct SpinLock<T> {
    locked: AtomicBool,
    value: UnsafeCell<T>,
}
```

作为一种预防措施，UnsafeCell 没有实现 `Sync`，这意味着我们的类型现在不再可以在线程之间共享，使其变得毫无用处。为了修复它，我们需要向编译器保证我们的类型实际上可以在线程之间共享是安全的。然而，因为锁可以用于在线程之间发送类型为 T 的值，我们限制这个承诺为那些可以安全发送给其它线程的类型。因此，我们（不安全地）为所有实现 `Send` 的 T 实现 `SpinLock<T>` 的 `Sync`，如下所示：

```rust
unsafe impl<T> Sync for SpinLock<T> where T: Send {}
```

注意，我们并不需要去要求 T 是 `Sync`，由于我们的 `SpinLock<T>` 仅一次允许一个线程访问它保护的 T。只有当我们同时允许多个线程访问权限时，就像读写锁对 reader 所做的那样，我们（另外）才需要 `T: Sync`。

下一步，现在我们的新函数需要接收一个 T 类型的值来初始化 `UnsafeCell`：

```rust
impl<T> SpinLock<T> {
    pub const fn new(value: T) -> Self {
        Self {
            locked: AtomicBool::new(false),
            value: UnsafeCell::new(value),
        }
    }

    // …
}
```

然后我们进入有趣的部分：锁定和解锁。我们做这一切的原因，是为了能够从 `lock()` 中返回 `&mut T`，例如，这样用户在使用我们的锁来保护它们的数据时，并不要求写不安全、未检查的代码。这意味着，我们现在在锁定的实现中不得不使用一个不安全的代码。`UnsafeCell` 可以通过其 `get()` 方法向我们提供指向其内容（`*mut T`）的原始指针，我们可以使用不安全块传唤到一个引用，如下所示：

```rust
    pub fn lock(&self) -> &mut T {
        while self.locked.swap(true, Acquire) {
            std::hint::spin_loop();
        }
        unsafe { &mut *self.value.get() }
    }
```

由于 `lock` 函数的函数签名在其输入和输出都包含引用，`&self` 和 `&mut T` 的生命周期都已经被省略并假定为相同的生命周期。（参见《Rust Book》中的“Chapter 10: Generic Types, Traits, and Lifetimes”的“Lifetime Elision”一节）。我们可以通过手动书写来明确这些生命周期，如下所示：

```rust
 pub fn lock<'a>(&'a self) -> &'a mut T { … }
```

这清楚地表明，返回引用的生命周期与 `&self` 的生命周期相同。这意味着我们已经声称，只要锁本身存在，返回的引用就是有效的。

如果我们假装 `unlock()` 不存在，这将是完全安全和健全的接口。SpinLock 可以被锁定，导致一个 `&mut T`，并且然后不再被再次锁定，这保证了这个独占引用确实是独占的。

然而，如果我们尝试重新**增加** `unlock()` 方法，我们需要一种方式去限制返回引用的生命周期，直到下一次调用 `unlock()`。如果编译器理解英语，或者它应该这样工作：

```rust
pub fn lock<'a>(&self) -> &'a mut T
    where
        'a ends at the next call to unlock() on self,
        even if that's done by another thread.
        Oh, and it also ends when self is dropped, of course.
        (Thanks!)
    { … }
```

不幸地是，这并不是有效的 Rust。我们必须试图向用户解释这个限制，而不是向编译器解释。为了将责任转移到用户身上，我们将 `unlock` 函数标记为不安全，并给他们留下一张纸条，解释他们需要做什么来保持健全：

```rust
/// 安全性：来自 lock() 的 &mut T 必须消失
/// （并且通过引用该 T 周围的字段来防止欺骗！）
pub unsafe fn unlock(&self) {
    self.locked.store(false, Release);
}
```

## 使用锁守卫的安全接口

（<a href="https://marabos.nl/atomics/building-spinlock.html#building-safe-spinlock" target="_blank">英文版本</a>）

为了能够提供一个完全安全地接口，我们需要将解锁操作绑定到 `&mut T` 的末尾。我们可以通过将此引用包装成我们自己的类型来做到这一点，该类型的行为类似于引用，但也实现了 Drop trait，以便在它被丢弃时做一些事情。

这一类型通常被称为*守卫*（guard），因为它有效地守卫了锁的状态，并且对该状态负责，直到它被丢弃。

我们的 `Guard` 类型将仅包含对 SpinLock 的引用，以便它既可以访问 UnsafeCell，也可以稍后重置 AtomicBool：

```rust
pub struct Guard<T> {
    lock: &SpinLock<T>,
}
```

然而，如果我们尝试编译它，编译器将告诉我们：

```txt
error[E0106]: missing lifetime specifier
   --> src/lib.rs
    |
    |         lock: &SpinLock<T>,
    |               ^ expected named lifetime parameter
    |
help: consider introducing a named lifetime parameter
    |
    ~     pub struct Guard<'a, T> {
    |                      ^^^
    ~         lock: &'a SpinLock<T>,
    |                ^^
    |
```

显然，这不是一个可以淘汰生命周期的地方。我们必须明确表示，引用的生命周期有限，正如编译器所建议的那样：

```rust
pub struct Guard<'a, T> {
    lock: &'a SpinLock<T>,
}
```

这保证了 Guard 不能超出 SpinLock 的生命周期。

下一步，我们在我们的 SpinLock 上改变 lock 方法，以返回 Guard：

```rust
pub fn lock(&self) -> Guard<T> {
    while self.locked.swap(true, Acquire) {
        std::hint::spin_loop();
    }
    Guard { lock: self }
}
```

我们的 Guard 类型没有构造函数，其字段是私有的，因此这是用户获得 Guard 的唯一方法。因此，我们可以有把握地假设 Guard 的存在意味着 SpinLock 已被锁定。

为了使 `Guard<T>` 行为类似一个（独占）引用，透明地允许访问 T，我们必须实现以下特殊的 Deref 和 DerefMut trait：

```rust
use std::ops::{Deref, DerefMut};

impl<T> Deref for Guard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // 安全性：Guard 的 存在
        // 保证了我们已经独占地锁定这个锁
        unsafe { &*self.lock.value.get() }
    }
}

impl<T> DerefMut for Guard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        // 安全性：Guard 的存在
        // 保证了我们已经独占地锁定这个锁
        unsafe { &mut *self.lock.value.get() }
    }
}
```

最后一步，我们为 Guard 实现 Drop，允许我们完全地引出不安全的 `unlock()` 方法：

```rust
impl<T> Drop for Guard<'_, T> {
    fn drop(&mut self) {
        self.lock.locked.store(false, Release);
    }
}
```

就这样，通过 Drop 和 Rust 类型系统的魔力，我们为我们的 SpinLock 类型提供了一个完全安全（和有用的）接口。

让我们尝试使用它：

```rust
fn main() {
    let x = SpinLock::new(Vec::new());
    thread::scope(|s| {
        s.spawn(|| x.lock().push(1));
        s.spawn(|| {
            let mut g = x.lock();
            g.push(2);
            g.push(2);
        });
    });
    let g = x.lock();
    assert!(g.as_slice() == [1, 2, 2] || g.as_slice() == [2, 2, 1]);
}
```

上面的程序展示了我们的 `SpinLock` 是多么容易使用。多亏了 `Deref` 和 `DerefMut`，我们可以直接在 guard 上调用 `Vec::push` 方法。多亏了 `Drop`，我们不必担心解锁。

通过调用 `drop(g)` 来丢弃 guard，也可以明确地解锁。如果你尝试过早地解锁，你将看见 guard 正在做它的工作时，发生编译器错误。例如，如果你在两个 `push(2)` 行之间插入 `drop(g);`，第二个 push 将无法编译，因为你此时已经丢弃 g 了：

```txt
error[E0382]: borrow of moved value: `g`
   --> src/lib.rs
    |
    |     drop(g);
    |          - value moved here
    |     g.push(2);
    |     ^^^^^^^^^ value borrowed here after move
```

多亏了 Rust 的类型系统，我们可以放心，在我们运行程序之前，这样的错误就已经被发现了。

## 总结

（<a href="https://marabos.nl/atomics/building-spinlock.html#summary" target="_blank">英文版本</a>）

* 自旋锁是在等待时忙碌循环或自选的 mutex。
* 自旋可以**减少**延迟，但也可能浪费时钟周期并降低性能。
* 自旋循环提示（`spin::hint::spin_loop()`）可以用于通知处理器自旋循环，这可能**增加**它的效率。
* `SpinLock<T>` 只需使用 `AtomicBool` 和 `UnsafeCell<T>` 即可实现，后者是*内部可变性*所必需的（见[第 1 章中的“内部可变性”](./1_Basic_of_Rust_Concurrency.md#内部可变性)）。
* 在解锁和锁定之间的 *happens-before 关系*是防止*数据竞争*的必要条件，否则会导致未定义行为。
* *Acquire* 和 *Release* 内存排序对这个用例是极合适的。
* 当做出必要的未检查的假设以避免未定义的行为时，可以通过将函数标记为不安全来将责任转移到调用者。
* `Deref` 和 `DerefMut` trait 可用于使类型像引用一样，透明地提供对另一个对象的访问。
* `Drop` trait 可以用于在对象被丢弃时，做一些事情，例如当它超出作用域或者它被传递给 `drop()`。
* *锁守卫*是一种特殊类型的有用设计模式，它被用于表示对锁定的锁的（安全）访问。由于 `Deref` trait，这种类型通常与引用的行为相似，并通过 `Drop` trait 实现自动解锁。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./5_Building_Our_Own_Channels.html">下一篇，第五章：构建我们自己的 Channel</a>
</p>
