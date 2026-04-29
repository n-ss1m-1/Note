## 第一部分：基础组件理解

### 问题 1：EventLoop 的核心机制

**Q**：EventLoop 如何实现 "one loop per thread" 模型？wakeupFd 的作用是什么？为什么需要 wakeup 机制？

**考察点**：线程模型设计、跨线程唤醒机制、竞态条件避免

**答案**：

1. **one loop per thread 实现**

- 用 `thread_local EventLoop* t_loopInThisThread` 作为线程局部实例。
- 构造时检查：若该指针非空，说明线程已有 loop，直接报错。
- 运行时用 `isInLoopThread()` 比较创建线程 tid 与当前线程 tid，保证一致。

2. **wakeupFd 作用**

- 用于**跨线程唤醒**。
- 注册 `wakeupChannel` 到 loop 所属 Poller，关注可读事件。
- 其他线程调用 `queueInLoop()` 递交任务时，向 `wakeupFd` 写 8 字节，触发可读事件，唤醒 loop。

3. **为什么需要 wakeup 机制**

- 解决 `epoll_wait` 阻塞导致的**任务处理延迟**。
- 无 wakeup 时，loop 只能等网络事件才能处理任务队列，可能长时间阻塞。
- 保证任务队列处理**及时**，提升服务器效率。

---

### 问题 2：Channel 与 Poller 的协作

**Q**：当有事件发生时，Channel 如何被通知？整个过程涉及哪几个组件的协作？请描述从 epoll_wait 返回事件到调用用户回调的完整流程。

**考察点**：事件分发机制、回调链、线程安全性

**流程**：

1. `epoll_wait()` 返回，拿到触发事件 `revents`。
2. `EPollPoller::fillActiveChannels()` 通过 `revents[i].data.ptr` 拿到对应 Channel。
3. 调用 `channel->set_revents(revents[i].events)`，将 Channel 加入活跃队列。
4. `Poller::poll()` 返回活跃 Channel 列表给 EventLoop。
5. EventLoop 遍历活跃 Channel，调用 `channel->handleEvent()`。
6. 根据 `revents` 类型，执行预设的读 / 写 / 错误 / 关闭回调。

**涉及组件**：EventLoop、Poller、EPollPoller、Channel

---

### 问题 3：智能指针的生命周期管理

**Q**：在 TcpConnection 中，为什么使用 `shared_from_this()` 而不是传递 `this` 指针？在 `handleClose` 中，为什么要创建一个临时的 `TcpConnectionPtr`？

**考察点**：生命周期、异步安全、悬垂指针

**答案**：

1. **用 `shared_from_this()` 而非 `this`**

- `shared_from_this()` 返回 `shared_ptr`，**增加引用计数**，保证回调执行期间对象存活。
- 直接传 `this` 无引用计数，可能在回调执行时对象已被销毁，导致**悬垂指针 / 崩溃**。
- 前提：TcpConnection 必须公有继承 `enable_shared_from_this`。

2. **handleClose 中临时 `TcpConnectionPtr`**

cpp

运行

```
void handleClose() {
  TcpConnectionPtr guard(shared_from_this());
  connectionCallback_(guard);
  closeCallback_(guard);
}
```

- 作用：**延长生命周期到函数结束**，防止回调链中自销毁。
- 即使 `closeCallback_` 移除连接、减少计数，`guard` 仍持有一份引用，保证全程安全。

---

## 第二部分：高级组件设计

### 问题 4：Buffer 的设计

**Q**：Buffer 如何解决 TCP 粘包 / 半包问题？为什么使用 `vector<char>` 而不是 `string` 作为底层容器？`readableBytes()`、`writableBytes()`、`prependableBytes()` 分别表示什么？

**考察点**：缓冲区设计、内存管理、性能

**答案**：

1. **粘包 / 半包处理**

- Buffer 不直接解决，而是提供**连续读写缓冲区**，让应用层累积数据、按协议解析。
- 应用层常用：长度前缀、分隔符、固定长度、协议头字段。

1. **vector< char > 优势**

- 不自动补 `\0`，可直接用于 `read/write` 系统调用。
- 内存连续、可动态扩容、性能更优。

3. **三区域含义**

plaintext

```
[ prependable ] [     readable     ] [     writable     ]
0        readerIndex_        writerIndex_        buffer_.size()
```

- `prependableBytes()`：预留头部空间 + 已读空间。
- `readableBytes()`：已接收未读取的数据长度。
- `writableBytes()`：可直接写入的空闲空间。

---

### 问题 5：TcpConnection 的状态机

**Q**：TcpConnection 有哪几个状态？状态转换的触发条件是什么？为什么需要 `kDisconnecting` 状态？

**考察点**：状态机、连接生命周期、优雅关闭

**答案**：

1. **四个状态**

- `kDisconnected`：已断开
- `kConnecting`：连接中
- `kConnected`：已连接
- `kDisconnecting`：半关闭 / 断开中

2. **状态转换**

- `kDisconnected → kConnecting`：构造时初始化。
- `kConnecting → kConnected`：`connectEstablished()`。
- `kConnected → kDisconnecting`：调用 `shutdown()`，等待发送完毕。
- `kConnected → kDisconnected`：对端关闭（读到 0 字节）、出错。
- `kDisconnecting → kDisconnected`：发送完成，调用 `handleClose()`。

3. **kDisconnecting 作用**

- 标记**半关闭状态**，保证缓冲区数据发送完再关闭写端。
- 实现**优雅关闭**，不丢数据。

---

### 问题 6：TcpServer 的线程模型

**Q**：当新连接到达时，TcpServer 如何选择 EventLoop 来处理这个连接？这样做有什么好处？在多线程环境下，如何安全地将连接从主线程转移到子线程？

**考察点**：多线程、负载均衡、线程间通信

**答案**：

1. **选择 subloop**

- 主 Reactor（main loop）负责 accept。
- 用 `threadPool_->getNextLoop()` **轮询**选择子 IO loop。

2. **好处**

- 负载均衡，连接均匀分配。
- 资源隔离，减少锁竞争。
- 一个连接阻塞不影响其他连接。

3. **安全转移连接**

- 主线程 accept 后，选中 subloop。
- 构造 `TcpConnection` 绑定 subloop。
- 后续回调、读写、关闭都通过 `runInLoop/queueInLoop` 抛到 subloop 执行。

---

## 第三部分：核心原理

### 问题 7：sendInLoop 的智能发送策略

**Q**：解释 `sendInLoop` 函数中的 "智能发送策略"：为什么先尝试直接 write，如果失败再缓冲 + 事件监听？这种设计解决了什么问题？

**考察点**：网络性能、非阻塞、流量控制

**答案**：

1. **先直接 write**

- 零拷贝、低延迟、立即发往内核。
- 数据小、缓冲区空时，一次写完效率最高。

2. **失败则缓冲 + 监听 EPOLLOUT**

- 原因：内核缓冲区满、对端接收慢、网络拥塞（`EAGAIN/EWOULDBLOCK`）。
- 数据存入 `outputBuffer_`，注册可写事件。
- 内核可写时，`epoll_wait` 唤醒，`handleWrite()` 继续发送。

3. **解决问题**

- 非阻塞、不挂起线程。
- 避免忙等，CPU 利用率更高。
- 平滑处理生产 / 消费速度不匹配。

---

### 问题 8：高水位回调机制

**Q**：什么是高水位回调？它的作用是什么？在什么情况下会触发？

**考察点**：流量控制、内存管理、背压

**答案**：

1. **定义**

- 发送缓冲区 `outputBuffer_` 数据量超过预设阈值时的回调。

2. **作用**

- **内存保护**：防止缓冲区无限膨胀。
- **背压（Back Pressure）**：通知应用层限速，匹配发送 / 接收速度。

3. **触发条件**

- 追加数据后，`readableBytes()` 从低于阈值变为**高于阈值**。
- 只触发一次，降到阈值以下再超限时才会再次触发。

---

## 补充知识点

### 1. 异常处理机制

略（可按项目补充）

### 2. EventLoopThreadPool 启动流程

**父线程**：

1. `EventLoopThreadPool::start()`
2. 创建 `EventLoopThread`，调用 `startLoop()`
3. 启动线程，用条件变量等待子线程创建 loop
4. 将 loop 存入 `loops_`

**子线程**：

1. `threadFunc()` 运行
2. 创建栈上 EventLoop
3. 条件变量通知父线程
4. 执行 `loop.loop()` 事件循环

### 3. 进程 / 线程 / 同步机制

- 线程启动：`Thread::start()` 创建 `std::thread`，用信号量同步 tid。
- `EventLoopThread` 用**条件变量**而非信号量：适合等待 “loop 已创建” 条件，可防虚假唤醒。

### 4. 为什么 EventLoop 在子线程栈上创建

- 符合 `one loop per thread`，无跨线程访问。
- 栈自动管理生命周期，无线程安全释放问题。
- 缓存局部性更好，性能更高。