# 版本演进口径：Go 1.24–1.26（GO-MOD）

上游 guide 最后更新于 2023-05，基线约 Go 1.20。下表是它完全没有的内容。**不补这一章，团队会按老写法写新代码。**

条款编号 `GO-MOD-<序号>`。「门槛」列指 `go.mod` 的 `go` 行最低要求。

---

## 1. 总表

| 编号 | 条款 |
|---|---|
| GO-MOD-01 | `go.mod` 的 `go` 行是本模块的语言版本下限。不得低于 1.24；不得使用超出该行的语言或标准库特性。升级时全仓一次改完，禁止同一模块内两套版本口径。 |

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

## 2. Go 1.24

### GO-MOD-02 `tool` 指令

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

### GO-MOD-03 `testing.B.Loop`

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

### GO-MOD-04 `omitzero`

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

### GO-MOD-05 `os.Root`

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

### GO-MOD-06 泛型类型别名

```go
// 1.24+ 合法
type Set[T comparable] = map[T]struct{}
type Handler[T any] = func(T) error
```

1.24 前只能定义新的具名类型（`type Set[T comparable] map[T]struct{}`），语义不同（不共享方法集）。

### GO-MOD-07 迭代器版字符串分割

```go
// 旧：分配一个切片
for _, part := range strings.Split(csv, ",") { use(part) }

// 新（1.24+）：零分配惰性迭代
for part := range strings.SplitSeq(csv, ",") { use(part) }

// 按行
for line := range strings.Lines(text) { use(line) }
```

需要保留切片时才用 `Split`；只是遍历一律用 `*Seq` 版本。

### GO-MOD-08 `t.Context()`

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

### GO-MOD-09 `runtime.AddCleanup`

```go
// 旧：SetFinalizer 只能设一次，且对象可能被复活
runtime.SetFinalizer(obj, func(o *Obj) { o.Close() })

// 新（1.24+）：可多次添加，不复活对象，支持循环引用
runtime.AddCleanup(obj, func(f *os.File) { f.Close() }, obj.file)
```

新代码一律 `AddCleanup`。`SetFinalizer` 只在不支持的旧模块里保留。

---

## 3. Go 1.25

### GO-MOD-10 `sync.WaitGroup.Go`

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

### GO-MOD-11 `testing/synctest`

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

### GO-MOD-12 容器感知的 `GOMAXPROCS`

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

## 4. Go 1.26

### GO-MOD-13 `new(expr)`

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

### GO-MOD-14 `errors.AsType[T]`

```go
// 旧
var pathErr *fs.PathError
if errors.As(err, &pathErr) { use(pathErr.Path) }

// 新（1.26+）：类型安全、无变量声明、更快
if pathErr, ok := errors.AsType[*fs.PathError](err); ok { use(pathErr.Path) }
```

### GO-MOD-15 `go fix` 现代化器

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

### GO-MOD-16 泛型约束自引用

```go
// 1.26+ 合法
type Adder[A Adder[A]] interface { Add(A) A }
func sum[A Adder[A]](xs ...A) A
```

1.26 前这种「约束引用自身」的写法不被允许。使用场景窄，只在确实需要 F-bounded 多态时用。

---

## 5. 升级时的改法顺序

1. 把 `go.mod` 的 `go` 行升到目标版本，`go mod tidy`，`go build ./...`。
2. 删除 `automaxprocs` 空导入与依赖（1.25+）。
3. `go fix ./...` 或按单个 fixer 分批执行；每轮改完 `go build ./... && go test ./...` 全绿再进下一轮。
4. 人工复核剩余项：基准测试改 `b.Loop`、JSON 标签改 `omitzero`、路径打开改 `os.Root`、测试改 `t.Context()` 与 `synctest`。
5. 删除只为旧版本保留的兼容分支与 `//go:build` 降级标签。

---

## 6. 禁止事项

| 禁止 | 原因 |
|---|---|
| 在 `go` 行低于特性门槛时使用该特性 | 构建失败或行为不一致 |
| 用 `//go:build go1.26` 局部绕过门槛 | 同一模块内版本割裂，维护成本高于收益 |
| 保留 `tools.go` 与 `tool` 指令并存 | 两套工具版本来源，必然漂移 |
| 继续引 `automaxprocs`（1.25+） | 冗余，且可能覆盖运行时的正确值 |
| 用 `go fix` 一次性改全仓 | 改动无法逐条复核，出问题无法定点回滚 |
