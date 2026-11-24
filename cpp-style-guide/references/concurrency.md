# 并发纪律（CPP-CONC）

上游 cppguide 对多线程几乎无实质条款。本章为增补，针对安全关键模块：联锁逻辑、状态机、采集与输出通道。

**总则**：并发正确性不靠「看起来没问题」，靠「有锁、有顺序、有谓词、有静态/动态检查」。四条缺一条即视为未完成。

---

## 1. 共享可变状态与数据竞争

| 编号 | 条款 |
|---|---|
| CPP-CONC-01 | 任何可能被两个及以上线程访问的可变状态，必须有锁或原子保护。禁止「读不加锁、写加锁」。 |
| CPP-CONC-02 | 数据竞争按 C++ 标准定义判定：两线程访问同一内存位置、至少一个写、无 happens-before 关系。不要求同时发生。 |
| CPP-CONC-03 | 声明成员/全局共享数据时，**紧邻注释**写明保护方式：`// guarded by mu_` 或 `std::atomic`。 |
| CPP-CONC-04 | 禁止通过「只读不写」绕过加锁。若写者存在，读者必须加锁或走原子。 |
| CPP-CONC-05 | 禁止把引用/指针逃逸到锁外：锁内取得的引用，不得在锁释放后使用（容器迭代器、元素指针同此）。 |
| CPP-CONC-06 | 禁止「锁内回调外部代码」。回调可能反向上锁，形成 AB-BA 死锁。锁内只做数据搬运，回调在锁外。 |

反例与正例：

```cpp
// 错：读不加锁，读到半更新状态
if (!ready_) { /* ... */ }

// 错：元素引用逃逸到锁外
Item* GetItem(int id) {
  std::lock_guard<std::mutex> lk(mu_);
  return &items_.at(id);          // 锁释放后 items_ 可能被重置
}

// 对：锁内拷贝需要的值，锁外使用
std::optional<Item> GetItem(int id) {
  std::lock_guard<std::mutex> lk(mu_);
  auto it = items_.find(id);
  if (it == items_.end()) return std::nullopt;
  return it->second;              // 按值返回
}
```

---

## 2. 锁的获取顺序

死锁的根因几乎都是「加锁顺序不一致」。用**分层锁序**从设计上消除。

| 编号 | 条款 |
|---|---|
| CPP-CONC-07 | 设计阶段给每类锁定层级（数字越小越外层）。只允许「由外层向内层」加锁，禁止反向。 |
| CPP-CONC-08 | 需要同时持有多把锁时，用 `std::scoped_lock`（C++17，内部走 `std::lock` 的退避算法），禁止连续 `lock_guard`。 |
| CPP-CONC-09 | 锁层级写在类注释与头文件顶部，例：`// Lock order: L1 config, L2 device, L3 buffer`。 |
| CPP-CONC-10 | 禁止在持锁时调用虚函数、信号槽、用户回调、`std::function`。 |
| CPP-CONC-11 | 禁止递归锁（`std::recursive_mutex`）。需要递归说明锁粒度错误，改设计。 |
| CPP-CONC-12 | 禁止在析构函数中获取可能被其他线程持有的锁。析构前应确保对象已从共享集合移除。 |

锁层级示例（本团队参考值，新模块在此表上增补并在类注释中登记）：

| 层级 | 锁 | 说明 |
|---|---|---|
| L1 | 全局配置锁 `config_mu_` | 只在启动与重配时持有 |
| L2 | 设备/通道锁 `device_mu_` | 串口、网络通道 |
| L3 | 缓冲区/队列锁 `queue_mu_` | 采集缓冲、发送队列 |
| L4 | 统计计数器 | 优先用 `std::atomic`，不加锁 |

```cpp
// 对：一次拿两把，顺序由 std::lock 决定，不会 AB-BA
std::scoped_lock lk(config_mu_, device_mu_);

// 错：顺序写死且两处相反
{ std::lock_guard<std::mutex> a(config_mu_); std::lock_guard<std::mutex> b(device_mu_); }
{ std::lock_guard<std::mutex> a(device_mu_); std::lock_guard<std::mutex> b(config_mu_); }
```

---

## 3. 条件变量：必须带谓词

| 编号 | 条款 |
|---|---|
| CPP-CONC-13 | `wait` 必须带谓词形式：`cv_.wait(lk, [&]{ return pred; })`。禁止裸 `cv_.wait(lk)`。 |
| CPP-CONC-14 | 谓词读取的状态必须由同一把互斥量保护。 |
| CPP-CONC-15 | `notify_one`/`notify_all` 可在持锁或不持锁时调用；但**修改谓词状态必须在持锁时**。 |
| CPP-CONC-16 | 无超时的等待只用于「必须等到」的场景。可退出路径必须用 `wait_for`/`wait_until` 并处理超时。 |
| CPP-CONC-17 | 禁止用条件变量做「事件计数」；计数用原子或信号量语义的封装。 |

```cpp
// 错：裸等待，虚假唤醒后继续执行
cv_.wait(lk);

// 错：谓词读的状态没被这把锁保护
cv_.wait(lk, [&]{ return shared_flag_; });   // shared_flag_ 是 atomic 或另一把锁保护

// 对
{
  std::unique_lock<std::mutex> lk(mu_);
  cv_.wait(lk, [&]{ return !queue_.empty() || stop_; });
  if (stop_ && queue_.empty()) return false;
  item = std::move(queue_.front());
  queue_.pop_front();
}
cv_.notify_one();   // 消费后再通知生产者，或按业务需要
```

带超时的等待：

```cpp
std::unique_lock<std::mutex> lk(mu_);
if (!cv_.wait_for(lk, 200ms, [&]{ return done_ || stop_; })) {
  return Status::Timeout();   // 超时是可预期结果，必须显式返回
}
```

---

## 4. 原子操作与内存序

| 编号 | 条款 |
|---|---|
| CPP-CONC-18 | 默认用 `std::memory_order_seq_cst`（不写内存序参数即为默认）。 |
| CPP-CONC-19 | 使用 `relaxed`/`acquire`/`release` 必须满足两条：① 有注释写明「与哪一处配对」② 有对应测试或 TSan 通过记录。 |
| CPP-CONC-20 | `std::atomic<T>` 只用于单个标量或指针。多字段一致性用锁，不要用多个原子拼。 |
| CPP-CONC-21 | 禁止对 `std::atomic` 做复合读改写后再判断（`if (x.load() == 0) x.store(1);`）。用 `compare_exchange_weak` 循环。 |
| CPP-CONC-22 | `std::atomic<bool>` 只用于标志位。计数用 `std::atomic<uint64_t>` 并注意溢出。 |
| CPP-CONC-23 | 原子变量不得被 `memcpy`/`memset` 操作，不得放进 `union`。 |

内存序速查：

| 场景 | 内存序 | 说明 |
|---|---|---|
| 通用标志、停止位 | `seq_cst` | 默认，不要优化 |
| 生产者发布数据后置标志 | `release` | 置标志那一次 |
| 消费者读标志后读数据 | `acquire` | 读标志那一次 |
| 纯统计计数 | `relaxed` | 只关心最终值，不建立顺序 |

```cpp
// 对：CAS 循环
uint32_t expected = state_.load();
while (!state_.compare_exchange_weak(expected, kRunning)) {
  if (expected == kFault) return Status::InvalidState();
  // expected 已被更新为当前值，继续重试
}

// 对：release/acquire 配对，注释到位
payload_ = BuildPayload();                       // 普通写
published_.store(true, std::memory_order_release); // 与消费者 acquire 配对
```

---

## 5. 线程生命周期

| 编号 | 条款 |
|---|---|
| CPP-CONC-24 | 禁止 `detach()`。线程必须有明确的所有者与汇合点。 |
| CPP-CONC-25 | `std::thread` 必须在析构前 `join()`。C++17 无 `std::jthread`，用 RAII 封装（见下）。 |
| CPP-CONC-26 | 线程入口函数不得抛出异常（异常已全局禁用）；线程体内所有失败路径显式返回。 |
| CPP-CONC-27 | 停止线程用「置停止标志 + 通知条件变量 + join」，禁止 `pthread_cancel` 或强制终止。 |
| CPP-CONC-28 | 线程数量上限写进配置并在启动时校验。禁止按消息动态创建无上限线程。 |
| CPP-CONC-29 | 每个线程命名（`pthread_setname_np` / `SetThreadDescription`），便于 `top -H` 与抓栈定位。 |

```cpp
// C++17 的 RAII 线程封装。C++20 std::jthread 可用后替换。
class ScopedThread {
 public:
  explicit ScopedThread(std::thread t) : t_(std::move(t)) {}
  ~ScopedThread() { if (t_.joinable()) t_.join(); }
  ScopedThread(const ScopedThread&) = delete;
  ScopedThread& operator=(const ScopedThread&) = delete;
  ScopedThread(ScopedThread&&) = default;
  ScopedThread& operator=(ScopedThread&&) = default;
 private:
  std::thread t_;
};
```

---

## 6. 初始化与单例

| 编号 | 条款 |
|---|---|
| CPP-CONC-30 | 单例用函数内静态对象（C++11 起初始化线程安全）。禁止手写双重检查锁。 |
| CPP-CONC-31 | 全局对象的构造不得依赖其他全局对象的构造顺序。需要顺序就用显式 `Init()` 编排。 |
| CPP-CONC-32 | 静态局部变量的析构可能发生在其他线程仍在使用时。需要「永不析构」语义时用指针 + `new`（泄漏受控）。 |

```cpp
// 对
Config& Config::Instance() {
  static Config inst;   // 线程安全初始化
  return inst;
}

// 错：手写 DCLP，无原子则数据竞争，有原子仍需 acquire/release
if (!inst_) { std::lock_guard<std::mutex> lk(mu_); if (!inst_) inst_ = new Config; }
```

---

## 7. 安全关键模块附加要求

| 编号 | 条款 |
|---|---|
| CPP-CONC-33 | 联锁判定与输出驱动之间的共享状态，必须有单调递增的序号或时间戳，用于事后判定「用的是不是同一拍数据」。 |
| CPP-CONC-34 | 关键线程必须有心跳。心跳超时由看门狗判定并进入安全侧，不由该线程自己判定。 |
| CPP-CONC-35 | 禁止在线程中无限期阻塞等待外部资源而不设超时。所有阻塞点必须有上界。 |
| CPP-CONC-36 | 并发逻辑变更必须附 TSan 运行结果（零报告）。仅「跑过测试」不作为并发正确性证据。 |

---

## 8. 检查与定位

```bash
# 数据竞争：编译时插桩，运行时检测
cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_FLAGS="-fsanitize=thread -g -O1" ..
ctest --output-on-failure

# 死锁现场：抓所有线程栈
gdb -p <pid> -batch -ex "thread apply all bt" > stacks.txt
# 或
pstack <pid> > stacks.txt

# 锁竞争热点
perf record -e sched:sched_switch -g -- ./app && perf report
```

TSan 报告为零才算通过。TSan 与 ASan 不能同时开启，分两次构建分别运行。

---

## 9. 反例清单

评审时优先查这十条，出现即返工。

| 反例 | 危害 |
|---|---|
| 裸 `cv_.wait(lk)` | 虚假唤醒后读到未就绪数据 |
| 读不加锁、写加锁 | 读到撕裂值 |
| 锁外使用锁内取得的元素指针/迭代器 | 悬垂引用 |
| 两处加锁顺序相反 | 死锁，且只在特定时序复现 |
| 锁内调用回调/虚函数 | 反向加锁死锁，或长时间持锁 |
| `detach()` 的线程访问成员变量 | 对象析构后访问已释放内存 |
| 多个原子字段表达一个不变量 | 读到不一致的组合 |
| `if (a.load()==0) a.store(1)` | 检查与设置之间被插队 |
| `std::recursive_mutex` 掩盖设计问题 | 锁粒度失控，性能与正确性双损 |
| 只在 x86 上验证 | ARM 弱内存序下暴露未定义行为 |
