# 测试代码的写法（GO-TCODE）

本章只管「测试代码怎么写才符合规范」。覆盖率指标、CI 门禁、报告生成不属编码规范，不在本章。

---

## 1. 文件与包

| 编号 | 条款 |
|---|---|
| GO-TCODE-01 | 测试文件与被测文件同目录，`xxx_test.go`。禁止单独的 `tests/` 目录（Go 按包组织）。 |
| GO-TCODE-02 | 优先同包测试（`package interlock`）；需要验证外部视角时用 `package interlock_test`。同一包内不混用两种。 |
| GO-TCODE-03 | 测试代码同样适用本规范全部条款。测试代码不是免责区。 |

---

## 2. 命名与结构

| 编号 | 条款 |
|---|---|
| GO-TCODE-04 | 命名 `TestXxx_场景_预期`，子测试用 `t.Run` 描述输入。 |
| GO-TCODE-05 | 用 `t.Helper()` 标记断言辅助函数，使失败行号指向调用处。 |
| GO-TCODE-06 | 断言用标准库即可；引入 `testify` 须全仓统一，禁止两套断言风格并存。 |
| GO-TCODE-07 | 失败信息写「期望什么、实际什么、输入是什么」，不写「assertion failed」。 |
| GO-TCODE-08 | 一个测试只验证一个行为。出现多个不相关断言说明该拆。 |

---

## 3. 表驱动

| 编号 | 条款 |
|---|---|
| GO-TCODE-09 | 同一逻辑多组输入用表驱动。 |
| GO-TCODE-10 | 每个用例必须能独立运行：`t.Run(name, ...)`，无共享状态时加 `t.Parallel()`。 |
| GO-TCODE-11 | 用例体只允许「调用 + 断言」。出现 `if`/`switch` 分叉说明该拆成多个测试函数。 |
| GO-TCODE-12 | 用例名用可读描述，不用 `case1`、`case2`。 |
| GO-TCODE-13 | 期望值写在表里，不写在断言逻辑里。 |

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

## 4. 基准测试

| 编号 | 条款 |
|---|---|
| GO-TCODE-14 | 一律用 `b.Loop()`（1.24+），禁止 `for i := 0; i < b.N; i++`。 |
| GO-TCODE-15 | 准备数据放 `b.Loop()` 之前，并用 `b.ReportAllocs()` 报告分配。 |
| GO-TCODE-16 | 禁止为了通过基准而写「编译器能优化掉」的代码。`b.Loop` 已保证不被优化。 |

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

## 5. 可重复性与并发

| 编号 | 条款 |
|---|---|
| GO-TCODE-17 | 测试必须可重复：不依赖执行顺序、外部网络、真实时间、机器性能。 |
| GO-TCODE-18 | 禁止在测试里用 `time.Sleep` 等 goroutine。用 channel 显式同步，或 `testing/synctest`（1.25+）。 |
| GO-TCODE-19 | 测试上下文用 `t.Context()`（1.24+），不手写 `context.Background()` + `cancel`。 |
| GO-TCODE-20 | 并发时序用 `testing/synctest` 确定性验证，不用 `sleep` 凑。 |

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

## 6. 边界与替身

| 编号 | 条款 |
|---|---|
| GO-TCODE-21 | 禁止为测试在生产代码里加 `if testing` 分支。用接口注入替身。 |
| GO-TCODE-22 | 需要 mock 的依赖必须是接口，且接口定义在消费方（见 GO-API-02）。 |
| GO-TCODE-23 | 测试替身放 `xxx_internal_test.go` 或 `internal/testutil`，不导出到公开 API。 |
| GO-TCODE-24 | 可以调未导出函数，但断言应针对行为，不针对实现细节。 |
