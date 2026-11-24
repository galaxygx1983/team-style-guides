---
name: go-style-guide
description: 团队 Go 编码规范与评审基线，派生自 Uber Go Style Guide 并做裁剪与增补。当用户要写、改、评审 Go 代码，或提到"Go 规范""uber go guide""Go 风格""Go 同行评审""表驱动测试""Go 1.24 / 1.25 / 1.26 新写法""tool 指令""b.Loop""omitzero""os.Root""new(expr)"时使用。输出可引用的条款编号（GO-区域-序号）、中文条款正文与三档评审结论。只看编码规则与评审口径，不含流程、部署、CI 内容。当任务是 C++ 规范（转 cpp-style-guide）时不要使用。
agent_created: true
---

# Go 编码规范（团队版）

## 目的

给团队 Go 代码一套可执行、可引用的编码规则，并把版本口径拉到当前工具链。上游是 uber-go/guide（更新至 2023-05），本文是裁剪版。

条款编号 `GO-<区域>-<序号>`。评审意见直接引编号，例：`违反 GO-CONC-01`。

## 基线

| 项 | 取值 |
|---|---|
| 最低版本 | Go 1.24（`go.mod` 的 `go 1.24.0`） |
| 推荐版本 | Go 1.26 |
| 格式 | 交 `gofmt` + `goimports`。格式条款不进本规范正文 |
| 命名 | 条款见 `references/naming-and-api.md`，机器化配置见 `assets/.golangci.yml` |

裁剪范围：删去纯风格项（行宽、声明分组、import 顺序）；性能章节只留原则与纪律，删具体技巧；删去函数式选项模式。

## 十一条红线

命中即阻断合入。

| 编号 | 红线 |
|---|---|
| GO-CONC-01 | 禁止 fire-and-forget goroutine。每个 goroutine 必须有明确的退出路径与汇合点。 |
| GO-CONC-02 | `sync.WaitGroup` 用 `wg.Go`（1.25+）或 `Add` + `defer Done`。禁止 `Add` 之后忘记 `Done`。 |
| GO-ERR-25 | 每个返回的 `error` 必须处理。禁止 `_ = f()` 丢弃，除非注释写明理由。 |
| GO-ERR-02a | 错误只处理一次：不要既记日志又返回。二选一。 |
| GO-SEC-01 | 禁止硬编码密钥、口令、令牌。含测试与示例。 |
| GO-SEC-02 | 外部输入拼路径必须用 `os.Root`（1.24+）或等价受限打开。禁止裸 `filepath.Join` 后用 `os.Open`。 |
| GO-SEC-12 | 禁止 `exec.Command("sh", "-c", 拼接串)`；参数逐个传。 |
| GO-SEC-15 | 渲染不可信数据必须用 `html/template`；禁止 `template.HTML(x)` 绕过转义。 |
| GO-MOD-01 | `go.mod` 的 `go` 行不得低于 1.24。禁止使用超出声明版本的语言特性。 |
| GO-NAM-09 | 导出标识符必须有文档注释，以标识符名开头。 |
| GO-API-20 | 禁止裸参数。超过 2 个同类型参数或任何布尔参数，改用具名结构体。 |

## 评审口径

评审意见引编号，按三档出结论。

| 档 | 内容 | 作者怎么回 |
|---|---|---|
| 必须改 | 红线命中项，以及评审人确认的实质缺陷 | 直接改，不进讨论 |
| 建议改 | 违反条款，但作者可能有正当理由 | 采纳写改法，不采纳写清理由 |
| 个人偏好 | 规范没约束的口味问题 | 可以照旧，不进结论 |

三条规矩：

1. 写代码的 AI 与评审的 AI 分开，换一个代理或换一个模型来评。
2. 红线命中不进讨论，直接改。
3. 「建议改」停在建议，不升级成必须改。

拿不准的意见单列一节，写清为什么拿不准，留到评审会上当面聊。

## 条款索引

| 文件 | 内容 |
|---|---|
| `references/naming-and-api.md` | 包名、命名、接口、接收者、结构体、函数、切片与映射、控制流、全局状态、`main` 退出、时间、`panic` 边界 |
| `references/errors.md` | 错误的产生、判断、只处理一次、不透明错误、边界转换 |
| `references/concurrency.md` | goroutine 生命周期、WaitGroup、channel、mutex、原子、context |
| `references/modern-go.md` | 1.24–1.26 新写法：`tool`、`b.Loop`、`omitzero`、`os.Root`、`new(expr)`、`errors.AsType`、`go fix` |
| `references/testing.md` | 测试代码的写法规则 |
| `references/security.md` | 密钥、注入、输入校验、密码学 |

## 与上游的取舍

| 上游条款 | 我们的做法 |
|---|---|
| 性能章节给具体技巧 | 只留原则与纪律，具体优化须附基准数据 |
| 函数式选项作为可选模式 | 不用，API 层统一配置结构体 |
| 行宽 99 字符 | 交 `gofmt` 与 lint |
