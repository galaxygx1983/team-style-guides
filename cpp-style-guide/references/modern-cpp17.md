# C++17 现代惯用法（CPP-M17）

上游对 C++17 关键设施只讲「能不能用」，不讲「何时别用」。本章补后者。条款编号 `CPP-M17-<序号>`。

**总原则**：新设施优先解决「表达意图」问题，不是用来缩短代码行数。用不用看意图是否更清楚，不看是否更短。

---

## 1. `std::optional`

| 编号 | 条款 |
|---|---|
| CPP-M17-01 | 用于表达「可能没有值」的**返回值**。例：`std::optional<Config> ParseConfig(...)`。 |
| CPP-M17-02 | 禁止用 `optional` 表达「失败原因」。有原因就有错误信息，用 `Status`。 |
| CPP-M17-03 | 禁止用 `optional` 做函数形参来表达「可选参数」。改用重载、具名配置结构、或默认值。 |
| CPP-M17-04 | 禁止 `optional<T&>`（C++17 不支持）。需要可选引用用 `T*`。 |
| CPP-M17-05 | 取值必须走 `if (opt)` 或 `opt.has_value()`。禁止 `*opt` 与 `opt.value()` 直接取值，除非上一行已判空且不会被改动。 |
| CPP-M17-06 | 禁止 `opt.value_or(default)` 掩盖「缺失」语义，除非缺失时默认值确实是正确的业务默认。 |
| CPP-M17-07 | 成员变量用 `optional` 时注释说明「无值」代表什么业务状态。 |

```cpp
// 对：返回值表达可能缺失
std::optional<DeviceHandle> FindDevice(DeviceId id);

// 错：用 optional 当错误通道
std::optional<Config> Load();   // 失败原因丢了

// 对：错误有原因
Status Load(Config* out);
```

---

## 2. `std::variant`

| 编号 | 条款 |
|---|---|
| CPP-M17-08 | 用于**封闭集合**的择一表达（状态、消息体、解析结果）。集合开放时用多态。 |
| CPP-M17-09 | 访问一律用 `std::visit` + 重载集合，禁止按 `index()` 分支。 |
| CPP-M17-10 | 禁止嵌套超过两层。三层说明模型错了，拆成具名类型。 |
| CPP-M17-11 | 替代「基类 + 两个派生类 + `dynamic_cast`」的场景优先用 `variant`，因为项目禁 RTTI。 |
| CPP-M17-12 | `std::get<T>` 抛 `bad_variant_access`，异常已禁用。改用 `std::get_if<T>` 并处理 `nullptr`。 |
| CPP-M17-13 | `variant` 的每个备选类型必须可区分，避免两个同构类型（如两个 `int`）并存。 |

```cpp
struct Idle {};
struct Running { uint32_t tick; };
struct Faulted { FaultCode code; };
using State = std::variant<Idle, Running, Faulted>;

// 对
State s = Running{42};
const char* name = std::visit([](const auto& v) -> const char* {
  using T = std::decay_t<decltype(v)>;
  if constexpr (std::is_same_v<T, Idle>)    return "Idle";
  else if constexpr (std::is_same_v<T, Running>) return "Running";
  else                                           return "Faulted";
}, s);

// 对：取值用 get_if
if (auto* r = std::get_if<Running>(&s)) { Use(r->tick); }

// 错：按 index 分支，增删备选即静默错位
if (s.index() == 1) { /* ... */ }
```

`overloaded` 辅助（放项目公共头，只写一次）：

```cpp
template <class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts> overloaded(Ts...) -> overloaded<Ts...>;   // C++17 推导指引
```

---

## 3. `std::string_view`

| 编号 | 条款 |
|---|---|
| CPP-M17-03a | 仅用于**函数形参**表达「只读字符串，不要求以 `\0` 结尾」。 |
| CPP-M17-14 | 禁止作为类成员、全局变量、容器元素长期保存。它不拥有内存。 |
| CPP-M17-15 | 禁止从返回 `string_view` 的函数返回指向临时 `std::string` 的视图。 |
| CPP-M17-16 | 禁止把 `string_view` 传给需要 C 字符串的 API（`strlen`、`printf %s`、`strcpy`）。先转 `std::string` 或显式传长度。 |
| CPP-M17-17 | `substr` 返回 `string_view`，其生命周期仍绑定原串，使用时注意原串是否还活着。 |
| CPP-M17-18 | 需要「可能不含 `\0`」且要长期持有，直接按值传 `std::string`。 |

```cpp
// 对：形参
Status ParseLine(std::string_view line, Record* out);

// 错：悬垂视图
std::string_view Bad() { std::string s = "x"; return s; }

// 错：成员长期持有
struct Name { std::string_view text; };   // 来源字符串释放即悬垂
```

---

## 4. 结构化绑定

| 编号 | 条款 |
|---|---|
| CPP-M17-19 | 用于 `std::pair`、`std::tuple`、`std::map` 遍历、多返回值函数。 |
| CPP-M17-20 | 遍历容器时必须用引用或 `const` 引用：`for (const auto& [k, v] : m)`。禁止按值绑定大对象。 |
| CPP-M17-21 | 需要写回时用 `auto&`；只读用 `const auto&`；明确要拷贝才用 `auto`。 |
| CPP-M17-22 | 需要把绑定名放进 lambda 时，C++17 **不允许**直接捕获结构化绑定名。改用捕获整个对象或先赋值给具名变量。 |
| CPP-M17-23 | 需要调试器可读或需显式类型时，不用结构化绑定，老实写两个变量。 |
| CPP-M17-24 | 绑定名要具语义：`[id, name]`，不要 `[a, b]`。 |

```cpp
// 对
for (const auto& [id, dev] : devices_) { Use(id, dev); }

// 错：按值绑定，整条记录拷贝
for (auto [id, dev] : devices_) { Use(id, dev); }

// 错（C++17）：lambda 捕获绑定名
auto [code, msg] = Split(r);
auto f = [code] { /* ... */ };      // 编译失败，改为 auto c = code; 再捕获 c
```

---

## 5. `if constexpr`

| 编号 | 条款 |
|---|---|
| CPP-M17-25 | 用于模板内的**编译期**分支，替代 SFINAE 与标签分派。 |
| CPP-M17-26 | 禁止用 `if constexpr` 做运行时配置分支。运行期分支用普通 `if`。 |
| CPP-M17-27 | 每个分支必须能独立通过编译；被丢弃的分支不实例化，但语法须正确。 |
| CPP-M17-28 | 分支超过三层时改用 `overloaded` + `visit` 或显式特化。 |

```cpp
template <class T>
void Serialize(const T& v, Buffer* out) {
  if constexpr (std::is_arithmetic_v<T>) {
    out->AppendRaw(&v, sizeof(v));
  } else {
    v.SerializeTo(out);
  }
}
```

---

## 6. 其他 C++17 设施

| 编号 | 条款 |
|---|---|
| CPP-M17-29 | `[[nodiscard]]` 标在返回 `Status`/`optional`/错误码的函数上。违反须编译告警（配 `-Werror=unused-result`）。 |
| CPP-M17-30 | `[[maybe_unused]]` 用于条件编译下可能未使用的形参与变量。禁止用它掩盖真实未使用。 |
| CPP-M17-31 | `[[fallthrough]]` 必须写在有意贯穿的 `case` 末尾。 |
| CPP-M17-32 | `inline` 变量用于头文件常量：`inline constexpr int kMaxPorts = 8;`。替代 `static const` 与 `extern` 声明对。 |
| CPP-M17-33 | 嵌套命名空间用 `namespace a::b::c {`。禁止逐层嵌套写法。 |
| CPP-M17-34 | `std::clamp` 替代手写夹取。`std::byte` 用于原始字节，不用 `char`/`unsigned char` 表达字节语义。 |
| CPP-M17-35 | `<filesystem>` 受限使用：路径成分不得来自外部输入；外部输入必须先经 `security.md` 的路径校验。禁用 `fs::path` 隐式拼接外部字符串。 |
| CPP-M17-36 | `std::optional`/`variant`/`string_view`/结构化绑定 的新增用法必须在代码评审中说明意图，不能以「新写法」为由引入。 |

---

## 7. 选型决策树

```
需要表达"可能没有值"
├─ 是返回值        → std::optional<T>
├─ 是错误路径      → Status（带原因）
├─ 是可选引用      → T*（可空指针）
└─ 是可选形参      → 重载 / 配置结构 / 默认值

需要表达"多种之一"
├─ 集合封闭、需穷尽 → std::variant + visit
└─ 集合开放、需扩展 → 抽象基类 + 虚函数

需要表达"只读字符串"
├─ 仅形参、不需持有 → std::string_view
└─ 需持有、需修改   → std::string
```

新增类型时先走这张树，再写代码。
