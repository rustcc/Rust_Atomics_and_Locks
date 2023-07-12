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

## 缓存[^3]

（<a href="https://marabos.nl/atomics/hardware.html#caching" target="_blank">英文版本</a>）

读取和写入内存是缓慢的，并且很容易花费执行数十甚至数百条指令的时间。这就是为什么所有高性能的处理器都实现了*缓存*，以尽可能避免与相对较慢的内存进行交互。现代处理器中内存缓存的具体实现细节复杂、有的是独有的，而且最重要的是，当我们编写软件时，这些细节大部分对我们来说都不相关。毕竟，*缓存*（cache）这个词源自法语单词 `caché`，意思是*隐藏*。尽管如此，在优化软件性能时，理解大多数处理器幕后如何实现缓存的基本原理时可能非常有用。（当然，我们并不需要借口去学习更多有关的主题。）

除了非常小的微控制器，几乎所有现代处理器都使用缓存。这样的处理器从不直接与主内存交互，而是通过它的缓存路由每个读取和写入请求。如果一个指令需要从内存中读取某些内容，处理器将向其缓存请求这些数据。如果数据已经在缓存中，缓存将快速响应并提供缓存的数据，从而避免与主内存交互。否则，它将不得不走一条慢路，即缓存可能需要向主内存请求相关数据的副本。一旦主内存响应，缓存不仅最终会响应原始的读取请求，同时也会记住这些数据，以便在下次请求这些数据时能更快地响应。如果缓存满了，它会通过丢弃一些它认为最不可能有用的旧数据来腾出空间。

当一个指令想要将某些内容写入内存时，缓存可能会决定保留修改后的数据，而不将其写入主内存。任何后续对相同内存地址的读取请求将得到修改后数据的副本，从而忽略主内存中过时的数据。仅有在需要从缓存中丢弃修改后的数据以腾出空间时，才会实际将数据写回主内存。

在大多数处理器架构中，缓存以 64 字节的分块读取和写入内存，即使只请求了一个字节。这些块通常被称为*缓存行*（cache line）。通过缓存该请求字节周围的整个 64 字节分块，任何后续需要访问该分块中的其他字节的指令都不必等待主内存。

### 缓存一致性

（<a href="https://marabos.nl/atomics/hardware.html#cache-coherence" target="_blank">英文版本</a>）

在现代处理器中，通常有不止一层缓存。第一层缓存，或者称为*一级*（L1）缓存，是最小且最快的。它不和主内存通信，而是和*二级*（L2）缓存通信，后者虽然更大，但速度慢一些。*L2 缓存*可能是与主内存通信的那个，或者可能还有另一个更大更慢的 L3 缓存——甚至可能有 L4 缓存。

添加额外的层并不会改变它们的工作方式；每一层都可以独立运行。然而，当存在多个处理器核心，每个核心都有自己的缓存时，情况就变得有趣了。在多核系统中，每个处理器核心通常有自己的 L1 缓存，而 L2 或 L3 缓存往往与部分或所有其他核心共享。

在这种条件下，原本的缓存实现会崩溃，因为缓存不能再假设它控制着所有与下一层的交互。如果一个缓存接受了写操作并将某个缓存行标记为已修改，而没有通知其他的缓存，那么缓存的状态可能会变得不一致。修改后的数据直到缓存将数据写入下一层之前，不会对其他核心可用，而且最终可能会与其他缓存中缓存的不同修改发生冲突。

为了解决这个问题，我们使用了一种叫做*缓存一致性协议*。这样的协议定义了如何准确地操作缓存并与其他缓存通信，以保持所有的状态一致。具体使用的协议根据架构、处理器模型，甚至每个缓存层都有所不同。

我们将讨论两种基本的缓存一致性协议。现代处理器使用这些协议的许多变体。

#### write-through 协议

（<a href="https://marabos.nl/atomics/hardware.html#the-write-through-protocol" target="_blank">英文版本</a>）

在缓存中，实施 write-through 缓存一致性协议，写操作不会被缓存，而是立即发送到下一层。其它缓存通过同一共享通信通道连接到下一层，这意味着它们可以观察到其它缓存与下一层的通信状况。当缓存观察到某个地址的写操作，而该地址当前已在缓存中，它会立即丢弃或更新自己的缓存行，以保持一致性。

使用这种协议，缓存永远不会包含任何处于已修改状态的缓存行。尽管这极大地简化了事情，但对于写操作，它丧失了缓存的优势。当仅针对读取进行优化时，这可能是一个很好的选择。

#### MESI 协议[^4]

（<a href="https://marabos.nl/atomics/hardware.html#mesi" target="_blank">英文版本</a>）

*MESI 缓存一致性协议*是由之后的四种可能状态命名的，它为缓存行定义了：已修改（Modified，M）、独占（Exclusive，E）、共享（Shared，S）和无效（Invalid，I）。已修改（M）用于包含已经修改数据的缓存行，但该数据尚未写入到内存（或下一级缓存）数据。独占（E）用于包含未修改数据的缓存行，且该数据没有缓存在任意其他缓存中（在同一级别）。共享（S）用于包含未修改数据的缓存行，这些缓存行可能也出现在一个或多个其他（同级别）的缓存中。无效（I）用于未使用（空的或被丢弃）的缓存行，它们不包含任何有用的数据。

使用此协议的缓存会与同级别的所有其他缓存进行通信。它们互相发送更新和请求，使它们能够保持一致性。

当一个缓存接收到一个它尚未缓存的地址的请求（也称为*缓存未命中*）时，它不会立即从下一层请求。相反，它首先询问其他（同级别的）缓存是否有可用的这个缓存行。如果没有，缓存将继续从（更慢的）下一层请求地址，并将结果标记为独占（E）。当此缓存行被写操作修改时，缓存可以将状态改为已修改（M），而不通知其他缓存，因为它知道其他缓存没有缓存相同的缓存行。

当请求一个已经在任何其他缓存中可用的缓存行时，结果是一个共享（S）的缓存行，可以直接从其他缓存获得。如果缓存行处于已修改（M）状态，它将首先被写入（或*刷新*）到下一层，然后再改变为共享（S）并共享。如果它处于独占（E）状态，它将立即被改变为共享（S）。

如果缓存想要独占访问权，而不是共享访问权（例如，因为它将在之后立即修改数据），其他缓存不会保持缓存行在共享（S）状态，而是通过将其更改为无效（I）来完全丢弃它。在这种情况下，结果是一个独占（E）的缓存行。

如果一个缓存需要对已经在共享（S）状态下可用的缓存行进行独占访问，它只需告诉其他缓存丢弃这个缓存行，然后再将其升级为独占（E）。

此协议有几种变体。例如，*MOESI* 协议添加了一个额外的状态，以允许在不立即将其写入下一层的情况下共享修改过的数据，而 *MESIF* 协议使用一个额外的状态，该状态决定哪个缓存可以响应多个缓存中可用的共享缓存行的请求。现代处理器通常使用更复杂的和专有的缓存一致性协议。

### 对性能的影响

（<a href="https://marabos.nl/atomics/hardware.html#performance-experiment" target="_blank">英文版本</a>）

尽管缓存大多数时候对我们是隐藏的，但缓存行为对我们的原子操作性能可能有重要影响。让我们尝试测量其中一些影响。

测量单个原子操作的速度非常棘手，因为它们速度极快。为了能得到一些有用的数据，我们必须重复一个操作，比如说，十亿次，然后测量总体花费的时间。例如，我们可以尝试测量十亿次加载 load 需要多少时间，就像这样：

```rust
static A: AtomicU64 = AtomicU64::new(0);

fn main() {
    let start = Instant::now();
    for _ in 0..1_000_000_000 {
        A.load(Relaxed);
    }
    println!("{:?}", start.elapsed());
}
```

不幸的是，这并没有按照我们的预期工作。

当通过优化之后运行这段代码时（例如，使用 `cargo run --release` 或者 `rustc -O`），我们将看见不合理的低测量时间。编译器足够智能知道发生了什么状况，它能够理解我们并没有使用加载的值，所以它决定完全优化掉*不需要的*循环。

为了避免这种情况，我们可以使用特殊的 `std::hint::black_box` 函数。这个函数接受任何类型的参数，它只是在不做任何优化的情况下返回这个参数。这个函数的特殊之处在于，编译器会尽可能不假设这个函数做的任何事情；它把这个函数当作一个可能做任何事情的“黑箱”来对待。

我们可以使用这个函数来避免某些可能使基准测试无效的优化。在这种情况下，我们可以将 load 操作的结果传递给 `black_box()`，以停止任何优化，这里假设我们实际上不需要加载值的优化。然而，这还不够，因为编译器可能仍然假设 A 总是 0，这使得 load 操作是不必要的。为了避免这种情况，我们可以在开始时将一个指向 A 的引用传递给 `black_box()`，这样编译器就不能再假设只有一个线程访问 A 了。毕竟，它必须假设 `black_box(&A)` 可能已经产生了一个与 A 交互的额外线程。

让我们试试看：

```rust
use std::hint::black_box;

static A: AtomicU64 = AtomicU64::new(0);

fn main() {
    black_box(&A); // 新增！
    let start = Instant::now();
    for _ in 0..1_000_000_000 {
        black_box(A.load(Relaxed)); // 新增！
    }
    println!("{:?}", start.elapsed());
}
```

这段代码在运行多次时，输出可能有点波动，但是在一台不是很新的 x86-64 电脑上，它似乎是大约 300 毫秒的结果。

为了观察任何缓存影响，我们将产生一个后台线程与原子变量交互。这样，我们可以看到它是否影响主线程的的 load 操作。

首先，让我们尝试一下，只需在后台线程上加载操作，如下所示：

```rust
static A: AtomicU64 = AtomicU64::new(0);

fn main() {
    black_box(&A);

    thread::spawn(|| { // 新增！
        loop {
            black_box(A.load(Relaxed));
        }
    });

    let start = Instant::now();
    for _ in 0..1_000_000_000 {
        black_box(A.load(Relaxed));
    }
    println!("{:?}", start.elapsed());
}
```

注意，我们没有测量后台线程上的操作性能。我们仍然仅是测量主线程上执行一百万的 load 操作的性能。

运行这个程序，导致与之前类似的测量结果：当在同一台 x86-64 计算机上进行测试时，它会在 300 毫秒左右波动。后台线程并不会对主线程有什么影响。它们大概都在一个单独的处理器内核上运行，但*两个*核心的缓存都包含 A 的副本，这允许非常快速的访问。

现在让我们更改后台线程来执行 store 操作：

```rust
static A: AtomicU64 = AtomicU64::new(0);

fn main() {
    black_box(&A);
    thread::spawn(|| {
        loop {
            A.store(0, Relaxed); // 新增！
        }
    });
    let start = Instant::now();
    for _ in 0..1_000_000_000 {
        black_box(A.load(Relaxed));
    }
    println!("{:?}", start.elapsed());
}
```

这次，我们确实看到了显著的差异。现在在 x86-64 架构上运行这个程序，导致的输出波动大概有 3 秒，是之前的十倍。最新的计算机将展示更小的差异，但仍然是可衡量的不同。例如，在最新的苹果 M1 处理器上，它从 350 毫秒上升到 500 毫秒，在最新的 x86-64 AMD 处理器上，它从 250 毫秒上升到 650 毫秒。

这种行为匹配我们对缓存一致性的理解：store 操作需要独占访问缓存行，这会减慢在其他核上不再共享缓存行的后续 load 操作。

<div class="box">
  <h2 style="text-align: center;">「比较并交换」操作失败</h2>
  <p>有趣的是，在大多数处理器架构中，当后台线程只进行「比较并交换」操作时，我们也能观察到和 store 操作相同的效果，即使所有的「比较并交换」操作都失败。</p>

  <p>为了验证这一点，我们可以将后台线程的 store 操作替换为一个永远不会成功的 compare_exchange 调用：</p>

  <pre>    …
        loop {
            // 从不成功，因为 A 从不会是 10
            black_box(A.compare_exchange(10, 20, Relaxed, Relaxed).is_ok());
        }
    …</pre>
  
  <p>因为 A 总是 0，<code>compare_exchange</code> 操作将从不成功。它将加载当前的 A 值，但是从不更新它到一个新值。</p>

  <p>人们可能合理的将这个行为与 load 操作等同，因为它从没有修改原子变量。然而，在大多数处理器架构中，无论比较是否成功，<code>compare_exchange</code> 的指令都将声明相关缓存行的独占访问权限。</p>

  <p>这意味着，对于我们在<a href="./4_Building_Our_Own_Spin_Lock.html">第四章</a>的 SpinLock 中不使用 compare_exchange（或 swap）来自旋循环可能更高效，而是首先使用 load 操作去检查锁是否已经释放锁。那样，我们可以避免不必要地声明相关缓存行的独占访问权限。</p>
</div>

由于缓存是按照缓存行进行的，而不是按照单个字节或变量进行的，所以我们应该能够看到使用相邻的变量而不是相同的变量也会产生相同的效果。为了验证这个，让我们使用三个原子变量而不是一个，让主线程仅使用中间的变量，并让后台线程只使用其他两个，如下所示：

```rust
static A: [AtomicU64; 3] = [
    AtomicU64::new(0),
    AtomicU64::new(0),
    AtomicU64::new(0),
];

fn main() {
    black_box(&A);
    thread::spawn(|| {
        loop {
            A[0].store(0, Relaxed);
            A[2].store(0, Relaxed);
        }
    });
    let start = Instant::now();
    for _ in 0..1_000_000_000 {
        black_box(A[1].load(Relaxed));
    }
    println!("{:?}", start.elapsed());
}
```

运行这个片段后，我们得到的结果和之前类似：在同样的 x86-64 计算机上，它还是需要花费数秒的时间。尽管 `A[0]`、`A[1]` 和 `A[2]` 仅被一个线程使用，我们仍然看到相同的效果，与两个线程使用同一个变量一样。原因在于，`A[1]` 和其他一或两个变量共享同一缓存行。运行后台线程的处理器核心反复地对包含 `A[0]` 和 `A[2]` 的缓存行（也包含 `A[1]`）声明独占访问权限，从而拖慢了对 `A[1]` 的“无关”操作。这种问题被称为伪共享[^1]。

我们可以通过将原子变量间隔更远来避免这个问题，这样每个变量都可以拥有自己的缓存行。如前所述，64 字节是一个合理的猜测值，用于表示缓存行的大小，所以让我们试着将我们的原子变量包装在一个 64 字节对齐的结构体中，如下所示：

```rust
#[repr(align(64))] // 这个结构体必须是 64 字节对齐
struct Aligned(AtomicU64);

static A: [Aligned; 3] = [
    Aligned(AtomicU64::new(0)),
    Aligned(AtomicU64::new(0)),
    Aligned(AtomicU64::new(0)),
];

fn main() {
    black_box(&A);
    thread::spawn(|| {
        loop {
            A[0].0.store(1, Relaxed);
            A[2].0.store(1, Relaxed);
        }
    });
    let start = Instant::now();
    for _ in 0..1_000_000_000 {
        black_box(A[1].0.load(Relaxed));
    }
    println!("{:?}", start.elapsed());
}
```

`#[repr(align)]` 属性允许我们告诉编译器我们的类型的（最小）对齐值，以字节为单位。由于 AtomicU64 仅有 8 字节，这将给我们的 Aligned 结构体添加 56 字节的填充

运行这个程序不再给出缓慢的结果。相反，我们得到的结果和完全没有后台线程一样：当在与之前同一台 x86-64 计算机上运行时，约需要 300 毫秒。

> 根据你正在尝试的处理器的类型，你可能需要使用 128 字节的对齐才能看到相同的效果。

上面的实验表明，建议不要把不相关的原子变量放得太近。例如，密集的小型 mutex 数组可能并不总是能够表现得和一个让 mutex 间距隔离更远的替代结构一样好。

另一方面，当多个（原子）变量相关并且经常快速连续访问时，将它们放在一起可能是有益的。例如，我们在[第4章](./4_Building_Our_Own_Spin_Lock.md)中的 `SpinLock<T>` 将 T 紧挨着 AtomicBool 存储，这意味着包含 AtomicBool 的缓存行也可能包含 T，因此对一个（独占）访问的声明也包括了另一个。这是否有益完全取决于情况。

## 重排

（<a href="https://marabos.nl/atomics/hardware.html#reordering" target="_blank">英文版本</a>）

一致性缓存，例如我们在本章前面探讨的 MESI 协议，通常不会影响程序的正确性，即使涉及多个线程。由一致性缓存引起的唯一可观察的差异归结为时间上的差异。然而，现代处理器实现了更多的优化，尤其是这些优化在涉及多个线程时可能对正确性产生重大影响。

在[第 3 章](./3_Memory_Ordering.md)开始时，我们简要地讨论了*指令重排*，即编译器和处理器如何改变指令的顺序。仅关注处理器，这里有一些指令，或者它们的效果，可能以*不同的顺序*发生的示例：

* *存储缓冲区*（store buffer）[^5]

  因为写入可能较慢，即使有缓存，处理器核心通常包含一个*存储缓冲区*。内存的写操作可以存储在这个存储缓冲区中，这非常快，这允许处理器立即继续执行随后的指令。然后，在后台，通过写入（L1）缓存完成写操作，这可能要慢得多。这样，处理器就不需要等待缓存一致性协议跳入动作，以得到相关缓存行的独占访问权限。

  只要采取对后续来自同一内存地址的读操作特别注意的处理，这对于作为同一处理器核心上的同一线程的一部分运行的指令来说，是完全不可见的。然而，对于一个短暂的时刻，写操作还没有对其他核心可见，这导致了从在不同核心上运行的不同线程看内存的视图不一致。

* *失效队列*（Invalidation queue）[^6]

  无论精确的一致性协议如何，并行方式运行的缓存都需要处理失效请求：这是一种指令，指示丢弃特定的缓存行，因为这个缓存行即将被修改并变得无效。作为性能优化，通常这样的请求并不会立即处理，而是排队等待（稍后）处理。当使用这样的失效队列时，缓存不再始终是一致的，因为缓存行可能在被丢弃前短暂地过时。然而，除了使单线程程序加快，这对单线程程序没有影响。唯一的影响是来自其他核心的写操作的可见性，这可能现在看起来像是（非常轻微的）延迟。

* *流水线*（pipeline）[^7]

  另一个极其常见的可以显著提高性能的处理器特性是*流水线*：如果可能的话，尽可能并行执行连续的指令。在一个指令完成执行之前，处理器可能已经开始执行下一个指令。现代处理器通常可以在第一个指令仍在处理的同时，开始执行一系列的指令。

  如果每个指令都在根据前一个指令的结果运行，这并没有什么帮助；它们仍然需要等待前一个的结果。但是，当一个指令可以独立于前一个指令执行时，它甚至可能先完成。例如，一个对寄存器仅仅进行递增的指令可能很快就完成，而前面开始的一个指令可能仍在等待从内存中读取数据，或者其他一些慢的操作。

  虽然这对单线程程序（除了速度）没有影响，但当一个操作内存的指令在其前面的指令完成执行之前就完成了，可能会导致与其他核的交互发生在预期顺序之外。

在很多方面，现代处理器可能以完全不同于预期的顺序执行指令。其中涉及许多专有技术，有些只有在发现可以被恶意软件利用的微妙错误时才公开。然而，当它们按预期工作时，它们都有一个共同点：除了时间，它们不会影响单线程程序，但可能导致与其他核心的交互看起来是不一致的顺序。

允许内存操作被重排的处理器架构也提供了通过特殊指令防止这种情况发生的方式。例如，这些指令可能强制刷新处理器的存储缓冲区，或者在继续之前完成任何流水线的指令。有时，这些指令只防止某种类型的重排。例如，可能有一种指令可以防止存储操作相对于彼此被重排，同时仍然允许 load 操作被重排。可能发生哪种类型的重排，以及如何防止它们，取决于处理器架构。

## 内存排序

（<a href="https://marabos.nl/atomics/hardware.html#memory-ordering" target="_blank">英文版本</a>）

当执行像 Rust 或 C 这样的语言中的任意原子操作时，我们会指定一个内存排序去告知编译器我们的排序需求。编译器将为处理器生成正确的指令，以防止它以某种方式重排指令，这将可能打破规则，使程序不正确。

允许哪种类型的指令重新排序屈居于内存的操作。对于非原子和 relaxed 原子操作，任意类型的重排是可接受的。在另一个极端情况下，顺序一致原子操作完全不允许任意类型的原子排序。

acquire 操作不能与随后的任意内存操作重排，而 release 操作不能与之前的任意内存操作重排。否则，可能在 acquire mutex 之前或者 release mutex 之后，访问一些受 mutex 保护的数据可能会导致数据竞争。

<div class="box">
  <h2 style="text-align: center;">Other-Multi-Copy Atomicity</h2>

  <p>在一些处理器架构（例如，可能在显卡中找到的那些）中，内存操作顺序的影响方式并不总是可以通过指令重排序来解释。在一个核上的两个连续的 store 操作的效果可能会按照相同的顺序在第二个核上变得可见，但在第三个核上的顺序可能恰恰相反。例如，由于缓存不一致或共享存储缓冲区，可能会发生这种情况。由于这并不能解释第二核和第三核的观察结果之间的不一致性情况，所以无法通过第一个核上的指令被重排序来解释这种情况。</p>

  <p>我们在<a href="./3_Memory_Ordering.html">第 3 章</a>中讨论的理论内存模型为此类处理器架构留出了空间，因为它不要求除顺序一致的原子操作之外的任何操作具有全局一致的顺序。</p>

  <p>我们在本章中聚焦的架构（x86-64 和 ARM64）是“other-multi-copy atomic”，这意味着一旦写操作对任何核可见，它们就同时对所有核可见。对于其他“other-multi-copy atomic”架构，内存排序只是指令重排序的问题。</p>
</div>

一些架构（例如 ARM64）被称为*弱排序*，因为它们允许处理器自由地重排任意的内存操作。另一方面，*强排序*架构（例如 x86-64）对哪些内存操作可以排序是非常严格的。

### x86-64：强排序

（<a href="https://marabos.nl/atomics/hardware.html#x86-64-strongly-ordered" target="_blank">英文版本</a>）

在 x86-64 处理器上，load 操作将从不随后的内存操作之后发生。类似的，该架构也不允许 store 操作在之前的内存操作之前发生。你可能在 x86-64 上看到的唯一一种重新排序是 store 操作被延迟到稍后的 load 操作之后。

> 由于 x86-64 架构的重排序限制，它通常被描述为强排序架构，尽管有些人更愿意保留这个这个术语描述所有保留内存操作排序的架构。

这些限制满足了 acquire-load（因为 load 从不和后续操作重排序）和 release-store（因为 store 从不和之前的操作重排序）的所有需要。这意味着在 x86-64 上，我们可以“免费的”获取 release 和 acquire 语义：release 和 acquire 与 relaxed 操作等同。

我们可以通过查来自[加载和存储](#加载和存储操作)以及 [x86 lock 前缀](#x86-lock-前缀)片段来验证这些，然而我们要将 Relaxed 改变到 Release、Acquire 或 AcqRel：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) {
    x.store(0, Release);
}</pre>
    <pre>pub fn a(x: &AtomicI32) -> i32 {
    x.load(Acquire)
}</pre>
    <pre>pub fn a(x: &AtomicI32) {
    x.fetch_add(10, AcqRel);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    mov dword ptr [rdi], 0
    ret</pre>
    <pre>a:
    mov eax, dword ptr [rdi]
    ret</pre>
    <pre>a:
    lock add dword ptr [rdi], 10
    ret</pre>
  </div>
</div>

不出所料，尽管我们指定了更强的内存顺序，但汇编是相同的。

我们可以得出结论，在 x86-64 上，忽略潜在的编译器优化，acquire 和 release 操作仅和 relaxed 操作一样便宜。或者，更准确地说，relaxed 操作和 acquire 和 release 操作一样昂贵。

让我们看看 SeqCst 发生了什么：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) {
    x.store(0, SeqCst);
}</pre>
    <pre>pub fn a(x: &AtomicI32) -> i32 {
    x.load(SeqCst)
}</pre>
    <pre>pub fn a(x: &AtomicI32) {
    x.fetch_add(10, SeqCst);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    xor eax, eax
    xchg dword ptr [rdi], eax
    ret</pre>
    <pre>a:
    mov eax, dword ptr [rdi]
    ret</pre>
    <pre>a:
    lock add dword ptr [rdi], 10
    ret</pre>
  </div>
</div>

这段代码的 load 和 fetch_add 操作仍然导致和之前相同的汇编，单 store 操作的汇编代码完全改变了。xor 指令看起来有点突兀，但这仅是通过自己异或将 eax 寄存器设置为 0 的常见方式，异或结果总是 0。`mov eax, 0` 指令将也达到同样的效果，但是需要更多的空间。

有趣的部分是 xchg 指令，它通常用于 swap 操作：一个同时检索旧值的 store 操作。

对于 SeqCst store，像之前常规的 mov 指令不能满足要求，因为它将允许稍后的 load 操作重新排序，打破全局一致性排序。通过将其改为也执行 load 的操作，即使我们不关心它加载的值，我们也可以获得额外的保证，即我们的指令不会与后续的内存操作重排序，从而解决了问题。

> SeqCst load 操作可以仍然是一个常规的 mov 指令，这正是因为 SeqCst store 被升级到 xchg。SeqCst 操作仅保证和其他 SeqCst 操作有全局一致性排序。SeqCst load 的 mov 可能仍然与前面的非 SeqCst store 操作的 mov 进行重排序，但这完全没有问题。

在 x86-64 上，store 操作是唯一一个在 SeqCst 和较弱的内存排序之间存在差异的原子操作。换句话说，除了 store 之外的 x86-64 SeqCst 操作与 Release、Acquire、AcqRel，甚至 Relaxed 操作的一样便宜。或者，如果你愿意，x86-64 使得除 store 之外的 Relaxed 操作和 SeqCst 操作一样昂贵。

### ARM64：弱排序

（<a href="https://marabos.nl/atomics/hardware.html#arm64-weakly-ordered" target="_blank">英文版本</a>）

在如 ARM64 这样的*弱排序*架构上，所有的内存操作都有可能彼此之间被重新排序。这意味着，不像 x86-64，acquire 和 release 操作不会和 relaxed 操作一样。

让我们看看在 ARM64 上对于 Release、Acquire 和 AcqRel 会发生什么：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) {
    x.store(0, Release);
}</pre>
    <pre>pub fn a(x: &AtomicI32) -> i32 {
    x.load(Acquire)
}</pre>
    <pre>pub fn a(x: &AtomicI32) {
    x.fetch_add(10, AcqRel);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    stlr wzr, [x0] #(1)
    ret</pre>
    <pre>a:
    ldar w0, [x0] #(2)
    ret</pre>
    <pre>a:
.L1:
    ldaxr w8, [x0] #(3)
    add w9, w8, #10
    stlxr w10, w9, [x0] #(4)
    cbnz w10, .L1
    ret</pre>
  </div>
</div>

与我们之前的 Relaxed 版本相比，这些改变是微妙的：

1. str（store 寄存器）现在是 stlr（store-release 寄存器）
2. ldr（load 寄存器）现在是 ldar（load-acquire 寄存器）
3. ldxr（load exclusive 寄存器）现在是 ldaxr（load-acquire exclusive 寄存器）
4. stxr（store exclusive 寄存器）现在是 stlxr（store-release exclusive 寄存器）

如上所示，ARM64 对于 acquire 和 release 排序有一个特殊的版本的 load 和 store 指令。不同于 ldr 或者 ldxr 指令，ldar 或者 ldxar 指令将从不与任意后续的内存操作重排。类似地，与 str 或者 stxr 指令不同，stlr 或 stxlr 指令将从不会和任何之前的内存操作重排。

使用仅有 Release 或 Acquire 排序的「获取并修改」操作，而非 AcqRel，将仅使用 stlxr 或 ldxar 指令分别配对一个常规的 ldxr 或 stxr 指令。

除了对 release 和 acquire 语义所需的限制外，任何特殊的 acquire 和 release 指令都永远不会与其他任何这些特殊指令重新排序，这也使它们适合用于 SeqCst。

如下面所示，升级到 SeqCst 会产生和之前完全一样的汇编代码：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a(x: &AtomicI32) {
    x.store(0, SeqCst);
}</pre>
    <pre>pub fn a(x: &AtomicI32) -> i32 {
    x.load(SeqCst)
}</pre>
    <pre>pub fn a(x: &AtomicI32) {
    x.fetch_add(10, SeqCst);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    stlr wzr, [x0]
    ret</pre>
    <pre>a:
    ldar w0, [x0]
    ret</pre>
    <pre>a:
.L1:
    ldaxr w8, [x0]
    add w9, w8, #10
    stlxr w10, w9, [x0]
    cbnz w10, .L1
    ret</pre>
  </div>
</div>

这意味着在 ARM64 上，顺序一致性操作的和 acquire 操作和 release 操作一样便宜。或者说，ARM64 的 Acquire、Release 和 AcqRel 操作和 SeqCst 一样昂贵。然而，与 x86-64 不同，Relaxed 操作相对较便宜，因为它们不会导致比必要的更强的排序保证。

<div class="box">
  <h2 style="text-align: center;">ARMv8.1 原子 Release 和 Acquire 指令</h2>

  正如我们在 [ARMv8.1 原子指令](#ARMv8.1-原子指令)中讨论的，ARM64 的 ARMv8.1 版本包括 CISC 风格的原子操作指令，如 ldadd（load 和 add）作为 ldxr/stxr 循环的替代。

  就像 load 和 store 操作带有 acquire 和 release 语义的特殊版本一样，这些指令也有对于更强内存排序的变体。因为这些指令既涉及到加载又涉及到存储，它们每一个都有三个额外的变体：一个用于 release（<code>-l</code>），一个用于 acquire（<code>-a</code>），和一个用于组合的 release 和 acquire（<code>-al</code>）语义。

  例如，对于 ldadd，还有 ldaddl、ldadda 和 ldaddal。类似地，cas 指令带有 casl、casa 和 casal 变体。

  就像 load 和 store 指令一样，组合的 release 和 acquire（-al）变体也足以用于 SeqCst 操作。
</div>

### 一个实验

（<a href="https://marabos.nl/atomics/hardware.html#reordering-experiment" target="_blank">英文版本</a>）

由强排序架构的普遍性带来的不幸后果是，某些类型的内存排序 bug 可能很容易被忽视。在需要 Acquire 或 Release 的地方使用 Relaxed 是不正确的，但在 x86-64 上，假设编译器没有重新排序你的原子操作，这可能最终在实践中偶然工作得很好。

> 请记住，不仅处理器可以导致事情无序发生。只要考虑到内存排序的约束，编译器也被允许重新排序它产生的指令。
>
> 实际上，编译器在涉及原子操作的优化上往往非常保守，但这在未来可能会发生改变。

这意味着人们可以轻易地编写不正确的并发代码，在 x86-64 上（意外地）运行得很好，但当在 ARM64 处理器编译和运行时可能会崩溃。

让我们试着做到这一点。

我们将创建一个自旋锁保护的计数器，但将所有的内存排序改为 Relaxed。让我们不费心创建自定义类型或者不安全的代码。相反，让我们仅使用 AtomicBool 作为锁和 AtomicUsize 作为计数器。

为确保编译器不会重新排序我们操作，我们将使用 `std::sync::atomic::compiler_fence()` 函数来通知编译器哪些操作应该是 Acquire 或 Release 的，但不告诉处理器。

我们将让四个线程反复锁定、增加 counter 和解锁——每个线程一百万次。把这些都放在一起，我们得到了以下代码：

```rust
fn main() {
    let locked = AtomicBool::new(false);
    let counter = AtomicUsize::new(0);

    thread::scope(|s| {
        // 产生 4 个线程，每个都迭代 100 万次
        for _ in 0..4 {
            s.spawn(|| for _ in 0..1_000_000 {
                // 使用错误的内存排序获取锁
                while locked.swap(true, Relaxed) {}
                compiler_fence(Acquire);

                // 持有锁的同时，非原子地增加 counter
                let old = counter.load(Relaxed);
                let new = old + 1;
                counter.store(new, Relaxed);

                // 使用错误的内存排序释放锁
                compiler_fence(Release);
                locked.store(false, Relaxed);
            });
        }
    });

    println!("{}", counter.into_inner());
}
```

如果锁工作正常，我们预期 counter 的最终值应该恰好是四百万。注意，增加 counter 的方式是非原子的，用的是单独的 load 和 store 操作，而不是单个 fetch_add 操作。这样做是确保自旋锁如果存在任何问题，可能会导致部分递增操作没有正确计入，从而使 counter 的总值降低。

在配备 x86-64 处理器的计算机上运行此程序几次：

```txt
4000000
4000000
4000000
```

不出所料，我们获得了“免费”的 Release 和 Acquire 语义，我们的错误不会造成任何问题。

在 2021 年的安卓手机和 Raspberry Pi 3 model B 上尝试这个，两者都使用 ARM64 处理器，结果是相同的输出：

```txt
4000000
4000000
4000000
```

这表明并非所有 ARM64 处理器都使用所有形式的指令重新排序，尽管我们不能根据这个实验假设太多。

在尝试使用 2021 款苹果的 iMac 时，它包含一个基于 ARM64 的 M1 处理器，我们得到了不同的结果：

```rust
3988255
3982153
3984205
```

我们之前隐藏的错误突然变成了一个实际问题——这个问题只在弱排序系统上可见。计数器仅仅偏离了大约 0.4%，这显示了这样的问题可能会有多么微妙。在现实生活的场景中，像这样的问题可能会长时间地保持未被发现。

> 当试图复现上述结果时，不要忘记启用优化（使用 `cargo run --release` 或 `rustc -O`）。如果没有优化，同样的代码通常会产生更多的指令，这可能会掩盖指令重排序的微妙影响。

### 内存屏障

（<a href="https://marabos.nl/atomics/hardware.html#memory-fences" target="_blank">英文版本</a>）

我们还有一种与内存排序相关的指令尚未看到：内存屏障。*内存屏障*（fence）或*内存屏障*（barrier）指令用于表示我们在[第三章的“屏障”](./3_Memory_Ordering.md#屏障fence2)部分讨论过的 `std::sync::atomic::fence`。

正如我们之前看到的，x86-64 和 ARM64 的内存排序都关乎指令重排的。屏障指令防止某些类型的指令被重排。

acquire 屏障必须防止之前的 load 操作与任何后续的内存操作进行重排序。同样，release 屏障必须防止后续的 store 操作与任何之前的内存操作进行重排序。顺序一致的屏障必须防止所有在其之前的内存操作与屏障之后的内存操作进行重排序。

在 x86-64 上，基本的内存排序语义已经满足了 acquire 和 release 屏障的需要。这是因为，该架构不允许发生这些屏障试图阻止的指令重排。

让我们深入了解一下四种不同屏障在 x86-64 和 ARM64 上编译为什么指令：

<div style="columns: 3;column-gap: 20px;column-rule-color: green;column-rule-style: solid;">
  <div style="break-inside: avoid">
    Rust 源码
    <pre>pub fn a() {
    fence(Acquire);
}</pre>
    <pre>pub fn a() {
    fence(Release);
}</pre>
    <pre>pub fn a() {
    fence(AcqRel);
}</pre>
    <pre>pub fn a() {
    fence(SeqCst);
}</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 x86-64
    <pre>a:
    ret</pre>
    <pre>a:
    ret</pre>
    <pre>a:
    ret</pre>
    <pre>a:
    mfence
    ret</pre>
  </div>
  <div style="break-inside: avoid">
    编译的 ARM64
    <pre>a:
    dmb ishld
    ret</pre>
    <pre>a:
    dmb ish
    ret</pre>
    <pre>a:
    dmb ish
    ret</pre>
    <pre>a:
    dmb ish
    ret</pre>
  </div>
</div>

不用惊讶，x86-64 上的 acquire 和 release 屏障不会生成任何指令。在这种架构上，我们可以“免费”获得 release 和 acquire 的语义。只有 SeqCst 屏障会导致生成 mfence（内存屏障）指令。这个指令确保在继续之前，所有的内存操作都已经完成。

在 ARM64 上，等效的指令是 `dmb ish`（data memory barrier, inner shared domain）。与 x86-64 不同，它也被用于 Release 和 AcqRel，因为这种架构不会隐式地提供 Acquire 和 Release 的语义。对于 Acquire，使用了一种影响稍微小一点的变体：`dmb ishld`。这种变体只等待 load 操作完成，但是允许先前的 store 操作自由地重新排序到它之后。

这与我们之前看到的原子操作类似，我们看到 x86-64 为我们“免费”提供了 Release 和 Acquire 的屏障，而在 ARM64 上，顺序一致的屏障的成本与 Release 屏障相同。

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
* 在 ARM64 上，acquire 和 release 语义不如 relaxed 操作便宜，但它们在没有额外成本的情况下也包括顺序一致（SeqCst）语义。

我们在本章节可以看见的汇编指令的总结可以在图 7-1 找到。

![ ](./picture/raal_0701.svg)

图 7-1。各种原子操作在 ARM64 和 x86-64 上编译为每个内存排序的指令概述。

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./8_Operating_System_Primitives.html">下一篇，第八章：操作系统原语</a>
</p>

[^1]: <https://zh.wikipedia.org/wiki/伪共享>
[^2]: <https://zh.wikipedia.org/zh-cn/偽陽性和偽陰性>
[^3]: <https://zh.wikipedia.org/wiki/缓存>
[^4]: <https://zh.wikipedia.org/wiki/MESI协议>
[^5]: <https://en.wikipedia.org/wiki/MESI_protocol#Store_Buffer>
[^6]: <https://en.wikipedia.org/wiki/MESI_protocol#Invalidate_Queues>
[^7]: <https://zh.wikipedia.org/wiki/流水线_(计算机)>
