# oi-testdata-maker

一个给 AI 编码代理（agent）用的 **OI / 算法竞赛造测试数据** 技能（skill）。

把一道题的**题面**（Markdown 或 PDF）、**标程**、可选的**官方题解**变成一组可直接用于评测的 `.in` / `.out` 测试数据，可选打包成 [Hydro](https://hydro.js.org/zh/docs/Hydro/user/problem-format) 导入 zip，并可产出与题面配套的**下发样例**与**排版好的题面 / 题解 PDF**。

> 核心约定：**标程是输出的唯一真值**，题面决定输入的约束。`.out` 一律由标程运行产生，不手工编写。

## 它解决什么问题

随手用随机数生成器造出来的数据通常"能跑但没用"：约束没覆盖全、边界没打到、一半测试点的答案是常数、某条题目规则在 20 个点里从没真正影响过答案。这个 skill 把造数据当成一件有验收标准的事来做，产出包含：

- **严格按分档表生成**的 20 个测试点（每档至少有一个点取到该档上限与边界）；
- **独立的输入校验器**：重新解析即将写盘的完整文本，检查范围 / 唯一性 / 无环 / 树连通 / 特殊性质，坏数据直接 `exit(1)`；
- **防骗分自检**：把「输出 0」「漏写某条规则」这类骗分写成程序真跑一遍算出得分，纯常数策略必须 ≤ 10 分，每条规则必须在 ≥ 60% 的测试点上改变答案；
- **SPJ 双比较器**：答案不唯一（构造题 / 多解题）时同时产出 Lemon 的 `checker.cpp` 与 Hydro 的 `chk.cc`，并做真实接口回归、差分测试与 sha256 数据完整性核对；
- **下发样例**：正式比赛会另发 3~5 组逐级放大的样例，`.ans` 由标程产生并独立复核，且断言与评测点不逐字节重复（否则等于泄题）；
- **题面 / 题解 PDF**：Markdown → HTML + MathJax → 无头 Chrome 打印，含公式占位、分页、表格列宽、打印后 LaTeX 残留自检等踩坑记录。

## 主要特性

| 能力 | 说明 |
|---|---|
| 分档表驱动 | 全局约束 + 每档规模 + 特殊性质（`K=0`、严格递增、链、黑点数上限、Σn 上限…）逐档映射到测试点编号 |
| 可复现 | `mt19937_64` + 命令行种子，按测试点号重播种；有独立 `validate` 模式复核磁盘上的数据 |
| 防骗分 | `cheese_*.cpp` + harness，量化「白送了多少分」而不是凭感觉 |
| SPJ | Lemon（`registerLemonChecker`，兼容经典 Lemon 与 LemonLime）+ Hydro（`registerTestlibCmd`）两套比较器，判定规则必须一致 |
| I/O 约定 | `filename:` 以题面封面表为准；标程没 `freopen` 不等于标准输入输出，不要据此改题面 |
| PDF 题面 | 无文本层 / CID 字体时的最小 PDF 提取方案（`/ToUnicode` CMap + 坐标还原），且明确要求"读不出来就问，不臆造" |
| 材料不全 | 缺标程则自建标程并配独立对拍；缺题解则先问用户是否要生成 |
| 开工前确认 | 动手生成前把存疑点整理成一份清单与用户确认，避免返工重造全部 20 个 `.out` |

## 安装

DSH 会从 `.dsh/skills/<skill-name>/` 加载技能，把本仓库的 `SKILL.md` 放到对应目录即可：

```bash
# 项目级（只对当前项目生效）
mkdir -p .dsh/skills/oi-testdata-maker
cp SKILL.md .dsh/skills/oi-testdata-maker/SKILL.md

# 或用户级
# Windows: %USERPROFILE%\.dsh\skills\oi-testdata-maker\SKILL.md
# macOS/Linux: ~/.dsh/skills/oi-testdata-maker/SKILL.md
```

`SKILL.md` 顶部的 YAML frontmatter（`name` / `description` / `whenToUse`）就是技能清单，DSH 启动时读取。

## 使用

直接对 agent 说：

> 用 oi-testdata-maker 给 `problems/xxx` 造数据，题面是 `statement.pdf`，标程在 `source/std.cpp`

或者只给题面，让 skill 走"材料不全"的流程（自建标程、询问是否要题解）。

需要转 Hydro 时说：

> 顺便打包成 Hydro 导入 zip

## 仓库结构

```
.
├── SKILL.md      # 技能本体（frontmatter + 完整流程规范）
├── README.md     # 本文件
├── LICENSE       # MIT
└── .gitignore
```

`SKILL.md` 的组织方式：

| 章节 | 内容 |
|---|---|
| §0 | 材料清点、缺标程 / 缺题解的处理、开工前存疑点确认 |
| §1 | 通读题面、核实分档表、判定是否 SPJ |
| §2 | 编写 C++ 生成器（种子、分档映射、对抗性配方） |
| §3 | 输入校验器 |
| §4 | SPJ：Lemon `checker.cpp` + Hydro `chk.cc` |
| §5 | 编译与生成 `.out` |
| §6 | 防骗分自检 |
| §7 | 转 Hydro 导入 zip（含 `filename:` 文件 IO 约定） |
| §8 | 交付清单与质量门 |
| §9 | 下发文件（附加样例）：3~5 组逐级放大、不与评测点重复 |
| §10 | 题面 / 题解文档：Markdown → 无头 Chrome → PDF |
| 常见坑 | 实际踩过的坑，含 PDF 提取、Lemon 退出码、clamp 送分、PowerShell 编码等 |

## 适用与不适用

**适用**：OI / ICPC 风格的算法题造数据；构造题 / 多解题的 SPJ；整理下发样例；把题面 / 题解排成 PDF；把已有数据整理成 Hydro 格式。

**不适用**：交互题（`interactive`）的专项流程、题面本身都还没有的情况、需要特殊评测资源的题目。这些场景 skill 里没有覆盖，不要假装覆盖了。

## 许可证

[MIT](LICENSE)
