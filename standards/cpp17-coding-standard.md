# C++17 编码规范（团队版）

| 文档属性 | 内容 |
|---|---|
| 文档名称 | C++17 编码规范（团队版） |
| 文档编号 | RAIL-STD-CPP-001 |
| 版本 | V1.0 |
| 发布日期 | 2026-11-23 |
| 适用范围 | 团队安全关键 C++ 模块的编写、改写与评审 |
| 上游依据 | Google C++ Style Guide（`cppguide.md`），裁剪版；C++17 标准 |
| 条款编号 | `CPP-<区域>-<序号>`（稳定引用标识） |
| 来源 | 团队规范技能包 `cpp-style-guide-20261123`（SKILL.md 与 references 全部章节合并重排） |

## 前言

本文件给出安全关键 C++ 模块一套可执行、可引用的编码规则。

上游是 Google C++ Style Guide（`cppguide.md`），本文件是裁剪版：只留 C++，删去其余 18 种语言指南；删去格式类条款（行宽、缩进、括号位置）；删去 C++20 及以后特性（附录标注暂不适用）；删去 Google 内部工具链假设。

本文件只规定编码规则，不含流程、部署与 CI 内容。评审意见直接引用条号或条款编号，例：`违反 CPP-CONC-13`。

## 1 范围

本文件规定团队安全关键 C++ 模块的头文件、作用域、类、函数、所有权、语言特性、命名、注释、并发、C++17 惯用法、测试写法与安全编码规则，适用于团队内新建、改写与评审的全部 C++17 代码。本文件不规定流程、部署与 CI 内容，格式类要求统一交由 `clang-format` 与 `clang-tidy` 执行。

## 2 规范性引用文件

下列文件中的内容通过文中的规范性引用构成本文件必不可少的条款。其中，注日期的引用文件，仅该日期对应的版本适用于本文件；不注日期的引用文件，其最新版本（包括所有的修改单）适用于本文件。

| 编号 | 文件 | 说明 |
|---|---|---|
| CPP-REF-01 | Google C++ Style Guide（`cppguide.md`） | 上游风格指南，本文件的条款来源，已裁剪 |
| CPP-REF-02 | ISO/IEC 14882（C++17） | 语言语义与标准库设施依据 |
| CPP-REF-03 | clang-format | 格式执行的机器化配置来源，见 `assets/.clang-format` |
| CPP-REF-04 | clang-tidy | 命名与静态检查的机器化配置来源，见 `assets/.clang-tidy` |

## 3 术语和定义

下列术语和定义适用于本文件。

| 术语 | 定义 |
|---|---|
| 条款编号 | 形如 `CPP-CONC-13` 的稳定标识，用于评审引用，不随章节调整而变化。 |
| 条号 | 本文件的章节内顺序号，形如 5.2.13，随文档结构调整可能变化。 |
| 数据竞争 | 两个及以上线程访问同一内存位置、至少一个为写、且无 happens-before 关系的行为；不要求同时发生。 |
| happens-before | C++ 内存模型中的执行顺序关系，决定某次访问是否与其他访问构成数据竞争。 |
| 所有权 | 对象生命周期与销毁责任的归属关系，用 `std::unique_ptr` / `std::shared_ptr` 表达。 |
| 谓词 | 条件变量等待所依据的状态判断式，必须与其所读状态受同一把互斥量保护。 |
| Status | 带失败原因的错误返回类型，用于替代异常的错误通道。 |
| RAII | 资源获取即初始化：资源生命周期与对象生命周期绑定。 |
| TOCTOU | 检查与使用之间的时间差攻击面，如「检查存在再打开」。 |
| 红线 | 命中即阻断合入的强制条款。 |

## 4 总体原则

### 4.1 总则

并发正确性不靠「看起来没问题」，靠「有锁、有顺序、有谓词、有静态/动态检查」，四条缺一条即视为未完成。安全缺陷绝大多数来自「信任了不该信任的输入」和「自己管理内存」，两条都从设计上消除，比事后补检查有效。新设施优先解决「表达意图」问题，用不用看意图是否更清楚，不看是否更短。

### 4.2 红线条款

下列条款命中即阻断合入。

| 条号 | 条款编号 | 红线 |
|---|---|---|
| 4.2.1 | CPP-CONC-01 | 同一线程外可观测的共享可变状态，必须有锁或原子保护。禁止无保护读写。 |
| 4.2.2 | CPP-CONC-07 | 多把锁必须固定顺序获取；能一次拿完就用 `std::scoped_lock`。 |
| 4.2.3 | CPP-CONC-13 | `condition_variable::wait` 必须带谓词。禁止裸 `wait(lk)`。 |
| 4.2.4 | CPP-SEC-01 | 禁止硬编码密钥、口令、令牌、证书私钥。含测试代码与注释。 |
| 4.2.5 | CPP-SEC-02 | 外部输入在边界处校验：长度、范围、编码、类型。禁止信任上游。 |
| 4.2.6 | CPP-SEC-03 | 禁止 `strcpy`/`strcat`/`sprintf`/`gets`；格式化串必须是字面量。 |
| 4.2.7 | CPP-MEM-03 | 禁止裸 `new`/`delete`。所有权用 `std::unique_ptr` / `std::shared_ptr` 表达。 |
| 4.2.8 | CPP-MEM-08 | 禁止返回指向局部变量的指针或引用；禁止保存 `std::string_view` 指向临时量。 |
| 4.2.9 | CPP-LANG-01 | 禁用异常。可失败路径返回 `Status`/`std::optional`/`bool`+出参，并在头文件注释写明失败条件。 |
| 4.2.10 | CPP-NAM-01 | 命名必须符合第 5.1 节表格：类型大驼峰、变量蛇形、类成员尾下划线、常量 `k` 前缀。 |

## 5 详细条款

### 5.1 保留条款（中文提炼）

来源：Google C++ Style Guide `cppguide.md`。本文只保留**与格式无关、与 Google 内部基建无关**的条款，删去 Pros/Cons 论证。条款编号 `CPP-<区域>-<序号>`。

格式类条款（行宽 80、缩进 2 空格、括号位置、空行、初始化列表换行）不在本文。执行 `clang-format` 即可。

---

#### 头文件（CPP-HDR）

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.1 | CPP-HDR-01 | 每个 `.cc` 应有对应 `.h`。仅放 `main()` 或多文件共享实现的 `.cc` 可无头文件。 |
| 5.1.2 | CPP-HDR-02 | 头文件必须自包含：任何 `.h` 都能被单独编译。 |
| 5.1.3 | CPP-HDR-03 | 用 `#pragma once`。不写 `#ifndef` 宏守卫。**这是与上游的分歧**（上游要求 `PROJECT_PATH_FILE_H_` 形式的宏守卫），理由见分歧表。 |
| 5.1.4 | CPP-HDR-04 | 用到什么就包含什么。禁止依赖传递包含；禁止多余包含。 |
| 5.1.5 | CPP-HDR-05 | 头文件只放声明。内联函数体仅限 10 行以内、且会被多处调用的简单访问器。 |
| 5.1.6 | CPP-HDR-06 | 尽量避免前置声明，优先包含头文件。确需减少编译依赖时，只前置声明项目自有类型，且只用于指针/引用。禁止前置声明 `std::` 下的任何符号（标准明定为未定义行为）。 |
| 5.1.7 | CPP-HDR-07 | 包含顺序：对应头文件 → C 系统头 → C++ 标准库 → 其他库 → 本项目。组间空一行，组内按字母序。 |

---

#### 作用域（CPP-SCP）

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.8 | CPP-SCP-01 | 命名空间内放代码，不放头文件级全局变量。头文件中禁止 `using namespace`。 |
| 5.1.9 | CPP-SCP-02 | 匿名命名空间替代 `static` 用于内部链接。头文件中禁止匿名命名空间。 |
| 5.1.10 | CPP-SCP-03 | 命名空间用 `namespace foo::bar {` 嵌套写法（C++17 支持）。 |
| 5.1.11 | CPP-SCP-04 | 变量在**最窄作用域**声明，且声明时初始化。禁止先声明后赋值。 |
| 5.1.12 | CPP-SCP-05 | 循环变量在 `for` 语句内声明。 |
| 5.1.13 | CPP-SCP-06 | 静态/全局对象必须可平凡构造，或提供 `Init()`/`Shutdown()`。禁止依赖动态初始化顺序。 |
| 5.1.14 | CPP-SCP-07 | 安全关键路径禁止 `thread_local`。线程池复用会串状态。 |

> 静态初始化顺序问题的落地做法：需要运行的全局逻辑，包装为 `Xxx& Instance()` 函数内静态对象（C++11 起线程安全），不要裸全局对象。

---

#### 类（CPP-CLS）

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.15 | CPP-CLS-01 | 构造函数只做「赋值 + 无失败可能的初始化」。有失败可能就改工厂函数或 `Init()`。 |
| 5.1.16 | CPP-CLS-02 | 单参数构造函数标 `explicit`，转换构造函数例外（须注释说明）。 |
| 5.1.17 | CPP-CLS-03 | 可拷贝类型必须显式写出或 `= delete` 拷贝构造/赋值。禁止隐式生成后不管。 |
| 5.1.18 | CPP-CLS-04 | 基类析构函数为 `public virtual` 或 `protected` 非虚。禁止 `public` 非虚。 |
| 5.1.19 | CPP-CLS-05 | 只有数据、无行为、无不变量 → `struct`；有不变量或非平凡成员函数 → `class`。 |
| 5.1.20 | CPP-CLS-06 | 禁止用 `std::pair`/`std::tuple` 表达有语义的组合。定义具名 `struct`。 |
| 5.1.21 | CPP-CLS-07 | 优先组合，其次继承。实现继承只用于「is-a 且需要虚函数」，禁止为复用代码继承。 |
| 5.1.22 | CPP-CLS-08 | 重载运算符仅在不引起歧义时用。禁止重载 `&&`、` |
| 5.1.23 | CPP-CLS-09 | 成员访问控制收紧：数据成员一律 `private`，除非 `struct` 且为纯数据聚合。 |
| 5.1.24 | CPP-CLS-10 | 类内声明顺序：`public:` → `protected:` → `private:`；段内：类型 → 常量 → 静态成员 → 构造/析构 → 方法 → 数据成员。 |

---

#### 函数（CPP-FUN）

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.25 | CPP-FUN-01 | 入参在前，出参在后。出参用指针（`T*`），入参用值或 `const T&`。 |
| 5.1.26 | CPP-FUN-02 | 函数体不超过 60 行（不含注释）。超过就拆，拆不动说明职责不清。 |
| 5.1.27 | CPP-FUN-03 | 重载须让读者一眼分辨语义；否则换名字。 |
| 5.1.28 | CPP-FUN-04 | 虚函数带默认参数会破坏多态分派语义，禁止。 |
| 5.1.29 | CPP-FUN-05 | 尾置返回类型（`auto f() -> T`）仅在返回类型依赖形参时用，或为对齐 lambda 写法。其他情况用前置返回类型。 |
| 5.1.30 | CPP-FUN-06 | `[[nodiscard]]` 用于返回错误码、`Status`、`optional` 的函数。丢弃返回值应产生编译警告。 |

---

#### 所有权与指针（CPP-MEM）

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.31 | CPP-MEM-01 | 动态对象单一固定所有者。所有权通过智能指针转移。 |
| 5.1.32 | CPP-MEM-02 | 独占所有权用 `std::unique_ptr`；共享所有权用 `std::shared_ptr`，并在注释写明为何需要共享。 |
| 5.1.33 | CPP-MEM-03 | 禁止裸 `new`/`delete`。禁止 `std::auto_ptr`。 |
| 5.1.34 | CPP-MEM-04 | 函数形参表达「不转移所有权」时用 `T*` 或 `const T&`，不用智能指针。 |
| 5.1.35 | CPP-MEM-05 | `std::shared_ptr` 不作为函数形参按值传递，除非要延长生命周期（此时注释说明）。 |
| 5.1.36 | CPP-MEM-06 | 裸指针只用于「非拥有、可空、需重绑定」的场合。不得用于表达所有权。 |
| 5.1.37 | CPP-MEM-07 | `std::make_unique` 优先于 `new`。C++17 已有 `make_unique`；`make_shared` 可合并分配。 |
| 5.1.38 | CPP-MEM-08 | 禁止返回指向局部变量或临时量的指针/引用。 |

---

#### 其他语言特性（CPP-LANG）

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.39 | CPP-LANG-01 | 禁用异常（`-fno-exceptions`）。可失败用返回值。第三方库抛出时在边界捕获并转 `Status`（该编译单元单独开异常）。 |
| 5.1.40 | CPP-LANG-02 | `noexcept` 只标在「确实不会抛且是移动构造/析构/简单访问器」的函数上。乱标会在违规时直接 `std::terminate`。 |
| 5.1.41 | CPP-LANG-03 | 禁用 RTTI（`-fno-rtti`）。需要运行时类型区分用 `std::variant` + `std::visit` 或显式类型标签。 |
| 5.1.42 | CPP-LANG-04 | 转型用 `static_cast`/`const_cast`/`reinterpret_cast`。禁止 C 风格转型。`dynamic_cast` 因禁用 RTTI 不可用。 |
| 5.1.43 | CPP-LANG-05 | `const` 能加就加：成员函数不改状态标 `const`；引用形参不改标 `const&`。 |
| 5.1.44 | CPP-LANG-06 | `constexpr` 用于编译期可算的常量和函数。C++17 的 `if constexpr` 见 `modern-cpp17.md`。 |
| 5.1.45 | CPP-LANG-07 | 整型：默认 `int`；明确需要范围时用 `<cstdint>` 的定宽类型。禁止用 `unsigned` 表达「非负」并参与减法。 |
| 5.1.46 | CPP-LANG-08 | 浮点比较用容差，不用 `==`。禁止用浮点做金额计算。 |
| 5.1.47 | CPP-LANG-09 | 宏只用于条件编译与头文件包含。禁止用宏定义常量、函数、类型（用 `constexpr`/`inline`/`using`）。 |
| 5.1.48 | CPP-LANG-10 | 空指针统一 `nullptr`。禁止 `NULL` 与 `0`。 |
| 5.1.49 | CPP-LANG-11 | `sizeof` 用于类型时加括号，用于变量时不加。 |
| 5.1.50 | CPP-LANG-12 | `switch` 分支规则分两种：① 非枚举条件必须有 `default`；若 `default` 不应被执行，按错误处理（记录并终止）。② 枚举条件**不写** `default`，让编译器在漏项时告警（配 `-Wswitch-enum`）。贯穿必须标 `[[fallthrough]]`；连续 `case` 标签之间无代码时免标注。 |
| 5.1.51 | CPP-LANG-13 | lambda 显式捕获，禁止 `[=]`/`[&]` 全域捕获。捕获 `this` 时写明 `[this]`。 |
| 5.1.52 | CPP-LANG-14 | 禁用 `<ratio>`、`<cfenv>`。`<filesystem>` 受限使用（见 SKILL.md 分歧表）。 |
| 5.1.53 | CPP-LANG-15 | 禁止非标准扩展（`-pedantic-errors`）。`__attribute__` 仅在封装为项目宏时允许。 |

---

#### 命名（CPP-NAM）

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.54 | CPP-NAM-01 | 命名必须符合下表全部类别规则。新增类别须先在此表登记，不得自行造风格。 |

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

#### 注释（CPP-CMT）

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.1.55 | CPP-CMT-01 | 注释解释「为什么」和「约束」，不复述代码做了什么。 |
| 5.1.56 | CPP-CMT-02 | 每个文件头写：文件用途、作者、创建日期、许可。 |
| 5.1.57 | CPP-CMT-03 | 公开类/函数必须写注释：做什么、参数含义、返回值、失败条件、线程安全性（是否可重入、是否需要外部加锁）。 |
| 5.1.58 | CPP-CMT-04 | 线程安全性必须显式声明四选一：不可重入 / 可重入 / 需外部加锁 / 内部加锁。 |
| 5.1.59 | CPP-CMT-05 | 数据成员注释在声明处或上一条，说明单位、量纲、合法范围。 |
| 5.1.60 | CPP-CMT-06 | `TODO` 格式：`// TODO(姓名或工单号): 说明`。禁止无主的 `TODO`。 |
| 5.1.61 | CPP-CMT-07 | 删除被注释掉的代码。需要留档用版本控制，不用注释。 |
| 5.1.62 | CPP-CMT-08 | 注释用中文。术语、API 名、代码标识符保留英文原形。 |

---

#### 与上游分歧表

| 上游条款 | 上游做法 | 我们的做法 | 决策依据 |
|---|---|---|---|
| 头文件守卫 | 要求 `PROJECT_PATH_FILE_H_` 形式的 `#ifndef` 宏守卫 | 用 `#pragma once` | 编译器均已支持；宏守卫要求人工维护唯一符号名，改名即失效，维护成本高于收益 |
| `<filesystem>` | 全禁 | 受限允许 | 工程需要；用「路径成分不来自外部输入」替代全禁。外部输入必须经 `security.md` 的路径校验 |
| `thread_local` | 慎用 | 安全关键路径禁用 | 线程池复用导致状态串扰，属静默故障 |
| `auto` | 偏保守，广泛禁止 | 迭代器/模板返回/lambda 可 `auto`；数值类型禁用 | 只在「类型影响正确性」处强制写全 |
| 异常 | 禁用 | 禁用，但允许第三方边界单元单独开启 | 与上游一致；补上边界处理办法 |
| C++ 版本 | C++20 | C++17 | 工具链与既有代码基线 |

#### 附录：C++20 及以上标注「暂不适用」

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

### 5.2 并发纪律

上游 cppguide 对多线程几乎无实质条款。本章为增补，针对安全关键模块：联锁逻辑、状态机、采集与输出通道。

**总则**：并发正确性不靠「看起来没问题」，靠「有锁、有顺序、有谓词、有静态/动态检查」。四条缺一条即视为未完成。

---

#### 共享可变状态与数据竞争

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.1 | CPP-CONC-01 | 任何可能被两个及以上线程访问的可变状态，必须有锁或原子保护。禁止「读不加锁、写加锁」。 |
| 5.2.2 | CPP-CONC-02 | 数据竞争按 C++ 标准定义判定：两线程访问同一内存位置、至少一个写、无 happens-before 关系。不要求同时发生。 |
| 5.2.3 | CPP-CONC-03 | 声明成员/全局共享数据时，**紧邻注释**写明保护方式：`// guarded by mu_` 或 `std::atomic`。 |
| 5.2.4 | CPP-CONC-04 | 禁止通过「只读不写」绕过加锁。若写者存在，读者必须加锁或走原子。 |
| 5.2.5 | CPP-CONC-05 | 禁止把引用/指针逃逸到锁外：锁内取得的引用，不得在锁释放后使用（容器迭代器、元素指针同此）。 |
| 5.2.6 | CPP-CONC-06 | 禁止「锁内回调外部代码」。回调可能反向上锁，形成 AB-BA 死锁。锁内只做数据搬运，回调在锁外。 |

反例与正例：

```cpp
// 错：读不加锁，读到半更新状态
if (!ready_) { /* ... */ }

// 错：元素引用逃逸到锁外
Item* GetItem(int id) {
  std::lock_guard<std::mutex> lk(mu_);
  return &items_.at(id);          // 锁释放后 items_ 可能被重置
}

// 对：锁内拷贝需要的值，锁外使用
std::optional<Item> GetItem(int id) {
  std::lock_guard<std::mutex> lk(mu_);
  auto it = items_.find(id);
  if (it == items_.end()) return std::nullopt;
  return it->second;              // 按值返回
}
```

---

#### 锁的获取顺序

死锁的根因几乎都是「加锁顺序不一致」。用**分层锁序**从设计上消除。

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.7 | CPP-CONC-07 | 设计阶段给每类锁定层级（数字越小越外层）。只允许「由外层向内层」加锁，禁止反向。 |
| 5.2.8 | CPP-CONC-08 | 需要同时持有多把锁时，用 `std::scoped_lock`（C++17，内部走 `std::lock` 的退避算法），禁止连续 `lock_guard`。 |
| 5.2.9 | CPP-CONC-09 | 锁层级写在类注释与头文件顶部，例：`// Lock order: L1 config, L2 device, L3 buffer`。 |
| 5.2.10 | CPP-CONC-10 | 禁止在持锁时调用虚函数、信号槽、用户回调、`std::function`。 |
| 5.2.11 | CPP-CONC-11 | 禁止递归锁（`std::recursive_mutex`）。需要递归说明锁粒度错误，改设计。 |
| 5.2.12 | CPP-CONC-12 | 禁止在析构函数中获取可能被其他线程持有的锁。析构前应确保对象已从共享集合移除。 |

锁层级示例（本团队参考值，新模块在此表上增补并在类注释中登记）：

| 层级 | 锁 | 说明 |
|---|---|---|
| L1 | 全局配置锁 `config_mu_` | 只在启动与重配时持有 |
| L2 | 设备/通道锁 `device_mu_` | 串口、网络通道 |
| L3 | 缓冲区/队列锁 `queue_mu_` | 采集缓冲、发送队列 |
| L4 | 统计计数器 | 优先用 `std::atomic`，不加锁 |

```cpp
// 对：一次拿两把，顺序由 std::lock 决定，不会 AB-BA
std::scoped_lock lk(config_mu_, device_mu_);

// 错：顺序写死且两处相反
{ std::lock_guard<std::mutex> a(config_mu_); std::lock_guard<std::mutex> b(device_mu_); }
{ std::lock_guard<std::mutex> a(device_mu_); std::lock_guard<std::mutex> b(config_mu_); }
```

---

#### 条件变量：必须带谓词

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.13 | CPP-CONC-13 | `wait` 必须带谓词形式：`cv_.wait(lk, [&]{ return pred; })`。禁止裸 `cv_.wait(lk)`。 |
| 5.2.14 | CPP-CONC-14 | 谓词读取的状态必须由同一把互斥量保护。 |
| 5.2.15 | CPP-CONC-15 | `notify_one`/`notify_all` 可在持锁或不持锁时调用；但**修改谓词状态必须在持锁时**。 |
| 5.2.16 | CPP-CONC-16 | 无超时的等待只用于「必须等到」的场景。可退出路径必须用 `wait_for`/`wait_until` 并处理超时。 |
| 5.2.17 | CPP-CONC-17 | 禁止用条件变量做「事件计数」；计数用原子或信号量语义的封装。 |

```cpp
// 错：裸等待，虚假唤醒后继续执行
cv_.wait(lk);

// 错：谓词读的状态没被这把锁保护
cv_.wait(lk, [&]{ return shared_flag_; });   // shared_flag_ 是 atomic 或另一把锁保护

// 对
{
  std::unique_lock<std::mutex> lk(mu_);
  cv_.wait(lk, [&]{ return !queue_.empty() || stop_; });
  if (stop_ && queue_.empty()) return false;
  item = std::move(queue_.front());
  queue_.pop_front();
}
cv_.notify_one();   // 消费后再通知生产者，或按业务需要
```

带超时的等待：

```cpp
std::unique_lock<std::mutex> lk(mu_);
if (!cv_.wait_for(lk, 200ms, [&]{ return done_ || stop_; })) {
  return Status::Timeout();   // 超时是可预期结果，必须显式返回
}
```

---

#### 原子操作与内存序

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.18 | CPP-CONC-18 | 默认用 `std::memory_order_seq_cst`（不写内存序参数即为默认）。 |
| 5.2.19 | CPP-CONC-19 | 使用 `relaxed`/`acquire`/`release` 必须满足两条：① 有注释写明「与哪一处配对」② 有对应测试或 TSan 通过记录。 |
| 5.2.20 | CPP-CONC-20 | `std::atomic<T>` 只用于单个标量或指针。多字段一致性用锁，不要用多个原子拼。 |
| 5.2.21 | CPP-CONC-21 | 禁止对 `std::atomic` 做复合读改写后再判断（`if (x.load() == 0) x.store(1);`）。用 `compare_exchange_weak` 循环。 |
| 5.2.22 | CPP-CONC-22 | `std::atomic<bool>` 只用于标志位。计数用 `std::atomic<uint64_t>` 并注意溢出。 |
| 5.2.23 | CPP-CONC-23 | 原子变量不得被 `memcpy`/`memset` 操作，不得放进 `union`。 |

内存序速查：

| 场景 | 内存序 | 说明 |
|---|---|---|
| 通用标志、停止位 | `seq_cst` | 默认，不要优化 |
| 生产者发布数据后置标志 | `release` | 置标志那一次 |
| 消费者读标志后读数据 | `acquire` | 读标志那一次 |
| 纯统计计数 | `relaxed` | 只关心最终值，不建立顺序 |

```cpp
// 对：CAS 循环
uint32_t expected = state_.load();
while (!state_.compare_exchange_weak(expected, kRunning)) {
  if (expected == kFault) return Status::InvalidState();
  // expected 已被更新为当前值，继续重试
}

// 对：release/acquire 配对，注释到位
payload_ = BuildPayload();                       // 普通写
published_.store(true, std::memory_order_release); // 与消费者 acquire 配对
```

---

#### 线程生命周期

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.24 | CPP-CONC-24 | 禁止 `detach()`。线程必须有明确的所有者与汇合点。 |
| 5.2.25 | CPP-CONC-25 | `std::thread` 必须在析构前 `join()`。C++17 无 `std::jthread`，用 RAII 封装（见下）。 |
| 5.2.26 | CPP-CONC-26 | 线程入口函数不得抛出异常（异常已全局禁用）；线程体内所有失败路径显式返回。 |
| 5.2.27 | CPP-CONC-27 | 停止线程用「置停止标志 + 通知条件变量 + join」，禁止 `pthread_cancel` 或强制终止。 |
| 5.2.28 | CPP-CONC-28 | 线程数量上限写进配置并在启动时校验。禁止按消息动态创建无上限线程。 |
| 5.2.29 | CPP-CONC-29 | 每个线程命名（`pthread_setname_np` / `SetThreadDescription`），便于 `top -H` 与抓栈定位。 |

```cpp
// C++17 的 RAII 线程封装。C++20 std::jthread 可用后替换。
class ScopedThread {
 public:
  explicit ScopedThread(std::thread t) : t_(std::move(t)) {}
  ~ScopedThread() { if (t_.joinable()) t_.join(); }
  ScopedThread(const ScopedThread&) = delete;
  ScopedThread& operator=(const ScopedThread&) = delete;
  ScopedThread(ScopedThread&&) = default;
  ScopedThread& operator=(ScopedThread&&) = default;
 private:
  std::thread t_;
};
```

---

#### 初始化与单例

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.30 | CPP-CONC-30 | 单例用函数内静态对象（C++11 起初始化线程安全）。禁止手写双重检查锁。 |
| 5.2.31 | CPP-CONC-31 | 全局对象的构造不得依赖其他全局对象的构造顺序。需要顺序就用显式 `Init()` 编排。 |
| 5.2.32 | CPP-CONC-32 | 静态局部变量的析构可能发生在其他线程仍在使用时。需要「永不析构」语义时用指针 + `new`（泄漏受控）。 |

```cpp
// 对
Config& Config::Instance() {
  static Config inst;   // 线程安全初始化
  return inst;
}

// 错：手写 DCLP，无原子则数据竞争，有原子仍需 acquire/release
if (!inst_) { std::lock_guard<std::mutex> lk(mu_); if (!inst_) inst_ = new Config; }
```

---

#### 安全关键模块附加要求

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.2.33 | CPP-CONC-33 | 联锁判定与输出驱动之间的共享状态，必须有单调递增的序号或时间戳，用于事后判定「用的是不是同一拍数据」。 |
| 5.2.34 | CPP-CONC-34 | 关键线程必须有心跳。心跳超时由看门狗判定并进入安全侧，不由该线程自己判定。 |
| 5.2.35 | CPP-CONC-35 | 禁止在线程中无限期阻塞等待外部资源而不设超时。所有阻塞点必须有上界。 |
| 5.2.36 | CPP-CONC-36 | 并发逻辑变更必须附 TSan 运行结果（零报告）。仅「跑过测试」不作为并发正确性证据。 |

---

#### 检查与定位

```bash
# 数据竞争：编译时插桩，运行时检测
cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_FLAGS="-fsanitize=thread -g -O1" ..
ctest --output-on-failure

# 死锁现场：抓所有线程栈
gdb -p <pid> -batch -ex "thread apply all bt" > stacks.txt
# 或
pstack <pid> > stacks.txt

# 锁竞争热点
perf record -e sched:sched_switch -g -- ./app && perf report
```

TSan 报告为零才算通过。TSan 与 ASan 不能同时开启，分两次构建分别运行。

---

#### 反例清单

评审时优先查这十条，出现即返工。

| 反例 | 危害 |
|---|---|
| 裸 `cv_.wait(lk)` | 虚假唤醒后读到未就绪数据 |
| 读不加锁、写加锁 | 读到撕裂值 |
| 锁外使用锁内取得的元素指针/迭代器 | 悬垂引用 |
| 两处加锁顺序相反 | 死锁，且只在特定时序复现 |
| 锁内调用回调/虚函数 | 反向加锁死锁，或长时间持锁 |
| `detach()` 的线程访问成员变量 | 对象析构后访问已释放内存 |
| 多个原子字段表达一个不变量 | 读到不一致的组合 |
| `if (a.load()==0) a.store(1)` | 检查与设置之间被插队 |
| `std::recursive_mutex` 掩盖设计问题 | 锁粒度失控，性能与正确性双损 |
| 只在 x86 上验证 | ARM 弱内存序下暴露未定义行为 |

### 5.3 C++17 现代惯用法

上游对 C++17 关键设施只讲「能不能用」，不讲「何时别用」。本章补后者。条款编号 `CPP-M17-<序号>`。

**总原则**：新设施优先解决「表达意图」问题，不是用来缩短代码行数。用不用看意图是否更清楚，不看是否更短。

---

#### `std::optional`

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.1 | CPP-M17-01 | 用于表达「可能没有值」的**返回值**。例：`std::optional<Config> ParseConfig(...)`。 |
| 5.3.2 | CPP-M17-02 | 禁止用 `optional` 表达「失败原因」。有原因就有错误信息，用 `Status`。 |
| 5.3.3 | CPP-M17-03 | 禁止用 `optional` 做函数形参来表达「可选参数」。改用重载、具名配置结构、或默认值。 |
| 5.3.4 | CPP-M17-04 | 禁止 `optional<T&>`（C++17 不支持）。需要可选引用用 `T*`。 |
| 5.3.5 | CPP-M17-05 | 取值必须走 `if (opt)` 或 `opt.has_value()`。禁止 `*opt` 与 `opt.value()` 直接取值，除非上一行已判空且不会被改动。 |
| 5.3.6 | CPP-M17-06 | 禁止 `opt.value_or(default)` 掩盖「缺失」语义，除非缺失时默认值确实是正确的业务默认。 |
| 5.3.7 | CPP-M17-07 | 成员变量用 `optional` 时注释说明「无值」代表什么业务状态。 |

```cpp
// 对：返回值表达可能缺失
std::optional<DeviceHandle> FindDevice(DeviceId id);

// 错：用 optional 当错误通道
std::optional<Config> Load();   // 失败原因丢了

// 对：错误有原因
Status Load(Config* out);
```

---

#### `std::variant`

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.8 | CPP-M17-08 | 用于**封闭集合**的择一表达（状态、消息体、解析结果）。集合开放时用多态。 |
| 5.3.9 | CPP-M17-09 | 访问一律用 `std::visit` + 重载集合，禁止按 `index()` 分支。 |
| 5.3.10 | CPP-M17-10 | 禁止嵌套超过两层。三层说明模型错了，拆成具名类型。 |
| 5.3.11 | CPP-M17-11 | 替代「基类 + 两个派生类 + `dynamic_cast`」的场景优先用 `variant`，因为项目禁 RTTI。 |
| 5.3.12 | CPP-M17-12 | `std::get<T>` 抛 `bad_variant_access`，异常已禁用。改用 `std::get_if<T>` 并处理 `nullptr`。 |
| 5.3.13 | CPP-M17-13 | `variant` 的每个备选类型必须可区分，避免两个同构类型（如两个 `int`）并存。 |

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

#### `std::string_view`

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.14 | CPP-M17-03a | 仅用于**函数形参**表达「只读字符串，不要求以 `\0` 结尾」。 |
| 5.3.15 | CPP-M17-14 | 禁止作为类成员、全局变量、容器元素长期保存。它不拥有内存。 |
| 5.3.16 | CPP-M17-15 | 禁止从返回 `string_view` 的函数返回指向临时 `std::string` 的视图。 |
| 5.3.17 | CPP-M17-16 | 禁止把 `string_view` 传给需要 C 字符串的 API（`strlen`、`printf %s`、`strcpy`）。先转 `std::string` 或显式传长度。 |
| 5.3.18 | CPP-M17-17 | `substr` 返回 `string_view`，其生命周期仍绑定原串，使用时注意原串是否还活着。 |
| 5.3.19 | CPP-M17-18 | 需要「可能不含 `\0`」且要长期持有，直接按值传 `std::string`。 |

```cpp
// 对：形参
Status ParseLine(std::string_view line, Record* out);

// 错：悬垂视图
std::string_view Bad() { std::string s = "x"; return s; }

// 错：成员长期持有
struct Name { std::string_view text; };   // 来源字符串释放即悬垂
```

---

#### 结构化绑定

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.20 | CPP-M17-19 | 用于 `std::pair`、`std::tuple`、`std::map` 遍历、多返回值函数。 |
| 5.3.21 | CPP-M17-20 | 遍历容器时必须用引用或 `const` 引用：`for (const auto& [k, v] : m)`。禁止按值绑定大对象。 |
| 5.3.22 | CPP-M17-21 | 需要写回时用 `auto&`；只读用 `const auto&`；明确要拷贝才用 `auto`。 |
| 5.3.23 | CPP-M17-22 | 需要把绑定名放进 lambda 时，C++17 **不允许**直接捕获结构化绑定名。改用捕获整个对象或先赋值给具名变量。 |
| 5.3.24 | CPP-M17-23 | 需要调试器可读或需显式类型时，不用结构化绑定，老实写两个变量。 |
| 5.3.25 | CPP-M17-24 | 绑定名要具语义：`[id, name]`，不要 `[a, b]`。 |

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

#### `if constexpr`

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.26 | CPP-M17-25 | 用于模板内的**编译期**分支，替代 SFINAE 与标签分派。 |
| 5.3.27 | CPP-M17-26 | 禁止用 `if constexpr` 做运行时配置分支。运行期分支用普通 `if`。 |
| 5.3.28 | CPP-M17-27 | 每个分支必须能独立通过编译；被丢弃的分支不实例化，但语法须正确。 |
| 5.3.29 | CPP-M17-28 | 分支超过三层时改用 `overloaded` + `visit` 或显式特化。 |

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

#### 其他 C++17 设施

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.3.30 | CPP-M17-29 | `[[nodiscard]]` 标在返回 `Status`/`optional`/错误码的函数上。违反须编译告警（配 `-Werror=unused-result`）。 |
| 5.3.31 | CPP-M17-30 | `[[maybe_unused]]` 用于条件编译下可能未使用的形参与变量。禁止用它掩盖真实未使用。 |
| 5.3.32 | CPP-M17-31 | `[[fallthrough]]` 必须写在有意贯穿的 `case` 末尾。 |
| 5.3.33 | CPP-M17-32 | `inline` 变量用于头文件常量：`inline constexpr int kMaxPorts = 8;`。替代 `static const` 与 `extern` 声明对。 |
| 5.3.34 | CPP-M17-33 | 嵌套命名空间用 `namespace a::b::c {`。禁止逐层嵌套写法。 |
| 5.3.35 | CPP-M17-34 | `std::clamp` 替代手写夹取。`std::byte` 用于原始字节，不用 `char`/`unsigned char` 表达字节语义。 |
| 5.3.36 | CPP-M17-35 | `<filesystem>` 受限使用：路径成分不得来自外部输入；外部输入必须先经 `security.md` 的路径校验。禁用 `fs::path` 隐式拼接外部字符串。 |
| 5.3.37 | CPP-M17-36 | `std::optional`/`variant`/`string_view`/结构化绑定 的新增用法必须在代码评审中说明意图，不能以「新写法」为由引入。 |

---

#### 选型决策树

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

### 5.4 测试代码的写法

本章只管「测试代码怎么写才符合规范」。覆盖率指标、CI 门禁、报告生成不属编码规范，不在本章。

---

#### 结构与命名

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.4.1 | CPP-TCODE-01 | 测试文件与被测文件同名加 `_test`：`interlock.cc` → `interlock_test.cc`，与被测文件同目录。 |
| 5.4.2 | CPP-TCODE-02 | 一个测试只验证一个行为。名字写清「场景 + 预期」。 |
| 5.4.3 | CPP-TCODE-03 | 命名 `TEST(模块, 行为_条件_预期)`，例 `TEST(Interlock, RejectsMove_WhenSignalRed)`。 |
| 5.4.4 | CPP-TCODE-04 | 结构三段：Arrange / Act / Assert，段间空行。禁止把断言混进准备段。 |
| 5.4.5 | CPP-TCODE-05 | 断言用 `EXPECT_*`（失败继续）与 `ASSERT_*`（失败中止）区分：前置条件用 `ASSERT`，后续仍可验证用 `EXPECT`。 |
| 5.4.6 | CPP-TCODE-06 | 测试代码同样适用本规范全部条款（命名、所有权、并发、安全）。测试代码不是免责区。 |

```cpp
TEST(Interlock, RejectsMove_WhenSignalRed) {
  Fixture f;                                  // Arrange
  f.SetSignal(Signal::kRed);
  f.SetRoute(Route::kMainToSiding);

  const Status st = f.interlock().RequestMove(Route::kMainToSiding);   // Act

  EXPECT_EQ(st.code(), StatusCode::kDenied);  // Assert
  EXPECT_FALSE(f.output().IsMoveCommandActive());
}
```

---

#### 可重复性

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.4.7 | CPP-TCODE-07 | 测试必须可重复：不依赖执行顺序、外部网络、真实时间、机器性能。 |
| 5.4.8 | CPP-TCODE-08 | 禁止在测试中 `sleep`。需要时间推进就注入时钟（见第 4 节）。 |
| 5.4.9 | CPP-TCODE-09 | 禁止测试私有成员。需要测私有逻辑，说明该逻辑应提为独立可测单元。 |
| 5.4.10 | CPP-TCODE-10 | 禁止 `try/catch` 吞掉失败。测试失败必须让测试失败。 |

---

#### 参数化与夹具

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.4.11 | CPP-TCODE-11 | 同一逻辑多组输入用 `TEST_P` + `INSTANTIATE_TEST_SUITE_P`，不复制粘贴多个 `TEST`。 |
| 5.4.12 | CPP-TCODE-12 | 参数化用例名必须带可读参数描述（第四个参数用 lambda 生成 `name`），失败时能直接定位输入。 |
| 5.4.13 | CPP-TCODE-13 | 共享准备逻辑放 `::testing::Test` 夹具的 `SetUp`/`TearDown`。禁止用全局变量在测试间传状态。 |
| 5.4.14 | CPP-TCODE-14 | 夹具 `TearDown` 必须恢复所有被改的全局状态（单例、静态配置）。 |

```cpp
class ParseTest : public ::testing::TestWithParam<std::tuple<std::string, bool>> {};

TEST_P(ParseTest, AcceptsOnlyValidFrames) {
  const auto& [input, expect_ok] = GetParam();
  EXPECT_EQ(ParseFrame(input).ok(), expect_ok);
}

INSTANTIATE_TEST_SUITE_P(
    Frames, ParseTest,
    ::testing::Values(
        std::make_tuple(std::string(""), false),
        std::make_tuple(std::string("AA55"), true)),
    [](const auto& info) {
      return std::get<1>(info.param) ? "Valid" : "Invalid";
    });
```

---

#### 时间与并发的可测性

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.4.15 | CPP-TCODE-15 | 被测代码不得直接调用 `std::this_thread::sleep_for` 或 `steady_clock::now`。通过注入的 `IClock` 获取时间。 |
| 5.4.16 | CPP-TCODE-16 | 测试用假时钟推进时间，不用真实等待。 |
| 5.4.17 | CPP-TCODE-17 | 并发用例用「栅栏 + 原子计数」控制时序，不用 `sleep` 凑时序。 |

```cpp
class FakeClock : public IClock {
 public:
  TimePoint Now() const override { return now_; }
  void Advance(Duration d) { now_ += d; }
 private:
  TimePoint now_{};
};
```

---

#### 替换与边界

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.4.18 | CPP-TCODE-18 | 只 mock 边界（I/O、时钟、网络、硬件寄存器）。禁止 mock 被测类自己的内部协作对象。 |
| 5.4.19 | CPP-TCODE-19 | 需要 mock 的依赖必须是接口，且接口定义在消费方。 |
| 5.4.20 | CPP-TCODE-20 | 每个 `assert`/`CHECK` 触发的进程终止，必须有 `EXPECT_DEATH` 测试覆盖。 |
| 5.4.21 | CPP-TCODE-21 | 禁止为测试在生产代码里加 `if (testing)` 分支。 |
| 5.4.22 | CPP-TCODE-22 | 断言消息写「期望什么、实际什么、输入是什么」，不写「assertion failed」。 |

### 5.5 安全审查维度

上游无任何安全条款。本章为增补，用于评审与自查。条款编号 `CPP-SEC-<序号>`。

**总原则**：安全缺陷绝大多数来自「信任了不该信任的输入」和「自己管理内存」。两条都从设计上消除，比事后补检查有效。

---

#### 密钥与凭据

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.1 | CPP-SEC-01 | 禁止硬编码密钥、口令、令牌、证书私钥、连接串。含测试代码、注释、示例配置、构建脚本。 |
| 5.5.2 | CPP-SEC-04 | 凭据从密钥管理服务、环境变量或受权限保护的文件读取。启动时校验存在性，缺失即拒绝启动，不进降级模式。 |
| 5.5.3 | CPP-SEC-05 | 凭据不得进入日志、异常消息、崩溃转储、命令行参数（命令行可被 `ps` 看到）。 |
| 5.5.4 | CPP-SEC-06 | 内存中的密钥用后清零：`std::fill` / `explicit_bzero` / `SecureZeroMemory`。`memset` 可能被优化掉。 |
| 5.5.5 | CPP-SEC-07 | 密钥不得出现在 Git 历史。已提交的密钥视为已泄露，必须轮换，不是回滚。 |

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

#### 注入类缺陷

##### 2.1 命令注入

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.6 | CPP-SEC-08 | 禁止把外部输入拼接进 `system()`、`popen()`、`exec*` 字符串。 |
| 5.5.7 | CPP-SEC-09 | 需要执行外部程序时用 `execv`/`posix_spawn` 传参数数组，不经过 shell。 |
| 5.5.8 | CPP-SEC-10 | 参数中的每个值做白名单校验（字符集 + 长度）。 |

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

##### 2.2 格式化串

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.9 | CPP-SEC-11 | 格式化串必须是字面量。禁止把外部输入当格式化串传。 |
| 5.5.10 | CPP-SEC-12 | 编译器开 `-Wformat -Wformat-security`，并把告警当错误。 |

```cpp
printf(user_msg);              // 错：user_msg = "%s%s%s%s" 直接崩溃或泄内存
printf("%s", user_msg);        // 对
```

##### 2.3 SQL / 查询注入

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.11 | CPP-SEC-13 | 禁止字符串拼接生成 SQL。一律用参数化查询或预编译语句。 |
| 5.5.12 | CPP-SEC-14 | 表名、列名等无法参数化的标识符，必须对照白名单映射，不接受外部原值。 |

##### 2.4 路径穿越

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.13 | CPP-SEC-15 | 外部输入参与路径构造前，必须规范化并校验前缀落在允许根目录内。 |
| 5.5.14 | CPP-SEC-16 | 校验在 `canonical` 之后做，不在原始字符串上做（`..` 与符号链接都能绕过字符串比较）。 |
| 5.5.15 | CPP-SEC-17 | 能限定根目录的平台优先用语言/系统提供的受限打开（如受限目录句柄），不要手写拼接校验。 |
| 5.5.16 | CPP-SEC-18 | 禁止把用户输入直接传给 `fopen`/`ifstream`/`fs::path` 构造。 |

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

##### 2.5 日志注入

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.17 | CPP-SEC-19 | 外部输入写日志前，转义换行与制表符，防止伪造日志行。 |
| 5.5.18 | CPP-SEC-20 | 日志中禁止输出完整凭据、会话令牌、身份证号、手机号（后四位可保留）。 |

---

#### 输入校验

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.19 | CPP-SEC-02 | 外部输入在**边界处**校验一次：长度、范围、编码、类型、枚举取值。内部函数不再重复校验，靠类型与前置条件保证。 |
| 5.5.20 | CPP-SEC-21 | 校验必须拒绝而非修正（`clamp` 掉越界值会掩盖攻击）。 |
| 5.5.21 | CPP-SEC-22 | 长度校验在拷贝之前，基于**目标缓冲区**容量判断。 |
| 5.5.22 | CPP-SEC-23 | 枚举输入按白名单判定，不用范围判断（`if (v >= 0 && v <= 9)` 允许了非法中间值）。 |
| 5.5.23 | CPP-SEC-24 | 协议解析必须限制最大帧长、最大字段数、最大嵌套层数，防资源耗尽。 |
| 5.5.24 | CPP-SEC-25 | 校验失败的返回值要能区分「格式错」「超范围」「不支持」，便于定位与统计。 |

```cpp
// 错：修正式校验
port = std::clamp(input_port, 1, 65535);

// 对：拒绝式校验
if (input_port < 1 || input_port > 65535) return Status::OutOfRange("port");
```

---

#### 内存与整数安全

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.25 | CPP-SEC-03 | 禁用 `strcpy`/`strcat`/`sprintf`/`gets`/`alloca`。用 `std::string`、`snprintf`、`std::span` 语义的封装。 |
| 5.5.26 | CPP-SEC-26 | 任何「长度 × 元素大小」的分配，先检查乘法溢出。 |
| 5.5.27 | CPP-SEC-27 | 定宽无符号整型的减法前判大小；禁止 `size - 1` 在 `size == 0` 时执行。 |
| 5.5.28 | CPP-SEC-28 | 释放后置空指针；但置空不是所有权证明，所有权用智能指针表达。 |
| 5.5.29 | CPP-SEC-29 | 禁止把 `reinterpret_cast` 用于跨类型别名访问（违反严格别名规则，UB）。 |
| 5.5.30 | CPP-SEC-30 | 读取外部数据到结构体时，禁止直接 `memcpy` 到含指针或虚表的类型。先按字段解析。 |
| 5.5.31 | CPP-SEC-31 | 禁止 `union` 类型双关读。用 `memcpy` 或 `std::bit_cast`（C++20，暂不可用）替代方案：逐字段解析。 |

```cpp
// 错：乘法溢出后分配过小
size_t total = count * sizeof(Item);
void* p = malloc(total);                 // count 极大时 total 回绕

// 对
if (count > (SIZE_MAX / sizeof(Item))) return Status::OutOfRange("count");
size_t total = count * sizeof(Item);
```

---

#### 竞态与 TOCTOU

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.32 | CPP-SEC-32 | 「检查存在再打开」必须改为「打开后再检查」（`O_CREAT |
| 5.5.33 | CPP-SEC-33 | 临时文件用 `mkstemp` 或等效安全接口，不用可预测名字。 |
| 5.5.34 | CPP-SEC-34 | 权限敏感文件创建时显式指定权限（`0600`），不依赖 umask。 |

---

#### 编译与链接加固

| 条号 | 条款编号 | 条款 |
|---|---|---|
| 5.5.35 | CPP-SEC-35 | 发布构建必须开启：`-fstack-protector-strong`、`-D_FORTIFY_SOURCE=2`、`-fPIE -pie`、`-Wl,-z,relro,-z,now`。 |
| 5.5.36 | CPP-SEC-36 | 禁止 `-fno-stack-protector`。确有性能需求须在源码注释与评审记录中说明，并限定到具体编译单元。 |
| 5.5.37 | CPP-SEC-37 | 第三方依赖固定版本并校验哈希；禁止跟踪分支头。禁止保留有已知 CVE 的依赖版本。 |

CMake 片段：

```cmake
add_compile_options(-fstack-protector-strong -D_FORTIFY_SOURCE=2 -fPIE)
add_link_options(-pie -Wl,-z,relro,-z,now)
```

---

#### 评审清单（逐项打勾）

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

## 6 实施与评审

### 6.1 评审引用规则

评审意见只引条款编号或本文件条号，不得以个人习惯为依据。同一模块内同一种问题用一种解法；新代码跟随既有写法。要改变既有写法，一次改完并在评审中说明理由，禁止新旧并存。

### 6.2 工具与门禁

格式与命名类要求交 `clang-format`、`clang-tidy` 执行，正文不重复规定。并发逻辑变更必须附 TSan 运行结果（零报告），仅「跑过测试」不作为并发正确性证据。安全审查按第 5.5 章的评审清单逐项打勾。

### 6.3 C++20 演进

C++20 及以后特性在第 5.1 章附录中标注「暂不适用」，一律不得使用。团队评估后再单独立项；立项前按替代方案实现（如 `std::span` → `gsl::span` 或 `(T*, size_t)`，`std::jthread` → 自封装 RAII 线程类）。

## 附录 A  条款编号索引

本表用于「按条款编号定位」的场景。条款编号是稳定的对外引用标识，条号随章节调整而变化。

| 条款编号前缀 | 覆盖内容 | 所在条号段 |
|---|---|---|
| CPP-HDR | 头文件 | 5.1.1 ~ 5.1.7 |
| CPP-SCP | 作用域与初始化 | 5.1.8 ~ 5.1.14 |
| CPP-CLS | 类 | 5.1.15 ~ 5.1.24 |
| CPP-FUN | 函数 | 5.1.25 ~ 5.1.30 |
| CPP-MEM | 所有权与指针 | 5.1.31 ~ 5.1.38 |
| CPP-LANG | 语言特性 | 5.1.39 ~ 5.1.53 |
| CPP-NAM | 命名 | 5.1.54 ~ 5.1.54 |
| CPP-CMT | 注释 | 5.1.55 ~ 5.1.62 |
| CPP-CONC | 共享状态、锁顺序、条件变量、原子、线程生命周期 | 5.2.1 ~ 5.2.36 |
| CPP-M17 | `optional`/`variant`/`string_view`/结构化绑定/`if constexpr` | 5.3.1 ~ 5.3.37 |
| CPP-TCODE | 测试代码写法 | 5.4.1 ~ 5.4.22 |
| CPP-SEC | 密钥、注入、输入校验、内存与整数安全、编译加固 | 5.5.1 ~ 5.5.37 |

## 附录 B  与上游的取舍与分歧

下列条目是团队决策，不是转写遗漏。

| 上游条款 | 我们的做法 |
|---|---|
| 头文件用 `PROJECT_PATH_FILE_H_` 宏守卫 | 用 `#pragma once` |
| `<filesystem>` 全禁 | 受限允许：仅受控路径，禁止拼接外部输入 |
| `thread_local` 慎用 | 安全关键路径禁用 |
| `auto` 偏保守 | 迭代器、模板返回、lambda 形参可 `auto`；数值类型禁止推断 |
| 异常禁用 | 禁用，但允许第三方边界单元单独开启 |
| C++ 版本为 C++20 | 锁定 C++17 |

## 附录 C  修订记录

| 版本 | 日期 | 修订内容 |
|---|---|---|
| V1.0 | 2026-11-23 | 由 `cpp-style-guide-20261123` 合并重排为规范文档体例，条款原文未改。 |

