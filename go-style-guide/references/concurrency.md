# 并发（GO-CONC）

来源：Uber Go Style Guide 的 goroutine 生命周期、channel、mutex 零值条款，加 Go 1.24/1.25 新原语。

**总则**：Go 的并发很容易写出「看起来对、跑起来错」的代码。每条 goroutine 都要能回答两个问题：**谁让它退出、谁等它退出**。答不出来就是泄漏。

---

## 1. goroutine 生命周期

| 编号 | 条款 |
|---|---|
| GO-CONC-01 | 禁止 fire-and-forget。每个 `go func()` 必须有明确的退出路径与汇合点（`Wait`、`<-done`、`errgroup.Wait`）。 |
| GO-CONC-03 | 需要「等一批任务完成」用 `errgroup.Group`；需要收集首个错误用 `errgroup.WithContext`。 |
| GO-CONC-04 | 禁止在 `init()` 中启动 goroutine。 |
| GO-CONC-05 | goroutine 入口必须 `defer recover()` 并记录堆栈，否则单个 panic 会终止整个进程。 |
| GO-CONC-06 | 传给 goroutine 的循环变量必须显式传入（Go 1.22 起每次迭代独立，但显式传参更利于阅读与兼容低版本模块）。 |
| GO-CONC-07 | goroutine 持有的资源（连接、文件、ticker）必须在该 goroutine 内 `defer` 关闭，不由外部关闭。 |

```go
// 错：fire-and-forget，退出无人管
go func() { ProcessQueue() }()

// 对：errgroup + 上下文取消
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error { return consume(ctx) })
g.Go(func() error { return produce(ctx) })
if err := g.Wait(); err != nil {
    return fmt.Errorf("pipeline: %w", err)
}
```

---

## 2. `sync.WaitGroup`

| 编号 | 条款 |
|---|---|
| GO-CONC-02 | Go 1.25+ 用 `wg.Go(fn)`，自动 `Add(1)` + `defer Done()`。少一次人为疏漏。 |
| GO-CONC-08 | 低版本模块用 `wg.Add(1)` + `defer wg.Done()`，且 `Add` 必须在 `go` 之前。 |
| GO-CONC-09 | `Add` 与 `Wait` 不得并发调用。先全部 `Add`，再 `Wait`。 |
| GO-CONC-10 | `wg.Go` 的 `fn` 不得 `panic`（`WaitGroup` 不捕获 panic）。 |

```go
// 1.25+
var wg sync.WaitGroup
for _, item := range items {
    wg.Go(func() { process(item) })
}
wg.Wait()

// 1.24
var wg sync.WaitGroup
for _, item := range items {
    item := item                    // 低版本需显式拷贝
    wg.Add(1)
    go func() { defer wg.Done(); process(item) }()
}
wg.Wait()
```

---

## 3. channel

| 编号 | 条款 |
|---|---|
| GO-CONC-11 | channel 容量只有 0（无缓冲，同步交接）或 1（解耦一次发送）。其他容量必须有注释说明为什么是这个数，并配基准或压测。 |
| GO-CONC-12 | channel 的关闭责任在**唯一发送方**。多方发送时用 `sync.Once` 或单独协调 goroutine 关闭。 |
| GO-CONC-13 | 接收方禁止关闭 channel。 |
| GO-CONC-14 | 向已关闭 channel 发送会 panic；发送前必须确认所有权（本 goroutine 是唯一发送方，或已用 `select` 判断 `done`）。 |
| GO-CONC-15 | 从已关闭 channel 读取返回零值且 `ok == false`。必须检查 `ok`。 |
| GO-CONC-16 | 禁止用 channel 做互斥（当锁用）。互斥用 `sync.Mutex`。 |
| GO-CONC-17 | 禁止用 `len(ch)` 做「队列是否为空」的判断来做业务决策。`len` 是快照，读到即可能过时。 |

```go
// 对：容量 1，解耦发送与处理节拍
jobs := make(chan Job, 1)

// 对：唯一发送方负责关闭
go func() {
    defer close(jobs)
    for _, j := range all {
        jobs <- j
    }
}()

// 对：接收检查 ok
for j := range jobs {
    handle(j)
}
```

---

## 4. `sync.Mutex` 与共享状态

| 编号 | 条款 |
|---|---|
| GO-CONC-18 | `sync.Mutex` 零值可用。禁止 `new(sync.Mutex)`、`&sync.Mutex{}`。 |
| GO-CONC-19 | 含 `sync.Mutex` 的结构体禁止按值拷贝。方法用指针接收者；不得作为值传参、不得放进 `sync.Map` 的 value。 |
| GO-CONC-20 | `defer mu.Unlock()` 优先，除非热点路径已实测证明 `defer` 开销显著（须附基准）。 |
| GO-CONC-21 | 锁粒度按数据而非按函数划分。同一把锁保护同一个不变量。 |
| GO-CONC-22 | 禁止持锁调用回调、外部接口、网络 I/O、channel 发送。 |
| GO-CONC-23 | 多把锁必须固定获取顺序。顺序写进结构体注释。 |
| GO-CONC-24 | 优先 `sync.RWMutex` 仅当读远多于写且有实测收益；否则普通 `Mutex` 更简单。 |
| GO-CONC-25 | 读多写少的缓存可考虑 `sync.Map`，但仅在「键集合基本稳定、每键读写频繁」时。普通场景用 `map` + `RWMutex`。 |

```go
// 对：零值可用，指针接收者
type Counter struct {
    mu sync.Mutex   // 保护 n
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}

// 错：按值拷贝含锁的结构体
func bad(c Counter) { c.Inc() }   // go vet 会报 copylocks
```

---

## 5. 原子操作

| 编号 | 条款 |
|---|---|
| GO-CONC-26 | 单个计数、标志用 `sync/atomic` 的 `atomic.Int64`、`atomic.Bool` 等类型化封装（1.19+），不用裸 `atomic.AddInt64(&x, 1)`。 |
| GO-CONC-27 | 禁止用原子操作拼出「多字段一致性」。多字段不变量用锁。 |
| GO-CONC-28 | `atomic.Value` 只存同一具体类型，且存进去的值必须不可变。 |
| GO-CONC-29 | 禁止 `atomic` 与普通读写混用同一变量。要么全原子，要么全加锁。 |

---

## 6. 上下文与取消

| 编号 | 条款 |
|---|---|
| GO-CONC-30 | `context.Context` 作为函数第一个参数，命名 `ctx`。禁止存字段、禁止放进结构体。 |
| GO-CONC-31 | 禁止传 `nil` context。不确定时用 `context.TODO()`。 |
| GO-CONC-32 | 每个阻塞操作（I/O、channel 收发、锁等待）都必须能被 `ctx.Done()` 打断。 |
| GO-CONC-33 | `context.WithCancel`/`WithTimeout` 的 `cancel` 必须 `defer` 调用，否则泄漏。 |
| GO-CONC-34 | 派生 context 不得比父 context 活得久。 |
| GO-CONC-35 | 超时值来自配置，不写字面量散落各处。 |

```go
// 对
func (s *Service) Fetch(ctx context.Context, id string) (*Record, error) {
    ctx, cancel := context.WithTimeout(ctx, s.timeout)
    defer cancel()
    select {
    case <-ctx.Done():
        return nil, fmt.Errorf("fetch %s: %w", id, ctx.Err())
    case r := <-s.results:
        return r, nil
    }
}
```

---

## 7. 并发测试

| 编号 | 条款 |
|---|---|
| GO-CONC-36 | 触及并发的代码必须 `go test -race` 通过。 |
| GO-CONC-37 | Go 1.25+ 用 `testing/synctest` 测并发：假时钟 + `synctest.Wait()`，把「靠 sleep 凑时序」变成确定性等待。 |
| GO-CONC-38 | 禁止在测试里用 `time.Sleep` 等 goroutine。要么 `synctest`，要么用 channel 显式同步。 |
| GO-CONC-39 | 泄漏测试：Go 1.26 可用 `GOEXPERIMENT=goroutineleakprofile` 或第三方 `goleak`，在 `TestMain` 里校验无残留 goroutine。 |

```go
// 1.25+：确定性并发测试
func TestPipelineDrains(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        done := make(chan struct{})
        go func() { defer close(done); runPipeline() }()
        synctest.Wait()          // 等待所有 goroutine 就绪
        select {
        case <-done:
        default:
            t.Fatal("pipeline did not finish")
        }
    })
}
```

---

## 8. 反例清单

| 反例 | 危害 |
|---|---|
| `go func(){...}()` 无汇合点 | goroutine 泄漏，进程退出时才暴露 |
| 循环变量直接捕获（低版本模块） | 全部 goroutine 看到最后一个值 |
| 接收方 `close(ch)` | 发送方 panic |
| 容量 100 的 channel 无理由 | 掩盖背压缺失，内存随时间涨 |
| 持锁做 I/O | 整条链路串行，P99 飙高 |
| 含 `Mutex` 的结构体按值传 | 锁失效，数据竞争 |
| `len(ch)` 判空做业务决策 | 判断即过时 |
| 忘了 `defer cancel()` | context 树泄漏，定时器不释放 |
| 用 `atomic` 保护多字段 | 读到不一致组合 |
| 只跑 `go test` 不跑 `-race` | 数据竞争上线才暴露 |
