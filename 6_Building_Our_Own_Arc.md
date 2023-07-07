# 第六章：构建我们自己的“Arc”

在[第一章“引用计数”](./1_Basic_of_Rust_Concurrency.md#引用计数)中，我们了解了 `std::sync::Arc<T>` 类型允许通过引用计数共享所有权。`Arc::new` 函数创建一个新的分配，就像 `Box::new`。然而，与 Box 不同的是，克隆 Arc 将共享原始分配，而不是创建一个新的。只有当 Arc 和所有其他的克隆被 drop，共享分配才会被 drop。

这种类型的实现所涉及的内存排序可能是非常有趣的。在本章中，我们将通过实现我们自己的 `Arc<T>` 将更多理论付诸实践。我们将开始一个基础的版本，然后将其扩展到支持循环结构的 *weak 指针*，并且最终将其优化为一个与标准库差不多的实现结束本章。

## 基础的引用计数

我们的第一个版本将使用单个 `AtomicUsize` 去计数 Arc 对象共享分配的数量。让我们开始使用一个持有计数器和 T 对象的结构体：

```rust
struct ArcData<T> {
    ref_count: AtomicUsize,
    data: T,
}
```

注意，该结构体不是公共的。它是我们 Arc 实现的内部实现细节。

接下来是 `Arc<T>` 结构体本身，它实际上仅是一个指向（共享的）`ArcData<T>` 的指针。

使用 `Box<ArcDate<T>>` 作为包装器，并使用标准的 Box 来处理 `ArcData<T>` 的分配可能很诱人。然而，Box 表示独占所有权，并不是共享所有权。我们不能使用引用，因为我们不仅要借用其他所有权的数据，并且它的生命周期（“直到此 Arc 的最后一个克隆被 drop”）无法直接表示为 Rust 的生命周期。

相反，我们将不得不使用指针，并手动处理分配以及所有权的概念。我们将使用 `std::ptr::NonNull<T>`，而不是 `*mut T` 或 `*const T`，它表示一个永远不会为空的指向 T 的指针。这样，使用 None 的空指针表示 `Option<Arc<T>>` 与 `Arc<T>` 的大小相同。

```rust
use std::ptr::NonNull;

pub struct Arc<T> {
    ptr: NonNull<ArcData<T>>,
}
```

使用一个引用或者 Box，编译器会自动地理解它会为哪个 T 实现 Send 和 Sync。然而，当使用原始指针或者 `NonNull`，除非我们明确告知，否则它会保守地认为它永远不会 Send 或 Sync。

发送 `Arc<T>` 跨线程会导致 T 对象被共享，这时要求 T 实现 Sync。类似地，将 `Arc<T>` 跨线程发送可能导致另一个线程 drop 该对象，从而将它转移到其他线程，这里要求 T 实现 Send。换句话说，如果 T 既是 Send 又是 Sync，那么 `Arc<T>` 应该是 Send。对于 Sync 来说也是完全相同的，因为一个共享的 `&Arc<T>` 可以克隆为一个新的 `Arc<T>`。

```rust
unsafe impl<T: Send + Sync> Send for Arc<T> {}
unsafe impl<T: Send + Sync> Sync for Arc<T> {}
```

对于 `Arc<T>::new`，我们必须使用引用计数为 1 的 `ArcData<T>` 创建一个新的分配。我们将使用 `Box::new` 创建新的分配，使用 `Box::leak` 放弃我们对此分配的独占所有权，以及使用 `NonNull::from` 将其转换为指针：

```rust
impl<T> Arc<T> {
    pub fn new(data: T) -> Arc<T> {
        Arc {
            ptr: NonNull::from(Box::leak(Box::new(ArcData {
                ref_count: AtomicUsize::new(1),
                data,
            }))),
        }
    }

    // …
}
```

我们知道只要 Arc 对象存在，指针将总是指向一个有效的 `ArcData<T>`。然然而，这不是编译器知道的或为我们检查的内容，所以通过指针访问 `ArcData` 需要不安全的代码。我们将添加一个私有的辅助函数去从 Arc 获取 ArcData，因为这是我们将执行多次的操作：

```rust
    fn data(&self) -> &ArcData<T> {
        unsafe { self.ptr.as_ref() }
    }
```

使用它，我们现在可以实现 `Deref` trait 以使得我们的 `Arc<T>` 能够像 T 的引用一样透明地操作：

```rust
impl<T> Deref for Arc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.data().data
    }
}
```

注意，我们并没有实现 `DerefMut`。因为 `Arc<T>` 表示共享所有权，我们不能无条件地提供 `&mut T`。

接下来：实现 Clone。在增加引用计数后，克隆的 Arc 将使用相同的指针：

```rust
impl<T> Deref for Arc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.data().data
    }
}
```

我们可以使用 Relaxed 内存排序去增加引用计数，因为没有对其他变量的操作在此原子操作之前或者之后严格地发生。在此操作之前，我们已经可以访问 Arc 包含的 T（通过原始的 Arc），在此操作之后，访问仍然没有改变（但现在至少有两个相同 Arc 对象可以访问数据）。

一个 Arc 需要被克隆多次才有可能导致引用计数溢出，但是在循环中运行 `std::men::forget()` 可以实现这一点。我们可以使用在[第二章的“示例：ID 分配”](./2_Atomics.md#示例id-分配)和[“示例：没有溢出的 ID 分配”](./2_Atomics.md#示例没有溢出的-id-分配)中讨论的任意技术来处理这个问题。

为了在正常（非溢出）情况下保持尽可能高效，我们将保留原始的 `fetch_add`，并在接近溢出时简单地中止整个过程：

```rust
        if self.data().ref_count.fetch_add(1, Relaxed) > usize::MAX / 2 {
            std::process::abort();
        }
```

> 并不会立刻中止进程，在一段时间内，另一个线程也可以调用 `Arc::clone`，进一步增加引用计数器。因此，仅仅检查 `usize::MAX - 1` 将是不够的。然而，使用 `usize::MAX / 2` 限制工作是好的：假设每个线程在内存中至少需要几字节的空间，那么并发存在 `usize::MAX / 2` 数量的线程是不可能的。

就像我们在克隆时增加计数器一样，我们需要在 drop Arc 时需要减少计数器。线程看到计数器从 1 到 0，这意味着该线程 drop 了最后一个 `Arc<T>`，并负责 drop 和释放 `ArcData<T>`。

我们将使用 `Box::from_raw` 去重新获得分配的独占所有权，然后立即使用 `drop()` 将其丢弃：

```rust
impl<T> Drop for Arc<T> {
    fn drop(&mut self) {
        // TODO: Memory ordering.
        if self.data().ref_count.fetch_sub(1, …) == 1 {
            unsafe {
                drop(Box::from_raw(self.ptr.as_ptr()));
            }
        }
    }
}
```

对于这个操作，我们不能使用 Relaxed 排序，因为我们需要确保当我们 drop 它时，没有任何东西仍然在访问数据。换句话说，每个前面的 Arc drop 操作都必须发生在最终 drop 之前。因此，最后的 `fetch_sub` 必须与之前的 `fetch_sub` 操作建立一个 happens-before 关系，我们可以使用 release 和 acquire 排序来实现这一点：例如，从 2 减少到 1 可以有效地“释放”数据，而从 1 减少到 0 则“获取”了对它的所有权。

我们可以使用 `AcqRel` 内存排序来覆盖这两种情况，但只有最后一个递减到 0 才需要 `Acquire`，而其他情况只需要 `Release`。为了提高效率，我们将在 `Release` 用于 `fetch_sub` 操作，并且仅在必要时使用单独的 `Acquire` 屏障：

```rust
        if self.data().ref_count.fetch_sub(1, Release) == 1 {
            fence(Acquire);
            unsafe {
                drop(Box::from_raw(self.ptr.as_ptr()));
            }
        }
```

### 测试它

为了测试我们的 Arc 是否按预期运行，我们可以编写一个单元测试，创建一个包含特殊对象的 `Arc`，让我们知道何时它被 drop 时：

```rust
# [test]
fn test() {
    static NUM_DROPS: AtomicUsize = AtomicUsize::new(0);

    struct DetectDrop;

    impl Drop for DetectDrop {
        fn drop(&mut self) {
            NUM_DROPS.fetch_add(1, Relaxed);
        }
    }

    // Create two Arcs sharing an object containing a string
    // and a DetectDrop, to detect when it's dropped.
    let x = Arc::new(("hello", DetectDrop));
    let y = x.clone();

    // Send x to another thread, and use it there.
    let t = std::thread::spawn(move || {
        assert_eq!(x.0, "hello");
    });

    // In parallel, y should still be usable here.
    assert_eq!(y.0, "hello");

    // Wait for the thread to finish.
    t.join().unwrap();

    // One Arc, x, should be dropped by now.
    // We still have y, so the object shouldn't have been dropped yet.
    assert_eq!(NUM_DROPS.load(Relaxed), 0);

    // Drop the remaining `Arc`.
    drop(y);

    // Now that `y` is dropped too,
    // the object should've been dropped.
    assert_eq!(NUM_DROPS.load(Relaxed), 1);
}
```

编译并运行良好，因此我们的 Arc 似乎按预期运行！虽然这令人兴奋，但并不能证明实现是完全正确的。建议使用涉及多个线程的长压力测试，以获得更多的信心。

<div class="box">
  <h2 style="text-align: center;">Miri</h2>
  <p>使用 Miri 运行测试是非常有用的。Miri 是一个实验性，但非常有用并且强力的工具，用于检查各种未定义形式的不安全代码。</p>

  <p>Miri 是 Rust 编译器中间级别中间表示的解释器。这意味着它不会将你的代码编译成本机处理器指令，而是通过当像类型和生命周期是仍然可用时进行解释。因此，Miri 运行程序的速度比正常运行速度慢很多，但能够检测许多导致未定义行为的错误。</p>

  <p>它包括检测数据竞争的实验性支持，这允许它检测内存排序问题。</p>

  <p>有关更多使用 Miri 的细节和指导，请参见<a href="https://github.com/rust-lang/miri">它的 GitHub 页面</a></p>
</div>

### 可变性

正如之前提及的，我们不能为我们的 Arc 实现 DerefMut。我们不能无条件地承诺对数据的独占访问（`&mut T`），因为它能够通过其他 Arc 对象访问。

然而，我们可以有条件地允许独占访问。我们可以创建一个方法，如果引用计数为 1，则提供 `&mut T`，这证明没有其他 Arc 对象可以用来访问相同的数据。

该函数我们将称它为 `get_mut`，它必须接受一个 `&mute Self` 以确保没有其他的相同的东西使用 Arc 获取 T。如果这个 Arc 仍然可以共享，知道只有一个 Arc 对象是没有意义的。

我们需要使用 acquire 内存排序去确保之前拥有 Arc 克隆的线程不再访问数据。我们需要与导致引用计数为 1 的每个单独的 drop 建立一个 happens-before 关系。

这仅在引用计数实际为 1 时才重要：如果引用计数高，我们将不再提供一个 `&mut T`，并且内存排序是无关紧要。因此，我们可以使用 relaxed load 操作，随后跟条件行的 acquire 屏障，如下所示：

```rust
    pub fn get_mut(arc: &mut Self) -> Option<&mut T> {
        if arc.data().ref_count.load(Relaxed) == 1 {
            fence(Acquire);
            // Safety: Nothing else can access the data, since
            // there's only one Arc, to which we have exclusive access.
            unsafe { Some(&mut arc.ptr.as_mut().data) }
        } else {
            None
        }
    }
```

该函数并不采用 `self` 参数，而是接受一个常规的参数（名称 `arc`）。这意味着，它仅可以 `Arc::get_mut(&mut a)` 这样的方式调用，而不以 `a.get_mut()` 方式调用。对于实现了 `Deref` 的类型来说，这是可取的，以避免与底层类型 `T` 上的同名方法产生歧义。

返回的可以引用会隐式地从参数重借用生命周期，这意味着只要返回 `&mut T` 仍然存在，原始的 Arc 就不能被其他代码使用，从而允许安全的可变性操作。

当 `&mut T` 的生命周期过期后，Arc 可以在此被使用以及与其他线程共享。也许有人可能会想知道，在之后访问数据的线程是否需要关注内存顺序。然而，这是用于与其他线程共享 Arc（或着新克隆）的机制负责的。（例如 mutex、channel 或者产生的新线程。）

## Weak 指针

### 测试它2

### 优化

## 总结

* `Arc<T>` 提供一个引用计数分配的共享所有权。
* 通过检查引用计数是否确实是一个 `Arc<T>`，可以有条件地提供独占访问（`&mut T`）。
* 增加原子引用计数可以使用 relaxed 操作，但是最终的递减必须与之前的递减同步。
* *weak 指针*（`Weak<T>`）可以用于避免循环。
* `NonNull<T>` 类型表示一个指向 T 的指针，但是从不是空。
* `ManuallyDrop<T>` 类型可以用于使用不安全代码时，手动决定何时 drop T。
* 一旦涉及一个以上的原子变量，事情就会变得更加复杂。
* 实现特定的（自旋）锁有时可能是同时对多个原子变量进行操作的有效策略。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./7_Understanding_the_Processor.html">下一篇，第七章：理解处理器</a>
</p>
