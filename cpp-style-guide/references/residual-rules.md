# 保留条款（中文提炼）

来源：Google C++ Style Guide `cppguide.md`。本文只保留**与格式无关、与 Google 内部基建无关**的条款，删去 Pros/Cons 论证。条款编号 `CPP-<区域>-<序号>`。

格式类条款（行宽 80、缩进 2 空格、括号位置、空行、初始化列表换行）不在本文。执行 `clang-format` 即可。

---

## 1. 头文件（CPP-HDR）

| 编号 | 条款 |
|---|---|
| CPP-HDR-01 | 每个 `.cc` 应有对应 `.h`。仅放 `main()` 或多文件共享实现的 `.cc` 可无头文件。 |
| CPP-HDR-02 | 头文件必须自包含：任何 `.h` 都能被单独编译。 |
| CPP-HDR-03 | 用 `#pragma once`。不写 `#ifndef` 宏守卫。**这是与上游的分歧**（上游要求 `PROJECT_PATH_FILE_H_` 形式的宏守卫），理由见分歧表。 |
| CPP-HDR-04 | 用到什么就包含什么。禁止依赖传递包含；禁止多余包含。 |
| CPP-HDR-05 | 头文件只放声明。内联函数体仅限 10 行以内、且会被多处调用的简单访问器。 |
| CPP-HDR-06 | 尽量避免前置声明，优先包含头文件。确需减少编译依赖时，只前置声明项目自有类型，且只用于指针/引用。禁止前置声明 `std::` 下的任何符号（标准明定为未定义行为）。 |
| CPP-HDR-07 | 包含顺序：对应头文件 → C 系统头 → C++ 标准库 → 其他库 → 本项目。组间空一行，组内按字母序。 |

---

## 2. 作用域（CPP-SCP）

| 编号 | 条款 |
|---|---|
| CPP-SCP-01 | 命名空间内放代码，不放头文件级全局变量。头文件中禁止 `using namespace`。 |
| CPP-SCP-02 | 匿名命名空间替代 `static` 用于内部链接。头文件中禁止匿名命名空间。 |
| CPP-SCP-03 | 命名空间用 `namespace foo::bar {` 嵌套写法（C++17 支持）。 |
| CPP-SCP-04 | 变量在**最窄作用域**声明，且声明时初始化。禁止先声明后赋值。 |
| CPP-SCP-05 | 循环变量在 `for` 语句内声明。 |
| CPP-SCP-06 | 静态/全局对象必须可平凡构造，或提供 `Init()`/`Shutdown()`。禁止依赖动态初始化顺序。 |
| CPP-SCP-07 | 安全关键路径禁止 `thread_local`。线程池复用会串状态。 |

> 静态初始化顺序问题的落地做法：需要运行的全局逻辑，包装为 `Xxx& Instance()` 函数内静态对象（C++11 起线程安全），不要裸全局对象。

---

## 3. 类（CPP-CLS）

| 编号 | 条款 |
|---|---|
| CPP-CLS-01 | 构造函数只做「赋值 + 无失败可能的初始化」。有失败可能就改工厂函数或 `Init()`。 |
| CPP-CLS-02 | 单参数构造函数标 `explicit`，转换构造函数例外（须注释说明）。 |
| CPP-CLS-03 | 可拷贝类型必须显式写出或 `= delete` 拷贝构造/赋值。禁止隐式生成后不管。 |
| CPP-CLS-04 | 基类析构函数为 `public virtual` 或 `protected` 非虚。禁止 `public` 非虚。 |
| CPP-CLS-05 | 只有数据、无行为、无不变量 → `struct`；有不变量或非平凡成员函数 → `class`。 |
| CPP-CLS-06 | 禁止用 `std::pair`/`std::tuple` 表达有语义的组合。定义具名 `struct`。 |
| CPP-CLS-07 | 优先组合，其次继承。实现继承只用于「is-a 且需要虚函数」，禁止为复用代码继承。 |
| CPP-CLS-08 | 重载运算符仅在不引起歧义时用。禁止重载 `&&`、`||`、`,`、一元 `&`。 |
| CPP-CLS-09 | 成员访问控制收紧：数据成员一律 `private`，除非 `struct` 且为纯数据聚合。 |
| CPP-CLS-10 | 类内声明顺序：`public:` → `protected:` → `private:`；段内：类型 → 常量 → 静态成员 → 构造/析构 → 方法 → 数据成员。 |

---

## 4. 函数（CPP-FUN）

| 编号 | 条款 |
|---|---|
| CPP-FUN-01 | 入参在前，出参在后。出参用指针（`T*`），入参用值或 `const T&`。 |
| CPP-FUN-02 | 函数体不超过 60 行（不含注释）。超过就拆，拆不动说明职责不清。 |
| CPP-FUN-03 | 重载须让读者一眼分辨语义；否则换名字。 |
| CPP-FUN-04 | 虚函数带默认参数会破坏多态分派语义，禁止。 |
| CPP-FUN-05 | 尾置返回类型（`auto f() -> T`）仅在返回类型依赖形参时用，或为对齐 lambda 写法。其他情况用前置返回类型。 |
| CPP-FUN-06 | `[[nodiscard]]` 用于返回错误码、`Status`、`optional` 的函数。丢弃返回值应产生编译警告。 |

---

## 5. 所有权与指针（CPP-MEM）

| 编号 | 条款 |
|---|---|
| CPP-MEM-01 | 动态对象单一固定所有者。所有权通过智能指针转移。 |
| CPP-MEM-02 | 独占所有权用 `std::unique_ptr`；共享所有权用 `std::shared_ptr`，并在注释写明为何需要共享。 |
| CPP-MEM-03 | 禁止裸 `new`/`delete`。禁止 `std::auto_ptr`。 |
| CPP-MEM-04 | 函数形参表达「不转移所有权」时用 `T*` 或 `const T&`，不用智能指针。 |
| CPP-MEM-05 | `std::shared_ptr` 不作为函数形参按值传递，除非要延长生命周期（此时注释说明）。 |
| CPP-MEM-06 | 裸指针只用于「非拥有、可空、需重绑定」的场合。不得用于表达所有权。 |
| CPP-MEM-07 | `std::make_unique` 优先于 `new`。C++17 已有 `make_unique`；`make_shared` 可合并分配。 |
| CPP-MEM-08 | 禁止返回指向局部变量或临时量的指针/引用。 |

---

## 6. 其他语言特性（CPP-LANG）

| 编号 | 条款 |
|---|---|
| CPP-LANG-01 | 禁用异常（`-fno-exceptions`）。可失败用返回值。第三方库抛出时在边界捕获并转 `Status`（该编译单元单独开异常）。 |
| CPP-LANG-02 | `noexcept` 只标在「确实不会抛且是移动构造/析构/简单访问器」的函数上。乱标会在违规时直接 `std::terminate`。 |
| CPP-LANG-03 | 禁用 RTTI（`-fno-rtti`）。需要运行时类型区分用 `std::variant` + `std::visit` 或显式类型标签。 |
| CPP-LANG-04 | 转型用 `static_cast`/`const_cast`/`reinterpret_cast`。禁止 C 风格转型。`dynamic_cast` 因禁用 RTTI 不可用。 |
| CPP-LANG-05 | `const` 能加就加：成员函数不改状态标 `const`；引用形参不改标 `const&`。 |
| CPP-LANG-06 | `constexpr` 用于编译期可算的常量和函数。C++17 的 `if constexpr` 见 `modern-cpp17.md`。 |
| CPP-LANG-07 | 整型：默认 `int`；明确需要范围时用 `<cstdint>` 的定宽类型。禁止用 `unsigned` 表达「非负」并参与减法。 |
| CPP-LANG-08 | 浮点比较用容差，不用 `==`。禁止用浮点做金额计算。 |
| CPP-LANG-09 | 宏只用于条件编译与头文件包含。禁止用宏定义常量、函数、类型（用 `constexpr`/`inline`/`using`）。 |
| CPP-LANG-10 | 空指针统一 `nullptr`。禁止 `NULL` 与 `0`。 |
| CPP-LANG-11 | `sizeof` 用于类型时加括号，用于变量时不加。 |
| CPP-LANG-12 | `switch` 分支规则分两种：① 非枚举条件必须有 `default`；若 `default` 不应被执行，按错误处理（记录并终止）。② 枚举条件**不写** `default`，让编译器在漏项时告警（配 `-Wswitch-enum`）。贯穿必须标 `[[fallthrough]]`；连续 `case` 标签之间无代码时免标注。 |
| CPP-LANG-13 | lambda 显式捕获，禁止 `[=]`/`[&]` 全域捕获。捕获 `this` 时写明 `[this]`。 |
| CPP-LANG-14 | 禁用 `<ratio>`、`<cfenv>`。`<filesystem>` 受限使用（见 SKILL.md 分歧表）。 |
| CPP-LANG-15 | 禁止非标准扩展（`-pedantic-errors`）。`__attribute__` 仅在封装为项目宏时允许。 |

---

## 7. 命名（CPP-NAM）

| 编号 | 条款 |
|---|---|
| CPP-NAM-01 | 命名必须符合下表全部类别规则。新增类别须先在此表登记，不得自行造风格。 |

| 类别 | 规则 | 示例 |
|---|---|---|
| 文件 | 全小写，`_` 分隔，`.cc` / `.h` | `http_server_logs.h` |
| 类型（类/结构/别名/枚举/模板形参/概念） | 大驼峰，无下划线 | `UrlTable`, `PropertiesMap` |
| 变量、函数形参、结构体成员 | 蛇形 | `a_local_variable` |
| 类数据成员 | 蛇形 + 尾下划线 | `table_name_` |
| 常量（`constexpr`/`const` 全局） | `k` + 大驼峰 | `kMaxRetries` |
| 函数 | 大驼峰（访问器可用蛇形小写以匹配成员名） | `OpenFile()`, `num_entries()` |
| 命名空间 | 全小写，`_` 分隔 | `rail::interlock` |
| 枚举值 | `k` + 大驼峰（`enum class` 内可省前缀） | `kOutOfMemory` |
| 宏 | 全大写下划线，项目前缀 | `RAIL_ASSERT` |

补充：

- 名字要能搜索。避免 `i`、`tmp`、`data`、`info` 这类全仓高频词做关键变量名。
- 名字长度与作用域成正比。5 行的 lambda 里 `i` 可以；头文件公开 API 里不行。
- 禁止用项目缩写造词，除非该缩写已在术语表登记。
- 禁止匈牙利命名法与类型前缀。

---

## 8. 注释（CPP-CMT）

| 编号 | 条款 |
|---|---|
| CPP-CMT-01 | 注释解释「为什么」和「约束」，不复述代码做了什么。 |
| CPP-CMT-02 | 每个文件头写：文件用途、作者、创建日期、许可。 |
| CPP-CMT-03 | 公开类/函数必须写注释：做什么、参数含义、返回值、失败条件、线程安全性（是否可重入、是否需要外部加锁）。 |
| CPP-CMT-04 | 线程安全性必须显式声明四选一：不可重入 / 可重入 / 需外部加锁 / 内部加锁。 |
| CPP-CMT-05 | 数据成员注释在声明处或上一条，说明单位、量纲、合法范围。 |
| CPP-CMT-06 | `TODO` 格式：`// TODO(姓名或工单号): 说明`。禁止无主的 `TODO`。 |
| CPP-CMT-07 | 删除被注释掉的代码。需要留档用版本控制，不用注释。 |
| CPP-CMT-08 | 注释用中文。术语、API 名、代码标识符保留英文原形。 |

---

## 9. 与上游分歧表

| 上游条款 | 上游做法 | 我们的做法 | 决策依据 |
|---|---|---|---|
| 头文件守卫 | 要求 `PROJECT_PATH_FILE_H_` 形式的 `#ifndef` 宏守卫 | 用 `#pragma once` | 编译器均已支持；宏守卫要求人工维护唯一符号名，改名即失效，维护成本高于收益 |
| `<filesystem>` | 全禁 | 受限允许 | 工程需要；用「路径成分不来自外部输入」替代全禁。外部输入必须经 `security.md` 的路径校验 |
| `thread_local` | 慎用 | 安全关键路径禁用 | 线程池复用导致状态串扰，属静默故障 |
| `auto` | 偏保守，广泛禁止 | 迭代器/模板返回/lambda 可 `auto`；数值类型禁用 | 只在「类型影响正确性」处强制写全 |
| 异常 | 禁用 | 禁用，但允许第三方边界单元单独开启 | 与上游一致；补上边界处理办法 |
| C++ 版本 | C++20 | C++17 | 工具链与既有代码基线 |

## 10. 附录：C++20 及以上标注「暂不适用」

上游已覆盖以下内容，本规范锁 C++17，**一律不得使用**。团队评估后再单独立项。

| 特性 | 标准 |
|---|---|
| Concepts / `requires` 约束 | C++20 |
| Modules（`import`/`export`） | C++20 |
| Coroutines（`co_await`/`co_return`） | C++20 |
| `std::span`、`std::format`、`std::jthread`、`std::source_location` | C++20 |
| 三路比较 `<=>`、指定初始化器（designated initializers） | C++20 |
| `std::expected`、`std::print` | C++23 |

替代方案：`std::span` → `gsl::span` 或 `(T*, size_t)`；`std::jthread` → 自封装 RAII 线程类（析构 join）；`std::format` → 项目统一格式化工具或 `snprintf` 封装；Concepts → SFINAE + `static_assert`。
