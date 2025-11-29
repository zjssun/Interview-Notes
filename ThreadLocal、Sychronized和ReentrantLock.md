>## ThreadLocal

`ThreadLoacl`是Java中所提供的线程本地存储机制。它能让每个线程拥有该变量的一个独立副本，从而避免了多线程之间的竞争和干扰。
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

### 应用场景
##### 保存线程上下文信息（代替参数传递）
