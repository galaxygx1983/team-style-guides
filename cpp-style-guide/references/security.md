# 安全审查维度（CPP-SEC）

上游无任何安全条款。本章为增补，用于评审与自查。条款编号 `CPP-SEC-<序号>`。

**总原则**：安全缺陷绝大多数来自「信任了不该信任的输入」和「自己管理内存」。两条都从设计上消除，比事后补检查有效。

---

## 1. 密钥与凭据

| 编号 | 条款 |
|---|---|
| CPP-SEC-01 | 禁止硬编码密钥、口令、令牌、证书私钥、连接串。含测试代码、注释、示例配置、构建脚本。 |
| CPP-SEC-04 | 凭据从密钥管理服务、环境变量或受权限保护的文件读取。启动时校验存在性，缺失即拒绝启动，不进降级模式。 |
| CPP-SEC-05 | 凭据不得进入日志、异常消息、崩溃转储、命令行参数（命令行可被 `ps` 看到）。 |
| CPP-SEC-06 | 内存中的密钥用后清零：`std::fill` / `explicit_bzero` / `SecureZeroMemory`。`memset` 可能被优化掉。 |
| CPP-SEC-07 | 密钥不得出现在 Git 历史。已提交的密钥视为已泄露，必须轮换，不是回滚。 |

```cpp
// 错
const char* kDbPassword = "Pa55w0rd!";

// 对：从环境或受控文件读，缺失即失败
Status LoadDbPassword(std::string* out) {
  const char* v = std::getenv("RAIL_DB_PASSWORD");
  if (v == nullptr || *v == '\0') return Status::FailedPrecondition("db password not set");
  *out = v;
  return Status::Ok();
}
```

凭据只能来自环境变量或受控文件；缺失即失败（fail closed），不得回退到内置默认口令。

---

## 2. 注入类缺陷

### 2.1 命令注入

| 编号 | 条款 |
|---|---|
| CPP-SEC-08 | 禁止把外部输入拼接进 `system()`、`popen()`、`exec*` 字符串。 |
| CPP-SEC-09 | 需要执行外部程序时用 `execv`/`posix_spawn` 传参数数组，不经过 shell。 |
| CPP-SEC-10 | 参数中的每个值做白名单校验（字符集 + 长度）。 |

```cpp
// 错
std::string cmd = "ping -c 1 " + host;   // host = "1.1.1.1; rm -rf /"
system(cmd.c_str());

// 对：参数数组 + 白名单
if (!IsValidHost(host)) return Status::InvalidArgument("bad host");
char* argv[] = {const_cast<char*>("ping"), const_cast<char*>("-c"),
                const_cast<char*>("1"), const_cast<char*>(host.c_str()), nullptr};
posix_spawn(&pid, "/bin/ping", nullptr, nullptr, argv, environ);
```

### 2.2 格式化串

| 编号 | 条款 |
|---|---|
| CPP-SEC-11 | 格式化串必须是字面量。禁止把外部输入当格式化串传。 |
| CPP-SEC-12 | 编译器开 `-Wformat -Wformat-security`，并把告警当错误。 |

```cpp
printf(user_msg);              // 错：user_msg = "%s%s%s%s" 直接崩溃或泄内存
printf("%s", user_msg);        // 对
```

### 2.3 SQL / 查询注入

| 编号 | 条款 |
|---|---|
| CPP-SEC-13 | 禁止字符串拼接生成 SQL。一律用参数化查询或预编译语句。 |
| CPP-SEC-14 | 表名、列名等无法参数化的标识符，必须对照白名单映射，不接受外部原值。 |

### 2.4 路径穿越

| 编号 | 条款 |
|---|---|
| CPP-SEC-15 | 外部输入参与路径构造前，必须规范化并校验前缀落在允许根目录内。 |
| CPP-SEC-16 | 校验在 `canonical` 之后做，不在原始字符串上做（`..` 与符号链接都能绕过字符串比较）。 |
| CPP-SEC-17 | 能限定根目录的平台优先用语言/系统提供的受限打开（如受限目录句柄），不要手写拼接校验。 |
| CPP-SEC-18 | 禁止把用户输入直接传给 `fopen`/`ifstream`/`fs::path` 构造。 |

```cpp
// 错
std::ifstream in(base + "/" + name);        // name = "../../etc/shadow"

// 对：规范化后校验前缀
Status OpenUnder(std::string_view base, std::string_view name, std::ifstream* out) {
  if (name.find('/') != std::string_view::npos ||
      name.find('\\') != std::string_view::npos) return Status::InvalidArgument("sep in name");
  std::error_code ec;
  auto root = std::filesystem::weakly_canonical(base, ec);
  auto full = std::filesystem::weakly_canonical(root / name, ec);
  if (ec) return Status::InvalidArgument("bad path");
  auto rel = full.lexically_relative(root);
  if (rel.empty() || *rel.begin() == "..") return Status::PermissionDenied("escape");
  out->open(full);
  return out->is_open() ? Status::Ok() : Status::NotFound("open failed");
}
```

### 2.5 日志注入

| 编号 | 条款 |
|---|---|
| CPP-SEC-19 | 外部输入写日志前，转义换行与制表符，防止伪造日志行。 |
| CPP-SEC-20 | 日志中禁止输出完整凭据、会话令牌、身份证号、手机号（后四位可保留）。 |

---

## 3. 输入校验

| 编号 | 条款 |
|---|---|
| CPP-SEC-02 | 外部输入在**边界处**校验一次：长度、范围、编码、类型、枚举取值。内部函数不再重复校验，靠类型与前置条件保证。 |
| CPP-SEC-21 | 校验必须拒绝而非修正（`clamp` 掉越界值会掩盖攻击）。 |
| CPP-SEC-22 | 长度校验在拷贝之前，基于**目标缓冲区**容量判断。 |
| CPP-SEC-23 | 枚举输入按白名单判定，不用范围判断（`if (v >= 0 && v <= 9)` 允许了非法中间值）。 |
| CPP-SEC-24 | 协议解析必须限制最大帧长、最大字段数、最大嵌套层数，防资源耗尽。 |
| CPP-SEC-25 | 校验失败的返回值要能区分「格式错」「超范围」「不支持」，便于定位与统计。 |

```cpp
// 错：修正式校验
port = std::clamp(input_port, 1, 65535);

// 对：拒绝式校验
if (input_port < 1 || input_port > 65535) return Status::OutOfRange("port");
```

---

## 4. 内存与整数安全

| 编号 | 条款 |
|---|---|
| CPP-SEC-03 | 禁用 `strcpy`/`strcat`/`sprintf`/`gets`/`alloca`。用 `std::string`、`snprintf`、`std::span` 语义的封装。 |
| CPP-SEC-26 | 任何「长度 × 元素大小」的分配，先检查乘法溢出。 |
| CPP-SEC-27 | 定宽无符号整型的减法前判大小；禁止 `size - 1` 在 `size == 0` 时执行。 |
| CPP-SEC-28 | 释放后置空指针；但置空不是所有权证明，所有权用智能指针表达。 |
| CPP-SEC-29 | 禁止把 `reinterpret_cast` 用于跨类型别名访问（违反严格别名规则，UB）。 |
| CPP-SEC-30 | 读取外部数据到结构体时，禁止直接 `memcpy` 到含指针或虚表的类型。先按字段解析。 |
| CPP-SEC-31 | 禁止 `union` 类型双关读。用 `memcpy` 或 `std::bit_cast`（C++20，暂不可用）替代方案：逐字段解析。 |

```cpp
// 错：乘法溢出后分配过小
size_t total = count * sizeof(Item);
void* p = malloc(total);                 // count 极大时 total 回绕

// 对
if (count > (SIZE_MAX / sizeof(Item))) return Status::OutOfRange("count");
size_t total = count * sizeof(Item);
```

---

## 5. 竞态与 TOCTOU

| 编号 | 条款 |
|---|---|
| CPP-SEC-32 | 「检查存在再打开」必须改为「打开后再检查」（`O_CREAT|O_EXCL`、`fstat` 已打开的描述符）。 |
| CPP-SEC-33 | 临时文件用 `mkstemp` 或等效安全接口，不用可预测名字。 |
| CPP-SEC-34 | 权限敏感文件创建时显式指定权限（`0600`），不依赖 umask。 |

---

## 6. 编译与链接加固

| 编号 | 条款 |
|---|---|
| CPP-SEC-35 | 发布构建必须开启：`-fstack-protector-strong`、`-D_FORTIFY_SOURCE=2`、`-fPIE -pie`、`-Wl,-z,relro,-z,now`。 |
| CPP-SEC-36 | 禁止 `-fno-stack-protector`。确有性能需求须在源码注释与评审记录中说明，并限定到具体编译单元。 |
| CPP-SEC-37 | 第三方依赖固定版本并校验哈希；禁止跟踪分支头。禁止保留有已知 CVE 的依赖版本。 |

CMake 片段：

```cmake
add_compile_options(-fstack-protector-strong -D_FORTIFY_SOURCE=2 -fPIE)
add_link_options(-pie -Wl,-z,relro,-z,now)
```

---

## 7. 评审清单（逐项打勾）

| # | 检查项 | 通过判据 |
|---|---|---|
| 1 | 有无硬编码凭据 | 全仓 grep 无明文密钥；gitleaks 零命中 |
| 2 | 外部输入是否在边界校验 | 每个入口有长度/范围/枚举校验，且拒绝而非修正 |
| 3 | 有无命令拼接 | 无 `system`/`popen` 传拼接串 |
| 4 | 格式化串 | 全部为字面量 |
| 5 | 路径构造 | 规范化后前缀校验，或有受限打开 |
| 6 | 长度/乘法溢出 | 每个分配前有溢出检查 |
| 7 | 日志内容 | 无凭据、无个人敏感信息、换行已转义 |
| 8 | 加密用法 | 无自研算法；随机数用 CSPRNG（`getrandom`/`BCryptGenRandom`），不用 `rand()` |
| 9 | 加固标志 | 发布构建开启第 6 节全部标志 |
| 10 | 依赖 | 版本固定、哈希校验、CVE 扫描通过 |

加密补充：禁止自研加密与自研随机数；哈希用 SHA-256 及以上；口令存储用 Argon2/scrypt/bcrypt，不用裸 SHA；TLS 最低 1.2 且禁用弱套件。`std::rand` 与 `std::random_device` 的实现质量不保证，安全用途一律用系统 CSPRNG。
