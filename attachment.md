## 索引

### A

* AArch64（参见 ARM64）
* ABA 问题，[#](./2_Atomics.md#index-ABAproblem)
* 终止进程，[#](./2_Atomics.md#index-abortingtheprocess)
* AcqRel，[#](./3_Memory_Ordering.md#index-AcqRel)
  * （参见 release 和 acquire 内存排序）
* acquire 内存排序（参见 release 和 acquire 内存排序）
* add 指令（ARM），[#](./7_Understanding_the_Processor.md#index-addinstructionARM)
* add 指令（x86），[#](./7_Understanding_the_Processor.md#index-addinstructionx86)
* 基于地址的等待（Windows），[#](./8_Operating_System_Primitives.md#基于地址的等待)
  * （参见 futex）
* 凭空出现的值，[#](./3_Memory_Ordering.md#凭空出现的值)
* alignment，[#](./7_Understanding_the_Processor.md#index-alignment)
* 分配（参见 ID 分配）
* AMD 处理器，[#](./7_Understanding_the_Processor.md#index-AMDprocessors)
* and 指令（x86），[#](./7_Understanding_the_Processor.md#index-x86-64instructions-and)
* Arc，[#](./1_Basic_of_Rust_Concurrency.md#index-Arc)
  * 构建我们自己的 Arc，[#](./6_Building_Our_Own_Arc.md#index-buildingourown-Arc)
  * 循环结构，[#](./6_Building_Our_Own_Arc.md#index-Arc-cyclicstructures)
  * get_mut，[#](./6_Building_Our_Own_Arc.md#index-Arc-getmut)
  * 内存排序，[#](./6_Building_Our_Own_Arc.md#index-Arc-memoryordering)，[#](./6_Building_Our_Own_Arc.md#index-happens-beforerelationships-inArc-2)，[#](./6_Building_Our_Own_Arc.md#index-Arc-memoryordering-3)，[#](./6_Building_Our_Own_Arc.md#index-Arc-memoryordering-4)
  * 命名克隆，[#](./1_Basic_of_Rust_Concurrency.md#命名克隆)
  * 用于 Channel 的情况，[#](./5_Building_Our_Own_Channels.md#index-Arc-usingforchannelstate)
  * weak 指针，[#](./6_Building_Our_Own_Arc.md#weak-指针)
    * 性能开销，[#](./6_Building_Our_Own_Arc.md#index-Arc-weakpointers-performancecost)
* arguments，consuming，[#](./5_Building_Our_Own_Channels.md#index-argumentsconsuming)
* ARM64（处理器架构），[#](./7_Understanding_the_Processor.md#index-ARM64processorarchitecture)
  * aarch64-unknown-linux-musl target，[#](./7_Understanding_the_Processor.md#index-ARM64processorarchitecture-aarch64-unknown-linux-musltarget)
  * other-multi-copy atomic，[#](./7_Understanding_the_Processor.md#index-ARM64processorarchitecture-other-multi-copyatomic)
  * weakly ordered，[#](./7_Understanding_the_Processor.md#arm64弱排序)
* ARM64 指令
  * add，#
  * ARMv8.1 atomic instructions，#，#
  * b.ne (branch if not equal)，#
  * cbnz (compare and branch on nonzero)，#
  * clrex (clear exclusive)，#
  * cmp (compare)，#
  * dmb (data memory barrier)，#
  * ldar (load-acquire register)，#
  * ldaxr (load-acquire exclusive register)，#
  * ldr (load register)，#
  * ldxr (load exclusive register)，#
  * load-linked and store-conditional instructions，#
  * mov (move)，#
  * overview，#
  * ret (return)，#
  * stlr (store-release register)，#
  * stlxr (store-release exclusive register)，#
  * str (store register)，#
  * stxr (store exclusive register)，#
* ARMv8 (see ARM64)
* ARMv8.1 atomic instructions，#，#
  * overview，#
* array::from_fn，#
* assembler，#
* assembly，#
  * inspecting compiler output，#
* atomic，#，#
  * compare-and-exchange operations，#
    * weak，#，#，#，#，#，#，#，#，#，#，#，#
  * fetch-and-modify operations，#
    * wrapping behavior (add and sub)，#，#，#，#
  * load and store operations，#
    * example，stop flag，#，#，#，#，#
  * memory ordering (see memory ordering)
  * reference counting (see Arc)
* atomic barriers (see fences)
* atomic fences (see fences)
* atomic types，#，#
  * compare_exchange，#
  * compare_exchange_weak，#
  * fetch_add，#
    * wrapping behavior，#
    * (see also overflows)
  * fetch_and，#
  * fetch_max，#
  * fetch_min，#
  * fetch_nand，#
  * fetch_or，#
  * fetch_store (see swap)
  * fetch_sub，#
    * wrapping behavior，#
    * (see also overflows)
  * fetch_update，#
  * fetch_xor，#
  * get_mut，#
  * load，#
  * store，#
  * swap，#
* atomic-wait crate，#
* AtomicBool，#
  * (see also atomic types)
  * locking using，#，#
* AtomicI8 (see atomic types)
* AtomicI16 (see atomic types)
* AtomicI32 (see atomic types)
* AtomicI64 (see atomic types)
* AtomicIsize (see atomic types)
* AtomicPtr，#
  * (see also atomic types)
  * compare-and-exchange，#
  * lazy initialization，#
* AtomicU8 (see atomic types)
* AtomicU16 (see atomic types)
* AtomicU32 (see atomic types)
* AtomicU64 (see atomic types)
* AtomicUsize (see atomic types)
* auto traits，#

### B

* b.ne (branch if not equal) instruction (ARM)，#
* barriers (see fences)
* basics，#
* benchmarking，#，#
  * black_box，avoiding optimizations with，#，#
* binary semaphore，#
* black_box，#，#
* blocking，#
  * channel，#
  * condition variables，#
  * futex wait operation，#
  * (see also futex)
  * mutexes，#
  * Once and OnceLock，#，#
  * semaphores，#
  * spin loop，#
  * thread parking (see thread parking)
* boolean (atomic) (see AtomicBool)
* borrowing，#
  * bending the rules，#
  * error，#
  * exclusive，#
  * from multiple threads (Sync)，#
  * immutable，#
  * (see also shared)
  * local variables in a thread，#
  * mutable，#
  * (see also exclusive)
  * shared，#
  * splitting，#
  * undefined behavior，#
* Box
  * from_raw，#，#
  * into_raw，#
  * leak，#，#
  * unmovable type，wrapping in，#
* btc (bit test and complement) instruction (x86)，#
* btr (bit test and reset) instruction (x86)，#
* bts (bit test and set) instruction (x86)，#
* building our own
  * Arc，#
  * channels，#
  * condition variables，#
  * mutexes，#
  * reader-writer locks，#
  * spin locks，#
* busy-looping，#
  * (see also spinning)

### C

* C standard library，#
  * (see also libc)
* cache coherence，#
  * protocol，#
    * write-through，#，#，#，#
* cache lines，#
  * performance experiment，#
* cache miss，#
* caching (processors)，#
  * (see also cache coherence)
  * compare-and-exchange operations，effect of，#
  * per core，#
  * performance experiment，#
* cargo-show-asm，#
* cas (compare and swap) instruction (ARM)，#
* casa (compare and swap，acquire) instruction (ARM)，#
* casal (compare and swap，acquire and release) instruction (ARM)，#
* casl (compare and swap，release) instruction (ARM)，#
* cbnz (compare and branch on nonzero) instruction (ARM)，#
* Cell，#
  * unsafe (see UnsafeCell)
* channels
  * blocking，#
  * borrowing，#
  * building our own，#
  * dropping，#
  * one-shot，#
  * safe interface，#
  * Sender and Receiver types，#，#
  * storing in Arc，#
    * avoiding，#
  * unsafe interface，#
* Clone trait，#，#，#，#，#
* closures
  * captured values
    * moving，#，#
  * spawning scoped threads using，#
  * spawning threads using，#
* clrex (clear exclusive) instruction (ARM)，#
* cmp (compare) instruction (ARM)，#
* cmpxchg (compare and exchange) instruction (x86)，#
* `#[cold]`，#
* compare-and-exchange operations (atomic)，#
  * on ARM64，#
  * caching，effect on，#
  * compiler optimization，#
  * example，ID allocation，#
  * example，lazy initialization，#，#
  * memory ordering，#
  * using for channel state，#
  * using for mutex state，#
  * using for reader-writer lock state，#
  * using on AtomicPtr，#
  * using to lock reference counter，#
  * weak，#
    * on ARM64，#
  * on x86-64，#
* Compiler Explorer，#
* compiler fence，#，#
* compiler optimization
  * black_box，avoiding with，#，#
  * `#[cold]`，#
  * of compare-and-exchange loops，#
  * enabling，#，#
  * `#[inline]` #
  * reordering，#
* complex instruction set computer (CISC)，#
* concurrency，basics，#
* condition variables，#
  * building our own，#
  * example，#
  * memory ordering，#
  * pthread_cond_t，#
  * thundering herd problem，#
  * timeout，#
  * using to build a channel，#
  * Windows，#
* Condvar，#
  * (see also condition variables)
* consume memory ordering，#
* consuming arguments by value，#
* contention (mutexes)，#，#
  * benchmarking，#
* Copy trait，#，#
  * atomic types，not implementing，#
  * moving，#
* critical section (Windows)，#
* current thread，#，#，#
* cyclic structures (Arc)，#

### D

* data races，#
  * avoiding using atomics，#，#
* Deref trait，#，#，#
* DerefMut trait，#，#，#，#
* disassembler，#，#
* dmb (data memory barrier) instruction (ARM)，#
* drop function，#，#，#
* Drop trait，#，#，#，#，#，#，#，#，#，#，#，#
* dword，#

### E

* --emit=asm (rustc)，#
* exclusive references，#

### F

* fair locks，#
* false sharing，#
* fences，#，#，#
  * on ARM64，#
  * compiler fence，#，#
  * instructions，#
  * process-wide memory barriers，#
  * on x86-64，#
* fetch-and-modify operations (atomic)，#
  * on ARM64，#
  * example，ID allocation，#
  * example，progress reporting，#
  * example，statistics，#
  * wrapping behavior (add and sub)，#
    * (see also overflows)
  * on x86-64，#
* fetch_store operation (atomic) (see swap operation)
* fetch_update (atomic)，#
* FlushProcessWriteBuffers (Windows)，#
* forgetting (see leaking)
* FreeBSD，umtx_op syscall，#
  * (see also futex)
* from_fn (array)，#
* futex，#
  * cross-platform futex-like functionality，#
  * example，#
  * memory safety，#
  * on other platforms，#
  * operations (Linux)，#
    * FUTEX_WAIT，#，#，#，#，#，#，#，#，#，#，#
  * requeuing，#，#
  * spurious wake-ups，#
  * timeout，#，#
  * wait operation，#
  * wake operation，#

### G

* globally consistent order，#
  * (see also sequentially consistent memory ordering)
* Godbolt，#
* good luck，#
* guards
  * dropping，#
  * join guard，#
  * mutex guard，#，#
  * read guard，#，#
  * spin lock guard，#
  * write guard，#，#

### H

* hand，things getting out of，#
* happens-before relationships，#
  * in Arc，#，#
  * between threads，#
  * locking and unlocking，#，#
  * spawning and joining threads，#
  * through a release-acquire pair，#，#
    * chaining，#
  * within the same thread，#
* hint::black_box，#，#
* hint::spin_loop，#

### I

* ID allocation
  * using compare_exchange_weak，#
  * using fetch_add，#
  * using fetch_update，#
* ideas and inspiration，#
* if let statement
  * lifetime of temporaries，#
* ignorance，blissful，#
* immutable references，#
  * (see also shared references)
* indivisible，#
* `#[inline]`，#
* inspiration，#
* Instant，#
* instruction reordering (see reordering)
* instructions，#
  * (see also ARM64 instructions; x86-64 instructions)
  * compare-and-exchange operations，#，#
  * fences，#
  * load and store operations，#
  * load-linked/store-conditional (LL/SC) instructions，#
  * memory ordering，#
  * overview，#
  * read-modify-write operations，#
* Intel processors，#
* interior mutability，#，#，#
* invalidation queues，#

### J

* jne (jump if not equal) instruction (x86)，#
* join method，#
* JoinGuard，#
* JoinHandle，#
* joining threads，#
  * happens-before relationship，#

### K

* kernel，#，#
  * interfacing with，#
  * kernel-managed objects (Windows)，#

### L

* L1/L2/L3/L4 cache，#
* label (assembly)，#
* lazy initialization
  * using compare_exchange，#
  * using compare_exchange and allocation，#
  * using load and store，#
* ldadd (load and add) instruction (ARM)，#
* ldadda (load and add，acquire) instruction (ARM)，#
* ldaddal (load and add，acquire and release) instruction (ARM)，#
* ldaddl (load and add，release) instruction (ARM)，#
* ldar (load-acquire register) instruction (ARM)，#
* ldaxr (load-acquire exclusive register) instruction (ARM)，#
* ldr (load register) instruction (ARM)，#
* ldxr (load exclusive register) instruction (ARM)，#
* leaking，#，#
  * by mistake，#
  * a MutexGuard，#
* “Leakpocalypse”，#
* libc，#
  * pthreads functionality in，#
* libpthread，#
  * (see also pthreads)
* lifetime
  * elision，#，#
  * in a struct，#
  * of mutex guard，#
  * specifying using plain English，#
  * static，#
* linked list，#
  * Linux
    * futex syscall，#
    * (see also futex)
  * arguments，#
  * futex_waitv syscall，#
  * interfacing with the kernel，#
  * libc，role of，#
  * membarrier syscall，#
  * process-wide memory barrier，#
  * RCU，#
* load and store operations (atomic)，#
  * on ARM64 and x86-64，#
  * compared to non-atomic operations，#，#
  * example，lazy initialization，#
  * example，progress reporting，#
  * example，stop flag，#
* load-linked/store-conditional (LL/SC) loop，#
  * on ARM64，#
  * compiler optimization，#
* lock poisoning，#
* lock prefix (x86)，#
* lock_api crate，#
* luck，good，#

### M

* machine code，#
* machine instructions (see instructions)
* macOS
  * futex-like functionality on，#
  * interfacing with the kernel，#，#
  * os_unfair_lock，#
* main thread，#
* ManuallyDrop，#
* MaybeUninit，#，#，#
* mem::forget，#
* membarrier syscall，#
* memory barriers (see fences)
* memory fences (see fences)
* memory model，#
* memory ordering，#，#
  * on ARM64，#
  * compiler fence，#，#
  * consume，#
  * experiment，using relaxed instead of release and acquire，#
  * fences，#，#，#
  * happens-before relationship，#，#
  * Miri，detecting problems with，#
  * misconceptons about，#
  * out-of-thin-air values，#
  * at processor level，#
  * reference counting，#，#，#，#，#
  * relaxed，#，#
  * release and acquire，#
    * (see also release and acquire memory ordering)
    * locking and unlocking，#
  * sequentially consistent，#
    * (see also sequentially consistent memory ordering)
  * specifying using plain English，#
  * total modification order，#，#，#，#
  * on x86-64，#
* MESI cache coherence protocol，#
* MESIF cache coherence protocol，#
* mfence (memory fence) instruction (x86)，#
* microinstructions，#
* Miri，#
* MOESI cache coherence protocol，#
* mov (move) instruction (ARM)，#
* mov (move) instruction (x86)，#
* movable，not
  * critical section (Windows)，#
  * Pin，#
  * pthread types，#
  * wrapping in Box，#
* move closure，#
* multi-copy atomicity，#
* mutability，interior (see interior mutability)
* mutable references，#
  * (see also exclusive references)
* Mutex，#，#
  * (see also mutexes)
* mutexes，#
  * building our own，#
  * as container，#
  * contention，#，#
  * example，#
  * fair，#
  * happens-before relationship，#
  * into_inner，#
  * lifetime of mutex guard，#
  * memory ordering，#
  * Mutex type in Rust，#
  * os_unfair_lock (macOS)，#
  * in other languages，#
  * poisoning，#
  * pthread
    * wrapping in Rust，#
  * pthread_mutex_t，#
  * recursive，#
  * robust，#
  * Send requirement，#
  * spin locks，#
    * (see also spin locks)
  * spinning，#，#
  * using to build a channel，#
* MutexGuard，#
  * dropping，#
  * lifetime of，#
* mutual exclusion (see mutexes)

### N

* name of a thread，#
* NetBSD，futex support，#
  * (see also futex)
* NonNull，#

### O

* -O flag (rustc)，#，#
* Once and OnceLock，#，#
* one-shot channels，#
* OpenBSD，limited futex support，#
  * (see also futex)
* operating systems，#
  * (see also Linux; macOS; Windows)
  * libraries shipped with，#
  * synchronization primitives，#
* optimization (see compiler optimization)
* or instruction (x86)，#，#
* Ordering，#，#
  * AcqRel，#
    * (see also release and acquire memory ordering)
  * Acquire，#
    * (see also release and acquire memory ordering)
  * Consume，#
  * Relaxed，#，#
  * Release，#
    * (see also release and acquire memory ordering)
  * SeqCst，#
    * (see also sequentially consistent memory ordering)
* os_unfair_lock (macOS)，#
* other-multi-copy atomicity，#
* out of order execution (see reordering)
* out-of-thin-air values，#
* output locking，#
* overflows (atomic)，#
  * (see also wrapping behavior)
  * aborting on，#
  * notification counter，#
  * panicking on，#
  * preventing (compare-and-exchange)，#
  * reference counter，#
  * usize，big enough，#
* overview of atomic instructions，#
* ownership
  * moving，#，#
  * sharing，#
  * transferring to another thread (Send)，#

### P

* panicking
  * poisoned mutexes，#
  * RefCell，borrowing，#
  * thread name in panic messages，#
  * using a Condvar with multiple mutexes，#
  * when joining a thread，#，#
  * when spawning a thread，#
* parking (see thread parking)
* parking lot-based locks，#
* parking_lot crate，#
* PhantomData，#，#
* Pin，#
* pipelining，#
* pointers
  * atomic (see AtomicPtr)
  * neither Send nor Sync，#
  * NonNull，#
* poisoning，lock，#
* POSIX，#
  * pthreads，#
* println，use of output locking，#
* priority inheritance，#
* priority inversion，#
* privacy (modules)，#
* process-wide memory barriers，#
* processes，#
* processor architecture，#
  * (see also ARM64; x86-64)
  * strongly ordered，#
  * weakly ordered，#
* processor caching (see caching)
* processor instructions (see instructions)
* processor registers，#
  * return value，#
* pthreads，#
  * pthread_cond_t，#，#
  * pthread_mutex_t，#
  * dropping while locked，#
  * pthread_rwlock_t，#
  * wrapping in Rust，#

### Q

* queue-based locks，#

### R

* racing，#
* Rc，#
* RCU (read，copy，update)，#，#
* reader-writer locks，#
  * avoiding accidental spinning，#
  * building our own，#
  * pthread_rwlock_t，#
  * Send requirement，#
  * SRW locks (Windows)，#
  * Sync requirement，#
  * writer starvation，#，#
* recursive locking，#，#
* reduced instruction set computer (RISC)，#
* RefCell，#
  * RwLock compared to，#
* reference counting，#
  * (see also Arc)
* references
  * exclusive，#
  * immutable，#
    * (see also shared)
  * mutable，#
    * (see also exclusive)
  * shared，#
* registers，#
  * return value，#
* relaxed memory ordering，#，#，#
  * counter-intuitive results，#
  * misconceptions about，#，#
  * out-of-thin-air values，#
  * reference counting，#
  * total modification order，#，#，#，#
* release and acquire memory ordering，#
  * acquire fence，#，#，#，#，#
  * on ARM64，#
  * example，lazy initialization，#
  * experiment，using relaxed instead，#
  * happens-before relationship，#，#
    * chaining，#
  * locking and unlocking，#，#
  * reference counting，#，#，#，#
  * release fence，#
  * on x86-64，#
* --release flag (cargo)，#，#
* reordering (instructions)，#，#
  * memory ordering，#
* `#[repr(align)]`，#
* requeuing waiting threads，#，#
* ret (return) instruction (ARM)，#
* ret (return) instruction (x86)，#
* robust mutexes，#
* rustup，#
* RwLock，#，#
  * (see also reader-writer locks)
* RwLockReadGuard，#
* RwLockWriteGuard，#

### S

* safe interface，#，#，#
* safety requirements of unsafe functions，#
* scheduler，#
* scoped threads，#
* semaphores，#
* Send trait，#，#，#
  * error，#
  * implementing for Arc，#
  * requirement by Mutex and RwLock，#
* SeqCst (see sequentially consistent memory ordering)
* sequence locks，#
* sequentially consistent memory ordering，#
  * on ARM64，#
  * fence，#
  * misconceptions about，#，#
  * on x86-64，#
* shadowing，#
* shared ownership，#
  * leaking，#
  * reference counting，#
  * statics，#
* shared references，#
  * mutating atomics through，#
* slim reader-writer locks (Windows)，#，#
* spawning threads，#
  * failing to，#
  * happens-before relationship，#
  * scoped，#
* spin locks
  * building our own，#
  * cache lines，effect of，#
  * compare-and-exchange，(not) using，#
  * experiment，using wrong memory ordering，#
  * guard，#
  * memory ordering，#
* spin loop hint，#，#
* spinning，#，#，#
  * avoiding accidental (reader-writer lock)，#
* splitting (borrowing)，#
* spurious wake-ups，#，#，#
* SRW locks (Windows)，#，#
* stack size，#
* starvation，#，#
* static lifetime，#
* statics，#
* stlr (store-release register) instruction (ARM)，#
* stlxr (store-release exclusive register) instruction (ARM)，#
* stop flag，#
* store buffers，#
* store operations (atomic) (see load and store operations)
* store-conditional (see load-linked/store-conditional)
* str (store register) instruction (ARM)，#
* stress，reducing，#
* strongly ordered architecture，#
* stxr (store exclusive register) instruction (ARM)，#
* sub (subtract) instruction (x86)，#
* swap operation (atomic)，#
  * locking using，#
* Sync trait，#，#
  * implementing for Arc，#
  * implementing for channel，#
  * implementing for mutex，#
  * implementing for reader-writer lock，#
  * implementing for spin lock，#
  * requirement by RwLock，#
* SYS_futex (Linux)，#
  * (see also futex)
  * arguments，#
* syscalls，#
  * avoiding，#，#

### T

* --target (rustc)，#
* teaching，#
* thin air，out of，#
* thread builder，#
* thread name，#
* Thread object，#
  * id，#
  * unpark，#，#
* thread parking，#，#，#，#
  * spurious wake-ups，#
* timeout，#
  * example，#
* thread safety，#，#
  * keeping objects on one thread，#
* ThreadId，#
* threads，#
  * joining，#
  * panicking，#，#
  * returning a value，#
  * scoped，#
  * spawning，#
* thundering herd problem，#
* time travel，#
* timeout
  * condition variables，#
  * futex，#，#
* thread parking，#
  * example，#
* total modification order，#，#，#，#

### U

* uncontended (mutexes)，#，#
  * benchmarking，#
* undefined behavior，#
  * borrowing，#
  * data races，#
  * Miri，detecting with，#
  * time travel，#
* uninitialized memory，#
* Unix systems
  * interfacing with the kernel，#
  * libc，role of，#
* unmovable
  * critical section (Windows)，#
  * Pin，#
  * pthread types，#
  * wrapping in Box，#
* unpark (Thread)，#
* unparking (see thread parking)
* unsafe code，#
* unsafe functions，#
* unsafe trait implementation，#
* UnsafeCell，#，#，#
  * get_mut，#
* unsound，#

### V

* VecDeque，#

### W

* waiting (see blocking)
* WaitOnAddress (Windows)，#
* WakeByAddressAll (Windows)，#
* WakeByAddressSingle (Windows)，#
* Weak (see Arc; weak pointers)
* weakly ordered architecture，#
  * experiment，using relaxed instead of release and acquire，#
* Windows，#
  * condition variables，#
  * critical section，#
  * FlushProcessWriteBuffers，#
  * interfacing with the kernel，#
  * kernel-managed objects，#
  * Native API，#
  * process-wide memory barrier，#
  * SRW locks，#，#
* WaitOnAddress，#
* WakeByAddressAll，#
* WakeByAddressSingle，#
* Win32 API，#
* windows crate，#
* windows-sys crate，#
* wrapping behavior (fetch_add and fetch_sub)，#
  * (see also overflows (atomic))
* wrapping unmovable object in Box，#
* write-through cache coherence protocol，#
* writer starvation，#，#

### X

* x86-64 (processor architecture)，#
  * other-multi-copy atomic，#
  * strongly ordered，#
  * x86_64-unknown-linux-musl target，#
* x86-64 instructions
  * add，#
  * and，#
  * btc (bit test and complement)，#
  * btr (bit test and reset)，#
  * bts (bit test and set)，#
  * cmpxchg (compare and exchange)，#
  * jne (jump if not equal)，#
  * lock prefix，#
  * mfence (memory fence)，#
  * mov (move)，#
  * or，#，#
  * overview，#
  * ret (return)，#
  * sub (subtract)，#
  * xadd (exchange and add)，#
  * xchg (exchange)，#，#
  * xor，#
* xadd (exchange and add) instruction (x86)，#
* xchg (exchange) instruction (x86)，#，#
* xor instruction (x86)，#

## 译注

| 英文                 | 中译           | 可能出现章节        |
| -------------------- | -------------- | ------------------- |
| allocation           | 内存分配       | 1、3、5、6、8、10   |
| atomic               | 原子           | all                 |
| benchmark            | 基准测试       | 4、9                |
| borrow               | 借用           | 1、4、5             |
| building block       | 基石           | 1、2、8、10         |
| cache coherence      | 缓存一致性     | 7                   |
| compare and exchange | 比较并交换     | 3、4、6、9          |
| condition variable   | 条件变量       | 1、2、5、8、9       |
| drop                 | 丢弃           | 1、3、4、5、6、9    |
| fetch and modify     | 获取并修改     | 1、3                |
| fence                | 屏障           | 3、6、7、8          |
| formalize            | 形式化的       | 3                   |
| guard                | 守卫           | 4、9                |
| happens-before       | happens-before | 3、4、5、6、7、9    |
| Invalidation queue   | 失效队列       | 7                   |
| leak                 | （内存）泄漏   | 1、3、5、8、10      |
| load operation       | load 操作     | 2、3、5、6、7、8、9 |
| lock（v）            | 锁定           | all                 |
| memory ordering      | 内存排序       | 2、3、4、5、6、7、9 |
| mutex                | 互斥锁         | 1、2、3、4、8、9    |
| mutation             | 可变性         | 1、2、4、6、8       |
| notify               | 通知           | 1、4、5、8、9       |
| notify_all           | notify all       | 9                   |
| notify_one           | notify one       | 9                   |
| park/block           | 阻塞           | all                 |
| pipeline             | 流水线         | 7                   |
| reader-writer lock   | 读写锁         | 1、3、4、8、9       |
| reader               | reader         | 1、4、8、9、10      |
| receiver             | 接收者         | 5                   |
| reference            | 引用           | all                 |
| spinLock             | 自旋锁         | 3、4、8、9          |
| sender               | 发送者         | 5                   |
| spurious             | 虚假的         | 1、2、5、8、9       |
| static               | 静态值         | 1                   |
| stop the world       | 停止其他活动   | 7                   |
| store buffer         | 存储缓冲区     | 7                   |
| store operation      | store 操作     | 2、3、5、6、7、8、9 |
| swap operation       | swap 操作      | 2、3、5、6、7、8、9 |
| syscall              | 系统调用       | 8、9                |
| unlock（v）          | 解锁           | all                 |
| unpark               | 释放           | all                 |
| use cases            | 用例           | all                 |
| wait(er)             | 等待（者）     | all                 |
| writer               | writer         | 1、4、8、9、10      |
