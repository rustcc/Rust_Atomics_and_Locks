# 第三章：内存排序[^1]

（<a href="https://marabos.nl/atomics/memory-ordering.html" targt="_blank">英文版本</a>）

在[第二章](./2_Atomics.md)，我们简要地谈到了内存排序的概念。在该章节，我们将研究这个主题，并探索所有可用的内存排序选项，并且，更重要地是，我们将学习如何使用它们。

## 重排和优化

（<a href="https://marabos.nl/atomics/memory-ordering.html#reordering-and-optimizations" targt="_blank">英文版本</a>）

处理器和编译器执行各种技巧，以便使你的程序运行地尽可能地快。例如，处理器可能会确定你程序中的两个连续指令不会相互影响，并且如果这样更快，就会按顺序执行它们。当一个指令在从主存中获取一些数据被短暂地阻塞了，几个后续地指令可能在在第一个指令结束之前被执行和完成，只要这不会更改你程序的行为。类似地，编译器可能会决定重排或者重写你程序的部分代码，如果它有理由相信这可能会导致更快地执行。但是，同样地，仅有在不更改你程序行为的情况下。

让我们来看看一下这个例子：

```rust
fn f(a: &mut i32, b: &mut i32) {
    *a += 1;
    *b += 1;
    *a += 1;
}
```

这里，编译器肯定会明白，操作的顺序并不重要，因为在这三个加操作之间没有发生任何依赖于 `*a` 或 `*b` 的操作（假设溢出检查被禁用）。因此，编译器可能会重新排序第二个和第三个操作，然后将前两个操作合并为单个加操作：

```rust
fn f(a: &mut i32, b: &mut i32) {
    *a += 2;
    *b += 1;
}
```

稍后，在执行优化编译程序的函数时，由于各种原因，处理器可能最终在执行第一次加操作之前，执行第二次加操作，可能是 `*b` 在缓存中可用，而 `*a` 在主内存可获取。

无论这些优化如何，结果都是相同的： `*a` 递增 2，`*b` 递增 1。它的递增的顺序对于你程序的其余部分完全不可见。

验证特定的重新排序或者其他优化并不影响程序的行为的逻辑并不需要考虑其他线程。在我们上面的示例中，这是极好的，因为独占引用（`&mut i32`）保证没有其他线程可以访问这个值。出现问题的唯一情况是，当共享的数据在线程之间发生改变。或者，换句话说，当使用原子操作时。这就是为什么，我们必须明确地告诉编译器和处理器，它们可以和不能使用我们的原子操作做什么，因为它们通常的逻辑忽略了线程之间的交互，并且可能允许的优化，会导致我们程序的结果改变。

有趣的问题是我们*如何*告诉它们。如果我们想要准确地阐明什么是可以接受的，什么是不可以接受的，并发程序将变得非常冗长并很容易出错，并且可能特定于架构：

```rust
let x = a.fetch_add(1,
    Dear compiler and processor,
    Feel free to reorder this with operations on b,
    but if there's another thread concurrently executing f,
    please don't reorder this with operations on c!
    Also, processor, don't forget to flush your store buffer!
    If b is zero, though, it doesn't matter.
    In that case, feel free to do whatever is fastest.
    Thanks~ <3
);
```

的确，我们仅能从一小部分选项中进行选择，这些选项由 `std::sync::atomic::Ordering` 枚举表示，每个原子操作都将其作为参数。可用选项的部分是非常有限的，但是经过精心挑选，可以适用大部分用例。排序是非常抽象的，并且不能直接反映实际编译器和处理器涉及的机制，例如指令重排。这使得你的并发代码可以脱离架构并且面向未来。它允许在不知道每个当前和未来处理器版本的信息的情况下进行验证。

在 Rust 中可用的排序：

* Relaxed 排序：`Ordering::Relaxed`
* Release 和 acquire 排序：`Ordering::{Release, Acquire, AcqRel}`
* 顺序一致性排序：`Ordering::SeqCst`

在 C++ 中，有一种叫做 *consume ordering*，它在 Rust 中被省略了，尽管如此，对它的讨论也是很有用的。

## 内存模型

（<a href="https://marabos.nl/atomics/memory-ordering.html#the-memory-model" targt="_blank">英文版本</a>）

不同的内存排序选项有一个严格的形式定义，以确保我们确切地知道我们允许假设什么，并且让编译器编写者确切知道它们需要向我们提供什么。为了将它与特定处理器架构的细节解耦，内存排序是根据抽象*内存模型*定义的。

Rust 的内存模型，它更多的抄自 C++，与任何现有的处理器架构不匹配，而是一个抽象模型，它有一套严格的规则，试图带面当前和未来所有架构的最大公约数吗，同时基于编译器足够的自由去进行程序分析和优化时作出有用的假设。

我们已经在[第一章的“借用和数据竞争”]看到内存模型的一部分，我们讨论了数据竞争如何导致未定义行为。Rust 的内存模型允许并发的原子存储，但将并发的非原子存储到相同的变量视为数据竞争，这将导致未定义行为。

然而，在大多数处理器架构中，原子存储之间和常规非原子存储之间并没有什么区别，我们将在[第七章](./7_Understanding_the_Processor.md)看到这些。人们可以争辩说，内存模型的限制性比必要性要强，但这些严格的规则使编译器和程序员更容易对程序进行推理，并为未来的发展留下了空间。

## Happens-Before 关系

（<a href="https://marabos.nl/atomics/memory-ordering.html#happens-before" targt="_blank">英文版本</a>）

内存模型定义了操作在 *happens-before 关系*发生的顺序。这意味着，作为一个抽象模型，它不涉及机器指令、缓存、缓冲区、时间、指令重排、编译优化等，而只定一个了一件事情在另一件事情之前保证发生的情况，并将其它一切的顺序都视为未定义的。

基础的 happens-before 规则是同一线程内的任何事情都按顺序发生。如果线程线程正在执行 `f(); g();`，那么 `f()` 在 `g()` 之前发生。

然而，在线程之间，发生在特定的情况下的 happens-before 关系笔记哦啊有限，例如，在创建和等待线程时，解锁和锁定 mutex，以及使用非 relaxed 的原子操作。Relaxed 内存排序时最基本的（也是性能最好的）内存排序，它本身并不会导致任何跨线程的 happens-before 关系。

为了探索这意味着什么，让我们看看以下示例，我们假设 a 和 b 有不同的线程并发执行：

```rust
static X: AtomicI32 = AtomicI32::new(0);
static Y: AtomicI32 = AtomicI32::new(0);

fn a() {
    X.store(10, Relaxed); // 1
    Y.store(20, Relaxed); // 2
}

fn b() {
    let y = Y.load(Relaxed); // 3
    let x = X.load(Relaxed); // 4
    println!("{x} {y}");
}
```

正如以上提及的，基础的 happens-before 规则是同一线程内的任何事情都按顺序发生。因在这个示例中，1 发生在 2 之前，并且 3 发生在 4 之前，正如 3-1 图片所示。因为我们使用 relaxed 内存排序，在我们的示例中并没有其它的 happens-before 关系。

![ ](https://github.com/fwqaaq/Rust_Atomics_and_Locks/raw/main/picture/raal_0301.png)
图3-1。示例代码中原子操作之间的 happens-before 关系。

如果 a 或 b 的任何一个在另一个开始之前完成，输出将是 0 0 或 10 20。如果 a 和 b 并发地运行，很容易看见输出是 10 0。发生这种操作的方式是，可能以以下顺序写运行：3 1 2 4。

更有趣地是，输出可以也是 0 20，尽管导致这个结果的操作不可能有全局地一致性顺序。当 3 被执行，它与 2 之间不存在 happens-before 关系，这意味着它可以加载 0 或 20。当 4 被执行，它与 1 之间不存在 happens-before 关系，这意味它可以加载 0 或 10。因此，输出 0 20 是一种有效的结果。

需要理解的重要和反直觉的是操作 3 加载值 20 并不能与 2 操作形成 happens-before 关系，即使这个值是由操作 2 存储的。我们对“之前”的概念的直觉是，在事情不一定按照全局一致性的顺序发生时会被打破，比如涉及指令重排的情况。

一个更有用并且直观，但是不太正式的理解是，从执行 b 的线程的视角来看，操作 1 和 2 可能以相反的顺序发生。

### spawn 和 join

（<a href="https://marabos.nl/atomics/memory-ordering.html#spawning-and-joining" targt="_blank">英文版本</a>）

产生的线程会创建一个 happens-before 关系，它将发生在 `spawn()` 之前的事件与新线程关联起来。同样地，join 线程创建一个 happens-before 关系，它将发生在 `join()` 调用之后的事件与被 join 的线程关联起来。

为了证明，以下示例中的断言不能失败：

```rust
static X: AtomicI32 = AtomicI32::new(0);

fn main() {
    X.store(1, Relaxed);
    let t = thread::spawn(f);
    X.store(2, Relaxed);
    t.join().unwrap();
    X.store(3, Relaxed);
}

fn f() {
    let x = X.load(Relaxed);
    assert!(x == 1 || x == 2);
}
```

由于 join 和产生操作形成的 happens-before 关系，我们肯定知道 X 的加载操作在第一个 store 之后，但在随后一个 store 之前，正如在图 3-2 所见。然而，它是否在第二个存储之前或之后观察值是不可预测的。换句话说，它可能是 1 或 2，但不是 0 或 3。

![ ](https://github.com/fwqaaq/Rust_Atomics_and_Locks/raw/main/picture/raal_0301.png)
图 3-2。示例代码中生成、join、存储和加载操作之间的 happens-before 关系。

## Relaxed 排序

（<a href="https://marabos.nl/atomics/memory-ordering.html#relaxed" targt="_blank">英文版本</a>）

当原子操作使用 relaxed 内存排序并不会提供任何 happens-before 关系，但是它们仍然保证了每个原子变量的*总的修改顺序*。这意味着，从线程的角度来看，*同一原子变量*的所有修改都是以相同的顺序进行的。

为了证明这意味着什么，我们假设 a 和 b 由不同的线程并发执行，考虑以下示例：

```rust
static X: AtomicI32 = AtomicI32::new(0);

fn a() {
    X.fetch_add(5, Relaxed);
    X.fetch_add(10, Relaxed);
}

fn b() {
    let a = X.load(Relaxed);
    let b = X.load(Relaxed);
    let c = X.load(Relaxed);
    let d = X.load(Relaxed);
    println!("{a} {b} {c} {d}");
}
```

在该示例中，仅有一个线程修改 X，这使得很轻松地能够看到 X 的修改顺序：0→5→15。它从 0 开始，然后变成 5，最终变成 15。线程并不能从 X 中观察到与此总修改不一致的任何值。这意味着“0 0 0 0”、“0 0 5 15”和“0 15 15 15”是来自另一个线程打印语句的可能的一些结果，而“0 5 0 15”或“0 0 10 15”的输出是不可能的。

即使原子变量有多个可能的修改顺序，所有线程也仅同意一个顺序。

让我们用两个单独的函数替换 a1 和 a2，我们假设它们分别由一个单独的线程执行：

```rust
fn a1() {
    X.fetch_add(5, Relaxed);
}

fn a2() {
    X.fetch_add(10, Relaxed);
}
```

假设这些是唯一修改 X 的线程，现在有两种修改顺序：要么是 0→5→15 或 0→10→15，这取决于哪个 fetch_add 操作先执行。无论哪种情况，所有线程都遵守相同的顺序。因此，即使我们即使我们有数百个额外的线程正在运行我们的 `b()` 函数，我们知道如果其中一个打印出 10，那么顺序必须是 0→10→15，而它们其中的任何一个都不可能打印出 5。反之亦然。

在[第二章](./2_Atomics.md)，我们看见几个用例示例，其中保证个别变量的总修改顺序就足够了，使用 Relaxed 内存排序足够了。然而，如果我们尝试任何超出这些示例更高级的东西，我们将很快发现，需要比 relaxed 更强的保证。

<div class="box">
  <h2 style="text-align: center;">凭空出现的值</h2>
  <p>在使用 Relaxed 内存排序时，由于缺乏顺序保证，当操作在循环方式下相互依赖时，可能会导致理论上的复杂情况。</p>

  <p>为了演示，这里有一个人为的例子，两个线程从一个原子加载一个值，并将其存储在另一个原子中：</p>

  <pre>static X: AtomicI32 = AtomicI32::new(0);
static Y: AtomicI32 = AtomicI32::new(0);

fn main() {
    let a = thread::spawn(|| {
        let x = X.load(Relaxed);
        Y.store(x, Relaxed);
    });
    let b = thread::spawn(|| {
        let y = Y.load(Relaxed);
        X.store(y, Relaxed);
    });
    a.join().unwrap();
    b.join().unwrap();
    assert_eq!(X.load(Relaxed), 0); // 可能失败？
    assert_eq!(Y.load(Relaxed), 0); // 可能失败
}</pre>

  <p>似乎很容易得出 X 和 Y 的值不会是除 0 以外的任何东西的结论，因为 store 操作仅从这项相同的原子中加载值，而这些原子仅是 0。</p>

  <p>然而，如果我们严格遵循理论内存模型，我们必须面对循环推理，并得出可怕的结论，我们可能错了。事实上，内存模型在技术上允许出现这样的结果，即最终 X 和 Y 都是 37，或者任意其它的值，导致断言失败。</p>

  <p>由于缺饭顺序保证，这两个线程的 load 操作可能都看到另一个线程 store 操作的结果，允许按操作顺序循环：我们在 Y 中存储 37，因为我们从 X 加载了 37，X 存储到 X，因为我们从 Y 加载了 37，这是我们在 Y 中存储的值。</p>

  <p>幸运的是，这种<i>凭空捏造</i>值的可能性在理论模型中被普遍认为是一个 bug，而不需要你在实践中考虑。如何在不允许这种异常情况的情况下形式化 relaxed 内存排序还是一个未解决的问题。尽管这对于形式化验证来说可能是一个问题，让许多理论家夜不能寐，但是我们其他人可以放心地使用 relaxed，因为在实践中不会发生这种情况。</p>
</div>

## Release 和 Acquire 排序

（<a href="https://marabos.nl/atomics/memory-ordering.html#release-and-acquire-ordering" targt="_blank">英文版本</a>）

*Release* 和 *Acquire* 内存排序通常成对使用，它们用于形成线程之间的 happens-before 关系。`Release` 内存排序适用于 store 操作，而 `Acquire` 内存排序适用于 load 操作。

当 acquire-load 操作观察 release-store 操作的结果时，就会形成 happens-before 关系。在这种情况下，store 操作极其之前的所有操作在时间上先于 load 操作和之后的所有操作。

当使用 Acquire 进行「获取并修改」或者「比较并交换」操作时，它仅适用于操作的 load 部分。类似地，Release 仅适用于操作的 store 部分。`AcqRel` 用于表示 Acquire 和 Release 的组合，这既能使 load 使用 Acquire，也能使 store 使用 Release。

让我们回顾一个示例，看看我们在实践中如何使用它们。在以下示例中，我们将一个 64 位整数从产生的线程发送到主线程。我们使用一个额外的原子布尔类型以指示主线程，整数已经被存储并且已经可以读取：

```rust
use std::sync::atomic::Ordering::{Acquire, Release};

static DATA: AtomicU64 = AtomicU64::new(0);
static READY: AtomicBool = AtomicBool::new(false);

fn main() {
    thread::spawn(|| {
        DATA.store(123, Relaxed);
        READY.store(true, Release); // 在这个 store 之前的所有操作都会在这个 store 操作之后可见
    });
    while !READY.load(Acquire) { // READY 的值变为 true，表示前面的存储操作对于其他线程可见
        thread::sleep(Duration::from_millis(100));
        println!("waiting...");
    }
    println!("{}", DATA.load(Relaxed));
}
```

当产生的线程完成数据存储时，它使用 release-store 去设置 `READY` 标识为真。当主线程通过它的 acquire-load 操作观察到，在这两个线程之间建立了一个 happens-before 关系，正如图 3-3 所示。此时，我们肯定知道在 release-store 到 READY 之前的所有操作对发生在 acquire-load 之后的所有操作都可见。具体而言，当主线程从 `DATA` 加载时，我们可以肯定它将加载由后台线程存储的值。该程序在最后一行只有一种输出结果：123。

![ ](https://github.com/fwqaaq/Rust_Atomics_and_Locks/raw/main/picture/raal_0303.png)

图 3-3。示例代码中原子操作之间的 happens-before 关系，展示了通过 acquire 和 release 操作形成的跨线程关系。

如果我们在这个示例为所有操作使用 relaxed 内存排序，主线程可能会看到 `READY` 翻转为 true，而之后仍然从 DATA 中加载 0。

> “Release”和“Acquire”的名称基于它们最基本用例：一个线程通过原子地存储一些值到原子变量来发布数据，而另一个线程通过原子地加载这个值来获取数据。这正是当我们解锁（释放）互斥体并随后在另一个线程上锁定（获取）它时发生的情况。

在我们的示例中，来自 READY 标识的 happens-before 关系保证了 DATA 的 store 和 load 操作不能并发地发生。这意味着我们实际上不需要这些操作是原子的。

然而，如果我们仅是为我们的数据变量尝试去使用常规的非原子类型，编译器将拒绝我们的程序，因为当另一个线程也在借用它们，Rust 的类型系统不允许我们修改它们。类型系统不会理解我们在这里创建的 happens-before 关系。一些不安全的代码是必要的，以向编译器承诺我们已经仔细考虑过这个问题，我们确信我们没有违反任何规则，如下所示：

```rust
static mut DATA: u64 = 0;
static READY: AtomicBool = AtomicBool::new(false);

fn main() {
    thread::spawn(|| {
        // 安全性：没有其他东西正在访问 DATA，
        // 因为我们还没有设置 READY 标识。
        unsafe { DATA = 123 };
        READY.store(true, Release); // 在这个 store 之前的所有操作都会在这个 store 操作之后可见
    });
    while !READY.load(Acquire) { // READY 的值变为 true，表示前面的存储操作对于其他线程可见
        thread::sleep(Duration::from_millis(100));
        println!("waiting...");
    }
    // 安全地：没有其他东西修改 DATA，因为 READY 已设置。
    println!("{}", unsafe { DATA });
}
```

<div class="box">
  <h2 style="text-align: center;">更正式地</h2>
  <p>当 acquire-load 操作观察 release-store 操作的结果时，就会形成 happens-before 关系。但那是什么意思？</p>

  <p>想象一下，两个线程都将一个 7 release-store 到相同的原子变量中，第三个线程从该变量中加载 7。第三个线程和第一个或者第二个线程有一个 happens-before 关系吗？这取决于它加载“哪个 7”：线程一还是线程二的。（或许一个不相关的 7）。这使我们得出的结论是，尽管 7 等于 7，但两个 7 与两个线程有一些不同。</p>

  <p>思考这个问题的方式是我们在<a href="#relaxed-排序">“Relaxed 排序”</a>中讨论的*总修改顺序*：发生在原子变量上的所有修改的有序列表。即使将相同的值多次写入相同的变量，这些操作中的每一个都以该变量的总修改顺序代表一个单独的事件。当我们加载一个值，加载的值与每个变量“时间线”上的特定点相匹配，这告诉我们我们可能会同步哪个操作。</p>

  <p>例如，如果原子总修改顺序是</p>

  <p>1. 初始化为 0</p>

  <p>2. Release-store 7（来自线程二）</p>
  
  <p>3. Release-store 6</p>

  <p>4. Release-store 7（来自线程一）</p>

  <p>然后，acquire-load 7 将与第二个线程的 release-store 或者最后一个事件的 release-store 同步。然而，如果我们之前（就 happens-before 关系而言）见过 6，我们知道我们看到的是最后一个 7，而不是第一个 7，这意味着我们现在与线程一有 happens-before 的关系，而不是线程二。</p>

  <p>还有一个额外的细节，即 release-stored 的值可能会被任意数量的「获取并修改」和「比较并交换」操作修改，但仍会导致与 acquire-load 读取最终结果的 happens-before 关系。</p>

  <p>例如，想象一个具有以下总修改顺序的原子变量：</p>

  <p>1. 初始化为 0</p>
  
  <p>2. Release-store 7</p>

  <p>3. Relaxed-fetch-and-add 1，改变 7 到 8</p>

  <p>4. Relaxed-fetch-and-add 1，改变 8 到 9</p>

  <p>5. Release-store 7</p>

  <p>6. Relaxed-swap 10，改变 7 到 10</p>

  <p>现在，如果我们在这个变量上执行 acquire-load 到 9，我们不仅与第四个操作（存储此值）建立了一个 happens-before 关系，同时也与第二个操作（存储 7）建立了该关系，即使第三个操作使用了 Relaxed 内存排序。</p>

  <p>相似地，如果我们在这个变量上执行 acquire-load 到 10，而该值是由一个 relaxed 操作写入的，我们仍然建立了与第五个操作（存储 7）的 happens-before 关系。因为它只是一个普通的存储操作（不是「获取并修改」或「比较并交换」操作），它打破了规则链：我们没有与其他操作建立 happens-before 关系。</p>
</div>

### 示例：锁定

（<a href="https://marabos.nl/atomics/memory-ordering.html#example-locking" targt="_blank">英文版本</a>）

互斥锁是 release 和 acquire 排序的最常见用例（参见[第一章的“锁：互斥锁和读写锁”](./1_Basic_of_Rust_Concurrency.md#锁互斥锁和读写锁)）。当锁定时，它们使用 acquire 排序的原子操作来检查是否它已解锁，同时也（原子地）改变状态到“锁定”。当解锁时，它们使用 release 排序设置状态到“解锁”。这意味着，在解锁 mutex 和随后锁定它有一个 happens-before 关系。

以下是这种模式的演示：

```rust
static mut DATA: String = String::new();
static LOCKED: AtomicBool = AtomicBool::new(false);

fn f() {
    if LOCKED.compare_exchange(false, true, Acquire, Relaxed).is_ok() {
        // 安全地：我们持有独占的锁，所以没有其他东西访问 DATA。
        unsafe { DATA.push('!') };
        LOCKED.store(false, Release);
    }
}

fn main() {
    thread::scope(|s| {
        for _ in 0..100 {
            s.spawn(f);
        }
    });
}
```

正如我们在[第二章“「比较并交换」操作”](./2_Atomics.md#比较并交换操作)简要地所见，「比较并交换」接收两个内存排序参数：一个用于比较成功且 store 发生的情况，一个用于比较失败且 `store` 没有发生的情况。在 f 中，我们试图去改变 `LOCKED` 的值从 false 到 true，并且只有在成功的情况下才能访问 DATA。所以，我们仅关心成功的内存排序。如果 `compare_exchange` 操作失败，那一定是因为 `LOCKED` 已经设置为 true，在这种情况下 f 不会做任何事情。这与常规 mutex 上的 `try_lock` 操作相匹配。

> 观察力强的读者可能已经注意到，「比较并交换」操作也可能是交换操作，因为在已锁定时将 true 替换为 true 不会改变代码的正确性：
>
> ```rust
> // 这也有效。
> if LOCKED.swap(true, Acquire) == false {
>    // …
> }
> ```

归功于 acquire 和 release 内存排序，我们肯定没有两个线程能并发地访问数据。正如在图 3-4 展示的，对 DATA 的任何先前访问都在随后使用 release-store 操作将 false 存储到 LOCKED 之前发生，然后在下一个 acquire-compare-exchange（或 acquire-swap）操作中将 false 更改为 true，然后在下一次访问 DATA 之前发生。

![ ](https://github.com/fwqaaq/Rust_Atomics_and_Locks/raw/main/picture/raal_0304.png)
图 3-4。锁定示例中原子操作之间的 happens-before 关系，显示了两个线程按顺序锁定和解锁。

在[第四章](./4_Building_Our_Own_Spin_Lock.md)，我们将把这个概念变成一个可重复使用的类型：自旋锁。

### 示例：使用间接的方式惰性初始化

（<a href="https://marabos.nl/atomics/memory-ordering.html#example-lazy-initialization-with-indirection" targt="_blank">英文版本</a>）

在[第二章的“示例：惰性一次性初始化”](./2_Atomics.md#示例惰性一次性初始化)中，我们实现一个全局变量的惰性初始化，使用「比较且交换」操作去处理多个线程竞争同时初始化值的情况。由于该值是非零的 64 位整数，我们能够使用 AtomicU64 来存储它，在初始化之前使用零作为占位符。

要对不适合单个原子变量的更大的数据类型做同样的事情，我们需要寻找替代方案。

在这个例子中，假设我们想保持非阻塞行为，这样线程就不会等待另一个线程，而是从第一个线程中竞争并获取值来完成初始化。这意味着我们仍然需要能够在单个原子操作中从“未初始化”到“完全初始化”。

正如软件工程的基本定理告诉我们的那样，计算机科学中的每个问题都可以通过添加另一层间接来解决，这个问题也不例外。由于我们无法将数据放入单个原子变量中，因此我们可以使用原子变量来存储指向数据的*指针*。

`AtomicPtr<T>` 是 `*mut T` 的原子版本：指向 T 的指针。我们可以使用空指针作为初始状态的占位符，并使用「比较并交换」操作将其原子地替换为指向新分配的、完全初始化的 T 的指针，然后可以由其他线程读取。

由于我们不仅共享包含指针的原子变量，还共享它所指向的数据，因此我们不能再像[第2章](./2_Atomics.md)那样使用 Relaxed 的内存排序。我们需要确保数据的分配和初始化不会与读取数据竞争。换句话说，我们需要在 store 和 load 操作上使用 release 和 acquire 排序，以确保编译器和处理器不会通过，例如，重新排序指针的存储和数据本身的初始化来破坏我们的代码。

对于一些名为 Data 的任意数据类型，这引出了以下实现：

```rust
use std::sync::atomic::AtomicPtr;

fn get_data() -> &'static Data {
    static PTR: AtomicPtr<Data> = AtomicPtr::new(std::ptr::null_mut());

    let mut p = PTR.load(Acquire);

    if p.is_null() {
        p = Box::into_raw(Box::new(generate_data()));
        if let Err(e) = PTR.compare_exchange(
            std::ptr::null_mut(), p, Release, Acquire
        ) {
            // 安全性：p 来自上方的 Box::into_raw，
            // 并且不能与其他线程共享
            drop(unsafe { Box::from_raw(p) });
            p = e;
        }
    }

    // 安全性：p 不是 null，并且指向一个正确初始化的值。
    unsafe { &*p }
}
```

如果我们以 acquire-load 操作从 PTR 得到的指针是非空的，我们假设它指向已初始化的数据，并构建对该数据的引用。

然而，如果它仍然为空，我们会生成新数据，并使用 `Box::new` 将其存储在新的内存分配中。然后，我们使用 `Box::into_raw` 将此 `Box` 转换为原始指针，因此我们可以尝试使用「比较并交换」操作将其存储到 PTR 中。如果另一个线程赢得初始化竞争，`compare_exchange` 将失败，因为 PTR 不再是空的。如果发生这种情况，我们将原始指针转回 Box，使用 `drop` 来释放内存分配，避免内存泄漏，并继续使用另一个线程存储在 PTR 中的指针。

在最后的不安全块中，关于安全性的注视表明我们的假设是指它指向的数据已经被初始化。注意，这包括对事情发生顺序的假设。为了确保我们的假设成立，我们使用 release 和 acquire 内存排序来确保初始化数据实际上在创建对其的引用之前已经发生。

我们在两个地方加载一个潜在的非空（即初始化）指针：通过 load 操作和当 compare_exchange 失败时的该操作。因此，如上所述，我们需要在 load 内存排序和 compare_exchange 失败内存排序上都使用 Acquire，以便能够与存储指针的操作进行同步。当 compare_exchange 操作成功时，会发生 store 操作，因此我们必须使用 Release 作为其成功的内存排序。

图 3-5 显示了三个线程调用 `get_data()` 的情况的操作和发生前关系的可视化。在这种情况下，线程 A 和 B 都观察到一个空指针并都试图去初始化原子指针。线程 A 赢得竞争，导致线程 B 的 compare_exchange 调用失败。线程 C 在通过线程 A 初始化之后观察原子指针。最终结果是，所有三个线程最终都使用由线程 A 分配的 box。

![ ](https://github.com/fwqaaq/Rust_Atomics_and_Locks/raw/main/picture/raal_0305.png)
图3-5。调用 `get_data()` 的三个线程之间的操作和发生前关系。

## Consume 排序

（<a href="https://marabos.nl/atomics/memory-ordering.html#consume" targt="_blank">英文版本</a>）

让我们仔细看看上一个示例中的内存排序。如果我们把严格的模型放在一边，从更实际的方面来思考它，我们可以说 release 排序阻止了数据的初始化与共享指针的 store 操作重新排序。这一点非常重要，因为否则其它线程可能会在数据完全初始化之前就能看到它。

类似地，我们可以说 acquire 排序为防止重新排序，使得数据在加载指针之前被访问。然而，人们可能合理地质疑，在实践中是否有意义。在地址加载之前，如何访问数据？我们可能会得出结论，若于 acquire 排序的内存排序可能足够。我们的结论是正确的：这种较弱的内存排序被称为 consume 排序。

consume 排序是 acquire 排序的一个轻量级、更高效的变体，其同步效果仅限于依赖于已加载值的操作。

这意味着如果你用 consume-load 从一个原子变量中加载一个通过 release 存储的值 x，那么基本上，这个 store 操作发生在依赖表达式（如 `*x`、`array[x]` 或 `table.lookup(x + 1)`）的求值之前，但不一定发生在独立操作（如读取另一个与 x 无关的变量）之前。

现在有好消息和坏消息。

好消息是，在所有现代处理器架构上，consume 排序是通过与 relaxed 排序完全相同的指令实现的。换句话说，consume 排序可以是“免费的”，而 acquire 内存排序在某些平台可能不是这样。

坏消息是，没有编译器真正实现 consume 排序。

事实证明，这种“依赖性”评估的概念不仅难以定义，而且在转换和优化程序时保持这些依赖性也很难。例如，编译器能够优化 x + 2 - x 为 2，有效地消除了对 x 的依赖。对于更复杂的表达式，如 `array[x]`，如果编译器能够对 x 或数组元素的可能值进行逻辑推断，那么可能会出现更微妙的变化。当考虑控制流，如 if 语句或函数调用时，问题讲变得更加复杂。

因此，编译器升级 consume 排序到 acquire 排序，仅是为了安全起见。C++20 标准甚至明确地反对使用消费排序，并指出，除了 acquire 排序之外，其他实现被证明是不可行的。

将来可能找到一个 consume 排序的有效定义和实现。然而，直到这一天到来之前，Rust 都不会暴露 `Ordering::Consume`。

## 顺序一致性排序

（<a href="https://marabos.nl/atomics/memory-ordering.html#seqcst" targt="_blank">英文版本</a>）

一个更强的内存排序时*顺序一致性*排序：`Ordering::SeqCst`。它包含了 acquire 排序（对于 load 操作）以及 release 排序（对于 store 操作）的所有保证，并且*也*保证了全局一致性操作。

这意味着在程序中使用 `SeqCst` 排序的每个操作是所有线程都同意的单个总顺序的一部分。这个总顺序与每个变量的总修改顺序一致。

严格来讲，由于它比 acquire 和 release 内存排序要强，因此顺序一致性 load 或者 store 操作可以取代一对 release-acquire 中的 acquire-load 或 release-store 操作，形成 happens-before 关系。换句话说，acquire-load 不仅可以与 release-store 形成 happens-before 关系，同时也可以和顺序一致的 store 形成 happens-before 关系，同样 release-store 也是如此。

> 仅有当 happens-before 关系的双方使用 SeqCst 时，才能保证与 SeqCst 操作的单个总顺序一致。

虽然这似乎是最容易推理的内存排序，但 SeqCst 排序在实践中几乎从来都没有必要。在几乎所有情况下，通常 acquire 和 release 排序就足够了。

以下是一个取决于顺序一致的有序操作的示例：

```rust
use std::sync::atomic::Ordering::SeqCst;

static A: AtomicBool = AtomicBool::new(false);
static B: AtomicBool = AtomicBool::new(false);

static mut S: String = String::new();

fn main() {
    let a = thread::spawn(|| {
        A.store(true, SeqCst);
        if !B.load(SeqCst) {
            unsafe { S.push('!') };
        }
    });

    let b = thread::spawn(|| {
        B.store(true, SeqCst);
        if !A.load(SeqCst) {
            unsafe { S.push('!') };
        }
    });

    a.join().unwrap();
    b.join().unwrap();
}
```

两个线程首先设置它们自己的原子布尔值到 true，以告知另一个线程它们正在获取 S，并且然后检查其它的原子变量布尔值，是否它们可以安全地获取 S，而没有导致数据竞争。

如果两个存储操作都发生在任一加载操作之前，则两个线程最终都无法访问 S。然而，两个线程都不可能访问 S 并导致未定义的行为，因为顺序一致的顺序保证了其中只有一个线程可以赢得竞争。在每个可能的单个总的顺序中，单个操作将是 store 操作，这阻止其他线程访问 S。

在实际情况中，几乎所有对 SeqCst 的使用都涉及一种类似的存储模式，在随后在同一线程上加载之前必须全局可见。对于这些情况，一个潜在的更有效的替代方案是将 relaxed 的操作与 SeqCst 屏障结合使用，我们接下来将探索。

## 屏障（Fence）[^2]

（<a href="https://marabos.nl/atomics/memory-ordering.html#fences" targt="_blank">英文版本</a>）

除了对原子变量的额外操作，我们还可以将内存排序应用于：原子屏障。

`std::sync::atomic::fence` 函数表示一个*原子屏障*，它可以是一个 Release 屏障、一个 Acquire 屏障，或者两者都是（AcqRel 或 SeqCst）。SeqCst 屏障还参与顺序一致性的全局排序。

原子屏障允许你从原子操作中分离内存排序。如果你想要应用内存排序到多个操作这可能是有用的，或者你想要有条件地应用内存排序。

本质上，release-store 可以拆分成 release 屏障，然后是（relaxed）store，并且 acquire-load 可以拆分成（relaxed）load，然后是 acquire 屏障：

<div style="columns: 2;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    release-acquire 关系的 store 操作，
    <pre>
    a.store(1, Release);</pre>
    可以由 release 屏障和随后的 relaxed store 组成：
    <pre>
    fence(Release);
    a.store(1, Relaxed);</pre>
  </div>
  <div style="break-inside: avoid">
    release-acquire 关系的 load 操作，
      <pre>
      a.load(Acquire);</pre>
    可以由 relaxed load 和随后的 acquire 屏障组成：
    <pre>
    a.load(Relaxed);
    fence(Acquire);</pre>
  </div>
</div>

不过，使用单独的屏障可能会导致额外的处理器指令，这可能会略微降低效率。

更重要地是，与 release-store 或者 acquire-load 不同，屏障不会与任意单个原子变量捆绑。这意味着单个屏障可以立刻用于多个变量。

从形式上讲，如果 release 屏障在同意线程上的位置紧跟着任何一个原子操作，而该原子操作存储的值被我们要同步的 acquire 操作观察到，那么该 release 屏障可以代替 release 操作，并建立一个 happens-before 关系。同样地，如果一个 acquire 屏障在同一线程上的位置紧接着之前的任何一个原子操作，而该原子操作加载的值是由 release 操作存储的，那么该 acquire 屏障可以替代任何一个 acquire 操作。

综上所述，这意味着如果在 release 屏障之后的任何 store 操作被在 acquire 屏障之前的任何 load 操作所观察，那么在 release 屏障和 acquire 屏障之间将建立 happens-before 关系。

例如，假设我们有一个线程执行一个 release 屏障，然后对不同的变量执行三个原子 store 操作，另一个线程从这些相同的变量执行三个 load 操作，然后是一个 acquire 栅栏，如下所示：

<div style="columns: 2;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    线程 1：
    <pre>
    fence(Release);
    A.store(1, Relaxed);
    B.store(2, Relaxed);
    C.store(3, Relaxed);</pre>
  </div>
  <div style="break-inside: avoid">
    线程 2：
    <pre>
    A.load(Relaxed);
    B.load(Relaxed);
    C.load(Relaxed);
    fence(Acquire);</pre>
  </div>
</div>

在这种情况下，如果线程 2 上的任意 load 操作从线程 1 上的相应 store 操作加载值，那么线程 1 上的 release 屏障和线程 2 上的 acquire 屏障之间将建立 happens-before 关系。

屏障不必直接在原子操作之前或者之后。它们可以在原子操作之间，包括控制流。这可以用于使栅栏具有条件行，类似于比较且交换操作具有成功和失败的排序。

例如，如果我们从使用 acquire 内存排序的原子变量中加载指针，我们可以使用屏障仅当指针不是空的时候应用 acquire 内存排序：

<div style="columns: 2;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    使用 acquire-load：
    <pre>let p = PTR.load(Acquire);
if p.is_null() {
    println!("no data");
} else {
    println!("data = {}", unsafe { *p });
}</pre>
  </div>
  <div style="break-inside: avoid">
    使用条件的 acquire 屏障：
    <pre>let p = PTR.load(Relaxed);
if p.is_null() {
    println!("no data");
} else {
    fence(Acquire);
    println!("data = {}", unsafe {*p });
}</pre>
  </div>
</div>

如果指针通常为空，这可能是有益的，在不需要时避免 acquire 内存排序。

让我们来看看一个更复杂的 release 和 acquire 屏障的用例：

```rust
use std::sync::atomic::fence;

static mut DATA: [u64; 10] = [0; 10];

const ATOMIC_FALSE: AtomicBool = AtomicBool::new(false);
static READY: [AtomicBool; 10] = [ATOMIC_FALSE; 10];

fn main() {
    for i in 0..10 {
        thread::spawn(move || {
            let data = some_calculation(i);
            unsafe { DATA[i] = data };
            READY[i].store(true, Release);
        });
    }
    thread::sleep(Duration::from_millis(500));
    let ready: [bool; 10] = std::array::from_fn(|i| READY[i].load(Relaxed));
    if ready.contains(&true) {
        fence(Acquire);
        for i in 0..10 {
            if ready[i] {
                println!("data{i} = {}", unsafe { DATA[i] });
            }
        }
    }
}
```

> `Std::array::from_fn` 是一种执行一定次数并将结果收集到数组中的简单方法。

在这个示例中，10 个线程做了一些计算，并存储它们的结果到一个（非原子）共享变量中。每个线程设置一个原子布尔值，以指示数据已经通过主线程准备好读取，使用一个普通的 release-store。主线程等待半秒，检查所有 10 个布尔值以查看哪些线程已完成，并打印任何准备好的结果。

主线程不使用 10 个 acquire-load 操作来读取布尔值，而是使用 relaxed 的操作和单个 acquire 屏障。它在读取数据之前执行屏障，但前提是有数据要读取。

虽然在这个特定的例子中，投入任何精力进行此类优化可能完全没有必要，但在构建高效的并发数据结构时，这种节省额外获取操作开销的模式可能很重要。

`SeqCst` 屏障既是 release 屏障也是 acquire 屏障（就像 `AcqRel`），同时也是顺序一致性操作的单个总顺序的一部分。然而，只有屏障是总顺序的一部分，但不一定是它之前或之后的原子操作。这意味着，与 release 或 acquire 操作不同，顺序一致性的操作不能拆分为 relaxed 操作和内存屏障。

<div class="box">
  <h2 style="text-align: center;">编译器屏障</h2>
  <p>除了常规的原子屏障，Rust 标准库还提供了<i>编译器屏障</i>：<code>std::sync::atomic::compiler_fence</code>。它的签名与我们上面讨论的这些常规 <code>fence()</code> 不同，但它的效果仅限于编译器。与原子屏障不同，例如，它并不会阻止处理器重排指令。在绝大多数屏障的用例中，编译器屏障是不够的。</p>

  <p>在实现 Unix 信号处理程序或嵌入式系统上的中断时，可能会出现可能的用例。这些机制可以突然中断一个线程，暂时在同一处理器内核上执行一个不相关的函数。由于它发生在同一处理器内核上，处理器可能影响内存排序的常规方式不适用。（更多细节请参考<a href="./7_Understanding_the_Processor.html">第七章</a>）在这种情况下，编译器屏障可能阻隔，这样可以节省一条指令并且希望提高性能。</p>

  <p>另一用例涉及进程级内存屏障。这种技术超出了 Rust 内存模型的范畴，并且仅在某些操作系统上受支持：在 Linux 上通过 membarrier 系统调用，在 Windows 上使用 FlushProcessWriterBuffers 函数。它有效地允许一个线程强制向所有并发运行的线程注入（顺序一致性）原子屏障。这使得我们可以使用轻量级的编译器屏障和重型的进程级屏障替换两个匹配的屏障。如果轻量级屏障一侧的代码执行效率更高，这可以提高整体性能。（请参阅 <code>crates.io</code> 上的 membarrier crate 文档，了解更多详细信息和在 Rust 中使用这种屏障的跨平台方法。）</p>

  <p>编译器屏障也可以是一个有趣的工具，用于探索处理器对内存排序的影响。</p>

  <p>在<a href="./7_Understanding_the_Processor.html#一个实验">第七章“一个实验”</a>中，我们将故意使用编译器屏障替换常规屏障。这将让我们在使用错误的内存排序时体验到处理器的微妙但潜在的灾难性影响。</p>
</div>

## 常见的误解

（<a href="https://marabos.nl/atomics/memory-ordering.html#common-misconceptions" targt="_blank">英文版本</a>）

围绕内存排序有很多误解。在我们结束本章之前，让我们回顾一下最常见的误解。

> 误区：我需要强大的内存排序，以确保更改“立即”可见。

一个常见的误解是，使用像 Relaxed 这样的弱内存排序意味着对原子变量的更改可能永远不会到达另一个线程，或者只有在显著延迟之后才会到达。“Relaxed”这个名字可能会让它听起来像什么都没发生，直到有什么东西迫使硬件被唤醒并执行本该执行的操作。

事实是，内存模型并没有说任何关于时机的事情。它仅定义了某些事情发生的顺序；而不是你要等待多久。假设一台计算机需要数年时间才能将数据从一个线程传输到另一个线程，这完全无法使用，但可以完美地满足内存模型。

在现实生活中，内存排序是关于重新排序指令等事情，这通常以纳秒规模发生。更强的内存排序不会使你的数据传输速度更快；它甚至可能会减慢你的程序速度。

> 误区：禁用优化意味着我不需要关心内存排序。

编译器和处理器在使事情按照我们预期的顺序发生方面起着作用。禁用编译器优化并不能禁用编译器中的每种可能的转换，也不能禁用处理器的功能，这些功能导致指令重新排序和类似的潜在问题行为。

> 误区：使用不重新排序指令的处理器意味着我不需要关心内存排序。

一些简单的处理器，比如小型微控制器中的处理器，只有一个核心，每次只能执行一条指令，并且按顺序执行。然而，虽然在这类设备上，出现错误的内存排序导致实际问题的可能性较低，但编译器仍然可能基于错误的内存排序做出无效的假设，导致代码出错。此外，还需要认识到，即使处理器不会乱序执行指令，它仍可能具有其他与内存排序相关的特性。

> 误区：Relaxed 的操作是免费的。

这是否成立取决于对“免费”一词的定义。的确，Relaxed 是最高效的内存排序，相比其他内存排序可以显著提升性能。事实上，对于所有现代平台，Relaxed 加载和存储操作编译成与非原子读写相同的处理器指令。

如果原子变量只在单个线程中使用，与非原子变量相比，速度上的差异很可能是因为编译器对非原子操作具有更大的自由度并更有效地进行优化。（编译器通常避免对原子变量进行大部分类型的优化。）

然而，从多个线程访问相同的内存通常比从单个线程访问要慢得多。当其他线程开始重复读取该变量时，持续写入原子变量的线程可能会遇到明显的减速，因为处理器核心和它们的缓存现在必须开始协作。

我们将在[第7章](./7_Understanding_the_Processor.md)中探讨这种效应。

> 误区：顺序一致的内存排序是一个很好的默认值，并且总是正确的。

抛开性能问题，顺序一致性内存排序通常被视为默认选择的理想内存排序类型，因为它具有强大的保证。确实，如果任何其他内存排序是正确的，那么 `SeqCst` 也是正确的。这可能让人觉得 `SeqCst` 总是正确的。然而，可能并发算法本身就是不正确的，不论使用哪种内存排序。

更重要的是，在阅读代码时，`SeqCst` 基本上告诉读者：“该操作依赖于程序中每个 `SeqCst` 操作的总顺序”，这是一个极其广泛的声明。如果可能的话，使用较弱的内存排序往往会使相同的代码更容易进行审查和验证。例如，Release 明确告诉读者：“这与同一变量的 acquire 操作相关”，在形成对代码的理解时涉及的考虑要少得多。

建议将 SeqCst 看作是一个警示标识。在实际代码中看到它通常意味着要么涉及到复杂的情况，要么简单地说是作者没有花时间分析其与内存排序相关的假设，这两种情况都需要额外的审查。

> 误区：顺序一致的内存排序可用于“release-store”或“acquire-load”。

虽然顺序一致性内存排序可以替代 Acquire 或 Release，但它并不能以某种方式创建 acquire-store 或 release-load。这些仍然是不存在的。Release 仅适用于存储操作，而 acquire 仅适用于加载操作。

例如，Release-store 与 SeqCst-store 不会形成任何 release-acquire 关系。如果你希望它们成为全局一致顺序的一部分，两个操作都必须使用 SeqCst。

## 总结

（<a href="https://marabos.nl/atomics/memory-ordering.html#summary" targt="_blank">英文版本</a>）

* 所有的原子操作可能没有全局一致的顺序，因为不同的线程视角可能会以不同的顺序发生。
* 然而，每个单独的原子变量都有它自己的*总修改顺序*，不管内存排序如何，所有线程都会达成一致意见。
* 操作顺序是通过 *happens-before* 关系来定义的。
* 在单个线程中，每个操作之间都会有一个 *happens-before* 关系。
* 创建一个线程的操作在顺序上发生在该线程的所有操作之前。
* 线程做的任何事情都会在 join 这个线程之前发生。
* 解锁 mutex 的操作在顺序上发生在再次锁定 mutex 的操作之前。
* 从 release 存储中以 acquire 加载值建立了一个 happens-before 关系。该值可以通过任意数量的「获取并修改」以及「比较并交换」操作修改。
* 如果存在的话，consume-load 将是 acquire-load 的轻量级版本。
* 顺序一致的排序导致全局一致的操作顺序，但几乎从来都没有必要，并且会使代码审查更加复杂。
* 屏障允许你组合多个操作的内存排序或有条件地应用内存排序。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./3_Memory_Ordering.html">下一篇，第四章：构建我们自己的自旋锁</a>
</p>

[^1]: <https://zh.wikipedia.org/wiki/内存排序>
[^2]: <https://zh.wikipedia.org/wiki/内存屏障>
