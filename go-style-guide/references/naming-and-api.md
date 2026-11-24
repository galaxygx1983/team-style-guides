# 命名与 API（GO-NAM / GO-API）

来源：Uber Go Style Guide 保留条款的中文转写。删去论证，只留决策。纯风格项（行宽、声明分组、import 顺序）已交给 `gofmt` 与 lint，不在本文。

---

## 1. 包与文件

| 编号 | 条款 |
|---|---|
| GO-NAM-01 | 包名短、全小写、单词连写不加下划线。例：`interlock`、`framesvc`，不用 `frame_service`。 |
| GO-NAM-02 | 包名不与导入路径末段重复语义。`rail/interlock` 包内不再叫 `interlock.XXX` 前缀。 |
| GO-NAM-03 | 包名避免 `util`、`common`、`helpers`、`base`。这类包必然变成垃圾桶。 |
| GO-NAM-04 | 文件名小写下划线：`frame_parser.go`。测试文件 `xxx_test.go`，同包测试不加 `_test` 包名（白盒），需要外部视角才用 `package xxx_test`。 |
| GO-NAM-05 | 一个包内禁止出现与包名同名的类型，例 `interlock.Interlock`。改名或改包名。 |

---

## 2. 标识符命名

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

| 编号 | 条款 |
|---|---|
| GO-NAM-06 | 禁止用 Go 内建名做标识符：`len`、`cap`、`new`、`make`、`copy`、`string`、`error`、`true`。 |
| GO-NAM-07 | 禁止用 Go 关键字与预声明类型名做变量名。 |
| GO-NAM-08 | 名字要能 `grep`。`i`、`tmp`、`data` 只允许出现在 10 行内的局部作用域。 |
| GO-NAM-09 | 导出标识符的文档注释以标识符名开头，完整句，句号结尾。 |
| GO-NAM-10 | 同一事物全仓一个词。`frame` 就是 `frame`，不要同时出现 `packet`、`message`、`frame` 指同一概念。 |

```go
// 对
// ParseFrame 解析一帧，返回帧体与消耗的字节数。
func ParseFrame(b []byte) (Frame, int, error)

// 错：注释不以名字开头，读者无法在 godoc 中定位
// 这个函数解析一帧。
func ParseFrame(b []byte) (Frame, int, error)
```

---

## 3. 接口

| 编号 | 条款 |
|---|---|
| GO-API-02 | 接口定义在使用方（消费者侧），不在实现方。 |
| GO-API-03 | 接口不超过 3 个方法。超过说明抽象错了，拆。 |
| GO-API-04 | 禁止定义「只有一个实现」的接口，除非用于测试替身或跨模块解耦，且注释写明。 |
| GO-API-05 | 禁止返回接口、接收结构体指针之外的抽象。构造函数返回具体类型，调用方自行决定是否抽象。 |
| GO-API-06 | 编译期校验接口实现：`var _ Reader = (*frameReader)(nil)`。 |
| GO-API-07 | 几乎不用「指向接口的指针」。`*io.Reader` 是错误写法。 |

```go
// 对：编译期断言
var _ FrameSource = (*fileFrameSource)(nil)

// 错：指向接口的指针
func Consume(r *io.Reader) error
```

---

## 4. 接收者

| 编号 | 条款 |
|---|---|
| GO-API-08 | 同一类型的全部方法用同一种接收者（全值或全指针），不得混用。 |
| GO-API-09 | 需要修改接收者、含 `sync.Mutex` 等不可拷贝字段、或类型较大 → 指针接收者。 |
| GO-API-10 | 小且不可变的值类型（`time.Time`、小结构）→ 值接收者。 |
| GO-API-11 | 接收者名短且一致：`c`、`s`、`f`。禁止 `this`、`self`、`me`。 |
| GO-API-12 | 值接收者方法中禁止修改字段（改了也只影响副本，属静默错误）。 |

---

## 5. 结构体

| 编号 | 条款 |
|---|---|
| GO-API-13 | 字面量必须写字段名：`T{A: 1}`。禁止位置字面量 `T{1}`。 |
| GO-API-14 | 设计零值可用：`var buf bytes.Buffer` 直接能用。不可用时提供 `NewXxx()` 并在注释写明零值不可用。 |
| GO-API-15 | `sync.Mutex` 零值可用。禁止 `mu := new(sync.Mutex)` 或 `&sync.Mutex{}`。 |
| GO-API-16 | 导出结构体禁止嵌入导出类型（会把内部方法提升为公开 API）。要复用就具名字段。 |
| GO-API-17 | 需要序列化的结构体必须写字段标签。JSON 用 `omitzero` 而非 `omitempty`（1.24+，见 `modern-go.md`）。 |
| GO-API-18 | 结构体字段顺序：导出在前，未导出在后；同组内按逻辑分组。 |
| GO-API-19 | 禁止把 `context.Context` 存进结构体字段。作为每个方法的第一个参数传入。 |

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

## 6. 函数

| 编号 | 条款 |
|---|---|
| GO-API-20 | 禁止裸参数：`func Move(from, to string, force, dry bool)` 调用处 `Move("a","b",true,false)` 不可读。超过 2 个同类型参数就用具名结构体。 |
| GO-API-21 | 布尔参数尤其禁止裸传。用选项结构体或拆成两个函数。 |
| GO-API-22 | 返回 `error` 时放最后一位。 |
| GO-API-23 | 避免命名返回值，除用于 `defer` 修改或文档性极强的情形。 |
| GO-API-24 | 参数与返回值超过 4 个，考虑引入结构体。 |
| GO-API-25 | 函数按「构造 → 导出方法 → 未导出方法」顺序排列；同一文件内按调用关系自上而下。 |

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

## 7. 切片、映射与拷贝

| 编号 | 条款 |
|---|---|
| GO-API-26 | 跨越 API 边界时拷贝切片与映射，防止调用方与实现共享底层数组。 |
| GO-API-27 | 接收切片参数时不要保留引用（除注明「借用」的场合）。 |
| GO-API-28 | `nil` 是合法切片：返回 `nil` 切片优于返回空切片，除非调用方对 JSON 输出有要求（`nil` → `null`，空切片 → `[]`）。 |
| GO-API-29 | 映射读取必须用 `v, ok := m[k]` 区分「不存在」与「零值」。 |

```go
// 对：边界拷贝
func NewFrame(body []byte) Frame {
    return Frame{body: append([]byte(nil), body...)}
}

// 错：共享底层数组，调用方改动影响已存储的 Frame
func NewFrame(body []byte) Frame { return Frame{body: body} }
```

---

## 8. 控制流与作用域

| 编号 | 条款 |
|---|---|
| GO-API-30 | 减少嵌套：错误分支提前 `return`，正常路径保持左对齐。 |
| GO-API-31 | 删除不必要的 `else`：`if` 分支已 `return`/`continue`/`break` 时，`else` 去掉。 |
| GO-API-32 | 变量作用域收到最窄：在 `if` 语句内初始化。 |

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

## 9. 全局状态与初始化

| 编号 | 条款 |
|---|---|
| GO-API-33 | 禁止可变全局变量。用依赖注入或显式传递。 |
| GO-API-34 | 禁止 `init()`。初始化放显式构造函数，在 `main` 中编排。唯一例外：注册类副作用（`database/sql` 驱动），须注释说明。 |
| GO-API-35 | 包级变量只放常量与不可变配置。 |
| GO-API-36 | 禁止在 `init()` 里启动 goroutine。 |

---

## 10. `main` 与退出

| 编号 | 条款 |
|---|---|
| GO-API-37 | `main()` 只做编排，不含业务逻辑。业务逻辑放 `run() error`，`main` 调用后处理退出。 |
| GO-API-38 | 只在一处调用 `os.Exit`（`defer` 不会执行，多处调用必然漏清理）。 |
| GO-API-39 | 退出码语义固定：`0` 成功、`1` 一般错误、`2` 用法错误。 |

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

## 11. 时间

| 编号 | 条款 |
|---|---|
| GO-API-40 | 一律 `time.Time` 与 `time.Duration`，禁止用 `int64` 传时间戳裸值。 |
| GO-API-41 | 禁止用 `time.Sleep` 做同步（等 goroutine、等锁、等条件）。用 channel 或 `sync` 原语。 |
| GO-API-42 | 需要可测的时间，注入 `Clock` 接口或用 `testing/synctest`（1.25+）。 |
| GO-API-43 | 时区：内部存储一律 UTC，只在展示层转本地时区。 |

---

## 12. `panic` 边界

| 编号 | 条款 |
|---|---|
| GO-API-44 | 库代码禁止 `panic`。用 `error` 返回。 |
| GO-API-45 | `panic` 只用于「程序不可能继续」的编程错误（不变量破坏），且必须在包文档说明。 |
| GO-API-46 | `recover` 只允许出现在 goroutine 入口与 HTTP 中间件边界。禁止用它掩盖错误。 |

---

## 13. 一致性

最后一条，也是最难的一条。

| 编号 | 条款 |
|---|---|
| GO-API-47 | 同一包内，同一种问题用一种解法。新代码跟随既有写法，即使你认为另一种更好。 |
| GO-API-48 | 要改变既有写法，全包一次改完并在评审中说明理由。禁止新旧并存。 |
| GO-API-49 | 评审意见不得以「我个人习惯」为依据，只能引条款或引基准数据。 |
