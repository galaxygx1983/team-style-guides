# team-style-guides

团队编码规范，两个技能包，配套《AI 辅助开发实操手册》第三章与 4.2 第六步使用。

| 技能 | 语言 | 条款编号 | 红线 |
|---|---|---|---|
| [`go-style-guide`](go-style-guide/) | Go 1.24+ | `GO-<区域>-<序号>` | 十一条 |
| [`cpp-style-guide`](cpp-style-guide/) | C++17 | `CPP-<区域>-<序号>` | 十条 |

两个包的形状一样：`SKILL.md` 是入口，含红线表与评审口径；`references/` 按主题分册放条款正文；`assets/` 放能直接落到仓库根的机器化配置。

## 装法

把仓库地址交给编码代理，让它自己装：

```
装这个仓库里的两个技能：go-style-guide、cpp-style-guide。
https://github.com/galaxygx1983/team-style-guides
装好把两边的红线条款号列出来，我核一下版本。
```

装完确认三件事：`go-style-guide` 和 `cpp-style-guide` 两个目录都在代理读技能的目录下；`assets/.golangci.yml`、`assets/.clang-format`、`assets/.clang-tidy` 都在；红线条数对得上（十一 / 十）。

## 来源与裁剪

两个包都是裁剪版，不是照搬：

- `go-style-guide` 上游是 [uber-go/guide](https://github.com/uber-go/guide)（更新至 2023-05）。删去纯风格项（行宽、声明分组、import 顺序），性能章节只留原则与纪律，删去函数式选项模式，并把版本口径拉到 Go 1.24 / 1.26。
- `cpp-style-guide` 上游是 Google C++ Style Guide。只保留与格式无关、与 Google 内部基建无关的条款，删去 Pros/Cons 论证，删去 C++20 及以后特性（附录标注暂不适用）。

改动上游条款时，请同步改 `references/` 里对应的分册，并在提交信息里写明改了哪条编号。

## 用法的三条纪律

1. **写代码的 AI 与评审的 AI 分开。** 换一个代理或换一个模型来评，写完自己审等于没审。
2. **红线命中不进讨论，直接改。** 红线是「命中即阻断合入」，不是建议。
3. **「建议改」停在建议。** 作者有正当理由可以不采纳，写清理由即可，不升级成必须改。

拿不准的意见单列一节，写清为什么拿不准，留到评审会上当面聊。

## 只做规范，不做流程

两个包都不含流程、部署、CI 内容。需求怎么定、工单怎么拆、改动怎么灰度，看《AI 辅助开发实操手册》；这里只回答「这段代码该写成什么样」。

## 目录

```
go-style-guide/
  SKILL.md                 入口：适用场景、基线、十一条红线、评审口径
  references/
    naming-and-api.md      命名与 API 设计
    errors.md              错误处理
    concurrency.md         并发
    security.md            安全
    modern-go.md           Go 1.24 / 1.25 / 1.26 新写法
    testing.md             测试
  assets/.golangci.yml     机器化配置，落到仓库根生效

cpp-style-guide/
  SKILL.md                 入口：适用场景、基线、十条红线、评审口径
  references/
    modern-cpp17.md        C++17 惯用法
    concurrency.md         并发
    security.md            安全
    testing.md             测试
    residual-rules.md      其余条款（命名、注释、内存等）
  assets/.clang-format     格式化配置
  assets/.clang-tidy       静态检查配置
```

## 许可

内部规范，仅供团队与协作方使用。上游 Uber Go Guide 与 Google C++ Style Guide 的原始条款归各自作者，本仓库只做裁剪与增补。
