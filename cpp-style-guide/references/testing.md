# 测试代码的写法（CPP-TCODE）

本章只管「测试代码怎么写才符合规范」。覆盖率指标、CI 门禁、报告生成不属编码规范，不在本章。

---

## 1. 结构与命名

| 编号 | 条款 |
|---|---|
| CPP-TCODE-01 | 测试文件与被测文件同名加 `_test`：`interlock.cc` → `interlock_test.cc`，与被测文件同目录。 |
| CPP-TCODE-02 | 一个测试只验证一个行为。名字写清「场景 + 预期」。 |
| CPP-TCODE-03 | 命名 `TEST(模块, 行为_条件_预期)`，例 `TEST(Interlock, RejectsMove_WhenSignalRed)`。 |
| CPP-TCODE-04 | 结构三段：Arrange / Act / Assert，段间空行。禁止把断言混进准备段。 |
| CPP-TCODE-05 | 断言用 `EXPECT_*`（失败继续）与 `ASSERT_*`（失败中止）区分：前置条件用 `ASSERT`，后续仍可验证用 `EXPECT`。 |
| CPP-TCODE-06 | 测试代码同样适用本规范全部条款（命名、所有权、并发、安全）。测试代码不是免责区。 |

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

## 2. 可重复性

| 编号 | 条款 |
|---|---|
| CPP-TCODE-07 | 测试必须可重复：不依赖执行顺序、外部网络、真实时间、机器性能。 |
| CPP-TCODE-08 | 禁止在测试中 `sleep`。需要时间推进就注入时钟（见第 4 节）。 |
| CPP-TCODE-09 | 禁止测试私有成员。需要测私有逻辑，说明该逻辑应提为独立可测单元。 |
| CPP-TCODE-10 | 禁止 `try/catch` 吞掉失败。测试失败必须让测试失败。 |

---

## 3. 参数化与夹具

| 编号 | 条款 |
|---|---|
| CPP-TCODE-11 | 同一逻辑多组输入用 `TEST_P` + `INSTANTIATE_TEST_SUITE_P`，不复制粘贴多个 `TEST`。 |
| CPP-TCODE-12 | 参数化用例名必须带可读参数描述（第四个参数用 lambda 生成 `name`），失败时能直接定位输入。 |
| CPP-TCODE-13 | 共享准备逻辑放 `::testing::Test` 夹具的 `SetUp`/`TearDown`。禁止用全局变量在测试间传状态。 |
| CPP-TCODE-14 | 夹具 `TearDown` 必须恢复所有被改的全局状态（单例、静态配置）。 |

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

## 4. 时间与并发的可测性

| 编号 | 条款 |
|---|---|
| CPP-TCODE-15 | 被测代码不得直接调用 `std::this_thread::sleep_for` 或 `steady_clock::now`。通过注入的 `IClock` 获取时间。 |
| CPP-TCODE-16 | 测试用假时钟推进时间，不用真实等待。 |
| CPP-TCODE-17 | 并发用例用「栅栏 + 原子计数」控制时序，不用 `sleep` 凑时序。 |

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

## 5. 替换与边界

| 编号 | 条款 |
|---|---|
| CPP-TCODE-18 | 只 mock 边界（I/O、时钟、网络、硬件寄存器）。禁止 mock 被测类自己的内部协作对象。 |
| CPP-TCODE-19 | 需要 mock 的依赖必须是接口，且接口定义在消费方。 |
| CPP-TCODE-20 | 每个 `assert`/`CHECK` 触发的进程终止，必须有 `EXPECT_DEATH` 测试覆盖。 |
| CPP-TCODE-21 | 禁止为测试在生产代码里加 `if (testing)` 分支。 |
| CPP-TCODE-22 | 断言消息写「期望什么、实际什么、输入是什么」，不写「assertion failed」。 |
