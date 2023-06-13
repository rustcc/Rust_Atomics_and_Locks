# 第九章：构建我们自己的「锁」

在该章节，我们将建造属于我们自己的 `mutex`、`condition variable` 以及 `reader-writer lock`。对于它们中的任何一个，我们都会从一个非常基础的版本开始，然后扩展它以使其更高效。

因为我们并不会使用来自标准库中的锁类型（因为这将是作弊行为），因此我们将不得不使用来自[第八章](./8_Operating_System_Primitives.md)的工具，才能够在不忙碌循环（`busy-looping`[^1]）的情况下使线程等待。然而，正如我们在该章节看到的，操作系统提供的可用工具因平台而异，因此很难去构建跨平台工作的东西。

幸运地是，更多现代化操作系统都支持类似 `futex` 的功能，或者至少支持唤醒（`wake`）和等待（`wait`）操作。正如我们在[第八章](./8_Operating_System_Primitives.md)看到的，Linux 自从 2003 年就一直支持 futex 系统调用，Winodws 自从 2012 年就支持 `WaitOnAddress` 系列功能，FreeBSD 自从 2016 年就将 `_umtx_op` 作为系统调用的一部分，等等。

最让人意外的是 macOS。尽管它的内核支持这些操作，但是它并没有暴露任意稳定、公共的 C 函数给我们使用。然而，macOS 附带了一个最新版本的 `libc++`（这是一个 C++ 标准库的实现）。该标准库包含对 C++20 的支持，该版本内置了非常基础对原子等待和唤醒操作（像 `std::atomic<T>::wait()`）。尽管由于各种原因，Rust 利用这些还非常的棘手，然而，这当然是可能的，这也可以让我们在 macOS 上访问基本的像 futex 的等待和唤醒功能。

我们将不再深入研究哪些复杂的细节，而是选择利用 `crates.io` 的 `atomic-wait` crate，为我们的「锁」原语提供基础的构建模块。该 crate 提供了三个函数：`wait()`、`wake_one()` 以及 `wake_all()`。它使用我们上面讨论的特定于平台规范的实现，为所有主要的平台实现了这些功能。这意味着我们只要坚持使用这三个函数，我们不再需要考虑任何平台的特定细节。

这些函数的行为就像我们在 Linux [第八章中的“Futex”](./8_Operating_System_Primitives.md#Futex)中实现的同名函数一样，不过让我们快速回顾一下如何工作的。

## Mutex

### 避免系统调用

### 进一步优化

### 基准测试

## Condition Variable

### 避免系统调用2

### 避免虚假唤醒

## Reader-Writer Lock

### 避免 writer 忙碌循环

### 避免 writer 陷入饥饿

## 总结

[^1]: <https://zh.wikipedia.org/wiki/忙碌等待>
