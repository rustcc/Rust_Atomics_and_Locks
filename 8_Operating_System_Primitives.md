# 第八章：操作系统原语

目前，我们主要聚焦在非阻塞的操作中。如果我们想要实现一些类似互斥锁或者条件变量的内容，也就是能够等待另一个线程去解锁或者通知它的内容，我们需要一种有效地阻塞当前线程的方式。

正如我们在[第四章](./4_Building_Our_Own_Spin_Lock.md)所见到的，我们可以不依赖操作系统，通过自旋，重复地一遍又一遍地尝试某些操作，自己实现阻塞，但这浪费大量的处理器时间。如果我们想要高效地进行阻塞，我们需要操作系统内核的帮助。

内核，或者更具体地说是其中的调度部分，负责决定哪个进程或者线程在何时运行，运行多长时间，并且在哪个处理器核心运行。尽管线程在等待某个事件发生时，内核可以停止，并给它任意的处理器时间，优先考虑其他能更好地利用这个有限资源的线程。

我们将需要一种方式来通知内核我们正在等待某个事件，并要求它将我们的线程置于睡眠状态，直到发生相关的事情。

## 使用内核接口

与内核进行通信的方式很大程度依赖于操作系统，甚至是它的版本。通常，如何工作的细节被一个库或者更多库所隐藏，这些库为我们处理这些细节。例如，使用 Rust 的标准库，我们可以仅调用 `File::open()` 去打开这个文件，而不必关心任何操作系统内核的细节。类似地，使用 C 标准库（`libc`）也可以调用标准的 `fopen()` 函数去打开一个文件。调用这样的函数最终会导致调用操作系统内核，也称为*系统调用*（syscall），通常通过专门的处理器指令来完成（在某些架构上，该指令甚至直接称为 syscall）。

通常期望程序（有时直接要求）不直接进行系统调用，而是利用操作系统携带的更高级别的库。在 Unix 系统中（例如那些基于 Linux 的），libc 扮演了与内核交换的标准接口的特殊角色。

POSIX[^1]（可移植操作系统接口）标准，包括了在类 Unix 系统上的 libc，以及对其额外的要求。例如，在 C 标准的 `fopen()` 函数之外，POSIX 还要求存在更低级别的 `open()` 和 `openat()` 函数来打开文件，这些函数通常直接对应一个系统调用。由于 libc 在类 Unix 系统上的特殊地位，使用其他语言编写的程序通常仍然使用 libc 来进行与内核的所有交互。

Rust 软件，包括标准库，通常通过相同名称的 libc crate 使用 libc 库。

尤其对于 Linux，系统调用接口被保证稳定，允许我们直接进行系统调用，而不使用 libc。尽管这不是最常见或最推荐的方式，但它正在逐渐变得更受欢迎。

然而，虽然 MacOS 也是一个 Unix 操作系统，跟随 POSIX 标准，但是它的内核系统调用接口并不稳固，并且我们并不建议直接使用它。程序被允许使用的唯一稳定接口是通过系统附带的库（如 libc、libc++）和其他库（用于 C、C++、Objective-C 和 Swift）提供的接口，这些是苹果公司的首选编程语言。

Windows 不遵循 POSIX 标准。它并没有携带一个拓展的 libc 作为主要的内核接口，而是携带了一系列独立的库，例如 `kernel32.dll`，它提供了 Windows 的特定功能，如用于打开文件的 `CreateFileW`。与在 macOS 上一样，我们不应使用未记录的较低级别函数或直接进行系统调用。

通过它们的库，操作系统为我们提供了需要与内核进行交互的同步原语，如互斥锁和条件变量。这些实现的哪一部分属于库/内核的一部分，在不同的操作系统中有很大的差异。例如，有时互斥锁的锁定和解锁操作直接对应一个内核系统调用，而在其他系统中，库会处理大部分操作，并且只在需要阻塞或唤醒线程时执行系统调用（后者往往更高效，因为进行系统调用可能较慢）。

## POSIX

作为 POSIX 线程扩展的一部分，更为人熟知的是 pthread，POSIX 规范了用于并发的数据类型和函数。尽管 libthread 在技术上是作为一个独立的系统库的一部分，但是如今它通常被直接包含在 libc 中。

除了线程的 spawn 和 join 功能（`pthread_create` 和 `pthread_join`）外，pthread 还提供了最常见的同步原语：互斥锁（`pthread_mutex_t`）、读写锁（`pthread_rwlock_t`）和条件变量（`pthread_cond_t`）。

* *pthread_mutex_t*

  Pthread 的互斥锁必须通过调用 `pthread_mutex_init()` 进行初始化，并使用 `pthread_mutex_destroy()` 进行销毁。初始化函数采用一个 `pthread_mutexattr_t` 类型的参数，该参数可用于配置互斥锁的某些属性。

  其中一个属性是互斥锁在*递归锁定*时的行为，其是指同一线程再次尝试锁定已经持有的锁时发生的情况。在默认设置（`PTHREAD_MUTEX_DEFAULT`）下使用递归锁定会导致未定义的行为，但也可以配置为产生错误（`PTHREAD_MUTEX_ERRORCHECK`）、死锁（`PTHREAD_MUTEX_NORMAL`）或成功的第二次锁定（`PTHREAD_MUTEX_RECURSIVE`）。

  通过 `pthread_mutex_lock()` 或 `pthread_mutex_trylock()` 来锁定这些互斥锁，通过 `pthread_mutex_unlock()` 来解锁。此外，与 Rust 的标准互斥锁不同的是，它们还支持通过 `pthread_mutex_timedlock()` 在有限的时间内进行锁定。

  可以通过分配值 `PTHREAD_MUTEX_INITIALIZER` 来静态初始化 `pthread_mutex_t`，而无需调用 `pthread_mutex_init()`。但是，这仅适用于具有默认设置的互斥锁。

* *pthread_rwlock_t*

  Pthread 的读写锁通过 `pthread_rwlock_init()` 和 `pthread_rwlock_destroy()` 进行初始化和销毁。与互斥锁类似，默认的 `pthread_rwlock_t` 可以使用 `PTHREAD_RWLOCK_INITIALIZER` 静态地初始化。

  与 pthread 互斥锁相比，pthread 读写锁通过它的初始化函数可配置的属性要少得多。特别要注意的是，尝试递归写锁定将始终导致死锁。

  然而，尝试递归获取额外的读锁是保证会成功的，即使有 writer 正在等待。这实际上排除了任何优先考虑 writer 而不是 reader 的高效实现，这就是为什么大多数 pthread 实现优先考虑 reader 的原因。

  它的接口与 `pthread_mutex_t` 几乎相同，包括支持时间限制，除了每个锁定函数都有两个变体：一个用于 reader（`pthread_rwlock_rdlock`），一个用于 writer（`pthread_rwlock_wrlock`）。也许令人惊讶的是，仅有一个解锁函数（`pthread_rwlock_unlock`），其用于解锁任一类型的锁。

* *pthread_cond_t*

  pthread 条件变量与 pthread 互斥锁一起使用。通过 `pthread_cond_init` 和 `pthread_cond_destroy` 进行初始化和销毁，并且可以配置一些属性。其中最值得注意的是，我们可以配置时间限制使用单调时钟[^2]（类似于 Rust 的 Instant）还是实时时钟[^3]（类似于 Rust 的 SystemTime，有时称为“挂钟时间”）。具有默认设置的条件变量（例如由 `PTHREAD_COND_INITIALIZER` 静态初始化的条件变量）使用实时时钟。

  通过 `pthread_cond_timedwait()` 等待此类条件变量，可选择设置时间限制。通过调用 `pthread_cond_signal()` 唤醒等待的线程，或者为了一次唤醒所有等待的线程，调用 `pthread_cond_broadcast()`。

Pthread 提供的其余同步原语是屏障（`pthread_barrier_t`）、自旋锁（`pthread_spinlock_t`）和一次性初始化（`pthread_once_t`），我们不会讨论。

### 在 Rust 中包裹

通过方便地将其 C 类型（通过 libc crate）包装在 Rust 结构体中，我们可以轻松地将这些 pthread 同步原语暴露给 Rust，例如：

```rust
pub struct Mutex {
    m: libc::pthread_mutex_t,
}
```

然而，这种方法存在一些问题，因为该 pthread 类型是为 C 设计的，而不是为 Rust 设计的。

首先，Rust 关于可变性和借用有一些规则，通常不允许在共享时进行修改。由于类似 `pthread_mutex_lock` 这样的函数总是可能对互斥锁进行修改，我们将需要内部可变性来确保这是可接受的。因此，我们需要将其包装在 UnsafeCell 中：

```rust
pub struct Mutex {
    m: UnsafeCell<libc::pthread_mutex_t>,
}
```

一个巨大的问题是关于*移动*。

在 Rust 中，我们所有时间都在移动对象。例如，通过从函数返回对象、将其作为参数传递或者简单地将其分配给新的位置。我们拥有的多所有东西（并且没有被其他东西借用），我们可以自由地移动它们到一个新位置。

然而，在 C 中，这并不是普遍正确的。在 C 中，类型通常依赖它的内存地址保持不变。例如，它可能包含一个指向自身的指针，或者在某个全局数据结构中存储一个指向自己的指针。在这种情况下，移动到一个新的位置可能导致未定义行为。

我们讨论的 pthread 类型不能保证它们是*可移动的*，在 Rust 中，这会带来很大的问题。即使是一个简单的惯用的 `Mutex::new()` 函数也是一个问题：它将返回一个 mutex 对象，这将移动到一个内存的新位置。

因为用户可能总是移动任何它们拥有的 mutex 对象到其他地方，我们要买需要承诺它别这么做，要买使接口不安全；或者我们需要取走所有权并且隐藏所有内容到一个包装类型后面（可以使用 `std::pin::Pin` 来完成）。这些都不是最好的解决方案，因为它们会影响我们的 mutex 类型的接口，使其用起来容易出错和/或不方便使用。

一个可以的解决方案是将 mutex 包装在一个 Box 中。通过将 pthread 的 mutex 放在它的自己分配的内存中，即使所有者被移动，它仍然在内存中的相同位置。

```rust
pub struct Mutex {
    m: Box<UnsafeCell<libc::pthread_mutex_t>>,
}
```

> 这就是 `std::sync::Mutex` 在 Rust 1.62 之前在所有 Unix 平台上实现的方式。

这个方式的缺点就是开销大：每个 mutex 都有自己分配的内存，为创建、销毁以及使用 mutex 增加了显著的开销。另一个缺点是它阻止了 new 函数编译时执行（`const`），这妨碍了拥有静态 mutex 的方式。

即使 `pthread_mutex_t` 是可移动的，`const fn new` 也可能仅使用默认设置来初始化，这导致了当递归锁定时的未定义行为。没有办法设计一个安全的接口来防止递归锁定，因此这意味着我们要使用 unsafe 标记锁定函数，以使用户承诺他们不会这样做。

当 drop mutex 时，在我们的 Box 方法中仍然存在一个问题。看起来，如果设计正确，就不可能在被锁定时 drop mutex，因为通过 MutexGuard 借用它时，不可能 drop 它。MutexGuard 必须先被 drop，解锁 Mutex。然而，在 Rust 中，安全地遗忘（或泄露）一个对象，而不将其 drop 是安全的。这意味着可以编写类似下面的代码：

```rust
fn main() {
    let m = Mutex::new(..);

    let guard = m.lock(); // Lock it ..
    std::mem::forget(guard); // .. but don't unlock it.
}
```

在以上示例中，`m` 将在作用域结束后被 drop，尽管它仍然被锁定。根据 Rust 的编译器来看，这是好的，因为 guard 已经被泄漏并且不能再使用。

然而，pthread 规定在已锁定的 mutex 调用 `pthread_mutex_destroy()` 并不能保证工作并且可能导致未定义行为。一种解决方案是当 drop 我买的 Mutex 时，首先试图去锁定（和解锁）pthread mutex，并且当它已经锁定时触发 panic（或泄漏 Box），但这甚至要更大的开销。

这些问题不仅适用于 `pthread_mutex_t`，还适用于我们讨论的其他类型。总体而言，pthread 的同步原语设计对 C 是好的，但是并不完全适合 Rust。

## Linux

### Futex

### Futex 操作

### 优先继承 Futex 操作

## macOS

macOS 部分的内核支持各种有用的低级并发相关的系统调用。然而，就像大多数操作系统一样，内核接口并不是稳定的，并且我们应该直接地使用它。

软件与 macOS 内核交互的唯一方式是通过系统携带的库。这些库包含它对 C（libc）、C++（libc++）、Objective-C 和 Swift 的标准库实现。

作为符合 POSIX 标准的 Unix 系统，macOS C 标准库包含一个完整的 pthread 实现。其他语言中的标准库锁通常在底层使用 pthread 原语。

在 macOS 上，与其他系统对比，Pthread 的锁相对低较慢。原因之一是 macOS 上的锁默认情况下是*公平锁*（Fair Lock），这意味着几个线程试图去锁定相同的 mutex 时，它们会按照到达的顺序一次获得锁定，就像一个完美的队列。尽管公平性可能是值得拥有的属性，但它会显著降低性能，特别是在高竞争的情况下。

### os_unfair_lock

除了 pthread 原语，macOS 10.12 引入了一种新的轻量级平台特定的互斥锁，它是不公平的：`os_unfair_lock`。它的大小仅有 32 位，可以使用 OS_UNFAIR_LOCK_INIT 常来那个静态地初始化，并且不需要销毁。它可以通过 `os_unfair_lock_lock()`（阻塞）或 `os_unfair_lock_trylock()`（非阻塞）来锁定它，并且通过 `os_unfair_lock_unlock()` 来解锁。

不幸的是，它没有条件变量，也没有 reader-writer 变体。

## Windows

Windows 操作系统携带了一系列库，它们一起形成了 *Windows API*，通常称之为“Win32 API”（甚至在 64 位系统也是）。它构成了一个在“Native 之上”的层：大部分是与内核没有交互的接口，我们不建议直接使用它。

通过微软官方提供的 windows 和 windows-sys crate，Windows API 可以为 Rust 程序所用，这在 `crates.io` 上是可获得的。

### 重量级内核对象

在 Windows 上可用的许多旧的同步原语完全由内核管理，这使得它们非常重量，并赋予它们与其他内核管理对象（例如文件）类似的属性。它们可以被多个进程使用，可以通过名称进行命名和定位，并且支持细粒度的权限，类似于文件。例如，可以允许一个进程等待某个对象，而不允许它通过该对象发送信号来唤醒其他进程。

这些重量级的内核管理同步对象包括 Mutex（可以锁定和解锁）、Event（可以发送信号和等待）以及 WaitableTimer（可以在选择的时间后或定期自动发送信号）。创建这样的对象会得到一个句柄（HANDLE），就像打开一个文件一样，可以轻松地传递并与常规的 HANDLE 函数一起使用，特别是一系列的等待函数。这些函数允许我们等待各种类型的一个或多个对象，包括重量级同步原语、进程、线程和各种形式的 I/O。

### 轻量级对象

在 Windows API 中，一个轻量级的同步原语包括是“临界区[^4]”（critical section）。

*临界区*这个术语指的是程序的一部分，即代码的“区段”，可能不允许超过一个线程进入。这种保护临界区段机制通常称之为互斥锁。然而，微软为这种机制使用“临界区”的名称，可能因为之前讨论的重量级 mutex 对象已经采用了“互斥锁”这个名称。

Winodows 中的 `CRITICAL_SECTION` 实际上是一个递归互斥锁，只是它使用了“enter”（进入）和“leave”（离开）而不是“lock”（锁定）和“unlock”（解锁）。作为递归互斥锁，它仅被设计用于保护其他的线程。它允许相同的线程多次锁定（或者“进入”）它，也要求该线程也必须相同的次数解锁（“离开”）。

当使用 Rust 包装该类型时，有些东西值得注意。成功地锁定（进入）`CRITICAL_SECTION` 不应该导致对其数据保护的独占引用（`&mut T`）。否则，线程可以使用此来创建对同一数据的两个独占引用，这会立即导致未定义行为。

CRITICAL_SECTION 使用 `InitializeCriticalSection()` 函数来初始化，使用 `DeleteCriticalSection()` 函数来销毁，并且不能被移动。通过 `EnterCriticalSection()` 或者 `TryEnterCriticalSection()` 来锁定，并且使用 `LeaveCriticalSection()` 解锁。

> 在 Rust 1.51 之前，Windows XP 上的 `std::sync::Mutex` 基于（Box 内存分配）CRITICAL_SECTION 对象。（Rust 1.51 放弃了对 Windows XP 的支持。）

#### 精简的读写（SRW）锁[^5]

从 Windows Vista（和 Windows Server 2008）开始，Windows API 包含了一个非常轻量级的优秀锁原语：*精简读写锁*，简称 *SRW 锁*。

SRWLOCK 类型仅是一个指针大小，可以用 `SRWLOCK_INIT` 静态初始化，并且不需要销毁。当不再被使用（借用），我们甚至允许移动它，使它成为 Rust 类型的理想选择。

它通过 `AcquireSRWLockExclusive()`、`TryAcquireSRWLockExclusive()` 和 `ReleaseSRWLockExclusive()` 提供了独占（writer）锁定和解锁，并通过 `AcquireSRWLockShared()`、`TryAcquireSRWLockShared()` 和 `ReleaseSRWLockShared()` 提供了共享（reader）锁定和解锁。通常可以将其用作普通的互斥锁，只需忽略共享（reader）锁定函数即可。

SRW 锁既不优先考虑 writer 也不优先考虑 reader。虽然不能保证，但是它试图去按顺序去服务所有锁请求，以减少性能下降。在已经持有一个共享（reader）锁定的线程上不要尝试获取第二个共享（reader）锁定。如果该操作在另一个线程的独占（writer）锁定操作之后排队，那么这样做可能会导致永久死锁，因为第一个线程已经持有的第一个共享（reader）锁定会阻塞第二个线程。

SRW 锁与条件变量一起引入了 Windows API。`CONDITION_VARIABLE` 仅占用一个指针的大小，可以使用 `CONDITION_VARIABLE_INIT` 进行静态初始化，不需要销毁。只要它没有被使用（被借用），我们也可以移动它。

条件变量不仅通过 SleepConditionVariableSRW 与 SRW 锁一起使用，还可以通过 SleepConditionVariableCS 与临界区一起使用。

唤醒等待线程要么通过 WakeConditionVariable 唤醒单个线程，要么通过 WakeAllConditionVariable 唤醒所有等待线程。

> 最初，标准库中使用的 Windows SRW 锁和条件变量被包装在 Box 中，以避免移动对象。直到我们在 2020 年要求之后，微软才记录了这些对象的可移动性保证。自 Rust 1.49 起，`std::sync::Mutex`、`std::sync::RwLock` 和 `std::sync::Condvar` 在 Windows Vista 及更高版本中直接封装了 SRWLOCK 或 CONDITION_VARIABLE，而无需进行任何内存的分配。

### 基于地址的等待

Windows 8（和 Windows Server 2012）引入了一种新的、更灵活的同步功能类型，非常类似于本章前面讨论的 Linux `FUTEX_WAIT` 和 `FUTEX_WAKE` 操作。

`WaitOnAdderss` 函数可以操作 8 位、16 位、32 位 或 64 位的原子变量。它采用了 4 个参数：原子变量地址、保存期望值的变量地址、原子变量大小（以字节为单位）以及在放弃之前的最大等待最大毫秒数（或者无限超时的 `u32::MAX`）。

就像 FUTEX_WAIT 操作一样，它将原子变量的值与预期值进行比较，如果匹配则进入睡眠状态，等待相应的唤醒操作。检查和睡眠操作相对于唤醒操作是原子发生的，这意味着没有唤醒信号会在两者之间丢失。

唤醒正在等待 `WaitOnAddress` 的线程可以通过 `WakeByAddressSingle` 来唤醒单个线程，或者通过 `WakeByAddressAll` 来唤醒所有等待的线程。这两个函数只接受一个参数：原子变量的地址，该地址也被传递给 `WaitOnAddress`。

Windows API 的一些（但不是全部）同步原语是使用这些函数实现的。更重要的是，它们是构建我们自己的原始物的绝佳基石，我们将在[第9章](./9_Building_Our_Own_Locks.md)中这样做。

## 总结

* *系统调用*（syscall）是进入操作系统内核的调用，与普通函数调用相比，相对较慢。
* 通常，程序不直接进行系统调用，而是通过操作系统的库（如 `libc`）与内核进行交互。在许多操作系统中，这是与内核进行交互的唯一支持方式。
* libc crate 提供了 Rust 代码访问 libc 的能力。
* 在 POSIX 系统上，libc 包含了不仅符合 C 标准所需的内容，还符合 POSIX 标准的内容。
* POSIX 标准包括 *pthread*，这是一个具有并发原语（如 `pthread_mutex_t`）的库。
* pthread 类型是为 C 设计的，而不是为 Rust 设计的。例如，它们不可移动，这可能是一个问题。
* Linux 有一个 *futex* 系统调用，支持在 AtomicU32 上进行几种等待和唤醒操作。等待操作验证原子的期望值，以避免错过通知。
* 除了 pthread，macOS 还提供了 `os_unfair_lock` 作为轻量级锁定原语。
* Windows 的重量级并发原语始终需要与内核进行交互，但可以在进程之间传递，并与标准的 Windows 等待函数一起使用。
* Windows 的轻量级并发原语包括“slim”读写锁（SRW 锁）和条件变量。这些可以很容易地在 Rust 中包装，因为它们是可移动的。
* Windows 还通过 WaitOnAddress 和 WakeByAddress 提供了类似 futex 的基本功能。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./9_Building_Our_Own_Locks.html">下一篇，第九章：构建我们自己的「锁」</a>
</p>

[^1]: [可移植操作系统接口](https://zh.wikipedia.org/wiki/可移植操作系统接口)
[^2]: 是个绝对时间。表示系统（或程序）启动后流逝的时间，更改系统的时间对它没有影响。每次系统（或程序）启动时，该值都归 0
[^3]: 挂钟时间，即现实世界里我们感知到的时间，如 2008-08-08 20:08:00。但对计算机而言，这个时间不一定是单调递增的。因为人觉得当前机器的时间不准，可以随意拨慢或调快。
[^4]: <https://zh.wikipedia.org/zh-cn/臨界區段>
[^5]: <https://learn.microsoft.com/zh-cn/windows/win32/sync/slim-reader-writer--srw--locks>

参考：<https://dashen.tech/2012/07/17/Wall-Clock与Monotonic-Clock/>
