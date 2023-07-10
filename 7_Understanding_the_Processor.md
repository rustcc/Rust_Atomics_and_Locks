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

或许令人惊讶的是，它的汇编与非原子版本完全相同。事实证明，mov 和 str 指令已经是原子的。它们要么发生，要么它们完全不会发生。显然，在 `&mut i32` 和 `&i32` 之间的任何差异仅对编译器检查和优化有关，但对于处理器是没有意外的——至少对于这两种架构上的 relaxed 的存储操作。

当我们查看 relaxed 的存储操作时，也是相同的：

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

尽管处理器没有明显地区别原子和非原子的存储和加载操作之间的不同，但是在我们的 Rust 代码中不能安全地忽视这些区别。如果我们使用一个 `&mut i32`，Rust 编译器可能假设没有其他线程可以并发地访问相同的 i32 类型，并且可能决定以某种方式去转换或者优化代码，使得 store 操作不再导致单个相应的 store 指令。例如，通过使用两个单独的 16 位指令进行非原子的 32 位 load 或 store 操作时非常正确的，尽管有些不寻常。

### 读、修改、写操作

（<a href="https://marabos.nl/atomics/hardware.html#rmw-ops" target="_blank">英文版本</a>）

#### x86 锁前缀

（<a href="https://marabos.nl/atomics/hardware.html#lock-prefix" target="_blank">英文版本</a>）

#### x86 比较和交换指令

（<a href="https://marabos.nl/atomics/hardware.html#x86-cas" target="_blank">英文版本</a>）

### LL 和 SC 指令

（<a href="https://marabos.nl/atomics/hardware.html#load-linked-and-store-conditional-instructions" target="_blank">英文版本</a>）

#### ARM 的 ldxr 和 stxr 指令

（<a href="https://marabos.nl/atomics/hardware.html#arm-load-exclusive-and-store-exclusive" target="_blank">英文版本</a>）

#### ARM 的「比较和交换操作」

（<a href="https://marabos.nl/atomics/hardware.html#compare-and-exchange-on-arm" target="_blank">英文版本</a>）

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

<p style="text-align: center; padding-block-start: 5rem;">
  <a href="./8_Operating_System_Primitives.html">下一篇，第八章：操作系统原语</a>
</p>
