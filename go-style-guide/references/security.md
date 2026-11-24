# 安全维度（GO-SEC）

上游无任何安全条款。本章为增补，用于评审与自查。条款编号 `GO-SEC-<序号>`。

**总原则**：Go 没有内存安全问题，但注入与凭据问题一样多。这两类全从设计上消除。

---

## 1. 密钥与凭据

| 编号 | 条款 |
|---|---|
| GO-SEC-01 | 禁止硬编码密钥、口令、令牌、私钥、DSN。含测试、示例、`_test.go`、`go:embed` 的文件。 |
| GO-SEC-04 | 凭据从环境变量、密钥管理服务或受权限保护的文件读取。启动时校验，缺失即 `return err`，不降级。 |
| GO-SEC-05 | 凭据不得进入日志、错误消息、pprof 输出、HTTP header 回显。 |
| GO-SEC-06 | 字符串不可清零（Go 的 string 不可变）。需要安全擦除时用 `[]byte` 并在用后 `for i := range b { b[i] = 0 }`；敏感临时量可考虑 1.26 的 `runtime/secret`（实验）。 |
| GO-SEC-07 | 已提交到 Git 的密钥视为已泄露，必须轮换，不是回滚。 |
| GO-SEC-08 | 禁止 `os.Getenv` 直接把凭据写进 struct 后 `%+v` 打印。实现 `String()` 方法返回脱敏形式。 |

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

## 2. 注入

### 2.1 SQL

| 编号 | 条款 |
|---|---|
| GO-SEC-09 | 禁止 `fmt.Sprintf` 拼 SQL。一律参数化：`db.QueryContext(ctx, "SELECT ... WHERE id = ?", id)`。 |
| GO-SEC-10 | 表名、列名、排序方向无法参数化，必须对照白名单映射，不接受外部原值。 |
| GO-SEC-11 | `IN (...)` 用 `sqlx.In` 或按参数个数动态生成占位符，不拼字符串。 |

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

### 2.2 命令执行

| 编号 | 条款 |
|---|---|
| GO-SEC-12 | 禁止 `exec.Command("sh", "-c", userInput)`。 |
| GO-SEC-13 | `exec.Command` 的参数逐个传，不拼成一个字符串。 |
| GO-SEC-14 | 参数值做白名单校验（字符集 + 长度）。 |

```go
// 错
exec.Command("sh", "-c", "ping -c 1 "+host)

// 对
exec.CommandContext(ctx, "ping", "-c", "1", host)   // 且 host 已白名单校验
```

### 2.3 模板

| 编号 | 条款 |
|---|---|
| GO-SEC-15 | 渲染不可信数据用 `html/template`。`text/template` 只用于非 HTML 输出（纯文本、配置文件）。 |
| GO-SEC-16 | 禁止 `template.HTML(userInput)` 绕过转义。 |
| GO-SEC-17 | 模板中禁止动态拼接模板字符串（`template.New().Parse(userInput)`），这是模板注入。 |

### 2.4 路径

| 编号 | 条款 |
|---|---|
| GO-SEC-02 | 外部输入参与路径，一律用 `os.Root`（1.24+）。 |
| GO-SEC-18 | 无 `os.Root` 时：`filepath.Clean` 后校验前缀，且校验基于绝对路径。禁止在原始字符串上比较。 |
| GO-SEC-19 | 禁止把外部输入直接传给 `os.Open`、`os.ReadFile`、`os.Create`、`http.ServeFile`。 |

```go
// 对（1.24+）
root, err := os.OpenRoot(baseDir)
if err != nil { return err }
defer root.Close()
f, err := root.Open(name)   // 越界由内核拒绝
```

### 2.5 反序列化

| 编号 | 条款 |
|---|---|
| GO-SEC-20 | 解码外部 JSON 时限制请求体大小（`http.MaxBytesReader`）。 |
| GO-SEC-21 | 禁止对外部数据用 `gob` 解码。`gob` 不做类型白名单，可触发非预期类型构造。 |
| GO-SEC-22 | `encoding/xml` 解析不可信输入时禁用 entity 展开，并限制嵌套深度。 |

---

## 3. 输入校验

| 编号 | 条款 |
|---|---|
| GO-SEC-02a | 在 HTTP handler / 消息入口处校验一次：长度、范围、枚举、格式。内部函数靠类型与前置条件。 |
| GO-SEC-23 | 校验拒绝而非修正。`math.Max(0, x)` 会掩盖攻击。 |
| GO-SEC-24 | 请求体、消息体、上传文件必须有大小上限。 |
| GO-SEC-25 | 枚举字段按白名单判定。 |
| GO-SEC-26 | 校验失败返回可区分的错误（格式错/超范围/不支持），不返回内部细节。 |
| GO-SEC-27 | 禁止用 `strconv.Atoi` 后不检查错误就使用结果。 |

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

## 4. 密码学

| 编号 | 条款 |
|---|---|
| GO-SEC-28 | 随机数：安全用途一律 `crypto/rand`。`math/rand` 只用于仿真与测试。 |
| GO-SEC-29 | 口令哈希用 `golang.org/x/crypto/argon2` 或 `bcrypt`。禁止裸 `sha256` 存口令。 |
| GO-SEC-30 | 禁止自研加密算法与自研协议。用标准库 `crypto/*`。 |
| GO-SEC-31 | 对称加密用 AES-GCM；禁止 ECB；IV 每次随机且不重用。 |
| GO-SEC-32 | 比较敏感值（MAC、令牌）用 `crypto/subtle.ConstantTimeCompare`，不用 `==`。 |
| GO-SEC-33 | TLS 最低 1.2；`tls.Config` 不设 `InsecureSkipVerify: true`（测试代码也禁止，用测试证书）。 |

```go
// 对
n, err := rand.Int(rand.Reader, big.NewInt(100))
if err != nil { return err }

// 对：常量时间比较
if subtle.ConstantTimeCompare(got, want) != 1 { return ErrUnauthorized }
```

---

## 5. HTTP 与并发安全

| 编号 | 条款 |
|---|---|
| GO-SEC-34 | 所有对外 handler 设 `ReadHeaderTimeout`、`ReadTimeout`、`WriteTimeout`、`IdleTimeout`。缺省值会让慢连接拖死服务。 |
| GO-SEC-35 | 响应头加 `X-Content-Type-Options: nosniff`；涉及 cookie 的加 `Secure`、`HttpOnly`、`SameSite`。 |
| GO-SEC-36 | CORS 白名单，禁止 `Access-Control-Allow-Origin: *` 配合凭据。 |
| GO-SEC-37 | 生产环境关闭 `pprof` 的公开暴露，或加鉴权。`/debug/pprof` 泄露堆栈与内存内容。 |
| GO-SEC-38 | 请求日志脱敏：不记录 `Authorization`、`Cookie`、请求体中的口令字段。 |

---

## 6. 依赖与构建

| 编号 | 条款 |
|---|---|
| GO-SEC-39 | 依赖固定版本（`go.mod` 精确版本 + `go.sum`）。禁止跟踪分支头。 |
| GO-SEC-40 | 禁止保留 `govulncheck ./...` 有报告（含间接依赖）的版本，合入前须为零。 |
| GO-SEC-41 | 禁止引入无维护、无 license 的依赖。新依赖须在评审中说明用途与替代方案。 |
| GO-SEC-42 | 二进制构建加 `-trimpath`，避免泄露构建机路径。 |
| GO-SEC-43 | 服务不得以 root 运行：`main` 里检查 `os.Geteuid() == 0` 则拒绝启动；需低位端口时改用 capability 或前置代理。 |

```bash
govulncheck ./...
go build -trimpath -ldflags="-s -w" ./cmd/service
```

---

## 7. 评审清单（逐项打勾）

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
