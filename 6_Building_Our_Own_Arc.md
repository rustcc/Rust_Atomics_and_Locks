# 第六章：构建我们自己的“Arc”

在[第一章“引用计数”](./1_Basic_of_Rust_Concurrency.md#引用计数)中，我们了解了 `std::sync::Arc<T>` 类型允许通过引用计数共享所有权。`Arc::new` 函数创建一个新的内存分配，就像 `Box::new`。然而，与 Box 不同的是，克隆 Arc 将共享原始的内存分配，而不是创建一个新的。只有当 Arc 和所有其他的克隆被丢弃，共享的内存分配才会被丢弃。

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

使用 `Box<ArcDate<T>>` 作为包装器，并使用标准的 Box 来处理 `ArcData<T>` 的内存分配可能很诱人。然而，Box 表示独占所有权，并不是共享所有权。我们不能使用引用，因为我们不仅要借用其他所有权的数据，并且它的生命周期（“直到此 Arc 的最后一个克隆被丢弃”）无法直接表示为 Rust 的生命周期。

相反，我们将不得不使用指针，并手动处理内存分配以及所有权的概念。我们将使用 `std::ptr::NonNull<T>`，而不是 `*mut T` 或 `*const T`，它表示一个永远不会为空的指向 T 的指针。这样，使用 None 的空指针表示 `Option<Arc<T>>` 与 `Arc<T>` 的大小相同。

```rust
use std::ptr::NonNull;

pub struct Arc<T> {
    ptr: NonNull<ArcData<T>>,
}
```

使用一个引用或者 Box，编译器会自动地理解它会为哪个 T 实现 Send 和 Sync。然而，当使用原始指针或者 `NonNull`，除非我们明确告知，否则它会保守地认为它永远不会 Send 或 Sync。

发送 `Arc<T>` 跨线程会导致 T 对象被共享，这时要求 T 实现 Sync。类似地，将 `Arc<T>` 跨线程发送可能导致另一个线程丢弃该对象，从而将它转移到其他线程，这里要求 T 实现 Send。换句话说，如果 T 既是 Send 又是 Sync，那么 `Arc<T>` 应该是 Send。对于 Sync 来说也是完全相同的，因为一个共享的 `&Arc<T>` 可以克隆为一个新的 `Arc<T>`。

```rust
unsafe impl<T: Send + Sync> Send for Arc<T> {}
unsafe impl<T: Send + Sync> Sync for Arc<T> {}
```

对于 `Arc<T>::new`，我们必须使用引用计数为 1 的 `ArcData<T>` 创建一个新的内存分配。我们将使用 `Box::new` 创建新的内存分配，使用 `Box::leak` 放弃我们对此内存分配的独占所有权，以及使用 `NonNull::from` 将其转换为指针：

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

接下来：实现 Clone。在**增加**引用计数后，克隆的 Arc 将使用相同的指针：

```rust
impl<T> Deref for Arc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.data().data
    }
}
```

我们可以使用 Relaxed 内存排序去递增引用计数，因为没有对其他变量的操作在此原子操作之前或者之后严格地发生。在此操作之前，我们已经可以访问 Arc 包含的 T（通过原始的 Arc），在此操作之后，访问仍然没有改变（但现在至少有两个相同 Arc 对象可以访问数据）。

一个 Arc 需要被克隆多次才有可能导致引用计数溢出，但是在循环中运行 `std::men::forget()` 可以实现这一点。我们可以使用在[第二章的“示例：ID 分配”](./2_Atomics.md#示例id-分配)和[“示例：没有溢出的 ID 分配”](./2_Atomics.md#示例没有溢出的-id-分配)中讨论的任意技术来处理这个问题。

为了在正常（非溢出）情况下保持尽可能高效，我们将保留原始的 `fetch_add`，并在接近溢出时简单地中止整个过程：

```rust
        if self.data().ref_count.fetch_add(1, Relaxed) > usize::MAX / 2 {
            std::process::abort();
        }
```

> 并不会立刻中止进程，在一段时间内，另一个线程也可以调用 `Arc::clone`，进一步**增加**引用计数器。因此，仅仅检查 `usize::MAX - 1` 将是不够的。然而，使用 `usize::MAX / 2` 限制工作是好的：假设每个线程在内存中至少需要几字节的空间，那么并发存在 `usize::MAX / 2` 数量的线程是不可能的。

就像我们在克隆时递增计数器一样，我们在丢弃 Arc 时需要递减计数器。线程看到计数器从 1 到 0，这意味着该线程丢弃了最后一个 `Arc<T>`，并负责丢弃和释放 `ArcData<T>`。

我们将使用 `Box::from_raw` 去重新获得内存的独占所有权，然后立即使用 `drop()` 将其丢弃：

```rust
impl<T> Drop for Arc<T> {
    fn drop(&mut self) {
        // TODO：内存排序
        if self.data().ref_count.fetch_sub(1, …) == 1 {
            unsafe {
                drop(Box::from_raw(self.ptr.as_ptr()));
            }
        }
    }
}
```

对于这个操作，我们不能使用 Relaxed 排序，因为我们需要确保当我们丢弃它时，没有任何东西仍然在访问数据。换句话说，每个之前的 Arc 丢弃操作都必须发生在最终丢弃之前。因此，最后的 `fetch_sub` 必须与之前的 `fetch_sub` 操作建立一个 happens-before 关系，我们可以使用 release 和 acquire 排序来实现这一点：例如，从 2 递减到 1 可以有效地“释放”数据，而从 1 递减到 0 则“获取”了对它的所有权。

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

为了测试我们的 Arc 是否按预期运行，我们可以编写一个单元测试，创建一个包含特殊对象的 `Arc`，让我们知道何时它被丢弃时：

```rust
#[test]
fn test() {
    static NUM_DROPS: AtomicUsize = AtomicUsize::new(0);

    struct DetectDrop;

    impl Drop for DetectDrop {
        fn drop(&mut self) {
            NUM_DROPS.fetch_add(1, Relaxed);
        }
    }

    // 创建两个 Arc，共享一个对象，包含一个字符串
    // 和一个 DetecDrop，以当它被丢弃时去检测。
    let x = Arc::new(("hello", DetectDrop));
    let y = x.clone();

    // 发送 x 到另一个线程，并在那里使用它。
    let t = std::thread::spawn(move || {
        assert_eq!(x.0, "hello");
    });

    // 这是并行的，y 应该仍然在这里可用。
    assert_eq!(y.0, "hello");

    // 等待线程完成。
    t.join().unwrap();

    // Arc，x 现在应该被丢弃。
    // 我们仍然有 y，因此对象仍然还没有被丢弃。
    assert_eq!(NUM_DROPS.load(Relaxed), 0);

    // 丢弃剩余的 `Arc`。
    drop(y);

    // 现在，`y` 也被丢弃，
    // 对象应该也被丢弃。
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

我们需要使用 acquire 内存排序去确保之前拥有 Arc 克隆的线程不再访问数据。我们需要与导致引用计数为 1 的每个单独的 `drop` 建立一个 happens-before 关系。

这仅在引用计数实际为 1 时才重要：如果引用计数高，我们将不再提供一个 `&mut T`，并且内存排序是无关紧要。因此，我们可以使用 relaxed load 操作，随后跟条件行的 acquire 屏障，如下所示：

```rust
    pub fn get_mut(arc: &mut Self) -> Option<&mut T> {
        if arc.data().ref_count.load(Relaxed) == 1 {
            fence(Acquire);
            // 安全性：没有任何其他东西可以访问 data，因为
            // 只有一个 Arc，我们拥有独占访问的权限。
            unsafe { Some(&mut arc.ptr.as_mut().data) }
        } else {
            None
        }
    }
```

该函数并不接收 `self` 参数，而是接受一个常规的参数（名称 `arc`）。这意味着，它仅可以 `Arc::get_mut(&mut a)` 这样的方式调用，而不以 `a.get_mut()` 方式调用。对于实现了 `Deref` 的类型来说，这是可取的，以避免与底层类型 `T` 上的同名方法产生歧义。

返回的可以引用会隐式地从参数重借用生命周期，这意味着只要返回 `&mut T` 仍然存在，原始的 Arc 就不能被其他代码使用，从而允许安全的可变性操作。

当 `&mut T` 的生命周期过期后，Arc 可以在此被使用以及与其他线程共享。也许有人可能会想知道，在之后访问数据的线程是否需要关注内存排序。然而，这是用于与其他线程共享 Arc（或着新克隆）的机制负责的。（例如 mutex、channel 或者产生的新线程。）

## Weak 指针

当表示在内存中多个对象组成的结构时，引用计数非常有用。例如，在树结构中的每个节点可以包含对其子节点的 Arc 引用。这样，当我们丢弃一个节点时，不再使用的孩子节点也会被（递归地）丢弃。

然而，对于*循环结构*来说，这会失效。如果一个子节点也包含对它父节点的 Arc 引用，那么当所有 Arc 引用都不存在时，两者都不会被丢弃，因为始终存至少有一个 Arc 引用仍然指向它们。

标准库的 Arc 提供了解决这个问题的办法：`Weak<T>`。`Weak<T>`（也被称为 *weak 指针*），行为有点像 `Arc<T>`，但是并不会阻止对象被丢弃。T 可以在多个 `Arc<T>` 和 `Weak<T>` 对象之间共享，但是当所有 `Arc<T>` 对象都消失时，不管是否还有 `Weak<T>` 对象，T 都会被丢弃。

着意味着 `Weak<T>` 可以没有 T 而存在，因此无法像 Arc 那样无条件地提供 `&T`。然而，为了获取给定 `Weak<T>` 中的 T，可以通过 `Arc<T>` 的 `upgrade()` 方法来升级。这个方法返回一个 `Option<Arc<T>>`，如果 T 已经被丢弃，则返回 None。

在基于 Arc 的结构中，可以使用 Weak 打破循环引用。例如，在树结构中的子节点使用 Weak，而不是使用 Arc 来引用它们的父节点。然后，尽管子节点存在，也不会阻止父节点被丢弃。

让我们来实现这个功能。

与之前一样，当 Arc 对象的数量到达 0 时，我们可以丢弃包含 T 的对象。然而，我们仍然不能丢弃和释放 ArcData，因为可能仍然有 weak 指针指向它。只有当最后一个 Weak 指针也不存在时，我们才能丢弃和释放 ArcData。

因此，我们将使用两个计数器：一个计算“引用 T 对象的数量”，另一个计算“引用 `ArcData<T>` 对象的数量”。换句话说，第一个计数器与之前相同：它计算 Arc 对象的数量，而第二个计数器计算 Arc 和 Weak 对象的数量。

我们还需要一种在 `ArcData<T>` 被 weak 指针使用时，允许我们丢弃包含的对象（T）的机制。我们将使用 `Option<T>`，这样当数据被丢弃时可以使用 None，并将其包装在 UnsafeCell 中进行内部可变性（[在第一章“内部可变性”中](./1_Basic_of_Rust_Concurrency.md#内部可变性)），以允许在 `ArcData<T>` 不是独占所有权时发生这种情况：

```rust
struct ArcData<T> {
    /// `Arc` 的数量
    data_ref_count: AtomicUsize,
    /// `Arc` 和 `Weak` 总共的数量。
    alloc_ref_count: AtomicUsize,
    /// 持有的数据。如果仅剩下 weak 指针，则是 `None`。
    data: UnsafeCell<Option<T>>,
}
```

如果我们认为 `Weak<T>` 时保持 `ArcData<T>` 存活的对象，那么将 `Arc<T>` 实现为包含 `Weak<T>` 的结构体可能是有意义的，因为 `Arc<T>` 需要做相同的事情，而且还有更多的功能。

```rust
pub struct Arc<T> {
    weak: Weak<T>,
}

pub struct Weak<T> {
    ptr: NonNull<ArcData<T>>,
}

unsafe impl<T: Sync + Send> Send for Weak<T> {}
unsafe impl<T: Sync + Send> Sync for Weak<T> {}
```

新函数与之前的基本相同，除了它现在有两个计数器可以同时初始化：

```rust
impl<T> Arc<T> {
    pub fn new(data: T) -> Arc<T> {
        Arc {
            weak: Weak {
                ptr: NonNull::from(Box::leak(Box::new(ArcData {
                    alloc_ref_count: AtomicUsize::new(1),
                    data_ref_count: AtomicUsize::new(1),
                    data: UnsafeCell::new(Some(data)),
                }))),
            },
        }
    }

    //…
}
```

就像之前一样，我们假设 ptr 字段总是指向有效的 `ArcData<T>`。这一次，我们将在 `Weak<T>` 上将该假设编码为私有 `data()` 辅助方法：

```rust
impl<T> Weak<T> {
    fn data(&self) -> &ArcData<T> {
        unsafe { self.ptr.as_ref() }
    }

    // …
}
```

在 `Arc<T>` 的 Deref 实现中，我们现在不得不使用 `UnsafeCell::get()` 拉锯得到 cell 内容的指针，并使用不安全的代码去承诺它此时可以共享。我们也需要 `as_ref().unwrap()` 去获取 `Option<T>` 引用。我们不必担心引发 panic，因为只有在没有 Arc 对象时 Option 才会为 None。

```rust
impl<T> Deref for Arc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        let ptr = self.weak.data().data.get();
        // 安全性：由于 Arc 包装 data，
        // data 存在并可以共享。
        unsafe { (*ptr).as_ref().unwrap() }
    }
}
```

`Weak<T>` 的克隆实现非常简单；它与我们之前 `Arc<T>` 的克隆实现几乎相同：

```rust
impl<T> Clone for Weak<T> {
    fn clone(&self) -> Self {
        if self.data().alloc_ref_count.fetch_add(1, Relaxed) > usize::MAX / 2 {
            std::process::abort();
        }
        Weak { ptr: self.ptr }
    }
}
```

在我们新 `Arc<T>` 的克隆实现中，我们需要同时递增两个计数器。我们将简单地使用 `self.weak.clone()` 为第一个计数器重用上面的代码，因此我们只需要手动递增第二个计数器：

```rust
impl<T> Clone for Arc<T> {
    fn clone(&self) -> Self {
        let weak = self.weak.clone();
        if weak.data().data_ref_count.fetch_add(1, Relaxed) > usize::MAX / 2 {
            std::process::abort();
        }
        Arc { weak }
    }
}
```

当计数器从 1 到 0 时，丢弃 Weak 应该递减它的计数，以及丢弃和释放 ArcData。这与我们之前 Arc 的 Drop 实现相同。

```rust
impl<T> Drop for Weak<T> {
    fn drop(&mut self) {
        if self.data().alloc_ref_count.fetch_sub(1, Release) == 1 {
            fence(Acquire);
            unsafe {
                drop(Box::from_raw(self.ptr.as_ptr()));
            }
        }
    }
}
```

丢弃 Arc 应该同时递减两个计数器。注意，其中一个计数器已经被自动地处理，因为每个 Arc 都包含一个 Weak，因此删除 Arc 也会删除一个 Weak。我们仅需要处理另一个计数器：

```rust
impl<T> Drop for Arc<T> {
    fn drop(&mut self) {
        if self.weak.data().data_ref_count.fetch_sub(1, Release) == 1 {
            fence(Acquire);
            let ptr = self.weak.data().data.get();
            // 安全性：data 引用计数是 0，
            // 因此没有任何东西可以访问它。
            unsafe {
                (*ptr) = None;
            }
        }
    }
}
```

在 Rust 中丢弃一个对象将首先运行它的 `Drop::drop` 函数（如果它实现了 `Drop`），然后递归地逐个地丢弃它的所有字段。

`get_mut` 方法中的检查基本上保持不变，除了现在需要考虑 weak 指针。看起来似乎可以在检查独占性时忽略 weak 指针，但是 `Weak<T>` 可以随时升级为 `Arc<T>`。因此，在给出 `&mut T` 之前，`get_mut` 必须检查是否还有其他 `Arc<T>` 或者 `Weak<T>` 指针：

```rust
impl<T> Arc<T> {
    // …

    pub fn get_mut(arc: &mut Self) -> Option<&mut T> {
        if arc.weak.data().alloc_ref_count.load(Relaxed) == 1 {
            fence(Acquire);
            // 安全性：没有任何东西可以访问 data，因为
            // 仅有一个 Arc，并且我们拥有独占访问权限，
            // 也没有 Weak 指针
            let arcdata = unsafe { arc.weak.ptr.as_mut() };
            let option = arcdata.data.get_mut();
            // 我们知道 data 是仍然可获得的，因为我们
            // 有一个 Arc 去包裹它，因此不会 panic。
            let data = option.as_mut().unwrap();
            Some(data)
        } else {
            None
        }
    }

    // …
}
```

接下来：是升级 weak 指针。当数据仍然存在时，才能升级 Weak 到 Arc。如果仅剩下 Weak 指针，则没有数据通过 Arc 共享了。因此，我们需要递增 Arc 的计数器，但只能在计数器不为 0 时才能这样做。我们将使用「比较并交换」循环（[第二章的“比较并交换操作”](./2_Atomics.md#比较并交换操作)）来做这些。

与之前一样，对于递增引用计数，relaxed 内存排序是好的。在这个原子操作之前或之后，没有其他变量的操作需要严格执行。

```rust
impl<T> Weak<T> {
    //…

    pub fn upgrade(&self) -> Option<Arc<T>> {
        let mut n = self.data().data_ref_count.load(Relaxed);
        loop {
            if n == 0 {
                return None;
            }
            assert!(n <= usize::MAX / 2);
            if let Err(e) =
                self.data()
                    .data_ref_count
                    .compare_exchange_weak(n, n + 1, Relaxed, Relaxed)
            {
                n = e;
                continue;
            }
            return Some(Arc { weak: self.clone() });
        }
    }
}
```

相反，从 `Arc<T>` 获得 `Weak<T>` 要简单得多：

```rust
impl<T> Arc<T> {
    // …

    pub fn downgrade(arc: &Self) -> Weak<T> {
        arc.weak.clone()
    }
}
```

### 测试它2

为了快速测试我们创建的内容，我们将修改之前的单元测试，以使用 weak 指针，并验证它们是否可以在预期的情况下升级：

```rust
#[test]
fn test() {
    static NUM_DROPS: AtomicUsize = AtomicUsize::new(0);

    struct DetectDrop;

    impl Drop for DetectDrop {
        fn drop(&mut self) {
            NUM_DROPS.fetch_add(1, Relaxed);
        }
    }

    // 创建一个 Arc，同时也创建两个 weak 指针。
    let x = Arc::new(("hello", DetectDrop));
    let y = Arc::downgrade(&x);
    let z = Arc::downgrade(&x);

    let t = std::thread::spawn(move || {
        // 此刻，Weak 指针应该被升级。
        let y = y.upgrade().unwrap();
        assert_eq!(y.0, "hello");
    });
    assert_eq!(x.0, "hello");
    t.join().unwrap();

    // data 仍然不应该被丢弃，
    // 并且 weak 指针应该被升级。
    assert_eq!(NUM_DROPS.load(Relaxed), 0);
    assert!(z.upgrade().is_some());

    drop(x);

    // 现在，data 已经被丢弃，并且
    // weak 指针应该不再被升级。
    assert_eq!(NUM_DROPS.load(Relaxed), 1);
    assert!(z.upgrade().is_none());
}
```

这也毫无问题地编译和运行，这给我们留下了一个非常可用的手工 Arc 实现。

### 优化

虽然 weak 指针是可用的，但 Arc 类型通常用于没有任何 weak 的情况下。我们上次实现的缺点是，克隆和丢弃 Arc 现在都需要两个原子操作，因为它们不得不递增或递减两个计数器。这使得 Arc 用于丢弃 weak 指针的开销增大，即使它们没有使用 weak 指针。

似乎解决的方案是分别计算 `Arc<T>` 和 `Weak<T>` 指针的计数，但那样我们将无法原子地检测这两个计数器是否为 0。为了理解这个问题，想象我们有一个线程执行以下令人恼火的函数：

```rust
fn annoying(mut arc: Arc<Something>) {
    loop {
        let weak = Arc::downgrade(&arc);
        drop(arc);
        println!("I have no Arc!"); // 1
        arc = weak.upgrade().unwrap();
        drop(weak);
        println!("I have no Weak!"); // 2
    }
}
```

该线程不断降级和升级一个 Arc，以至于它反复地循环未持有 Arc（1）和未持有 Weak（2）的片刻。如果我们同时检查两个计数器，查看是否还有线程仍然使用的内存，如果我们不幸地在它的第一个输出语句（1）期间检查 `Arc` 计数，但在第二个输出语句（2）期间检查 `Weak` 计数器，该线程能够隐藏它的存在。

在我们上次实现中，我们通过将每个 `Arc` 也计为 `Weak` 来解决了这个问题。一种更微妙的解决是将所有的 Arc 指针合并为一个单独的 Weak 指针进行计数。这样，只要周围仍然有一个 Arc 对象，weak 指针计数器（`alloc_ref_count`）将从不会到达 0，就像在我们上次实现中一样，但是克隆的 Arc 不需要触及该计数器。只有当最后一个 Arc 被丢弃是，weak 指针计数才会递减。

让我们尝试它。

这次，我们不能简单地将 `Arc<T>` 实现为对 `Weak<T>` 的包装，所以两者都将包装一个非空指针到内存分配中：

```rust
pub struct Arc<T> {
    ptr: NonNull<ArcData<T>>,
}

unsafe impl<T: Sync + Send> Send for Arc<T> {}
unsafe impl<T: Sync + Send> Sync for Arc<T> {}

pub struct Weak<T> {
    ptr: NonNull<ArcData<T>>,
}

unsafe impl<T: Sync + Send> Send for Weak<T> {}
unsafe impl<T: Sync + Send> Sync for Weak<T> {}
```

因为我们正在优化我们的实现，我们也能通过使用 `std::mem::ManyallyDrop<T>` 来稍微减小 `ArcData<T>` 的大小。我们使用 `Option<T>` 是为了能够在丢弃数据时，将 `Some(T)` 替换为 None，但实际上我们并不需要单独的 None 状态去告诉我们数据消失了，因为 `Arc<T>` 的存在或者缺失已经告诉我们这一点。`ManyallyDrop<T>` 占用了与 T 相同的数量的空间，但是这允许我们任意时刻通过不安全地调用 `ManuallyDrop::drop()` 来手动丢弃 T：

```rust
use std::mem::ManuallyDrop;

struct ArcData<T> {
    /// `Arc` 的数量
    data_ref_count: AtomicUsize,
    /// `Weak` 的数量，如果有任意的 `Arc` 的数量，加上 1。
    alloc_ref_count: AtomicUsize,
    /// 持有的数据。如果仅剩下 weak 指针，就丢弃它。
    data: UnsafeCell<ManuallyDrop<T>>,
}
```

`Arc::new()` 函数几乎持不变，像之前一样初始化两个计数器，但是现在使用 `ManuallyDrop::new()`，而不是 `Some()`：

```rust
impl<T> Arc<T> {
    pub fn new(data: T) -> Arc<T> {
        Arc {
            ptr: NonNull::from(Box::leak(Box::new(ArcData {
                alloc_ref_count: AtomicUsize::new(1),
                data_ref_count: AtomicUsize::new(1),
                data: UnsafeCell::new(ManuallyDrop::new(data)),
            }))),
        }
    }

    // …
}
```

Deref 的实现不能再在 Weak 类型上使用私有数据方法，因此我们将在 `Arc<T>` 上添加相同的私有辅助函数：

```rust
impl<T> Arc<T> {
    // …

    fn data(&self) -> &ArcData<T> {
        unsafe { self.ptr.as_ref() }
    }

    // …
}

impl<T> Deref for Arc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        // 安全性：因为有一个 Arc 包裹 data，
        // data 存在不能呗共享。
        unsafe { &*self.data().data.get() }
    }
}
```

`Weak<T>` 的克隆和 `Drop` 实现与我们上次实现完全相同。包括私有的 `Weak::data` 辅助函数，这里是为了完整性：

```rust
impl<T> Weak<T> {
    fn data(&self) -> &ArcData<T> {
        unsafe { self.ptr.as_ref() }
    }

    // …
}

impl<T> Clone for Weak<T> {
    fn clone(&self) -> Self {
        if self.data().alloc_ref_count.fetch_add(1, Relaxed) > usize::MAX / 2 {
            std::process::abort();
        }
        Weak { ptr: self.ptr }
    }
}

impl<T> Drop for Weak<T> {
    fn drop(&mut self) {
        if self.data().alloc_ref_count.fetch_sub(1, Release) == 1 {
            fence(Acquire);
            unsafe {
                drop(Box::from_raw(self.ptr.as_ptr()));
            }
        }
    }
}
```

现在我终于来到这个新的优化实现的重点内容——克隆 `Arc<T>` 现在只需要操作一个计数器：

```rust
impl<T> Clone for Arc<T> {
    fn clone(&self) -> Self {
        if self.data().data_ref_count.fetch_add(1, Relaxed) > usize::MAX / 2 {
            std::process::abort();
        }
        Arc { ptr: self.ptr }
    }
}
```

类似地，丢弃 `Arc<T>` 现在也只需要递减一个计数器，除了看到最后一个 `drop` 操作会将计数器从 1 递减到 0。在这中情况下，weak 指针计数也需要递减，以便在没有 weak 指针时到达 0。我们通过简单地创建一个无关紧要的 `Weak<T>`，然后立即丢弃它来实现这一点：

```rust
impl<T> Drop for Arc<T> {
    fn drop(&mut self) {
        if self.data().data_ref_count.fetch_sub(1, Release) == 1 {
            fence(Acquire);
            // 安全性：data 引用计数是 0，
            // 所以没有东西再访问 data。
            unsafe {
                ManuallyDrop::drop(&mut *self.data().data.get());
            }
            // 现在，没有 `Arc<T>` 了，
            // 丢弃所有表示 `Arc<T>` 的隐式 weak 指针。
            drop(Weak { ptr: self.ptr });
        }
    }
}
```

`Weak<T>` 上的 upgrade 方法基本保持不变，只是不再克隆 weak 指针，因为它不再需要递增 weak 计数器。仅当内存分配中至少有一个 `Arc<T>` 才会成功，这意味着 Arc 对象已经计入了 weak 计数器。

```rust
impl<T> Weak<T> {
    // …

    pub fn upgrade(&self) -> Option<Arc<T>> {
        let mut n = self.data().data_ref_count.load(Relaxed);
        loop {
            if n == 0 {
                return None;
            }
            assert!(n <= usize::MAX / 2);
            if let Err(e) =
                self.data()
                    .data_ref_count
                    .compare_exchange_weak(n, n + 1, Relaxed, Relaxed)
            {
                n = e;
                continue;
            }
            return Some(Arc { ptr: self.ptr });
        }
    }
}
```

到目前为止，这与我们之前的实现差距是非常小的。然而，问题出现在我们仍然要实现最后两个方法：`downgrade` 和 `get_mut`。

与之前不同，`get_mut` 方法现在需要检查是否都设置为 1，以决定是否指进剩下一个 `Arc<T>` 以及没有保留 `Weak<T>`，因为一个 weak 指针计数现在可以表示多个 `Arc<T>` 指针。读取计数器是发生在（稍微）不同时间的两个分开的操作，所以我们不得不非常小心，以确保不会错过任何并发的 downgrade，就像我们在[“优化”](#优化)一节示例的开头所见。

如果我们首先检查 `data_ref_count` 是 1，那么我们在检查另一个计数器之前，可能错过随后的 `upgrade()`。但是，如果我们姜茶 `alloc_ref_count` 是 1，那么在检查另一个计数器之前，可能错过随后的 `downgrade()`。

摆脱这个困境的方法是通过“锁定”weak 指针计数器来暂时阻塞 `downgrade()` 操作。为此，我们不需要像 mutex 那样的东西。我们可以使用一个特殊的值，如  `usize::MAX`，来表示 weak 指针计数器的特殊“锁定”状态。它只会在加载另一个计数器之前很短暂地被锁定，因此 downgrade 方法只需在它解锁之前自旋，以防止在正好与 `get_mut` 并发运行的极端情况下出现问题。

因此，在 `get_mut` 方法中，我们首先需要检查 `alloc_ref_count` 是否为 1，并在确实为 1 的情况下将其替换为 `usize::MAX`。这是 compare_exchange 的任务。

然后，我们需要检查其他计数器是否也为 1，之后我们可以立即解锁弱 weak 指针计数器。如果第二个计数器也为 1，我们就能知道我们有独占访问内存分配和数据的权限，可以返回一个 `&mut T`。

```rust
    pub fn get_mut(arc: &mut Self) -> Option<&mut T> {
        // Acquire 与 Weak::drop 的 Release 递减操作匹配，以确保任意的
        // 指针升级在下一个 data_ref_count.load 中可见。
        if arc.data().alloc_ref_count.compare_exchange(
            1, usize::MAX, Acquire, Relaxed
        ).is_err() {
            return None;
        }
        let is_unique = arc.data().data_ref_count.load(Relaxed) == 1;
        // Release 与 `downgrade` 中的 Acquire 操作匹配，以确保任意
        // 在 `downgrade` 之后对 data_ref_count 的改变都不会
        // 改变以上 is_unique 的结果。
        arc.data().alloc_ref_count.store(1, Release);
        if !is_unique {
            return None;
        }
        // Acquire 去匹配 Arc::drop 的 Release 递减操作，以确保没有
        // 其他东西正在访问 data。
        fence(Acquire);
        unsafe { Some(&mut *arc.data().data.get()) }
    }
```

正如你所预期的那样，锁定操作（compare_exchange）将使用 `Acquire` 内存排序，而解锁操作（store）将使用 `Release` 内存排序。

如果我们为 `compare_exchange` 操作使用 `Relaxed` 内存排序，那么在从 `data_ref_count` 加载时，可能无法看到新升级的 `Weak` 指针的新值，尽管 `compare_exchange` 已经确认每个 `Weak` 指针都已经被丢弃。

如果我们为 store 操作使用 `Relaxed` 内存排序，那么之前的加载操作可能会观察到未来的 `Arc::drop` 结果，而该 `Arc` 仍然可以降级。

`Acquire` 屏障与之前相同：它与 `Arc::Drop` 中的 `release-decrement` 操作同步，以确保通过之前的 Arc 克隆的每次访问都发生在新的独占访问之前。

最后一部分是 `downgrade` 方法，它将检查特殊的 `usize::MAX` 值，以查看 weak 指针计数器是否被锁定，并在解锁之前自旋等待。就像在 upgrade 实现中一样，我们将在递增之前使用「比较并交换」循环来检查特殊值和溢出：

```rust
    pub fn downgrade(arc: &Self) -> Weak<T> {
        let mut n = arc.data().alloc_ref_count.load(Relaxed);
        loop {
            if n == usize::MAX {
                std::hint::spin_loop();
                n = arc.data().alloc_ref_count.load(Relaxed);
                continue;
            }
            assert!(n <= usize::MAX / 2);
            // Acquire 与 get_mut 的 release-store 操作同步。
            if let Err(e) =
                arc.data()
                    .alloc_ref_count
                    .compare_exchange_weak(n, n + 1, Acquire, Relaxed)
            {
                n = e;
                continue;
            }
            return Weak { ptr: arc.ptr };
        }
    }
```

我们为 `compare_exchange_weak` 操作使用 `acquire` 内存排序，它与 `get_mut` 函数中的 `release-store` 同步。否则，可能会出现在 `get_mut` 函数解锁计数器之前，后续的 `Arc::drop` 操作的效果对正在运行 `get_mut` 的线程可见。

换句话说，在这里，acquire 的「比较和交换」操作有效地“锁定”了 get_mut，阻止其成功。后续的 `Weak::drop` 操作可以使用 `release` 内存排序将计数器递减回 1，从而有效地“解锁”。

> 我们刚刚制作的 `Arc<T>` 和 `Weak<T>` 的优化实现与 Rust 标准库中包含的实现几乎相同。

如果我们运行与以前完全相同的测试（[“测试它”](#测试它2)），我们看到这个优化的实现也会编译并通过我们的测试。

> 如果你觉得为这个优化的实现做出正确的内存排序决定很困难，请不要担心。许多并发数据结构比这个更容易正确地实现。本章的 Arc 实现，特别是因为它在内存排序方面具有棘手的微妙之处。

## 总结

* `Arc<T>` 提供一个引用计数分配的共享所有权。
* 通过检查引用计数是否确实是一个 `Arc<T>`，可以有条件地提供独占访问（`&mut T`）。
* 递增原子引用计数可以使用 relaxed 操作，但是最终的递减必须与之前的递减同步。
* *weak 指针*（`Weak<T>`）可以用于避免循环。
* `NonNull<T>` 类型表示一个指向 T 的指针，但是从不为空。
* `ManuallyDrop<T>` 类型可以用于使用不安全代码时，手动决定何时丢弃 T。
* 一旦涉及一个以上的原子变量，事情就会变得更加复杂。
* 实现特定的（自旋）锁有时可能是同时对多个原子变量进行操作的有效策略。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./7_Understanding_the_Processor.html">下一篇，第七章：理解处理器</a>
</p>
