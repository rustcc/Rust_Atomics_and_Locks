# 第七章：理解处理器

（<a href="https://marabos.nl/atomics/hardware.html" target="_blank">英文版本</a>）

虽然第 [2](./2_Atomics.md) 章和第 [3](./3_Memory_Ordering.md) 章的全部理论，已经可以让我们去编写正确的并发代码，但是更进一步，大概了解一下在处理器层面，一切都是怎么运行的，这对我们也会非常有帮助。在这一章节，我们将会探索原子操作最终编译后的机器码是什么样、不同架构的处理器之间的差异、为什么会有一个弱版本的 `compare_exchange` 存在、内存排序在单个指令的最低级含义，并且缓存是如何和这一切联系的。

这一章的目标不是为了理解每种处理器架构的相关细节。如果这样的话，就需要我们去查阅大量大量的书籍，其中有些内容或许还没被著成书，或者根本没被公开。相反，本章节的目标是让大家对于原子操作在处理器层面是如何工作的这一点，有一个大概的认知，并且在实现或者优化涉及到关于原子的代码的时候，可以做出更多有依据的决策。当然，也仅是满足我们对处理器幕后如何工作的好奇心——暂时从抽象的理论中休息一下吧。

为了尽可能的具体化，我们只关注两种特定的处理器架构：

* *X86-64*：

  Intel 和 AMD 实现的 64 位 x86 架构处理器主要用于笔记本、台式机、服务器和一些游戏机主机。最初的 x86 架构的 16 位版本和之后非常流行的 32 位版本是 Intel 开发的，而 64 位的版本，也就是我们今天称作 x86-64 版本，最初是 AMD 开发的一种扩展版本，因此也常叫作 AMD64。Intel 也开发了自己的 64 位架构：IA-64，但最终采用了更为流行的 AMD 的 x86 扩展版本（名称为 IA-32E、EM64T 以及稍后的 64）。

* *ARM64*：

  ARM 架构的 64 位版本用于几乎所有的现代移动设备、高性能的嵌入式系统、现代化的笔记本电脑和台式机中。它也被称为 AArch64，并被被引入作为 ARMv8 的一部分。ARM 的早期版本（32 位）在很多方面也是类似地，广泛地使用在各种应用中。可以想象在各种嵌入式系统中，从汽车到电子 COVID 测试，这些流行的微控制器都是基于 ARMv6 和 ARMv7。

这两种架构在许多方面都是不同的。最重要的是，他们以不同的方式去实现原子化。理解这两种架构中原子化如何工作的，将会给我们一些更加普遍的理解，并且这些也可以适用于许多其他的处理器架构。

## 处理器指令

（<a href="https://marabos.nl/atomics/hardware.html#instructions" target="_blank">英文版本</a>）

我们可以通过仔细观察编译器的输出，以及处理器将执行的确切指令，来大致了解处理器级别的工作方式。

<div class="box">
  <h2 style="text-align: center;">汇编语言简介</h2>
  <p>当编译任何以编译语言（比如 rust 或者 C）编写的软件的时候，你的代码将会翻译成可以被处理器执行的<i>机器指令</i>，最终处理器将会以此执行你的程序。这些指令高度特定于你编译程序的处理器架构。</p>

  <p>这些机器指令也被称作<i>机器码</i>，是以二进制格式编码的，对于人类来说，完全可读。<i>汇编</i>是人类可阅读的代表。每一个指令用一行文本表示，通常以一个单词或者缩写表示指令，后面再跟上它的参数或者操作数。汇编器（<code>assembler</code>）将一段汇编文本转换成二进制表示，反汇编器（<code>disassembler</code>）则相反。</p>

  <p>像 Rust 语言，编译之后，源代码的大部分结构将会丢失。依据组织结构的层级，函数与函数调用或许仍然能被识别出来。但是，像结构体或者枚举这些类型会被简化为字节和地址，循环和条件处理会被转变为基本跳转或者分支指令的扁平结构。</p>

  <p>这里有一个示例，展示汇编长什么模样的一小段代码片段，用于某个假设的架构里面的：</p>

  <pre>
  ldr x, 1234 # 从内存地址的 1234 中取值，赋给 x  
  li y, 0     # 将 y 设置为 0
  inc x       # 增加 x
  add y, x    # 将 x 加 y
  mul x, 3    # 将 x 乘以 3
  cmp y, 10   # y 与 10 进行比较
  jne -5      # 如果不等于的话，指令跳转，回退 5 个指令
  str 1234, x # 将 x 存储到内存地址1234处</pre>

  <p>这个示例中，x 和 y 是寄存器（<code>register</code>）。寄存器是处理器的一部分，不属于主内存，通常保存单个数值或者内存地址。在 64 位的架构中，它们一般是 64 位的大小。在不同的架构中，寄存器的个数可能是不一样的，但通常个数也是非常有限的。寄存器基本用于计算过程中的临时存储，是将数据写回内存前存放中间结果的地方。</p>

  <p>对于特定内存地址的常量，例如上面示例中的 -5 和 1234，经常是以人类更容易阅读的<i>标签</i>来代替。在将汇编转换成机器码的时候，汇编器会自动将他们替换为实际的地址。</p>

  <p>使用标签的话，上面的示例可能是这样的：</p>

  <pre>
           ldr x, SOME_VAR
           li y, 0
  my_loop: inc x
           add y, x
           mul x, 3
           cmp y, 10
           jne my_loop
           str SOME_VAR, x</pre>

  <p>由于标签名只是汇编的一部分，并不是二进制机器码的，所以反汇编器不知道原始的标签名是什么，将机器码转换到汇编的时候，极大可能地只是生成一个无意义的标签名，比如 <code>label1</code> 和 <code>var2</code>。</p>

  <p>对于所有不同架构下的完整汇编学习课程，并不在本书的范围内，也不是阅读本章的必备条件。常规的理解已经足够去弄懂这些示例代码了，我们只阅读汇编，不去写它。每个示例中相关的指令已经解释的足够详细，对于先前没有汇编经验的人来说，也可以跟得上。</p>
</div>

为了去看 Rust 编译器产生的确切机器码，我们有几个选项。我们可以像平常一样编译我们的代码，然后使用反汇编器（比如 objdump），将生成的二进制文件转成汇编。利用编译阶段编译器产生的调试信息，反汇编器可以生成与 Rust 程序源码中原始函数名字对应的标签。这种方式有一个缺点，就是需要一个支持不同处理器架构的反汇编器。虽然 Rust 的编译器支持许多处理器架构，但是很多反汇编器只支持它所被编译的某一种处理器架构。

一种更为直接的方式是使用 `rustc` 的 `--emit=asm` 命令行参数，让编译器直接生成汇编代码，而不是生成二进制代码。这种方式的缺点是会产生很多不相关的输出信息，包含一些我们不需要的反汇编器和调试工具的信息。

也有一些很棒的工具，比如 [cargo-show-asm](https://crates.io/crates/cargo-show-asm)，它可以与 `cargo` 互操，并且在使用了对应的命令行参数后，会自动编译你的代码，找出你感兴趣的函数对应的汇编，并且将包含的具体指令行进行高亮处理。

对于一些相对比较小的代码片段，最简单最推荐的方式是使用 web 服务，比如特别棒的 [Compiler Explorer by Matt Godbolt](https://godbolt.org/)。这个网站上可以使用包含 Rust 在内的多种语言编写代码，并且很直观地可以看到使用选择的编译器版本所产生的汇编代码。该网站甚至使用颜色展示出哪一行 Rust 代码对应哪一行汇编代码，即使是代码被优化过，这种对应关系仍然存在。

由于我们想要观察在不同处理器架构下的汇编代码，因此需要指定 Rust 编译器想要编译的目标架构。我们将使用 x86-64 的 `x86_64-unknown-linux-musl` 和 ARM64 的 `aarch64-unknown-linux-musl`。这两个在网站 Compiler Explorer 上是直接被支持的。如果你想要在本地编译，比如使用上面提到的 `cargo-show-asm` 或者其他的方式，你必须确保在目标机器上，已经安装了 Rust 标准库，这些标准库通常使用 `rustup` 的 target 进行添加。

在任何情况下，使用编译器标识 `--target` 来选择编译的目标机器架构，比如 `--target=aarch64-unknown-linux-musl`。如果你不指定任何目标架构，编译器将会基于你当前的平台进行自动选择。（在网站 Compiler Explorer 的例子中，机器平台是这个网站所在的服务器，当前是 `x86_64-unknown-linux-gnu`）。

此外，建议使用 `-O` 标识以开启优化（或者当使用 `Cargo` 时使用 `--release`），因为这样启用优化并禁止溢出检查，这可以显著地减少我们将查看较小函数产生的汇编代码。

让我门尝试一下，看看以下函数在 x86-64 和 ARM64 上的汇编代码是什么样的：

```rust
pub fn add_ten(num: &mut i32) {
    *num += 10;
}
```

使用 `-O --target=aarch64-unknown-linux-musl` 作为上述任何方式的编译参数，我们将得到 ARM64 架构下的汇编代码：

```s
add_ten:
    ldr w8, [x0]
    add w8, w8, #10
    str w8, [x0]
    ret
```

`x0` 寄存器包含我们函数的参数，即增加 10 的 i32 内存地址。首先，`ldr` 指令从内存地址加载 32 位值到 w8 寄存器。然后，add 指令增加 10 到 w8 并且将结果存回至 w8。随后，str 指令将 w8 指令存回到相同的内存地址。最后，ret 指令标记函数结束，并导致处理器跳回并继续执行调用 add_ten 的函数。

如果我们为 `x86_64-unknown-linux-musl` 编译完全相同的代码，我们将得到这样的东西：

```s
add_ten:
    add dword ptr [rdi], 10
    ret
```

这次，叫做 rdi 的寄存器用于 num 参数。更有趣地是，在 `x86_64`，一个单独的 add 指令可以完成 ARM64 上需要 3 个指令才能完成的操作：加载、加和存储值。

在*复杂指令集计算机*（CISC）架构上的情况，例如 x86。这种架构上的指令通常有很多变体，例如操作寄存器或直接操作特定大小的内存。（汇编指令中的 dword 指定 32 位操作。）

相对地，*精简指令集计算机*（RISC）架构上通常有非常少变体的精简指令集，例如 ARM。大多数指令仅能操作寄存器，并且加载和存储到内存需要单独的指令。这允许更简单的处理器，这可以降低成本，有时甚至提高性能。

这种差异在原子性的取值和修改指令中尤为突出，我们很快就会看到。

> 虽然编译器通常非常聪明，但它们并不总是生成最优的汇编，尤其当涉及原子操作时。如果你正在尝试并发现你对汇编中看似不必要的复杂性感到困惑的情况，这通常只是意味着编译器的未来版本有更多的优化空间。

### 加载和存储操作

（<a href="https://marabos.nl/atomics/hardware.html#load-store-ops" target="_blank">英文版本</a>）

在我们深入研究更高级的内容之前，让我们首先看看最基本的原子操作：加载和存储指令。

通过 `&mut i32` 进行的采用的常规非原子指令，在 `x86-64` 和 `ARM64` 上只需要一个指令，如下所示：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &mut i32) {
    *x = 0;
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    mov dword ptr [rdi], 0
    ret</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 ARM64
    <pre>a:
    str wzr, [x0]
    ret</pre>
  </div>
</div>

在 x86-64 上，非常通用的 mov 指令用于将数据从一个地方复制（“移动”）到另一个地方；在这个示例下，是从 0 常数复制到内存。在 ARM64 上，`str`（存储寄存器）指令将一个 32 位寄存器存储到内存中。在这个示例下，使用的是特殊的 wzr 寄存器，它总是包含 0。

如果我们更改代码，而是使用 relaxed 排序的原子操作，我们将得到以下结果：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &mut i32) {
    *x = 0;
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    mov dword ptr [rdi], 0
    ret</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 ARM64
    <pre>a:
    mov dword ptr [rdi], 0
    ret</pre>
  </div>
</div>

或许令人惊讶的是，它的汇编与非原子版本完全相同。事实证明，mov 和 str 指令已经是原子的。它们要么发生，要么它们完全不会发生。显然，在 `&mut i32` 和 `&i32` 之间的任何差异仅对编译器检查和优化有关，但对于处理器是没有意外的——至少对于这两种架构上的 relaxed store 操作。

当我们查看 relaxed store 操作时，也是相同的：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &i32) -> i32 {
    *x
}</pre>
   <pre>pub fn a(x: &AtomicI32) -> i32 {
    x.load(Relaxed)
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    mov eax, dword ptr [rdi]
    ret</pre>
    <pre>a:
    mov eax, dword ptr [rdi]
    ret</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 ARM64
    <pre>a:
    mov eax, dword ptr [rdi]
    ret</pre>
    <pre>a:
    ldr w0, [x0]
    ret</pre>
  </div>
</div>

在 x86-64 上，mov 指令被再次使用，这次用于从内存中的值复制到 32 位 eax 寄存器。在 ARM64 上，使用 ldr（加载寄存器）指令将内存中的值加载到 w0 寄存器。

> 32 位的 eax 和 w0 寄存器用于返回函数的 32 位返回值。（对于 64 位值，使用 64 位的 rax 和 x0 寄存器。）

尽管处理器没有明显地区别原子和非原子的 store 和 load 操作之间的不同，但是在我们的 Rust 代码中不能安全地忽视这些区别。如果我们使用一个 `&mut i32`，Rust 编译器可能假设没有其他线程可以并发地访问相同的 i32 类型，并且可能决定以某种方式去转换或者优化代码，使得 store 操作不再导致单个相应的 store 指令。例如，通过使用两个单独的 16 位指令进行非原子的 32 位 load 或 store 操作时非常正确的，尽管有些不寻常。

### 读并修改并写操作

（<a href="https://marabos.nl/atomics/hardware.html#rmw-ops" target="_blank">英文版本</a>

对于*读并修改并写操作*来说，事情将变得更加有趣。正如本章早期讨论的那样，在类似 ARM64 的 RISC 架构中，非原子的读并修改并写操作通常编译为三个分开的指令（读、修改以及写），但在类似 x86-64 的 CISC 架构中，通常编译为一个单独的指令。这个简短的示例如下：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &mut i32) {
    *x += 10;
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    add dword ptr [rdi], 10
    ret</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 ARM64
    <pre>a:
    ldr w8, [x0]
    add w8, w8, #10
    str w8, [x0]
    ret</pre>
  </div>
</div>

我们甚至可以在查看相应的原子操作之前，合理的假设这次会看到一个非原子和原子版本之间的区别。ARM64 版本显然不是原子的，因为它的 load 和 store 操作发生在不同的步骤中。

尽管不能从汇编本身直接明显地看出，但 x86-64 版本并不是原子的。`add` 指令将由处理器在幕后分割成几个指令，分别的步骤为加载值和存储结果。在单核计算机中，这是无关紧要的，因为在指令之间切换处理器核心仅在线程之间发生。然而，当多个核心并行执行指令，我们在不考虑执行单个指令设计的多个步骤的情况下，就不能假设所有指令都以原子地方式发生。

#### x86 lock 前缀

（<a href="https://marabos.nl/atomics/hardware.html#lock-prefix" target="_blank">英文版本</a>）

为了支持多核系统，英特尔引入了一个称为 `lock` 的处理器前缀。它被用于像 `add` 这样的指令修饰符，以使它们的操作变得原子化。

lock 前缀最初是会导致处理器在指令执行期间，临时阻止所有其他核心访问内存。尽管这是一个简单和有效的方式，可以使在其他的核心看来像原子操作，但每次原子操作都停止其他活动（stop the world）是非常低效的。新的处理器对 lock 前缀的处理方式有更先进的 lock 前缀实现，它们并不会停止其他核心处理不相关的内存，并且允许核心在等待某块内存变得可获得期间，继续做有用的事情。

lock 前缀只能应用于非常有限数量的指令，包括 add、sub、and、or 以及 xor，这些都是非常有用的、能够以原子方式完成的操作。`xchg`（exchange）指令对英语原子交换操作，有一个隐式的 lock 前缀：无论 lock 前缀如何，它的行为都像 lock xchg/

让我们通过改变我们的最后一个示例来操作 AtomicI32，看看 lock add 的操作：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) {
    x.fetch_add(10, Relaxed);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    lock add dword ptr [rdi], 10
    ret</pre>
  </div>
</div>

正如预期的那样，与非原子版本唯一的区别就是 lock 前缀。

在上面的示例中，我们忽略了 fetch_add 返回的值，也就是操作之前 x 的值。然而，如果我们使用了这个值，add 指令就不够用了。add 指令可以向之后的指令提供一点有用的信息，比如更新后的值是否为零或负数，但它并未提供完整的（原始或更新的）值。相反，可以使用另一个指令：`xadd`（“交换并添加”），它将原来加载的值放入一个寄存器。

我们可以通过对我们的代码做一个小修改，使其返回 fetch_add 返回的值，来看到它的实际效果：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) -> i32 {
    x.fetch_add(10, Relaxed);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    mov eax, 10
    lock xadd dword ptr [rdi], eax
    ret</pre>
  </div>
</div>

现在使用一个包含 10 的寄存器，而不是常量 10。xadd 指令将重用该寄存器以存储旧值。

不幸的是，除了 xadd 和 xchg，其他可以加 lock 前缀的指令（例如 sub、and 以及 or）都没有这样的变体。例如，没有 xsub 指令。对于减法，这不是问题，因为 xadd 可以使用负数值。然而，对于 and 和 or 没有这样的替代方案。

对于 and、or 以及 xor 操作，只影响单个比特位，如 `fetch_or(1)` 或者 `fetch_and(1)`，可以使用 bts（比特测试和设置）、btr（比特测试和复位）以及 btc（比特测试和补码）指令。这些指令也允许一个 lock 前缀，只改变一个比特位，并且使该比特位的前一个值对后续的指令可用，如条件跳转。

当这些操作影响多个比特位，它们不能由单个 x86-64 指令表示。类似地，fetch_max 和 fetch_min 操作也没有相应的 x86-64 指令。对于这些操作，我们需要一个不同与简单 lock 前缀的策略。

#### x86 比较并交换指令

（<a href="https://marabos.nl/atomics/hardware.html#x86-cas" target="_blank">英文版本</a>）

在[第二章“比较并交换操作”](./2_Atomics.md#比较并交换操作)中，我们看到任何原子「获取并修改」操作都可以实现为一个「比较并交换」循环。对于由单个 x86-63 指令表示的操作，编译器可以使用这种方式，因为该架构确实包含一个（lock 前缀）的 cmpxcchg（比较并交换）指令。

我们可以通过将最后一个示例从 fetch_add 更改为 fetch_or 来在操作中看到这一点：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) -> i32 {
    x.fetch_or(10, Relaxed)
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    mov eax, dword ptr [rdi]
.L1:
    mov ecx, eax
    or ecx, 10
    lock cmpxchg dword ptr [rdi], ecx
    jne .L1
    ret</pre>
  </div>
</div>

第一条 mov 指令从原子变量加载值到 eax 寄存器。以下 mov 和 or 指令是将该值复制到 ecx 和应用二进制或操作，以至于 eax 包含旧值，ecx 包含新值。紧随之后的 cmpxchg 指令行为完全类似于 Rust 中 `compare_exchange` 方法。它的第一个参数是要操作的内存地址（原子变量），第二个参数（ecx）是新值，期望的值隐式地从 eax 取出，并且返回值隐式地存储在 eax 中。它还设置了一个状态标识，后续的指令可以用根据操作是否成功，来有条件的进行分支跳转。在这种情况下，使用 jne（如果与期待值不等则跳转）指令跳回 `.L1` 标签，在失败时再次尝试。

以下是 Rust 中等效的「比较并交换」循环的样子，就像我们在[第2章的“比较和交换操作”](./2_Atomics.md#比较并交换操作)中看到的那样：

```rust
pub fn a(x: &AtomicI32) -> i32 {
    let mut current = x.load(Relaxed);
    loop {
        let new = current | 10;
        match x.compare_exchange(current, new, Relaxed, Relaxed) {
            Ok(v) => return v,
            Err(v) => current = v,
        }
    }
}
```

编译此代码会导致与 `fetch_or` 版本完全相同的汇编。这表明，至少在 x86-64 上，它们在各方面确实都是相等的。

> 在 x86_64，在 compare_exchange 和 compare_exchange_weak 之间是有没有区别的。两者都编译为 lock cmpxchg 指令。

### LL 和 SC 指令

（<a href="https://marabos.nl/atomics/hardware.html#load-linked-and-store-conditional-instructions" target="_blank">英文版本</a>）

在 RISC 架构上最接近「比较并交换」循环的是 *load-linked/store-conditional*（LL/SC）循环。它包括两个特殊的指令，这两个指令成对出现：LL 指令的行为更像常规的 load 指令；SC 指令，其行为更像常规的 store 指令。它们都成对的使用，两个指令都针对同一个内存地址。与常规的 load 和 store 指令的主要区别是 store 是有条件的：如果自从 LL 指令以来任何其他线程已经覆盖了该内存，它就会拒绝存储到内存。

这两条指令允许我们从内存加载一个值，修改它，并只有在仅在没有人从我们加载它以来覆盖该值的情况下，才将新值存回。如果失败，我们可以简单地重试。一旦成功，我们可以安全地假装整个操作是原子的，因为它没有被打断。

使这些指令可行且高效的关键有两点：（1）一次只能追踪一个内存地址（每个核心），（2）存储条件允许有伪阴性[^2]，意味着即使没有任何东西改变这个特定的内存片段，它也可能失败存储。

这使得可以在跟踪内存更改时不那么精确，但可能需要通过 LL/SC 循环额外花几个周期。访问内存的跟踪可以不按字节，而是按 64 字节的分块，或者按千字节，甚至整个内存作为一个整体。不够精确的内存跟踪导致更多不必要的 LL/SC 循环，显著降低性能，但也降低了实现的复杂性。

采用一个极端的想法，一个基本的、假设的单核系统可以使用一种策略，即完全不跟踪对内存的写入。相反，它可以跟踪中断或上下文切换，这些事件可以导致处理器切换到另一个线程。如果在一个没有任何并行性的系统中，没有发生这样的事件，它可以安全地假设没有其他线程可能触及到内存。如果发生了这样的事件，它可以仅假设最糟糕的情况，拒绝存储，并希望在循环的下一次迭代中有更好的运气。

#### ARM 的 ldxr 和 stxr 指令

（<a href="https://marabos.nl/atomics/hardware.html#arm-load-exclusive-and-store-exclusive" target="_blank">英文版本</a>）

在 ARM64 中，或者至少是 ARMv8 的第一个版本，没有任何原子「获取并修改」或者「比较并交换」操作可以通过单个指令表示。对于 RISC 的性质，load 和 store 步骤与计算和比较是分离的。

ARM64 的 LL 和 SC 指令被称为 ldxr（加载独占寄存器）和 stxr（存储独占寄存器）。此外，clrex（清理独占）指令可以用作停止跟踪对内存的写入，而不存储任何东西的 stxr 替代。

为了看到它们的实际效果，让我们看看在 ARM64 上进行原子加时会发生什么：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) {
    x.fetch_add(10, Relaxed);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 ARM64
    <pre>a:
    mov eax, dword ptr [rdi]
.L1:
    ldxr w8, [x0]
    add w9, w8, #10
    stxr w10, w9, [x0]
    cbnz w10, .L1
    ret</pre>
  </div>
</div>

我们得到的东西看起来非常类似于我们之前得到的非原子版本（在[“读并修改并写操作”](#读并修改并写操作)）：一个 load 指令、一个 add 指令以及一个 store 指令。load 和 store 指令已经被替换为它们的“独占”LL/SC 版本，并且出现一个新的 cbnz（非 0 比较和分支）指令。如果成功，stxr 指令在 w10 存储一个 0，如果失败，则存储 1。cbnz 指令利用这点，如果操作失败，则重启整个操作。

注意，与 x86-64 上的 lock add 不同，我们不需要做任何特殊的处理检索旧值。在以上示例中，操作成功后，旧值将仍然在寄存器 w8 可获得，所以并不需要相 xadd 这样的特殊指令。

这钟 LL/SC 模式是非常灵活的：它不仅可以用于像 add 和 or 这样有限的操作集，而是可以用于几乎任何操作。我们可以通过在 ldxr 和 stxr 指令之间放入相应的指令，轻松地实现原子 fetch_divide 或 fetch_shift_left 操作。然而，如果它们之间有太多的指令，中断的几率就会越来越高，导致额外的周期。通常，编译器会尝试在 LL/SC 模式中的指令数量尽可能少，防止 LL/SC 循环频繁失败可能从不成功，并且以及可能无限自旋的情况。

<div class="box">
  <h2 style="text-align: center;">ARMv8.1 原子指令</h2>
  <p>ARMv8.1 的后续版本 ARM64，还包括新的 CISC 风格指令，用于常见的原子操作。例如，新的 ldadd（加载并添加）指令等效于一个原子 fetch_add 操作，无需使用 LL/SC 循环。它甚至包括像 fetch_max 这样的操作指令，这在 x86-64 上并不存在。</p>

  <p>它还包括一个与 <code>com⁠pare_​exchange</code> 相对应的 cas（比较并交换）指令。当使用此指令时，<code>compare_exchange</code> 和 <code>compare_exchange_weak</code> 之间没有差别，就像在 x86-64 上一样。</p>

  <code>尽管 LL/SC 模式非常灵活，并且很好地适应了一般的 RISC 模式，但这些新指令可以更高效，因为它们可以通过特定的硬件进行更轻松的优化。</code>
</div>

#### ARM 的「比较和交换」操作

（<a href="https://marabos.nl/atomics/hardware.html#compare-and-exchange-on-arm" target="_blank">英文版本</a>）

`compare_exchange` 操作通过使用条件分支指令在比较失败时跳过 store 指令，这与 LL/LC 模式的映射非常恰当。让我们来看看生成汇编的代码：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) {
    x.compare_exchange_weak(5, 6, Relaxed, Relaxed);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 ARM64
    <pre>a:
    ldxr w8, [x0]
    cmp w8, #5
    b.ne .L1
    mov w8, #6
    stxr w9, w8, [x0]
    ret
.L1:
    clrex
    ret</pre>
  </div>
</div>

> 注意，compare_and_exchange 操作通常用于一个循环，如果该比较失败，则循环重复。然而，对于该示例，我们仅调用一次并且忽略它的返回值，这让我们在无干扰的情况下查看汇编。

ldxr 指令加载了值，然后立即通过 cmp（比较）指令将其与预期的值 5 进行比较。如果值不符合预期，`b.ne`（如果与预期值不等则跳转分支）指令会导致跳转到 `.L1` 标签，在此刻，clrex 指令用于中止 LL/SC 模式。如果值是 5，流程将通过 mov 和 stxr 指令继续，将新的值 6 存储到内存中，但这只会在与此同时没有任何东西覆盖 5 的情况下发生。

请记住，stxr 允许有伪阴性；即使 5 没有被覆盖，这里也可能失败。这没问题，因为我们正在使用 `compare_exchange_weak`，它也允许有伪阴性。事实上，这就是为什么存在 `compare_exchange` 的 weak 版本。

如果我们将 `compare_exchange_weak` 替换为 `compare_exchange`，我们得到的汇编代码几乎完全相同，除了在操作失败时会有额外的分支来重新启动操作：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) {
    x.compare_exchange(5, 6, Relaxed, Relaxed);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 ARM64
    <pre>a:
    mov w8, #6
.L1:
    ldxr w9, [x0]
    cmp w9, #5
    b.ne .L2
    stxr w9, w8, [x0]
    cbnz w9, .L1
    ret
.L2:
    clrex
    ret</pre>
  </div>
</div>

正如预期的那样，现在有一个额外的 cbnz（非 0 的比较和分支）指令，在失败时去重新开始 LL/LC 循环。此外，mov 指令已经移出循环，以保证循环尽可能地短。

<div class="box">
  <h2 style="text-align: center;">「比较并交换」循环的优化</h2>
  <p><a>正如我们在<a href="#x86-比较并交换指令">x86-比较并交换指令</a>中看到的，在 x86-64 上，<code>fetch_or</code> 操作和等效的 <code>compare_exchange</code> 循环编译成了完全相同的指令。人们可能期望在 ARM 上也会发生同样的情况，至少在 <code>compare_exchange_weak</code> 上，因为加载和「弱比较并交换」操作可以直接映射到 LL/SC 指令。</a></p>

  <p>不幸的是，当前（截至 Rust 1.66.0）的情况并非如此。</p>

  <p>虽然随着编译器的不断改进，这种情况可能会在未来发生变化，但编译器要安全地将手动编写的「比较并交换」循环转化为相应的 LL/SC 循环还是相当困难的。其中一个原因是，可以放在 stxr 和 ldxr 指令之间的指令的数量和类型是有限的，这不是编译器在应用其他优化时需要考虑的内容。在像「比较并交换」这样的模式还可以识别时，表达式将编译成的确切指令还不清楚，这使得对于一般情况来说，这是一个非常棘手的优化问题。</p>

  <p>因此，直到在我们有更聪明的编译器之前，如果可能的话，建议使用专用的「获取并修改」方法，而不是「比较并交换」循环。</p>
</div>

## 缓存

（<a href="https://marabos.nl/atomics/hardware.html#caching" target="_blank">英文版本</a>）

### 缓存一致性

（<a href="https://marabos.nl/atomics/hardware.html#cache-coherence" target="_blank">英文版本</a>）

#### 写入协议

（<a href="https://marabos.nl/atomics/hardware.html#the-write-through-protocol" target="_blank">英文版本</a>）

#### MESI 协议

（<a href="https://marabos.nl/atomics/hardware.html#mesi" target="_blank">英文版本</a>）

### 对性能的影响

（<a href="https://marabos.nl/atomics/hardware.html#performance-experiment" target="_blank">英文版本</a>）

## 重排

（<a href="https://marabos.nl/atomics/hardware.html#reordering" target="_blank">英文版本</a>）

## 内存排序

（<a href="https://marabos.nl/atomics/hardware.html#memory-ordering" target="_blank">英文版本</a>）

### x86-64：强排序

（<a href="https://marabos.nl/atomics/hardware.html#x86-64-strongly-ordered" target="_blank">英文版本</a>）

### ARM64：弱排序

（<a href="https://marabos.nl/atomics/hardware.html#arm64-weakly-ordered" target="_blank">英文版本</a>）

### 一个实验

（<a href="https://marabos.nl/atomics/hardware.html#reordering-experiment" target="_blank">英文版本</a>）

### 内存屏障

（<a href="https://marabos.nl/atomics/hardware.html#memory-fences" target="_blank">英文版本</a>）

## 总结

（<a href="https://marabos.nl/atomics/hardware.html#summary" target="_blank">英文版本</a>）

* 在 x86-64 和 ARM64 中，relaxed load 和 store 操作与它们的非原子版本等同。
* 常见的原子「获取并修改」和「比较并交换」操作在 x86-64（以及从 ARM8.1 开始的 ARM64）上有它们自己的指令。
* 在 x86-64 中，对于等效的指令原子操作会编译到「比较并交换」循环。
* 在 ARM64 中，任意ID原子操作都可以通过 ll/sc 指令循环表示：如果试图的内存操作被中断，循环会自动重新开始。
* 缓存的操作时缓存行，一般是 64 字节。
* 缓存使用缓存一致性协议保持一致，例如通过写入或者 MESI。
* 填充可以通过防止伪共享[^1]来提高性能，例如通过 `#[repr(align(64)]`。
* load 操作可能比失败的「比较并交换」操作要便宜得多，部分原因是后者通常需要对缓存行进行独占访问。
* 指令重排序在单线程程序内部是不可见的。
* 在大多数架构上，包括 x86-64 和 ARM64，内存排序是为了防止某些类型的指令重排。
* 在 x86-64 上，每个内存操作都具有 acquire 和 release 语义，使其与 relaxed 操作一样便宜或昂贵。除存储和屏障外的所有其他操作也具有顺序一致的语义，无需额外成本。
* 在 ARM64 上，acquire 和 release 语义不如 relaxed 操作便宜，但也包括顺序一致（例如，SeqCst）语义，无需额外成本。

我们在本章节可以看见的汇编指令的总结可以在图 7-1 找到。

![ ](./picture/raal_0701.svg)

图 7-1。各种原子操作在 ARM64 和 x86-64 上编译为每个内存排序的指令概述。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./8_Operating_System_Primitives.html">下一篇，第八章：操作系统原语</a>
</p>

[^1]: <https://zh.wikipedia.org/wiki/伪共享>
[^2]: <https://zh.wikipedia.org/zh-cn/偽陽性和偽陰性>
