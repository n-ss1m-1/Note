# 一、std::thread 线程核心（API + 用法）

## 1. 核心 API

cpp

运行

```
#include <thread>
```

1. **创建线程**
    
    `thread t(函数名, 参数...)`
2. **等待线程结束（阻塞）**
    
    `t.join()`
3. **分离线程（后台运行）**
    
    `t.detach()`
4. **判断是否可等待**
    
    `t.joinable()`
5. **获取线程 ID**
    
    `t.get_id()`

## 2. 必记规则

- 线程对象必须调用 `join()` 或 `detach()`，否则程序崩溃
- 线程传参：默认值拷贝，**传引用必须用 `std::ref`**
- `detach()` 风险：主线程退出后，分离线程访问悬空指针会崩溃

## 3. 极简示例

cpp

运行

```
void func(int a) { cout << a << endl; }
thread t(func, 10);
t.join();
```

---

# 二、std::mutex 互斥锁（基础 API）

## 1. 核心 API

cpp

运行

```
#include <mutex>
```

1. `mtx.lock()`：阻塞加锁
2. `mtx.unlock()`：手动解锁
3. `mtx.try_lock()`：非阻塞，尝试加锁（成功返回 true）

## 2. 锁的种类（了解）

- `mutex`：基础互斥锁
- `recursive_mutex`：递归锁（同一线程可多次加锁）
- `timed_mutex`：带超时的锁

## 3. 风险

手动 `lock/unlock` 忘记解锁 → **死锁**

---

# 三、std::lock_guard（RAII 自动锁）

## 1. 核心特点

- **RAII 机制**：构造时自动加锁，析构时自动解锁
- **无手动解锁**，最安全、最简单
- **不可拷贝、不可移动**
- 作用域结束自动释放锁

## 2. API（只有构造 / 析构）

cpp

运行

```
mutex mtx;
lock_guard<mutex> lock(mtx); // 自动加锁
// 出作用域 自动解锁
```

---

# 四、std::unique_lock（RAII 灵活锁）

## 1. 核心特点

- 继承 `lock_guard` 所有优点，**功能更强大**
- 支持**手动加锁 / 解锁**
- 支持**延迟加锁**
- 支持**try_lock / 超时锁**
- **必须配合条件变量 `condition_variable` 使用**
- 可移动，不可拷贝

## 2. 核心 API

cpp

运行

```
mutex mtx;
unique_lock<mutex> lock(mtx); // 立即加锁
unique_lock<mutex> lock(mtx, defer_lock); // 延迟加锁
lock.lock();    // 手动加锁
lock.unlock();  // 手动解锁
lock.try_lock();// 尝试加锁
```

---

# 五、lock_guard vs unique_lock（面试必考！）

表格

|特性|lock_guard|unique_lock|
|:--|:--|:--|
|灵活性|低（全自动）|高（手动 + 自动）|
|手动解锁|❌ 不支持|✅ 支持|
|延迟加锁|❌|✅|
|条件变量配合|❌|✅（必须用）|
|性能|略高|略低|
|适用场景|简单加锁|复杂同步、条件等待|

---

# 六、高频面试题（直接背答案）

## 1. lock_guard 和 unique_lock 的区别？

- `lock_guard`：自动加锁解锁，**不可手动操作**，简单高效。
- `unique_lock`：**灵活可控**，支持手动锁、延迟锁、条件变量，功能更强。

## 2. 为什么条件变量必须用 unique_lock？

因为 `wait()` 会**自动解锁 + 重新加锁**，只有 `unique_lock` 支持动态解锁 / 加锁。

## 3. thread 中 join 和 detach 的区别？

- `join()`：主线程**阻塞等待**子线程结束，安全无悬空风险。
- `detach()`：子线程**后台运行**，主线程不等待，易出现悬空指针崩溃。

## 4. 什么是死锁？如何避免？

- 死锁：两个线程互相持有对方需要的锁，永久阻塞。
- 避免：
    
    1. 固定加锁顺序
    2. 尽量用 `lock_guard/unique_lock`
    3. 减少锁的粒度
    

## 5. 同一个 mutex 可以加锁两次吗？

- 普通 `mutex`：**不可以**，会导致死锁。
- `recursive_mutex`：可以，同一线程可递归加锁。

## 6. 多线程读写共享变量必须加锁吗？

必须加锁！否则会出现**数据竞争**，导致程序未定义行为。

---

# 七、一句话终极总结

- `thread`：创建线程，必须 `join/detach`，传引用用 `ref`。
- `mutex`：互斥锁，保护共享资源，防止数据竞争。
- `lock_guard`：自动锁，简单安全，无手动操作。
- `unique_lock`：灵活锁，支持手动解锁，**配合条件变量使用**。