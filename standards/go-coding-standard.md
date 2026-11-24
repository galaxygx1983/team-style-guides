# Go 编码规范（团队版）

| 文档属性 | 内容 |
|---|---|
| 文档名称 | Go 编码规范（团队版） |
| 文档编号 | RAIL-STD-GO-001 |
| 版本 | V1.0 |
| 发布日期 | 2026-11-23 |
| 适用范围 | 团队全部 Go 代码的编写、改写与评审 |
| 上游依据 | uber-go/guide（更新至 2023-05），裁剪版；Go 官方语言规范 1.24–1.26 |
| 条款编号 | `GO-<区域>-<序号>`（稳定引用标识） |
| 来源 | 团队规范技能包 `go-style-guide-20261123`（SKILL.md 与 references 全部章节合并重排） |

## 前言

本文件给出团队 Go 代码的可执行编码规则，并把版本口径拉到当前工具链。

上游是 uber-go/guide（更新至 2023-05），本文件是裁剪版：删去纯风格项（行宽、声明分组、import 顺序）；性能章节只留原则与纪律，删具体技巧；删去函数式选项模式。

本文件只规定编码规则，不含流程、部署与 CI 内容。评审意见直接引用条号或条款编号，例：`违反 GO-CONC-01`。

## 1 范围

本文件规定团队 Go 代码的命名、API 设计、错误处理、并发、版本演进、测试写法与安全编码规则，适用于团队内新建、改写与评审的全部 Go 代码（Go 1.24 及以上）。本文件不规定流程、部署与 CI 内容，格式类要求统一交由 `gofmt`、`goimports` 与 lint 工具执行。

## 2 规范性引用文件

下列文件中的内容通过文中的规范性引用构成本文件必不可少的条款。其中，注日期的引用文件，仅该日期对应的版本适用于本文件；不注日期的引用文件，其最新版本（包括所有的修改单）适用于本文件。

| 编号 | 文件 | 说明 |
|---|---|---|
| GO-REF-01 | uber-go/guide | 上游风格指南，本文件的条款来源，已裁剪 |
| GO-REF-02 | Go Language Specification | Go 1.24–1.26 语言语义依据 |
| GO-REF-03 | Go Release Notes | 1.24、1.25、1.26 新特性与门槛依据 |
| GO-REF-04 | golangci-lint | 命名与静态检查的机器化配置来源，见 `assets/.golangci.yml` |

## 3 术语和定义

下列术语和定义适用于本文件。

| 术语 | 定义 |
|---|---|
| 条款编号 | 形如 `GO-CONC-01` 的稳定标识，用于评审引用，不随章节调整而变化。 |
| 条号 | 本文件的章节内顺序号，形如 5.3.7，随文档结构调整可能变化。 |
| goroutine | Go 运行时调度的轻量并发执行单元，由 `go` 语句创建。 |
| 汇合点 | 等待 goroutine 结束的位置，如 `WaitGroup.Wait`、channel 接收、`errgroup.Wait`。 |
| 零值 | 变量未显式初始化时的取值。Go 与 `sync.Mutex`、`bytes.Buffer` 等类型的零值可直接使用。 |
| 哨兵错误 | 包级定义的具名错误变量，调用方用 `errors.Is` 判断。 |
| 只处理一次 | 一个错误只由一处负责：记日志、转返回值、转用户消息、吞掉，四选一。 |
| 受限根目录 | `os.Root`（Go 1.24+）提供的目录句柄，越界路径在系统调用层被拒绝。 |
| fire-and-forget | 启动 goroutine 后不设退出路径与汇合点的写法，属禁止用法。 |
| 红线 | 命中即阻断合入的强制条款。 |

## 4 总体原则

### 4.1 总则

错误是控制流的一部分，不是日志素材，每个错误有且只有一个处理者。Go 的并发很容易写出「看起来对、跑起来错」的代码：每条 goroutine 都要能回答两个问题——谁让它退出、谁等它退出；答不出来就是泄漏。上游 guide 最后更新于 2023-05，基线约 Go 1.20，本文件第 5.4 章补齐 1.24–1.26 的新写法，不补这一章，团队会按老写法写新代码。

### 4.2 红线条款

下列条款命中即阻断合入。

| 条号 | 条款编号 | 红线 |
|---|---|---|
| 4.2.1 | GO-CONC-01 | 禁止 fire-and-forget goroutine。每个 goroutine 必须有明确的退出路径与汇合点。 |
| 4.2.2 | GO-CONC-02 | `sync.WaitGroup` 用 `wg.Go`（1.25+）或 `Add` + `defer Done`。禁止 `Add` 之后忘记 `Done`。 |
| 4.2.3 | GO-ERR-25 | 每个返回的 `error` 必须处理。禁止 `_ = f()` 丢弃，除非注释写明理由。 |
| 4.2.4 | GO-ERR-02a | 错误只处理一次：不要既记日志又返回。二选一。 |
| 4.2.5 | GO-SEC-01 | 禁止硬编码密钥、口令、令牌。含测试与示例。 |
| 4.2.6 | GO-SEC-02 | 外部输入拼路径必须用 `os.Root`（1.24+）或等价受限打开。禁止裸 `filepath.Join` 后用 `os.Open`。 |
| 4.2.7 | GO-SEC-12 | 禁止 `exec.Command("sh", "-c", 拼接串)`；参数逐个传。 |
| 4.2.8 | GO-SEC-15 | 渲染不可信数据必须用 `html/template`；禁止 `template.HTML(x)` 绕过转义。 |
| 4.2.9 | GO-MOD-01 | `go.mod` 的 `go` 行不得低于 1.24。禁止使用超出声明版本的语言特性。 |
| 4.2.10 | GO-NAM-09 | 导出标识符必须有文档注释，以标识符名开头。 |
| 4.2.11 | GO-API-20 | 禁止裸参数。超过 2 个同类型参数或任何布尔参数，改用具名结构体。 |

## 5 详细条款

### 5.1 命名与 API

来源：Uber Go Style Guide 保留条款的中文转写。删去论证，只留决策。纯风格项（行宽、声明分组、import 顺序）已交给 `gofmt` 与 lint，不在本文。

---

#### 包与文件

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.1 | GO-NAM-01 | 包名短、全小写、单词连写不加下划线。例：`interlock`、`framesvc`，不用 `frame_service`。 |
| 5.1.2 | GO-NAM-02 | 包名不与导入路径末段重复语义。`rail/interlock` 包内不再叫 `interlock.XXX` 前缀。 |
| 5.1.3 | GO-NAM-03 | 包名避免 `util`、`common`、`helpers`、`base`。这类包必然变成垃圾桶。 |
| 5.1.4 | GO-NAM-04 | 文件名小写下划线：`frame_parser.go`。测试文件 `xxx_test.go`，同包测试不加 `_test` 包名（白盒），需要外部视角才用 `package xxx_test`。 |
| 5.1.5 | GO-NAM-05 | 一个包内禁止出现与包名同名的类型，例 `interlock.Interlock`。改名或改包名。 |

---

#### 标识符命名

| 类别 | 规则 | 示例 |
|---|---|---|
| 导出标识符 | `MixedCaps`，不以包名开头 | `interlock.Controller` |
| 未导出标识符 | `mixedCaps` | `parseFrame` |
| 未导出全局变量 | 加 `_` 前缀 | `_defaultTimeout` |
| 错误变量 | `ErrXxx`（导出）/ `errXxx`（未导出） | `ErrFrameTooShort` |
| 错误类型 | `XxxError` | `ParseError` |
| 常量 | `MixedCaps`，不用全大写 | `MaxFrameSize`，不用 `MAX_FRAME_SIZE` |
| 接口（单方法） | 方法名 + `er` | `Reader`、`Closer` |
| 缩写 | 全大或全小，不混写 | `URL`、`ID`、`httpURL`、`userID` |

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.6 | GO-NAM-06 | 禁止用 Go 内建名做标识符：`len`、`cap`、`new`、`make`、`copy`、`string`、`error`、`true`。 |
| 5.1.7 | GO-NAM-07 | 禁止用 Go 关键字与预声明类型名做变量名。 |
| 5.1.8 | GO-NAM-08 | 名字要能 `grep`。`i`、`tmp`、`data` 只允许出现在 10 行内的局部作用域。 |
| 5.1.9 | GO-NAM-09 | 导出标识符的文档注释以标识符名开头，完整句，句号结尾。 |
| 5.1.10 | GO-NAM-10 | 同一事物全仓一个词。`frame` 就是 `frame`，不要同时出现 `packet`、`message`、`frame` 指同一概念。 |

```go
// 对
// ParseFrame 解析一帧，返回帧体与消耗的字节数。
func ParseFrame(b []byte) (Frame, int, error)

// 错：注释不以名字开头，读者无法在 godoc 中定位
// 这个函数解析一帧。
func ParseFrame(b []byte) (Frame, int, error)
```

---

#### 接口

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.11 | GO-API-02 | 接口定义在使用方（消费者侧），不在实现方。 |
| 5.1.12 | GO-API-03 | 接口不超过 3 个方法。超过说明抽象错了，拆。 |
| 5.1.13 | GO-API-04 | 禁止定义「只有一个实现」的接口，除非用于测试替身或跨模块解耦，且注释写明。 |
| 5.1.14 | GO-API-05 | 禁止返回接口、接收结构体指针之外的抽象。构造函数返回具体类型，调用方自行决定是否抽象。 |
| 5.1.15 | GO-API-06 | 编译期校验接口实现：`var _ Reader = (*frameReader)(nil)`。 |
| 5.1.16 | GO-API-07 | 几乎不用「指向接口的指针」。`*io.Reader` 是错误写法。 |

```go
// 对：编译期断言
var _ FrameSource = (*fileFrameSource)(nil)

// 错：指向接口的指针
func Consume(r *io.Reader) error
```

---

#### 接收者

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.17 | GO-API-08 | 同一类型的全部方法用同一种接收者（全值或全指针），不得混用。 |
| 5.1.18 | GO-API-09 | 需要修改接收者、含 `sync.Mutex` 等不可拷贝字段、或类型较大 → 指针接收者。 |
| 5.1.19 | GO-API-10 | 小且不可变的值类型（`time.Time`、小结构）→ 值接收者。 |
| 5.1.20 | GO-API-11 | 接收者名短且一致：`c`、`s`、`f`。禁止 `this`、`self`、`me`。 |
| 5.1.21 | GO-API-12 | 值接收者方法中禁止修改字段（改了也只影响副本，属静默错误）。 |

---

#### 结构体

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.22 | GO-API-13 | 字面量必须写字段名：`T{A: 1}`。禁止位置字面量 `T{1}`。 |
| 5.1.23 | GO-API-14 | 设计零值可用：`var buf bytes.Buffer` 直接能用。不可用时提供 `NewXxx()` 并在注释写明零值不可用。 |
| 5.1.24 | GO-API-15 | `sync.Mutex` 零值可用。禁止 `mu := new(sync.Mutex)` 或 `&sync.Mutex{}`。 |
| 5.1.25 | GO-API-16 | 导出结构体禁止嵌入导出类型（会把内部方法提升为公开 API）。要复用就具名字段。 |
| 5.1.26 | GO-API-17 | 需要序列化的结构体必须写字段标签。JSON 用 `omitzero` 而非 `omitempty`（1.24+，见 `modern-go.md`）。 |
| 5.1.27 | GO-API-18 | 结构体字段顺序：导出在前，未导出在后；同组内按逻辑分组。 |
| 5.1.28 | GO-API-19 | 禁止把 `context.Context` 存进结构体字段。作为每个方法的第一个参数传入。 |

```go
// 错：嵌入导出类型，internal 方法被提升
type Server struct {
    http.Server   // Close、ListenAndServe 等全部成为 Server 的公开方法
}

// 对
type Server struct {
    httpSrv *http.Server
}
```

---

#### 函数

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.29 | GO-API-20 | 禁止裸参数：`func Move(from, to string, force, dry bool)` 调用处 `Move("a","b",true,false)` 不可读。超过 2 个同类型参数就用具名结构体。 |
| 5.1.30 | GO-API-21 | 布尔参数尤其禁止裸传。用选项结构体或拆成两个函数。 |
| 5.1.31 | GO-API-22 | 返回 `error` 时放最后一位。 |
| 5.1.32 | GO-API-23 | 避免命名返回值，除用于 `defer` 修改或文档性极强的情形。 |
| 5.1.33 | GO-API-24 | 参数与返回值超过 4 个，考虑引入结构体。 |
| 5.1.34 | GO-API-25 | 函数按「构造 → 导出方法 → 未导出方法」顺序排列；同一文件内按调用关系自上而下。 |

```go
// 错
func Sync(from, to string, force, dryRun bool) error
Sync("a", "b", true, false)

// 对
type SyncOptions struct {
    From   string
    To     string
    Force  bool
    DryRun bool
}
func Sync(opts SyncOptions) error
```

---

#### 切片、映射与拷贝

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.35 | GO-API-26 | 跨越 API 边界时拷贝切片与映射，防止调用方与实现共享底层数组。 |
| 5.1.36 | GO-API-27 | 接收切片参数时不要保留引用（除注明「借用」的场合）。 |
| 5.1.37 | GO-API-28 | `nil` 是合法切片：返回 `nil` 切片优于返回空切片，除非调用方对 JSON 输出有要求（`nil` → `null`，空切片 → `[]`）。 |
| 5.1.38 | GO-API-29 | 映射读取必须用 `v, ok := m[k]` 区分「不存在」与「零值」。 |

```go
// 对：边界拷贝
func NewFrame(body []byte) Frame {
    return Frame{body: append([]byte(nil), body...)}
}

// 错：共享底层数组，调用方改动影响已存储的 Frame
func NewFrame(body []byte) Frame { return Frame{body: body} }
```

---

#### 控制流与作用域

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.39 | GO-API-30 | 减少嵌套：错误分支提前 `return`，正常路径保持左对齐。 |
| 5.1.40 | GO-API-31 | 删除不必要的 `else`：`if` 分支已 `return`/`continue`/`break` 时，`else` 去掉。 |
| 5.1.41 | GO-API-32 | 变量作用域收到最窄：在 `if` 语句内初始化。 |

```go
// 对
if err := f(); err != nil {
    return err
}
doNext()

// 错
if err := f(); err != nil {
    return err
} else {
    doNext()
}
```

---

#### 全局状态与初始化

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.42 | GO-API-33 | 禁止可变全局变量。用依赖注入或显式传递。 |
| 5.1.43 | GO-API-34 | 禁止 `init()`。初始化放显式构造函数，在 `main` 中编排。唯一例外：注册类副作用（`database/sql` 驱动），须注释说明。 |
| 5.1.44 | GO-API-35 | 包级变量只放常量与不可变配置。 |
| 5.1.45 | GO-API-36 | 禁止在 `init()` 里启动 goroutine。 |

---

#### `main` 与退出

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.46 | GO-API-37 | `main()` 只做编排，不含业务逻辑。业务逻辑放 `run() error`，`main` 调用后处理退出。 |
| 5.1.47 | GO-API-38 | 只在一处调用 `os.Exit`（`defer` 不会执行，多处调用必然漏清理）。 |
| 5.1.48 | GO-API-39 | 退出码语义固定：`0` 成功、`1` 一般错误、`2` 用法错误。 |

```go
func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, "fatal:", err)
        os.Exit(1)
    }
}

func run() error {
    // ... 全部业务，defer 在此层生效
    return nil
}
```

---

#### 时间

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.49 | GO-API-40 | 一律 `time.Time` 与 `time.Duration`，禁止用 `int64` 传时间戳裸值。 |
| 5.1.50 | GO-API-41 | 禁止用 `time.Sleep` 做同步（等 goroutine、等锁、等条件）。用 channel 或 `sync` 原语。 |
| 5.1.51 | GO-API-42 | 需要可测的时间，注入 `Clock` 接口或用 `testing/synctest`（1.25+）。 |
| 5.1.52 | GO-API-43 | 时区：内部存储一律 UTC，只在展示层转本地时区。 |

---

#### `panic` 边界

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.53 | GO-API-44 | 库代码禁止 `panic`。用 `error` 返回。 |
| 5.1.54 | GO-API-45 | `panic` 只用于「程序不可能继续」的编程错误（不变量破坏），且必须在包文档说明。 |
| 5.1.55 | GO-API-46 | `recover` 只允许出现在 goroutine 入口与 HTTP 中间件边界。禁止用它掩盖错误。 |

---

#### 一致性

最后一条，也是最难的一条。

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.56 | GO-API-47 | 同一包内，同一种问题用一种解法。新代码跟随既有写法，即使你认为另一种更好。 |
| 5.1.57 | GO-API-48 | 要改变既有写法，全包一次改完并在评审中说明理由。禁止新旧并存。 |
| 5.1.58 | GO-API-49 | 评审意见不得以「我个人习惯」为依据，只能引条款或引基准数据。 |

### 5.2 错误处理

来源：Uber Go Style Guide 错误章节，加 `errors.AsType`（Go 1.26）。

**总则**：错误是控制流的一部分，不是日志素材。每个错误有且只有一个处理者。

---

#### 错误的产生

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.1 | GO-ERR-01 | 静态、无参数的错误用包级哨兵变量：`var ErrFrameTooShort = errors.New("frame too short")`。 |
| 5.2.2 | GO-ERR-02 | 需要携带上下文的错误，用 `fmt.Errorf("解析帧头: %w", err)` 包装，`%w` 保留错误链。 |
| 5.2.3 | GO-ERR-03 | `%v` 会切断错误链，`errors.Is`/`As` 失效。只有在「不打算让调用方判断这个错误」时才用 `%v`。 |
| 5.2.4 | GO-ERR-04 | 需要调用方按类型判断的错误，定义具名类型并实现 `Error() string`。 |
| 5.2.5 | GO-ERR-05 | 错误消息小写开头，不加标点。因为会被拼接。 |
| 5.2.6 | GO-ERR-06 | 错误消息不重复上下文：`fmt.Errorf("read config: %w", err)`，不要在 `err` 里再写一次 `read config`。 |
| 5.2.7 | GO-ERR-07 | 错误消息不得包含凭据、完整路径、SQL 原文、堆栈。 |

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

#### 错误的判断

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.8 | GO-ERR-08 | 判断特定哨兵错误用 `errors.Is`，禁止 `err == ErrXxx`（包装后就失效）。 |
| 5.2.9 | GO-ERR-09 | 提取特定类型用 `errors.As`（1.24 及以前）或 `errors.AsType[T]`（1.26+）。 |
| 5.2.10 | GO-ERR-10 | `errors.AsType` 是泛型版本，类型安全、少一次声明、更快。模块 `go` 行 ≥ 1.26 时优先用它。 |
| 5.2.11 | GO-ERR-11 | 禁止用字符串匹配判断错误类型（`strings.Contains(err.Error(), "timeout")`）。 |

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

#### 只处理一次

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.12 | GO-ERR-25 | 每个返回的 `error` 必须处理：返回、包装、记日志、或显式忽略并写理由。禁止静默丢弃。`errcheck` 零报告为准。 |
| 5.2.13 | GO-ERR-02a | 一个错误只能被处理一次。「处理」指：记日志、转成返回值、转成用户可见消息、吞掉。四选一。 |
| 5.2.14 | GO-ERR-12 | 禁止「既记日志又返回」：调用方还会再记一次，同一错误在日志里出现多次，掩盖真实层次。 |
| 5.2.15 | GO-ERR-13 | 分层原则：底层返回（不记日志），顶层记录并转换。 |
| 5.2.16 | GO-ERR-14 | 明确要忽略的错误，写 `_ = f()` 并紧邻注释理由。禁止无注释丢弃。 |
| 5.2.17 | GO-ERR-15 | 可预期的「未找到」类错误不算失败，不要刷错误日志（用 `debug` 级或计数）。 |

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

#### 不透明错误与内部实现

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.18 | GO-ERR-16 | 库不应把内部错误直接透出（暴露实现细节与依赖）。转成自己的哨兵错误或具名类型再返回。 |
| 5.2.19 | GO-ERR-17 | 需要保留原始错误供诊断时，用 `%w` 包装并同时满足第 16 条的做法：定义包装类型，`Unwrap()` 返回原始错误。 |
| 5.2.20 | GO-ERR-18 | 导出的错误变量必须有文档注释，说明触发条件。 |

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

#### 边界转换

跨层、跨进程时必须转换成边界自己的错误语义，不要把内部错误直接返回给上层或用户。

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.21 | GO-ERR-19 | 存储层错误 → 领域错误 → API 错误，三层各自定义，转换点写在一处。 |
| 5.2.22 | GO-ERR-20 | HTTP 层错误映射集中在一处（中间件或 mapper），禁止散落在 handler 里。 |
| 5.2.23 | GO-ERR-21 | 对外 API 的错误消息不得包含内部标识（表名、文件路径、主机名）。 |

---

#### 错误与 `panic` 的边界

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.24 | GO-ERR-22 | 可恢复的失败一律 `error`。`panic` 只用于不变量破坏。 |
| 5.2.25 | GO-ERR-23 | `recover` 只允许在 goroutine 入口与 HTTP 中间件，且必须记录堆栈后转为错误返回或终止该请求。 |
| 5.2.26 | GO-ERR-24 | 禁止 `panic(err)` 把 error 当 panic 抛。这是控制流误用。 |

---

#### 检查清单

| # | 检查项 | 判据 |
|---|---|---|
| 1 | 每个 `error` 返回被处理 | `errcheck` 零报告 |
| 2 | 无「既记又返」 | 人工评审，grep `log.*err` 后跟 `return err` |
| 3 | 哨兵错误用 `errors.Is` | 无 `== Err` 比较 |
| 4 | 类型判断用 `As`/`AsType` | 无字符串匹配 |
| 5 | 包装用 `%w` | 需要调用方判断的错误链未断 |
| 6 | 错误消息无敏感信息 | 抽检日志输出 |
| 7 | 导出错误有注释 | `golint`/`revive` 的导出注释检查通过 |

### 5.3 并发

来源：Uber Go Style Guide 的 goroutine 生命周期、channel、mutex 零值条款，加 Go 1.24/1.25 新原语。

**总则**：Go 的并发很容易写出「看起来对、跑起来错」的代码。每条 goroutine 都要能回答两个问题：**谁让它退出、谁等它退出**。答不出来就是泄漏。

---

#### goroutine 生命周期

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.1 | GO-CONC-01 | 禁止 fire-and-forget。每个 `go func()` 必须有明确的退出路径与汇合点（`Wait`、`<-done`、`errgroup.Wait`）。 |
| 5.3.2 | GO-CONC-03 | 需要「等一批任务完成」用 `errgroup.Group`；需要收集首个错误用 `errgroup.WithContext`。 |
| 5.3.3 | GO-CONC-04 | 禁止在 `init()` 中启动 goroutine。 |
| 5.3.4 | GO-CONC-05 | goroutine 入口必须 `defer recover()` 并记录堆栈，否则单个 panic 会终止整个进程。 |
| 5.3.5 | GO-CONC-06 | 传给 goroutine 的循环变量必须显式传入（Go 1.22 起每次迭代独立，但显式传参更利于阅读与兼容低版本模块）。 |
| 5.3.6 | GO-CONC-07 | goroutine 持有的资源（连接、文件、ticker）必须在该 goroutine 内 `defer` 关闭，不由外部关闭。 |

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

#### `sync.WaitGroup`

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.7 | GO-CONC-02 | Go 1.25+ 用 `wg.Go(fn)`，自动 `Add(1)` + `defer Done()`。少一次人为疏漏。 |
| 5.3.8 | GO-CONC-08 | 低版本模块用 `wg.Add(1)` + `defer wg.Done()`，且 `Add` 必须在 `go` 之前。 |
| 5.3.9 | GO-CONC-09 | `Add` 与 `Wait` 不得并发调用。先全部 `Add`，再 `Wait`。 |
| 5.3.10 | GO-CONC-10 | `wg.Go` 的 `fn` 不得 `panic`（`WaitGroup` 不捕获 panic）。 |

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

#### channel

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.11 | GO-CONC-11 | channel 容量只有 0（无缓冲，同步交接）或 1（解耦一次发送）。其他容量必须有注释说明为什么是这个数，并配基准或压测。 |
| 5.3.12 | GO-CONC-12 | channel 的关闭责任在**唯一发送方**。多方发送时用 `sync.Once` 或单独协调 goroutine 关闭。 |
| 5.3.13 | GO-CONC-13 | 接收方禁止关闭 channel。 |
| 5.3.14 | GO-CONC-14 | 向已关闭 channel 发送会 panic；发送前必须确认所有权（本 goroutine 是唯一发送方，或已用 `select` 判断 `done`）。 |
| 5.3.15 | GO-CONC-15 | 从已关闭 channel 读取返回零值且 `ok == false`。必须检查 `ok`。 |
| 5.3.16 | GO-CONC-16 | 禁止用 channel 做互斥（当锁用）。互斥用 `sync.Mutex`。 |
| 5.3.17 | GO-CONC-17 | 禁止用 `len(ch)` 做「队列是否为空」的判断来做业务决策。`len` 是快照，读到即可能过时。 |

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

#### `sync.Mutex` 与共享状态

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.18 | GO-CONC-18 | `sync.Mutex` 零值可用。禁止 `new(sync.Mutex)`、`&sync.Mutex{}`。 |
| 5.3.19 | GO-CONC-19 | 含 `sync.Mutex` 的结构体禁止按值拷贝。方法用指针接收者；不得作为值传参、不得放进 `sync.Map` 的 value。 |
| 5.3.20 | GO-CONC-20 | `defer mu.Unlock()` 优先，除非热点路径已实测证明 `defer` 开销显著（须附基准）。 |
| 5.3.21 | GO-CONC-21 | 锁粒度按数据而非按函数划分。同一把锁保护同一个不变量。 |
| 5.3.22 | GO-CONC-22 | 禁止持锁调用回调、外部接口、网络 I/O、channel 发送。 |
| 5.3.23 | GO-CONC-23 | 多把锁必须固定获取顺序。顺序写进结构体注释。 |
| 5.3.24 | GO-CONC-24 | 优先 `sync.RWMutex` 仅当读远多于写且有实测收益；否则普通 `Mutex` 更简单。 |
| 5.3.25 | GO-CONC-25 | 读多写少的缓存可考虑 `sync.Map`，但仅在「键集合基本稳定、每键读写频繁」时。普通场景用 `map` + `RWMutex`。 |

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

#### 原子操作

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.26 | GO-CONC-26 | 单个计数、标志用 `sync/atomic` 的 `atomic.Int64`、`atomic.Bool` 等类型化封装（1.19+），不用裸 `atomic.AddInt64(&x, 1)`。 |
| 5.3.27 | GO-CONC-27 | 禁止用原子操作拼出「多字段一致性」。多字段不变量用锁。 |
| 5.3.28 | GO-CONC-28 | `atomic.Value` 只存同一具体类型，且存进去的值必须不可变。 |
| 5.3.29 | GO-CONC-29 | 禁止 `atomic` 与普通读写混用同一变量。要么全原子，要么全加锁。 |

---

#### 上下文与取消

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.30 | GO-CONC-30 | `context.Context` 作为函数第一个参数，命名 `ctx`。禁止存字段、禁止放进结构体。 |
| 5.3.31 | GO-CONC-31 | 禁止传 `nil` context。不确定时用 `context.TODO()`。 |
| 5.3.32 | GO-CONC-32 | 每个阻塞操作（I/O、channel 收发、锁等待）都必须能被 `ctx.Done()` 打断。 |
| 5.3.33 | GO-CONC-33 | `context.WithCancel`/`WithTimeout` 的 `cancel` 必须 `defer` 调用，否则泄漏。 |
| 5.3.34 | GO-CONC-34 | 派生 context 不得比父 context 活得久。 |
| 5.3.35 | GO-CONC-35 | 超时值来自配置，不写字面量散落各处。 |

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

#### 并发测试

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.36 | GO-CONC-36 | 触及并发的代码必须 `go test -race` 通过。 |
| 5.3.37 | GO-CONC-37 | Go 1.25+ 用 `testing/synctest` 测并发：假时钟 + `synctest.Wait()`，把「靠 sleep 凑时序」变成确定性等待。 |
| 5.3.38 | GO-CONC-38 | 禁止在测试里用 `time.Sleep` 等 goroutine。要么 `synctest`，要么用 channel 显式同步。 |
| 5.3.39 | GO-CONC-39 | 泄漏测试：Go 1.26 可用 `GOEXPERIMENT=goroutineleakprofile` 或第三方 `goleak`，在 `TestMain` 里校验无残留 goroutine。 |

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

#### 反例清单

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

### 5.4 版本演进口径：Go 1.24–1.26

上游 guide 最后更新于 2023-05，基线约 Go 1.20。下表是它完全没有的内容。**不补这一章，团队会按老写法写新代码。**

条款编号 `GO-MOD-<序号>`。「门槛」列指 `go.mod` 的 `go` 行最低要求。

---

#### 总表

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.4.1 | GO-MOD-01 | `go.mod` 的 `go` 行是本模块的语言版本下限。不得低于 1.24；不得使用超出该行的语言或标准库特性。升级时全仓一次改完，禁止同一模块内两套版本口径。 |

| 版本 | 特性 | 条款 | 门槛 |
|---|---|---|---|
| 1.24 | `tool` 指令（工具写进 `go.mod`） | GO-MOD-02 | 1.24 |
| 1.24 | `testing.B.Loop` | GO-MOD-03 | 1.24 |
| 1.24 | `omitzero` JSON 标签 | GO-MOD-04 | 1.24 |
| 1.24 | `os.Root`（受限目录） | GO-MOD-05 | 1.24 |
| 1.24 | 泛型类型别名 | GO-MOD-06 | 1.24 |
| 1.24 | `strings.SplitSeq` / `FieldsSeq` / `Lines` | GO-MOD-07 | 1.24 |
| 1.24 | `t.Context()` | GO-MOD-08 | 1.24 |
| 1.24 | `runtime.AddCleanup` 替代 `SetFinalizer` | GO-MOD-09 | 1.24 |
| 1.25 | `sync.WaitGroup.Go` | GO-MOD-10 | 1.25 |
| 1.25 | `testing/synctest` | GO-MOD-11 | 1.25 |
| 1.25 | 容器感知的 `GOMAXPROCS`（默认开启） | GO-MOD-12 | 1.25 |
| 1.26 | `new(expr)` | GO-MOD-13 | 1.26 |
| 1.26 | `errors.AsType[T]` | GO-MOD-14 | 1.26 |
| 1.26 | `go fix` 现代化器 | GO-MOD-15 | 1.26 |
| 1.26 | 泛型约束可自引用 | GO-MOD-16 | 1.26 |

---

#### Go 1.24

#### 5.4.2　GO-MOD-02　`tool` 指令

工具依赖写进 `go.mod`，不再用 `tools.go` + 空导入。

```bash
# 添加工具
go get -tool golang.org/x/tools/cmd/stringer
go get -tool github.com/golangci/golangci-lint/cmd/golangci-lint

# 运行
go tool stringer -type=State

# 安装 go.mod 里全部工具
go install tool
```

```go
// 删除 tools.go（老写法）
//go:build tools
package tools
import _ "golang.org/x/tools/cmd/stringer"
```

理由：`tools.go` 的空导入会让工具出现在包依赖图里，且版本靠 `go.mod` 的 indirect 行维持，容易漂移。`tool` 指令是显式声明。

#### 5.4.3　GO-MOD-03　`testing.B.Loop`

```go
// 旧
func BenchmarkParse(b *testing.B) {
    data := loadFixture()
    for i := 0; i < b.N; i++ {
        Parse(data)
    }
}

// 新（1.24+）
func BenchmarkParse(b *testing.B) {
    data := loadFixture()          // 只执行一次，且不计入耗时
    for b.Loop() {
        Parse(data)
    }
}
```

理由：`b.N` 循环会被编译器优化掉被测代码（结果未被使用），且准备代码可能被重复执行。`b.Loop` 由运行时保证不被优化、准备只跑一次。**所有基准测试一律改用它。**

#### 5.4.4　GO-MOD-04　`omitzero`

```go
// 错：time.Time 的零值不是「空」，omitempty 不生效
type Event struct {
    At    time.Time `json:"at,omitempty"`     // 零值会被序列化成 "0001-01-01T00:00:00Z"
}

// 对（1.24+）
type Event struct {
    At    time.Time     `json:"at,omitzero"`
    Count int           `json:"count,omitzero"`   // 0 被省略
    Tags  []string      `json:"tags,omitzero"`    // nil/空被省略
}
```

选型：

| 场景 | 用哪个 |
|---|---|
| `string`/`int`，空字符串与 0 都要省略 | `omitempty` 也可，但 `omitzero` 语义更准 |
| `time.Time`、`time.Duration`、结构体 | **必须 `omitzero`** |
| 需要区分「未设置」与「设为零值」 | 用指针 + `omitzero`（指针 nil 才省略） |

#### 5.4.5　GO-MOD-05　`os.Root`

外部输入参与路径时，用 `os.Root` 把操作限制在一个目录子树内，`../` 与越界符号链接在系统调用层被拒。

```go
// 错：拼接后被穿越
p := filepath.Join(uploadDir, name)   // name = "../../etc/passwd"
data, err := os.ReadFile(p)

// 对（1.24+）
root, err := os.OpenRoot(uploadDir)
if err != nil {
    return err
}
defer root.Close()

f, err := root.Open(name)   // 越界返回错误：path escapes from parent
if err != nil {
    return err
}
defer f.Close()
```

`os.Root` 支持 `Open`、`Create`、`OpenFile`、`Stat`、`Mkdir`、`Remove`、`Rename` 等。凡是「路径成分来自外部」的场景一律用它，替代手写 `filepath.Clean` + 前缀比较。

#### 5.4.6　GO-MOD-06　泛型类型别名

```go
// 1.24+ 合法
type Set[T comparable] = map[T]struct{}
type Handler[T any] = func(T) error
```

1.24 前只能定义新的具名类型（`type Set[T comparable] map[T]struct{}`），语义不同（不共享方法集）。

#### 5.4.7　GO-MOD-07　迭代器版字符串分割

```go
// 旧：分配一个切片
for _, part := range strings.Split(csv, ",") { use(part) }

// 新（1.24+）：零分配惰性迭代
for part := range strings.SplitSeq(csv, ",") { use(part) }

// 按行
for line := range strings.Lines(text) { use(line) }
```

需要保留切片时才用 `Split`；只是遍历一律用 `*Seq` 版本。

#### 5.4.8　GO-MOD-08　`t.Context()`

```go
// 旧
func TestFoo(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
}

// 新（1.24+）：测试结束自动取消
func TestFoo(t *testing.T) {
    ctx := t.Context()
}
```

#### 5.4.9　GO-MOD-09　`runtime.AddCleanup`

```go
// 旧：SetFinalizer 只能设一次，且对象可能被复活
runtime.SetFinalizer(obj, func(o *Obj) { o.Close() })

// 新（1.24+）：可多次添加，不复活对象，支持循环引用
runtime.AddCleanup(obj, func(f *os.File) { f.Close() }, obj.file)
```

新代码一律 `AddCleanup`。`SetFinalizer` 只在不支持的旧模块里保留。

---

#### Go 1.25

#### 5.4.10　GO-MOD-10　`sync.WaitGroup.Go`

```go
// 旧
var wg sync.WaitGroup
wg.Add(1)
go func() { defer wg.Done(); work() }()
wg.Wait()

// 新（1.25+）
var wg sync.WaitGroup
wg.Go(func() { work() })
wg.Wait()
```

`wg.Go` 内部完成 `Add(1)` 与 `defer Done()`。注意：`fn` 内 `panic` 不被捕获，仍需自行 `recover`。

#### 5.4.11　GO-MOD-11　`testing/synctest`

测试并发代码时，把 `time` 换成假时钟，`synctest.Wait()` 等待所有 goroutine 进入阻塞状态，消除 `time.Sleep` 造成的 flaky。

```go
func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ctx, cancel := context.WithTimeout(t.Context(), time.Second)
        defer cancel()

        started := make(chan struct{})
        go func() {
            close(started)
            <-ctx.Done()
        }()

        synctest.Wait()   // 此时 goroutine 已阻塞在 ctx.Done()
        // 假时钟推进，超时立即生效，不真等 1 秒
        time.Sleep(time.Second)
        synctest.Wait()

        if ctx.Err() == nil {
            t.Fatal("expected timeout")
        }
    })
}
```

1.24 的实验版 `synctest.Run` 已被 `synctest.Test` 取代，不要在 1.25+ 代码里用旧 API。

#### 5.4.12　GO-MOD-12　容器感知的 `GOMAXPROCS`

Go 1.25 起，Linux 上运行时默认按 cgroup CPU 限额设置 `GOMAXPROCS`。所以 1.25+ 的代码不要再引 `automaxprocs`：

```go
// 删除（1.25+ 不再需要，且可能覆盖运行时的正确值）
import _ "go.uber.org/automaxprocs"
```

| 版本 | 写法 |
|---|---|
| 1.25+ | 不引 `automaxprocs`。确需覆盖时在 `main` 里显式 `runtime.GOMAXPROCS(n)`，并注释取值依据 |
| 1.24 及以前 | 可保留 `automaxprocs`，或在 `main` 里读 cgroup 限额后设 `runtime.GOMAXPROCS` |

---

#### Go 1.26

#### 5.4.13　GO-MOD-13　`new(expr)`

```go
// 旧：需要临时变量或 newInt 辅助函数
v := 30
cfg := Config{Timeout: &v}

func newInt(x int) *int { return &x }
cfg := Config{Timeout: newInt(30)}

// 新（1.26+）
cfg := Config{Timeout: new(30)}
```

典型用途：`encoding/json`、protobuf 里用 `*T` 表示可选字段。

```go
type Person struct {
    Name string `json:"name"`
    Age  *int   `json:"age,omitzero"`
}
data, _ := json.Marshal(Person{Name: "x", Age: new(36)})
```

禁止写法：`new(int(30))`（多余转换）、`x := 30; new(x)`（老套路）。

#### 5.4.14　GO-MOD-14　`errors.AsType[T]`

```go
// 旧
var pathErr *fs.PathError
if errors.As(err, &pathErr) { use(pathErr.Path) }

// 新（1.26+）：类型安全、无变量声明、更快
if pathErr, ok := errors.AsType[*fs.PathError](err); ok { use(pathErr.Path) }
```

#### 5.4.15　GO-MOD-15　`go fix` 现代化器

`go fix` 在 1.26 被重写为「现代化器集合」，与 `gopls` 共用分析框架。旧的历史 fixer 已全部移除。

```bash
# 查看可用现代化器
go tool fix help

# 全部应用（会改写源码，先做好可比对的备份）
go fix ./...

# 只迁移 new(expr)
go fix -newexpr ./...

# 只迁移 range-over-int
go fix -rangeint ./...
```

已知现代化器（部分）：`minmax`（改用 `min`/`max`）、`rangeint`（三句式 for → `for range n`）、`stringscut`（`if i := strings.Index...` → `strings.Cut`）、`newexpr`（`new(expr)`）。

使用纪律：

1. 只在 `go.mod` 的 `go` 行 ≥ 对应版本时应用（现代化器自身也遵守这条）。
2. 每轮只应用一个 fixer，跑完 `go build ./... && go test ./...` 全绿再进下一个。
3. 不接受 `go fix` 产生的、语义等价但可读性变差的改写；手工改回并加 `//nolint` 说明。

#### 5.4.16　GO-MOD-16　泛型约束自引用

```go
// 1.26+ 合法
type Adder[A Adder[A]] interface { Add(A) A }
func sum[A Adder[A]](xs ...A) A
```

1.26 前这种「约束引用自身」的写法不被允许。使用场景窄，只在确实需要 F-bounded 多态时用。

---

#### 升级时的改法顺序

1. 把 `go.mod` 的 `go` 行升到目标版本，`go mod tidy`，`go build ./...`。
2. 删除 `automaxprocs` 空导入与依赖（1.25+）。
3. `go fix ./...` 或按单个 fixer 分批执行；每轮改完 `go build ./... && go test ./...` 全绿再进下一轮。
4. 人工复核剩余项：基准测试改 `b.Loop`、JSON 标签改 `omitzero`、路径打开改 `os.Root`、测试改 `t.Context()` 与 `synctest`。
5. 删除只为旧版本保留的兼容分支与 `//go:build` 降级标签。

---

#### 禁止事项

| 禁止 | 原因 |
|---|---|
| 在 `go` 行低于特性门槛时使用该特性 | 构建失败或行为不一致 |
| 用 `//go:build go1.26` 局部绕过门槛 | 同一模块内版本割裂，维护成本高于收益 |
| 保留 `tools.go` 与 `tool` 指令并存 | 两套工具版本来源，必然漂移 |
| 继续引 `automaxprocs`（1.25+） | 冗余，且可能覆盖运行时的正确值 |
| 用 `go fix` 一次性改全仓 | 改动无法逐条复核，出问题无法定点回滚 |

### 5.5 测试代码的写法

本章只管「测试代码怎么写才符合规范」。覆盖率指标、CI 门禁、报告生成不属编码规范，不在本章。

---

#### 文件与包

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.1 | GO-TCODE-01 | 测试文件与被测文件同目录，`xxx_test.go`。禁止单独的 `tests/` 目录（Go 按包组织）。 |
| 5.5.2 | GO-TCODE-02 | 优先同包测试（`package interlock`）；需要验证外部视角时用 `package interlock_test`。同一包内不混用两种。 |
| 5.5.3 | GO-TCODE-03 | 测试代码同样适用本规范全部条款。测试代码不是免责区。 |

---

#### 命名与结构

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.4 | GO-TCODE-04 | 命名 `TestXxx_场景_预期`，子测试用 `t.Run` 描述输入。 |
| 5.5.5 | GO-TCODE-05 | 用 `t.Helper()` 标记断言辅助函数，使失败行号指向调用处。 |
| 5.5.6 | GO-TCODE-06 | 断言用标准库即可；引入 `testify` 须全仓统一，禁止两套断言风格并存。 |
| 5.5.7 | GO-TCODE-07 | 失败信息写「期望什么、实际什么、输入是什么」，不写「assertion failed」。 |
| 5.5.8 | GO-TCODE-08 | 一个测试只验证一个行为。出现多个不相关断言说明该拆。 |

---

#### 表驱动

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.9 | GO-TCODE-09 | 同一逻辑多组输入用表驱动。 |
| 5.5.10 | GO-TCODE-10 | 每个用例必须能独立运行：`t.Run(name, ...)`，无共享状态时加 `t.Parallel()`。 |
| 5.5.11 | GO-TCODE-11 | 用例体只允许「调用 + 断言」。出现 `if`/`switch` 分叉说明该拆成多个测试函数。 |
| 5.5.12 | GO-TCODE-12 | 用例名用可读描述，不用 `case1`、`case2`。 |
| 5.5.13 | GO-TCODE-13 | 期望值写在表里，不写在断言逻辑里。 |

```go
func TestParseFrame(t *testing.T) {
    tests := []struct {
        name    string
        input   []byte
        wantLen int
        wantErr error
    }{
        {"空输入", nil, 0, ErrFrameTooShort},
        {"仅帧头", []byte{0xAA, 0x55}, 2, ErrFrameTooShort},
        {"完整帧", []byte{0xAA, 0x55, 0x01, 0x00}, 4, nil},
        {"校验和错", []byte{0xAA, 0x55, 0x01, 0xFF}, 0, ErrChecksum},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := ParseFrame(tt.input)
            if !errors.Is(err, tt.wantErr) {
                t.Fatalf("err = %v, want %v", err, tt.wantErr)
            }
            if len(got) != tt.wantLen {
                t.Fatalf("len = %d, want %d", len(got), tt.wantLen)
            }
        })
    }
}
```

反例（分支复杂的测试体，应拆分）：

```go
// 错：用例体里有 if/switch，难以定位失败原因
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        switch tt.mode {
        case "json":
            // ...
        case "binary":
            // ...
        }
    })
}
```

---

#### 基准测试

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.14 | GO-TCODE-14 | 一律用 `b.Loop()`（1.24+），禁止 `for i := 0; i < b.N; i++`。 |
| 5.5.15 | GO-TCODE-15 | 准备数据放 `b.Loop()` 之前，并用 `b.ReportAllocs()` 报告分配。 |
| 5.5.16 | GO-TCODE-16 | 禁止为了通过基准而写「编译器能优化掉」的代码。`b.Loop` 已保证不被优化。 |

```go
func BenchmarkParseFrame(b *testing.B) {
    data := loadFixture(b)
    b.ReportAllocs()
    for b.Loop() {
        ParseFrame(data)
    }
}
```

---

#### 可重复性与并发

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.17 | GO-TCODE-17 | 测试必须可重复：不依赖执行顺序、外部网络、真实时间、机器性能。 |
| 5.5.18 | GO-TCODE-18 | 禁止在测试里用 `time.Sleep` 等 goroutine。用 channel 显式同步，或 `testing/synctest`（1.25+）。 |
| 5.5.19 | GO-TCODE-19 | 测试上下文用 `t.Context()`（1.24+），不手写 `context.Background()` + `cancel`。 |
| 5.5.20 | GO-TCODE-20 | 并发时序用 `testing/synctest` 确定性验证，不用 `sleep` 凑。 |

```go
func TestPipelineDrains(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        done := make(chan struct{})
        go func() { defer close(done); runPipeline() }()
        synctest.Wait()
        select {
        case <-done:
        default:
            t.Fatal("pipeline did not finish")
        }
    })
}
```

---

#### 边界与替身

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.21 | GO-TCODE-21 | 禁止为测试在生产代码里加 `if testing` 分支。用接口注入替身。 |
| 5.5.22 | GO-TCODE-22 | 需要 mock 的依赖必须是接口，且接口定义在消费方（见 GO-API-02）。 |
| 5.5.23 | GO-TCODE-23 | 测试替身放 `xxx_internal_test.go` 或 `internal/testutil`，不导出到公开 API。 |
| 5.5.24 | GO-TCODE-24 | 可以调未导出函数，但断言应针对行为，不针对实现细节。 |

### 5.6 安全维度

上游无任何安全条款。本章为增补，用于评审与自查。条款编号 `GO-SEC-<序号>`。

**总原则**：Go 没有内存安全问题，但注入与凭据问题一样多。这两类全从设计上消除。

---

#### 密钥与凭据

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.1 | GO-SEC-01 | 禁止硬编码密钥、口令、令牌、私钥、DSN。含测试、示例、`_test.go`、`go:embed` 的文件。 |
| 5.6.2 | GO-SEC-04 | 凭据从环境变量、密钥管理服务或受权限保护的文件读取。启动时校验，缺失即 `return err`，不降级。 |
| 5.6.3 | GO-SEC-05 | 凭据不得进入日志、错误消息、pprof 输出、HTTP header 回显。 |
| 5.6.4 | GO-SEC-06 | 字符串不可清零（Go 的 string 不可变）。需要安全擦除时用 `[]byte` 并在用后 `for i := range b { b[i] = 0 }`；敏感临时量可考虑 1.26 的 `runtime/secret`（实验）。 |
| 5.6.5 | GO-SEC-07 | 已提交到 Git 的密钥视为已泄露，必须轮换，不是回滚。 |
| 5.6.6 | GO-SEC-08 | 禁止 `os.Getenv` 直接把凭据写进 struct 后 `%+v` 打印。实现 `String()` 方法返回脱敏形式。 |

```go
// 错
type Config struct {
    DBPassword string
}
log.Printf("config: %+v", cfg)   // 口令进日志

// 对：实现 String 脱敏
func (c Config) String() string {
    return fmt.Sprintf("Config{DBUser:%q, DBPassword:***}", c.DBUser)
}
```

---

#### 注入

##### 2.1 SQL

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.7 | GO-SEC-09 | 禁止 `fmt.Sprintf` 拼 SQL。一律参数化：`db.QueryContext(ctx, "SELECT ... WHERE id = ?", id)`。 |
| 5.6.8 | GO-SEC-10 | 表名、列名、排序方向无法参数化，必须对照白名单映射，不接受外部原值。 |
| 5.6.9 | GO-SEC-11 | `IN (...)` 用 `sqlx.In` 或按参数个数动态生成占位符，不拼字符串。 |

```go
// 错
q := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", name)

// 对
row := db.QueryRowContext(ctx, "SELECT id FROM users WHERE name = $1", name)

// 对：标识符白名单
var orderCols = map[string]bool{"created_at": true, "name": true}
if !orderCols[col] { return ErrBadColumn }
q := "SELECT * FROM users ORDER BY " + col   // col 已过白名单
```

##### 2.2 命令执行

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.10 | GO-SEC-12 | 禁止 `exec.Command("sh", "-c", userInput)`。 |
| 5.6.11 | GO-SEC-13 | `exec.Command` 的参数逐个传，不拼成一个字符串。 |
| 5.6.12 | GO-SEC-14 | 参数值做白名单校验（字符集 + 长度）。 |

```go
// 错
exec.Command("sh", "-c", "ping -c 1 "+host)

// 对
exec.CommandContext(ctx, "ping", "-c", "1", host)   // 且 host 已白名单校验
```

##### 2.3 模板

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.13 | GO-SEC-15 | 渲染不可信数据用 `html/template`。`text/template` 只用于非 HTML 输出（纯文本、配置文件）。 |
| 5.6.14 | GO-SEC-16 | 禁止 `template.HTML(userInput)` 绕过转义。 |
| 5.6.15 | GO-SEC-17 | 模板中禁止动态拼接模板字符串（`template.New().Parse(userInput)`），这是模板注入。 |

##### 2.4 路径

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.16 | GO-SEC-02 | 外部输入参与路径，一律用 `os.Root`（1.24+）。 |
| 5.6.17 | GO-SEC-18 | 无 `os.Root` 时：`filepath.Clean` 后校验前缀，且校验基于绝对路径。禁止在原始字符串上比较。 |
| 5.6.18 | GO-SEC-19 | 禁止把外部输入直接传给 `os.Open`、`os.ReadFile`、`os.Create`、`http.ServeFile`。 |

```go
// 对（1.24+）
root, err := os.OpenRoot(baseDir)
if err != nil { return err }
defer root.Close()
f, err := root.Open(name)   // 越界由内核拒绝
```

##### 2.5 反序列化

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.19 | GO-SEC-20 | 解码外部 JSON 时限制请求体大小（`http.MaxBytesReader`）。 |
| 5.6.20 | GO-SEC-21 | 禁止对外部数据用 `gob` 解码。`gob` 不做类型白名单，可触发非预期类型构造。 |
| 5.6.21 | GO-SEC-22 | `encoding/xml` 解析不可信输入时禁用 entity 展开，并限制嵌套深度。 |

---

#### 输入校验

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.22 | GO-SEC-02a | 在 HTTP handler / 消息入口处校验一次：长度、范围、枚举、格式。内部函数靠类型与前置条件。 |
| 5.6.23 | GO-SEC-23 | 校验拒绝而非修正。`math.Max(0, x)` 会掩盖攻击。 |
| 5.6.24 | GO-SEC-24 | 请求体、消息体、上传文件必须有大小上限。 |
| 5.6.25 | GO-SEC-25 | 枚举字段按白名单判定。 |
| 5.6.26 | GO-SEC-26 | 校验失败返回可区分的错误（格式错/超范围/不支持），不返回内部细节。 |
| 5.6.27 | GO-SEC-27 | 禁止用 `strconv.Atoi` 后不检查错误就使用结果。 |

```go
// 错
port, _ := strconv.Atoi(r.URL.Query().Get("port"))   // 失败时 port = 0

// 对
portStr := r.URL.Query().Get("port")
port, err := strconv.Atoi(portStr)
if err != nil || port < 1 || port > 65535 {
    http.Error(w, "invalid port", http.StatusBadRequest)
    return
}
```

---

#### 密码学

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.28 | GO-SEC-28 | 随机数：安全用途一律 `crypto/rand`。`math/rand` 只用于仿真与测试。 |
| 5.6.29 | GO-SEC-29 | 口令哈希用 `golang.org/x/crypto/argon2` 或 `bcrypt`。禁止裸 `sha256` 存口令。 |
| 5.6.30 | GO-SEC-30 | 禁止自研加密算法与自研协议。用标准库 `crypto/*`。 |
| 5.6.31 | GO-SEC-31 | 对称加密用 AES-GCM；禁止 ECB；IV 每次随机且不重用。 |
| 5.6.32 | GO-SEC-32 | 比较敏感值（MAC、令牌）用 `crypto/subtle.ConstantTimeCompare`，不用 `==`。 |
| 5.6.33 | GO-SEC-33 | TLS 最低 1.2；`tls.Config` 不设 `InsecureSkipVerify: true`（测试代码也禁止，用测试证书）。 |

```go
// 对
n, err := rand.Int(rand.Reader, big.NewInt(100))
if err != nil { return err }

// 对：常量时间比较
if subtle.ConstantTimeCompare(got, want) != 1 { return ErrUnauthorized }
```

---

#### HTTP 与并发安全

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.34 | GO-SEC-34 | 所有对外 handler 设 `ReadHeaderTimeout`、`ReadTimeout`、`WriteTimeout`、`IdleTimeout`。缺省值会让慢连接拖死服务。 |
| 5.6.35 | GO-SEC-35 | 响应头加 `X-Content-Type-Options: nosniff`；涉及 cookie 的加 `Secure`、`HttpOnly`、`SameSite`。 |
| 5.6.36 | GO-SEC-36 | CORS 白名单，禁止 `Access-Control-Allow-Origin: *` 配合凭据。 |
| 5.6.37 | GO-SEC-37 | 生产环境关闭 `pprof` 的公开暴露，或加鉴权。`/debug/pprof` 泄露堆栈与内存内容。 |
| 5.6.38 | GO-SEC-38 | 请求日志脱敏：不记录 `Authorization`、`Cookie`、请求体中的口令字段。 |

---

#### 依赖与构建

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.6.39 | GO-SEC-39 | 依赖固定版本（`go.mod` 精确版本 + `go.sum`）。禁止跟踪分支头。 |
| 5.6.40 | GO-SEC-40 | 禁止保留 `govulncheck ./...` 有报告（含间接依赖）的版本，合入前须为零。 |
| 5.6.41 | GO-SEC-41 | 禁止引入无维护、无 license 的依赖。新依赖须在评审中说明用途与替代方案。 |
| 5.6.42 | GO-SEC-42 | 二进制构建加 `-trimpath`，避免泄露构建机路径。 |
| 5.6.43 | GO-SEC-43 | 服务不得以 root 运行：`main` 里检查 `os.Geteuid() == 0` 则拒绝启动；需低位端口时改用 capability 或前置代理。 |

```bash
govulncheck ./...
go build -trimpath -ldflags="-s -w" ./cmd/service
```

---

#### 评审清单（逐项打勾）

| # | 检查项 | 通过判据 |
|---|---|---|
| 1 | 硬编码凭据 | gitleaks 零命中；`grep -r "password\s*=" 无明文` |
| 2 | SQL 拼接 | 无 `Sprintf` 生成 SQL；查询全参数化 |
| 3 | 命令执行 | 无 `sh -c` 拼接；参数逐个传 |
| 4 | 模板 | HTML 输出用 `html/template`；无 `template.HTML(x)` |
| 5 | 路径 | 外部路径走 `os.Root` 或规范化前缀校验 |
| 6 | 输入上限 | 请求体、上传、解析深度均有上限 |
| 7 | 随机数 | 安全用途全 `crypto/rand` |
| 8 | 口令存储 | argon2/bcrypt，非裸哈希 |
| 9 | 敏感比较 | 常量时间比较 |
| 10 | 日志脱敏 | 无凭据、无个人敏感信息 |
| 11 | HTTP 超时 | 四个超时全设 |
| 12 | 依赖漏洞 | `govulncheck` 零报告 |

## 6 实施与评审

### 6.1 评审引用规则

评审意见只引条款编号或本文件条号，不得以个人习惯为依据。同一包内同一种问题用一种解法；新代码跟随既有写法。要改变既有写法，全包一次改完并在评审中说明理由，禁止新旧并存。

### 6.2 工具与门禁

格式与命名类要求交 `gofmt`、`goimports` 与 lint 执行，正文不重复规定。触及并发的代码必须 `go test -race` 通过；安全审查按第 5.6 章的评审清单逐项打勾；依赖漏洞以 `govulncheck` 零报告为准。

### 6.3 版本升级

升级 `go.mod` 的 `go` 行时，按第 5.4 章给出的顺序执行：先升版本行并 `go mod tidy`，再删 `automaxprocs` 空导入，再按单个 fixer 分批跑 `go fix`，每轮 `go build ./... && go test ./...` 全绿再进下一轮，最后人工复核基准测试、JSON 标签、路径打开与并发测试写法，删除只为旧版本保留的兼容分支。

## 附录 A  条款编号索引

本表用于「按条款编号定位」的场景。条款编号是稳定的对外引用标识，条号随章节调整而变化。

| 条款编号前缀 | 覆盖内容 | 所在条号段 |
|---|---|---|
| GO-NAM | 包名、标识符命名 | 5.1.1 ~ 5.1.10 |
| GO-API | 接口、接收者、结构体、函数、切片与映射、控制流、全局状态、`main` 退出、时间、`panic` 边界、一致性 | 5.1.11 ~ 5.1.58 |
| GO-ERR | 错误产生、判断、只处理一次、不透明错误、边界转换、`panic` 边界 | 5.2.1 ~ 5.2.26 |
| GO-CONC | goroutine 生命周期、WaitGroup、channel、mutex、原子、context、并发测试 | 5.3.1 ~ 5.3.39 |
| GO-MOD | Go 1.24–1.26 新写法 | 5.4.1 ~ 5.4.16 |
| GO-TCODE | 测试代码写法 | 5.5.1 ~ 5.5.24 |
| GO-SEC | 密钥、注入、输入校验、密码学、HTTP、依赖与构建 | 5.6.1 ~ 5.6.43 |

## 附录 B  与上游的取舍与分歧

下列条目是团队决策，不是转写遗漏。

| 上游条款 | 我们的做法 |
|---|---|
| 性能章节给具体技巧 | 只留原则与纪律，具体优化须附基准数据 |
| 函数式选项作为可选模式 | 不用，API 层统一配置结构体 |
| 行宽 99 字符 | 交 `gofmt` 与 lint |

## 附录 C  修订记录

| 版本 | 日期 | 修订内容 |
|---|---|---|
| V1.0 | 2026-11-23 | 由 `go-style-guide-20261123` 合并重排为规范文档体例，条款原文未改。 |

