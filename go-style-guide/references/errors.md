# 错误处理（GO-ERR）

来源：Uber Go Style Guide 错误章节，加 `errors.AsType`（Go 1.26）。

**总则**：错误是控制流的一部分，不是日志素材。每个错误有且只有一个处理者。

---

## 1. 错误的产生

| 编号 | 条款 |
|---|---|
| GO-ERR-01 | 静态、无参数的错误用包级哨兵变量：`var ErrFrameTooShort = errors.New("frame too short")`。 |
| GO-ERR-02 | 需要携带上下文的错误，用 `fmt.Errorf("解析帧头: %w", err)` 包装，`%w` 保留错误链。 |
| GO-ERR-03 | `%v` 会切断错误链，`errors.Is`/`As` 失效。只有在「不打算让调用方判断这个错误」时才用 `%v`。 |
| GO-ERR-04 | 需要调用方按类型判断的错误，定义具名类型并实现 `Error() string`。 |
| GO-ERR-05 | 错误消息小写开头，不加标点。因为会被拼接。 |
| GO-ERR-06 | 错误消息不重复上下文：`fmt.Errorf("read config: %w", err)`，不要在 `err` 里再写一次 `read config`。 |
| GO-ERR-07 | 错误消息不得包含凭据、完整路径、SQL 原文、堆栈。 |

```go
// 哨兵错误
var (
    ErrFrameTooShort = errors.New("frame too short")
    ErrChecksum      = errors.New("checksum mismatch")
)

// 带上下文的包装
if len(b) < headerLen {
    return fmt.Errorf("parse frame: %w", ErrFrameTooShort)
}

// 具名类型（携带数据）
type RangeError struct {
    Field string
    Value int
    Max   int
}
func (e *RangeError) Error() string {
    return fmt.Sprintf("%s out of range: %d > %d", e.Field, e.Value, e.Max)
}
```

---

## 2. 错误的判断

| 编号 | 条款 |
|---|---|
| GO-ERR-08 | 判断特定哨兵错误用 `errors.Is`，禁止 `err == ErrXxx`（包装后就失效）。 |
| GO-ERR-09 | 提取特定类型用 `errors.As`（1.24 及以前）或 `errors.AsType[T]`（1.26+）。 |
| GO-ERR-10 | `errors.AsType` 是泛型版本，类型安全、少一次声明、更快。模块 `go` 行 ≥ 1.26 时优先用它。 |
| GO-ERR-11 | 禁止用字符串匹配判断错误类型（`strings.Contains(err.Error(), "timeout")`）。 |

```go
// 1.24 及以前
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
    log.Printf("path=%s", pathErr.Path)
}

// 1.26+（推荐）
if pathErr, ok := errors.AsType[*fs.PathError](err); ok {
    log.Printf("path=%s", pathErr.Path)
}

// 哨兵
if errors.Is(err, ErrFrameTooShort) {
    return StatusInvalid
}
```

---

## 3. 只处理一次

| 编号 | 条款 |
|---|---|
| GO-ERR-25 | 每个返回的 `error` 必须处理：返回、包装、记日志、或显式忽略并写理由。禁止静默丢弃。`errcheck` 零报告为准。 |
| GO-ERR-02a | 一个错误只能被处理一次。「处理」指：记日志、转成返回值、转成用户可见消息、吞掉。四选一。 |
| GO-ERR-12 | 禁止「既记日志又返回」：调用方还会再记一次，同一错误在日志里出现多次，掩盖真实层次。 |
| GO-ERR-13 | 分层原则：底层返回（不记日志），顶层记录并转换。 |
| GO-ERR-14 | 明确要忽略的错误，写 `_ = f()` 并紧邻注释理由。禁止无注释丢弃。 |
| GO-ERR-15 | 可预期的「未找到」类错误不算失败，不要刷错误日志（用 `debug` 级或计数）。 |

```go
// 错：既记日志又返回
if err := save(); err != nil {
    log.Printf("save failed: %v", err)   // 上层还会再记一次
    return err
}

// 对：只返回，由上层决定记录方式
if err := save(); err != nil {
    return fmt.Errorf("save record: %w", err)
}

// 对：明确忽略并写理由
_ = conn.Close()   // 关闭失败不影响已有结果，连接即将丢弃
```

---

## 4. 不透明错误与内部实现

| 编号 | 条款 |
|---|---|
| GO-ERR-16 | 库不应把内部错误直接透出（暴露实现细节与依赖）。转成自己的哨兵错误或具名类型再返回。 |
| GO-ERR-17 | 需要保留原始错误供诊断时，用 `%w` 包装并同时满足第 16 条的做法：定义包装类型，`Unwrap()` 返回原始错误。 |
| GO-ERR-18 | 导出的错误变量必须有文档注释，说明触发条件。 |

```go
// 对：对外不透明，对内可追溯
var ErrStoreUnavailable = errors.New("store unavailable")

func (s *Store) Get(id string) (*Record, error) {
    r, err := s.db.Query(id)
    if err != nil {
        return nil, fmt.Errorf("%w: %v", ErrStoreUnavailable, err)  // 对外只认哨兵
    }
    return r, nil
}
```

---

## 5. 边界转换

跨层、跨进程时必须转换成边界自己的错误语义，不要把内部错误直接返回给上层或用户。

| 编号 | 条款 |
|---|---|
| GO-ERR-19 | 存储层错误 → 领域错误 → API 错误，三层各自定义，转换点写在一处。 |
| GO-ERR-20 | HTTP 层错误映射集中在一处（中间件或 mapper），禁止散落在 handler 里。 |
| GO-ERR-21 | 对外 API 的错误消息不得包含内部标识（表名、文件路径、主机名）。 |

---

## 6. 错误与 `panic` 的边界

| 编号 | 条款 |
|---|---|
| GO-ERR-22 | 可恢复的失败一律 `error`。`panic` 只用于不变量破坏。 |
| GO-ERR-23 | `recover` 只允许在 goroutine 入口与 HTTP 中间件，且必须记录堆栈后转为错误返回或终止该请求。 |
| GO-ERR-24 | 禁止 `panic(err)` 把 error 当 panic 抛。这是控制流误用。 |

---

## 7. 检查清单

| # | 检查项 | 判据 |
|---|---|---|
| 1 | 每个 `error` 返回被处理 | `errcheck` 零报告 |
| 2 | 无「既记又返」 | 人工评审，grep `log.*err` 后跟 `return err` |
| 3 | 哨兵错误用 `errors.Is` | 无 `== Err` 比较 |
| 4 | 类型判断用 `As`/`AsType` | 无字符串匹配 |
| 5 | 包装用 `%w` | 需要调用方判断的错误链未断 |
| 6 | 错误消息无敏感信息 | 抽检日志输出 |
| 7 | 导出错误有注释 | `golint`/`revive` 的导出注释检查通过 |
