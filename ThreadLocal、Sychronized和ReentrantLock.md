>## ThreadLocal

`ThreadLoacl`在Java中的核心作用是**提供线程内的局部变量**。它能让每个线程拥有该变量的一个独立副本，从而避免了多线程之间的竞争和干扰。
### 底层原理
#### 核心结构：数据存在哪里？
**`ThreadLocal` 本身并不存储数据**，它只是一个访问入口，真正存储数据的是 **`Thread` 线程对象本身**
- 每个 `Thread` 对象内部都有一个名为 `threadLocals` 的成员变量。
    
- 这个变量的类型是 `ThreadLocalMap`（它是 `ThreadLocal` 的静态内部类）。
    
- 当我们调用 `set()` 时，其实是把数据存到了**当前线程**的 `threadLocals` 那个 Map 里
#### 映射关系与冲突解决
在 `ThreadLocalMap` 中：
- **Key**：是当前的 `ThreadLocal` 对象实例。
    
- **Value**：是我们想要保存的业务数据。
值得注意的是，`ThreadLocalMap` 和我们常用的 `HashMap` 不同。它在解决 Hash 冲突时，没有使用链表法，而是采用了**线性探测法**。也就是说，如果计算出的槽位被占了，它会顺着数组往后找，直到找到一个空位为止。这也意味着 `ThreadLocalMap` 不适合存储大量数据。
#### 弱引用与内存泄漏
`ThreadLocalMap` 的 Entry 对 **Key（ThreadLocal 对象）使用的是弱引用**，但对 **Value 使用的是强引用**。
- **设计初衷**：如果外部对 `ThreadLocal` 对象的引用没了，GC 发生时，作为 Key 的 `ThreadLocal` 会被自动回收，变成 `null`，这样尽量避免 Key 的内存泄漏。
    
- **潜在风险**：虽然 Key 回收了，但只要线程还在运行（比如在线程池中），Value 依然有一条强引用链：`Thread -> ThreadLocalMap -> Entry -> Value`。
    
- **后果**：这会导致 `Entry` 中出现 `Key` 为 `null`，但 `Value` 还在的情况。这个 Value 既无法被访问（因为 Key 没了），也无法被 GC 回收，从而造成**内存泄漏**。
所以，我们在使用 `ThreadLocal` 时，标准范式是在 `try-finally` 代码块的 `finally` 中，**显式调用 `remove()` 方法**。这不仅是为了清理数据，更是为了切断 Value 的引用链，防止内存泄漏和线程复用导致的数据脏读。
### 应用场景
##### 保存线程上下文信息（代替参数传递）
