---
name: cpp-style-guide
description: 团队 C++17 编码规范与评审基线，派生自 Google C++ Style Guide 并做裁剪与增补。当用户要写、改、评审 C++ 代码，或提到"C++ 规范""cppguide""Google C++ 风格""C++ 评审""C++ 同行评审""C++17 惯用法""std::optional 该不该用""智能指针怎么用""加锁顺序""数据竞争"时使用。输出可引用的条款编号（CPP-区域-序号）、中文条款正文与三档评审结论。只看编码规则与评审口径，不含流程、部署、CI 内容。当任务是 Go 规范（转 go-style-guide）时不要使用。
agent_created: true
---

# C++17 编码规范（团队版）

## 目的

给安全关键 C++ 模块一套可执行、可引用的编码规则。上游是 Google C++ Style Guide（`cppguide.md`），本文是裁剪版。

条款编号 `CPP-<区域>-<序号>`。评审意见直接引编号，例：`违反 CPP-CONC-13`。

## 基线

| 项 | 取值 |
|---|---|
| 语言标准 | C++17（`-std=c++17`）。C++20/23 特性标注「暂不适用」 |
| 异常 / RTTI | 禁用（`-fno-exceptions`、`-fno-rtti`） |
| 格式 | 交 `clang-format`，配置见 `assets/.clang-format`。格式条款不进本规范正文 |
| 命名 | 条款见 `references/residual-rules.md` 第 7 节，机器化配置见 `assets/.clang-tidy` |

裁剪范围：只留 C++，删去其余 18 种语言指南；删去格式类条款（行宽、缩进、括号位置）；删去 C++20 及以后特性（附录标注暂不适用）；删去 Google 内部工具链假设。

## 十条红线

命中即阻断合入。

| 编号 | 红线 |
|---|---|
| CPP-CONC-01 | 同一线程外可观测的共享可变状态，必须有锁或原子保护。禁止无保护读写。 |
| CPP-CONC-07 | 多把锁必须固定顺序获取；能一次拿完就用 `std::scoped_lock`。 |
| CPP-CONC-13 | `condition_variable::wait` 必须带谓词。禁止裸 `wait(lk)`。 |
| CPP-SEC-01 | 禁止硬编码密钥、口令、令牌、证书私钥。含测试代码与注释。 |
| CPP-SEC-02 | 外部输入在边界处校验：长度、范围、编码、类型。禁止信任上游。 |
| CPP-SEC-03 | 禁止 `strcpy`/`strcat`/`sprintf`/`gets`；格式化串必须是字面量。 |
| CPP-MEM-03 | 禁止裸 `new`/`delete`。所有权用 `std::unique_ptr` / `std::shared_ptr` 表达。 |
| CPP-MEM-08 | 禁止返回指向局部变量的指针或引用；禁止保存 `std::string_view` 指向临时量。 |
| CPP-LANG-01 | 禁用异常。可失败路径返回 `Status`/`std::optional`/`bool`+出参，并在头文件注释写明失败条件。 |
| CPP-NAM-01 | 命名必须符合第 7 节表格：类型大驼峰、变量蛇形、类成员尾下划线、常量 `k` 前缀。 |

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
| `references/residual-rules.md` | 头文件、作用域、类、函数、所有权、语言特性、命名、注释 |
| `references/concurrency.md` | 锁顺序、数据竞争、条件变量、原子与内存序、线程生命周期 |
| `references/modern-cpp17.md` | `optional`/`variant`/`string_view`/结构化绑定/`if constexpr` 的使用与禁用场景 |
| `references/testing.md` | 测试代码的写法规则 |
| `references/security.md` | 密钥、注入、输入校验、内存与整数安全 |

## 与上游的分歧

四处团队决策，不是笔误。

| 上游条款 | 我们的做法 |
|---|---|
| 头文件用 `PROJECT_PATH_FILE_H_` 宏守卫 | 用 `#pragma once` |
| `<filesystem>` 全禁 | 受限允许：仅受控路径，禁止拼接外部输入 |
| `thread_local` 慎用 | 安全关键路径禁用 |
| `auto` 偏保守 | 迭代器、模板返回、lambda 形参可 `auto`；数值类型禁止推断 |
