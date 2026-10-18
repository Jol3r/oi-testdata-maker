---
name: oi-testdata-maker
description: Use when creating rigorous .in/.out test data for OI / algorithm-contest problems from their problem statement (PDF or Markdown), standard solution(s), and optionally the official solution document — then validating inputs, producing outputs with the standard solution, and optionally packaging the result into Hydro import zips. Also covers special-judge (SPJ / 构造题 / 多解题) problems, which additionally need a custom checker for Lemon (`checker.cpp`, registerLemonChecker) and for Hydro (`chk.cc`, testlib). Also handles incomplete inputs — if no standard solution exists, write one yourself (§0.2); if no editorial exists, ask the user whether to generate one (§0.3); and settle every open question with the user before generating anything (§0.4). Beyond the scored test set it also produces the distributed sample files handed out with the statement (3~5 progressively larger `*.in`/`*.ans` pairs, cross-checked against an independent implementation and asserted not byte-identical to any scored case, §9), and renders statements and editorials to PDF through Markdown + MathJax + headless Chrome when the machine has no LaTeX/pandoc/LibreOffice (§10). It also pins the file-IO convention (`filename:`) against the statement cover table (§7).
whenToUse: User asks to 造数据/出数据/生成测试数据 based on 题面(statement)+标程(std)+题解(solution)，asks to 转换成 hydro 格式 / 打包导入 hydro，asks to 为构造题/SPJ 写 checker（Lemon 的 checker.cpp、Hydro 的 chk.cc），asks to 生成或整理下发样例 / 附加样例 / *.ans 与样例一览 README，asks to 把题面或题解 Markdown 排成 PDF（无 LaTeX / pandoc 环境、公式渲染、封面表、分页），或 asks about 文件输入输出 filename 约定。也适用于材料不全的情况（只有题面、缺标程或题解），以及"先确认再开工"的协作流程。
---

# OI Test Data Maker（根据题面 / 标程 / 题解造严谨测试数据，可转 Hydro）

目标：把一道（或多道）OI 题做成一组 20 个测试点的 `.in/.out` 测试数据，输入严格满足题面约束与各测试点规模、强度充分、生成后可复现，并（可选）打包成 [Hydro Problem Format](https://hydro.js.org/zh/docs/Hydro/user/problem-format) zip。若该题答案不唯一（构造题 / SPJ），还要按 §4 产出 Lemon 与 Hydro 两套自定义比较器。

关键前提：**标程是输出的唯一真值**，题面决定输入的约束。`.out` 一律由标程运行产生，不得手工编写；当题面描述与标程语义看起来矛盾时，以标程输出为准，并向用户说明疑点。

**三条开工铁律**（细节见 §0）：

1. **缺标程 → 自己写一份**（§0.2）。绝不允许"没有标程就照题面手写 `.out`"；自建标程必须标注来源并配独立对拍。
2. **缺题解 → 问用户要不要生成**（§0.3）。题解不是造数据的必需料，不阻塞流程，但也**不要擅自开写**。
3. **开工前先把不确定的地方问清楚**（§0.4）。在写出第一个 `.in/.out` 之前，把所有存疑点整理成一张确认清单交给用户，得到答复后再正式生成；不要边生成边问。

## 0. 材料清点与开工前确认（缺失处理 + 存疑点门禁）

### 0.1 先做一次材料清点

| 材料 | 缺失时的处理 |
|---|---|
| 题面（Markdown / PDF） | **必需**，没有就先向用户索要；无法解析全文也属"不可用"（§1.2） |
| 标程 `std` | 自己写一份（§0.2），并在交付物中标注 |
| 官方题解 / 题解 PDF | **询问用户是否生成**（§0.3），默认不阻塞 |
| 分档表（各测试点规模） | 题面没给就**必须问用户**：要用户提供，或授权你自行设计一版并回报 |
| 目标 `data` 目录 / 命名约定 | 沿用用户给的；没有就用默认约定（§2、§7）并在 README 记录 |
| 评测机与 SPJ 归属 | 不确定就问（§0.4）：LemonLime / 经典 Lemon / Hydro，是否要导出 zip |

### 0.2 没有标程：自己生成一份标程

- 产出 `std.cpp`（放 `source/std.cpp` 或 `data/<prob>/std.cpp`），严格按题面语义与数据范围实现，复杂度必须撑得住最大档（不能只会暴力）。
- 编译加 `-O2 -std=c++17`，深递归再加 `-Wl,--stack,1073741824`（§5）。
- **必须另写一份独立实现做对拍**（暴力 / 换思路的第二个正解），在小规模随机点上逐点比对。标程自身错了会一次性污染全部 20 个 `.out`，这是最贵的错误，不能靠"看起来对"放过。
- 冒烟：自建标程跑题面样例，输出必须与样例输出一致。**不一致就停下来问用户**——不要改样例去迎合标程，也不要改标程去迎合样例。
- 题面存在多种合理读法时（是否允许重复、是否取模、是否要求字典序最小、多组数据的上限…），**先把读法逐条问清再动手**；选定后写进 README 的「语义裁定」一节，并注明依据的是题面哪一句。
- 标程一旦确定即成为唯一真值（见上文关键前提）。交付时**必须显式标注"标程为本次自建、非官方"**，并列出所有靠推测补全的点，避免后来被当成官方标程。
- 自建标程会放大 SPJ 误判风险（§1.4）：答案唯一性拿不准时一律问用户。

### 0.3 没有题解：询问用户是否需要生成

- 题解**不影响造数据**，缺失不阻塞 §2 起的任何步骤；但也不要默认跳过或默认代写。
- 用一句话问清：「未找到题解，需要我补一份吗？」并给出选项：
  - **不需要**：只交付数据（默认，最省时）；
  - **需要**：生成 `<prob>-solution.md`，含题意重述、算法思路、复杂度分析、正确性说明、参考实现、以及测试点分档与部分分设计说明。
- 若用户选择生成，题解里的每个结论都要能追溯到题面或标程；不臆造复杂度、不编证明，拿不准的地方标注「存疑」。
- 题解基于自建标程时，必须注明"对应本次自建标程的解法，可能与官方解法不同"。
- 要不要把题解一并打进 Hydro 包，也要问（默认：只放题面与数据）。

### 0.4 正式生成前的存疑点确认（硬性门禁）

**执行时点**：通读题面、核实完约束（§1）之后，写生成器 / 产出第一个 `.in/.out`（§2）之前。§0.1–§0.3 是开工前的材料准备，可与 §1 的通读交错进行；§0.4 一经答复就进入正式生成。

在写生成器、产出任何 `.in/.out` 之前，先把所有存疑点整理成**一份编号清单**交给用户确认。常见存疑点：

- **题面读取**：PDF 无文本层 / 字体无 `/ToUnicode`、图片版题面、分档数字与上标只能靠坐标猜（§1.2）；
- **分档表**：题面未写各测试点规模、分档边界不明确、Σn / 总点数上限未给；
- **约定**：测试点个数（默认 20）、分值（默认均分）、文件命名、输入输出格式细节（行尾空格、多解输出顺序、是否需要输出方案数）；
- **语义**：是否 SPJ / 答案是否唯一（§1.4）、题面与标程冲突、多组数据的 `T` 上限；
- **环境**：评测机类型、时限与内存、是否导出 Hydro 包、是否有官方数据可对照；
- **交付范围**：是否需要防骗分 harness（§6）、`cheese_*.cpp`、README、题解（§0.3）。

要求：

- 每项都写出**你的默认值 / 建议**，让用户能只回复"按默认"；用户未回复的项按默认执行，并在 README 的「保留意见」一节逐条记录。
- 宁可一次问 8 条，也不要分 5 轮问；提问用编号清单或选项化提问，避免开放式长问。
- **不要边生成边问**：档位、语义、评测机一变，20 个 `.in` 与 `.out` 往往要全部重造（§5 的耗时与 sha256 复核都会失效）。
- 只有在用户明确说过"先做起来、有疑问再回来问"时，才允许按默认值先跑，同时把不确定项在 README 标为**待确认**，并提醒重跑代价。

## 1. 收集材料并核实约束

1. 先按 §0.1 清点材料、按 §0.2/§0.3 补齐标程与题解、按 §0.4 完成开工前确认；然后通读每道题的：题面（Markdown 或 PDF）、`source/std` 标程、（如有）官方题解/题解 PDF、以及目标 `data` 目录结构。
2. 若题面 / 题解是 **PDF** 且无法直接读文本（无文本层或字体用 CID/Identity 编码）：
   - 用 Python + `zlib` 手写最小 PDF 文本提取器：解析对象、FlateDecode 内容流、每个字体的 `/ToUnicode` CMap（`bfchar`/`bfrange`），按字体将字形码解码为 Unicode。
   - 保留每个字形的 `(x, y)` 坐标用于版面还原：按 y 聚类成行、行内按 x 排序。
   - 常见陷阱：PDF 会为“样例”等代码块加左侧行号列（x 明显偏左），必须按 x 列切掉；上下标 / 次方（如 `10^5`、`2·10^5`）会被拆到相邻基线，需结合列中心与上/下标偏移还原；表格列要用固定列 x 区间归类。
   - **不臆造**：读不出来就标记不确定，宁可向用户求证，也不要编数字。
3. 汇总每个测试点的“分档表”（如 `1~3 n≤5`、`4~6 n≤1e3 K=0`…）：全局约束 + 每档 n/坐标/参数上限 + 特殊性质（K=0、x 严格递增、c_i=1、所有 p 相等/互异、链、黑点数上限、Σn 上限…）。生成器必须按档位产出。
4. 顺手判定这道题是不是 **SPJ（答案不唯一）**——这决定后面要不要写比较器（§4）：
   - 题面出现「答案不唯一」「任意一组满足条件的构造均视为正确」「输出任意合法方案」「允许误差」「多解/方案题」等字样；
   - 标程是“构造出一种合法解”而不是“算出唯一值”（如固定输出一组构造、方案与题面样例不同）；
   - 输出是图/方案/序列本身（而非其长度、最值、计数）。
   - 反例：只有唯一正确答案（最短路长度、方案数、字典序最小者…）→ 仍用默认全文比较。
   - **不确定就问用户**：把该题误判成 SPJ（或漏判）都会让整场评测失效。

## 2. 编写 C++ 生成器（每个题一个）

约定：`gen_<problem>.cpp`，可执行文件支持两种模式：

- `gen.exe [seed]`：生成第 1..20 个测试点 → `1.in … 20.in`（写盘前/后都校验）；
- `gen.exe validate [seed]`：重新解析磁盘上已有 `*.in` 并全部校验（独立复核用）。

生成要点：

- 用 `mt19937_64`，命令行种子决定一切；每题内部再按测试点号重播种（如 `seed + id*1000003`），保证可复现。
- 测试点 id 1..20 与题面分档行一一映射；每档至少有一个测试取到该档上限（n=上限、坐标取到边界 0 与上限），其余覆盖中/小规模。
- 强度与对抗性：不只是随机。常见手法：
  - vis 类：S 在区间外左/右、区间内、贴近某点 ±1、取端点；K 取 0、1、n、n-1、n/2、随机；坐标聚簇/散布/含端点；需要“严格递增”档就排序输出。
  - 图论类：链 + 随机前向边、分层 DAG、随机 DAG、菱形“平局陷阱”；标签用小字母表（制造字典序平局）、全相等、全互不相同、大随机。保证无环、无重边（有序对至多一条）。
  - 键盘/状态类：c_i 全 1、L=1、所有 c_i 相等、所有 s_i 相等、同键远距字符交替（强制切换）、单键循环；p/q/t 取极端与相反大小关系（q<p 与 q>p、t=1 与 t 最大）两套都覆盖。
  - 树上类：随机树、长主干、链（1-2-…-n，深度 n-1）、星形；颜色黑白交替/前段黑/限黑点数；k=1、k=n、k 随机。
  - 每组/每题的总量约束（如 Σn≤5e5、m≤4e5）必须在生成前算好并封顶。
- 唯一性等硬约束一定遵守：坐标互异且 S 不在点集中；字符 1..m 恰出现一次；链即 max 度 ≤2 的连通树等。

## 3. 校验器（生成器内置，另存独立模式）

> 注意区分两类“校验器”：本节说的是**校验输入数据**（生成器的一部分，坏数据直接 `exit(1)`）；评价选手答案的自定义比较器（SPJ）见 §4。

- 校验必须**重新解析即将写盘的完整文本**（而不是信任内存变量），再检查：
  - 数值范围与档位上限、符号与数量完全一致（无多余/缺失 token）；
  - 唯一性 / 不重边 / 无环（Kahn）/ 树连通性与边数 n-1 / 颜色串仅含 0/1 且长度正确 / Σn 上限；
  - 该档特殊性质（K=0、递增、c=1、L=1、c 全等、s 全等、标签互异、链、黑点数…）。
  - 不满足就 `stderr` 报错并 `exit(1)`，绝不放行坏数据。
- 生成后独立再跑一遍 `validate` 模式，双重确认磁盘文件本身合法。

## 4. SPJ（构造题 / 多解题）：Lemon `checker.cpp` + Hydro `chk.cc`

答案不唯一时，全文比较会把合法解判错，必须提供自定义比较器。约定产出**两份**（判定规则必须一致）：

| 产物 | 放置位置 | 评测机 | 入口 |
|---|---|---|---|
| `checker.cpp`（编译成 `checker.exe`） | `data/<prob>/`（数据真源目录，与 `testlib.h` 同目录） | Lemon / LemonLime | lemon 模式（六参数自写分数文件）或 `registerLemonChecker` / `registerTestlibCmd` |
| `chk.cc` | `hydro/<prob>/testdata/`（与 `config.yaml` 同目录） | Hydro 导入包 | `registerTestlibCmd(argc, argv)` |

- `.out` 仍然由标程生成并随包提供（Hydro 的 `config.yaml` 必须写 `output: i.out`），但**两个比较器都不要读它**：构造题答案不唯一，把 `.out` 当真值会让“换标程/换构造”直接失效。
- 两份实现是独立代码，改一份必须同步另一份，并各自回归（§7）。

### 4.1 Lemon 侧：`checker.cpp`

先确认目标评测机风格，再决定写法：

- **LemonLime**（现代版，SPJ 基于自带的 `testlib.h`＝GitPinkRabbit 的 [Testlib-for-Lemons](https://github.com/GitPinkRabbit/Testlib-for-Lemons)，入口 `registerLemonChecker`）；
- **经典 Lemon**（老版，SPJ 自己 `fopen(argv[5],"w")` 写分数、`fopen(argv[6],"w")` 写报告）。

两者的**参数完全一致**，所以优先用 Testlib-for-Lemons 的 `registerLemonChecker` 写法——同一份 `checker.exe` 在两种评测机上都能跑：

```
checker.exe <in> <out> <ans> <full_score> <score_file> <report_file>
  argv[1] 标准输入   argv[2] 选手输出   argv[3] 标准答案（SPJ 不读）
  argv[4] 本测试点满分（整数）           argv[5] 分数文件         argv[6] 报告文件
```

骨架（把判定内核换成自己的规则即可）：

```cpp
#include "testlib.h"
#include <bits/stdc++.h>
using namespace std;

// 本地复核模式：必须在 return 前关掉 testlib 的收尾守卫
static int runValidate(const string &dir) { /* 自己 fopen 读 <dir>/i.in、i.out 并判定 */ }

int main(int argc, char *argv[]) {
    if (argc >= 2 && string(argv[1]) == "--validate") {   // 本地用法放最前
        disableFinalizeGuard();
        return runValidate(argc >= 3 ? argv[2] : ".");
    }
    registerLemonChecker(argc, argv);   // 必须在 main 的第一条可执行语句
    setName("<problem>: one-line rule summary");

    int n  = inf.readInt(1, 1000000, "n");
    int m  = ouf.readInt(1, 300, "m");
    vector<int> a(m);
    for (int i = 0; i < m; i++) a[i] = ouf.readInt(1, 1000000, format("a[%d]", i + 1).c_str());
    ouf.seekEof();                      // 多余输出判 _pe；不需要就删掉这行

    /* ... 判定 ... */
    if (!ok) quitf(_wa, "x = %lld ...", witness);
    quitf(_ok, "n = %d, m = %d: ...", n, m);
}
```

要点与易错处：

- `registerLemonChecker(argc, argv)` 要求 **argc 恰好为 7**；否则 testlib 直接报 `Number of arguments for lemon checkers must be 7.`，不写分数文件。因此不要去支持 `checker.exe <in> <out>` 这类两参数本地用法（会被拒），本地自测走 `--validate`。
- **分数与退出码全部交给 testlib**（它打开 `argv[5]` 作为分数文件）：`quitf(_ok,…)` → 写 `argv[4]`；`_wa` / `_pe` / `_unexpected_eof` / `_fail` → 写 `0`；程序总是 `exit(0)`。**Lemon 不看退出码**，只看分数文件——所以不要自己 `return 1` 表达 WA。需要部分分用 `quitp(points, …)`（LemonLime 支持 `_points`），只有对/错的题不要用。
- 用 `inf` / `ouf` 读数据：`ouf.readInt(min, max, "a[i]")` 自带范围校验，报告里是 `Integer a[1] violates the range [1, 1000000]`；常见内建消息还有 `Unexpected end of file - int32 expected`（输出为空/个数不足）、`Extra information in the output file`（`ouf.seekEof()` 抓到多余 token）、`Expected integer, but "abc" found`（非整数/溢出）。
- **报告与分数文件只写 ASCII**：不同 Lemon 版本按 UTF-8 / 本地代码页解码报告文件的行为不一致，中文可能乱码。中文解释写进题目目录的 `README.md`（做一张「报告内容 ↔ 中文含义 ↔ 分值」对照表）。
- 判定复杂度要撑住最坏数据并**实测**（例：`m≤300` 的“不超过 3 个数之和”枚举 O(m³/6)、1 MiB 标记数组，15–30 ms）。
- 编译：`g++ -O2 -std=c++17 -o checker.exe checker.cpp`（LemonLime 要求 ≥C++11，实测 `-std=c++11` 也可）；`testlib.h` 必须与 `checker.cpp` 同目录。
- LemonLime 配置：「试题」右键 → **比较模式 → 自定义校验器模式** → 填相对 data 目录的路径（如 `construct\checker.exe`）。
- 「多重集 + 选出不超过 k 个」这类规则要按**不同位置**（同一值写两遍才能用两遍）判定；题面样例里出现重复值通常就是这个语义的证据，README 要写清楚，并给出反例（如 `{1,1,1,1}` 在 `n=4` 时必须 WA、`{1,2}` 在 `n=4` 时必须 WA）。

### 4.2 Hydro 侧：`chk.cc`

- 标准 testlib：`#include "testlib.h"` + `registerTestlibCmd(argc, argv)`；`argv[1]=inf`、`argv[2]=ouf`、`argv[3]=ans`。只读 `inf`／`ouf`，`ans` 不用。
- `testdata/config.yaml` 加两行：`checker_type: testlib`、`checker: chk.cc`（同目录文件名）；20 个 `subtasks` 与 `input/output` 保持原样。
- 打包与复核脚本（如 `tools/make_hydro.py`、`tools/verify_hydro.py`）通常**硬编码**了比较器文件名：新增/改名时要同步这些脚本，并把 `chk.cc` 一起打进 zip、断言 zip 内存在该条目。
- README 里显式记录两份比较器的差异（例：Lemon 版用 `ouf.seekEof()` 判脏输出为 `_pe`，Hydro 版忽略 `m` 之后的额外 token）。
- 落地形态参考：一道构造题的产物就是「数据真源目录下的 `checker.cpp` + Hydro 包内 `testdata/chk.cc`」这一对文件，两边共用同一套判定规则、在同一边界语料上各自回归（§8 的 SPJ 五步）。

### 4.3 LemonLime 侧四个必查项（否则一律报“校验器没有被正确配置”）

LemonLime 手册（<https://project-lemonlime.github.io/Project_LemonLime/contest/>，见“自定义校验器（SPJ）说明”）
与源码（`src/core/judgingthread.cpp` 的 `lemonSpecialJudge()`）合起来看，只有四条会踩：

1. **路径相对 `data` 目录**：`specialJudge` 填的是相对**数据目录**的路径（评测时拼成 `<比赛目录>/data/<这里>`），
   测试点的 `inputFiles` / `outputFiles` 同样相对 `data`。填绝对路径、或在别处建目录联接，一律判
   `InvalidSpecialJudge`（实测 result 8），而且校验器**根本不会启动**（不会留下任何运行痕迹）。
2. **lemon 模式**：Lemon 按顺序传六个参数 `in out ans full_score score_file report_file`，
   **分数文件必须由校验器自己创建**并写入一个非负整数；`argv[3]` 可以不读（答案不唯一时本来就不该读）。
   也可以用 testlib 模式（`registerTestlibCmd`），但 lemon 模式不依赖 testlib 版本，也不会因为
   打不开答案文件而在 `_fail` 分支里崩——本 skill 实测推荐 lemon 模式。
3. **静态链接**：Lemon 启动校验器时 PATH 里没有 MinGW 的 bin 目录，动态链接版会因缺
   `libstdc++-6.dll` / `libgcc_s_seh-1.dll` 直接退出（`0xC0000135`），表现为启动失败。
   用 `g++ -O2 -std=c++14 -static` 编译。
4. **可执行文件的完整性标记**（最隐蔽，值得单独记）：Windows 启动进程时，**进程的完整性级别取可执行文件
   自身的标记**。如果校验器是在一个带 Low 标记的目录里编出来的（受限沙箱、被 DLP/ACL 工具收紧过的工作区都算），
   它启动出的进程也是 Low；Low 进程写不了 `%TEMP%`（Medium），而 Lemon 的 `_score` / `_message` 恰好在那里 ——
   校验器能启动、也 `exit 0`，但分数文件建不出来，Lemon 读不到 `_score` 就报 `InvalidSpecialJudge`。
   没有管理员权限也能修：**把校验器编到一个与数据目录同盘、但不在受限目录里的文件夹，再用 `Move-Item`
   移进来**（同盘移动保留安全描述符）；或者反过来，把整个比赛目录放到受限目录之外。

`InvalidSpecialJudge` 的判定语义（源码）：`!judge->waitForStarted(-1)` → 启动失败；
`!scoreFile.open(ReadOnly)` / 分数读不出 / 分数为负 → 也算它。`judge->exitCode() != 0` 报的是
`SpecialJudgeRunTimeError`（另一个 result code）。所以看到“校验器没有被正确配置”先查上面两类，
不要去改判定逻辑。

**验证方式要能证明“写得进分数文件”**：让校验器按 Lemon 的真实调用跑一次，把 `score_file` 指到
`%TEMP%` 下的临时目录，断言文件真的出现且内容等于满分；只跑本地 `--validate` 只能证明判定逻辑正确，
证明不了评测机启动得了它、也证明不了它能写分数文件。建议把这条做成回归脚本里的固定一步。

## 5. 编译与生成 .out

- 编译：`g++ -O2 -std=c++17`。若标程有深递归（如树上 DFS、5e5 链），编译时加 `-Wl,--stack,1073741824`，否则会栈溢出。
- 在目标数据目录内执行生成器，然后对每个 `i`：`std.exe < i.in > i.out`，检查退出码。
- 先做“样例一致性”：把题面样例输入喂给标程，输出必须与题面样例输出一致，这是整条链路的冒烟测试。（SPJ 题例外：标程输出合法即可，不必与题面样例逐字节相同——如 `1 2 3` vs 298 个数构造都对。）
- 冒烟后再批量跑全量大测试，留意耗时（大图/大 L 可能几秒）。
- 结果核查：`.in/.out` 各 20 个且非空；行数符合预期（如按 n 行 / 按组数 / 单行）；输出体量与答案数量级匹配。（SPJ 题若 20 个 `.out` 逐字节相同，通常是标程固定输出一组构造，属正常，但要在 README 说明。）

## 6. 防骗分自检（常数输出 / 单规则缺失 / 固定套路）

数据强度不只是「覆盖了题面约束」，还包括「**不能白送分**」。典型事故：一半测试点的答案恰好是 0，于是 `cout << 0` 直接拿 50 分；或者某条规则（会员券、保底 clamp、取模、绝对值）在 20 个点里几乎从不影响答案，于是「漏写这条规则」的错解被判成接近满分。这一节是**必做项**，不是可选项。

### 6.1 做法：把骗分写成程序，跑一遍，算出得分

- 每题写一个 `tools/cheese_<prob>.cpp`，用 `argv[1]` 选择策略，从标准输入读测试点、按策略输出；再用一个 harness（`tools/anticheese.py` 之类）对每个策略跑完 20 个测试点，与 `.out` 按**和判题一致的口径**比较（忽略行末空格与文末换行），得分 = 命中点数 × 每点分值。
- 输出一张表：策略 / 得分 / 上限 / 命中的测试点编号 / 是否题面固有；**任一策略超过上限就让 harness 退出非零**，这样它可以进 CI 或一键复现脚本。
- 每个策略都要有一句人能看懂的自述（`print 0`、`sum + m（忘掉全部优惠）`…），表直接抄进 README。

### 6.2 至少要覆盖的策略族

1. **纯常数**：`0`、`1`、`-1`、`n`、`m`、`x`、`1e9`、读入的某个数、答案值域的边界（mod 题的 `P-1`）。上限 **≤ 10 分**（即至多 2 个测试点）。
2. **输入里现成可见的量**：`n`、`m`、`Σa`、`max`、`n*m`、只加不减的线性式子。这些最容易被随手写出来。
3. **忽略一条规则的部分解**：漏满减 / 漏会员券 / 漏保底 clamp / 忘记取模 / 丢掉绝对值（或只保留一半符号）/ 忘乘 `n` 或 `m` / 忘排序。这类是**合法部分分**——它拿多少分**恰好等于该规则在多少个测试点上真正影响答案**，所以这是对「规则必要性」的直接测量。
4. **复杂度骗分**：暴力/低复杂度在小档必然 AC（这是刻意设计的部分分），但必须实测它在大档超时；反过来若暴力在小档就超限，说明分档没起到作用。
5. **格式骗分**：只输出第一问、只输出一行、输出 `-1`、输出空行、输出多余内容。
6. **对拍式骗分**（多解题）：固定输出题面样例、只构造一种极端形状。

### 6.3 硬性判据

- 所有**纯常数**策略 ≤ 10 分。
- **每条规则至少在 60% 的测试点上改变答案**：即对每个测试点，构造「缺这条规则」的版本，统计它与正解不同的点数；低于 60% 就回去改生成器的配方（把该规则真正用起来）。
- 每个**分档**至少有一个测试点会被该档"应有的部分分解法"命中（否则这一档等于没区分度），且这些命中的点必须正好等于该档规定的那些点。
- 不允许出现「某个常数命中 ≥ 3 个测试点」而不写明理由的情况。

### 6.4 有些送分是题面固有的，改数据改不掉，必须如实记录

遇到下面两类，**不要**为了把表格压好看而删掉测试点或改坏分档——那会把真正的考点一起删掉；正确做法是在 harness 里把该策略标注成 `forced`（写清命中点与原因），cap 写成"题面允许的上限"，并在 README 里逐条解释：

- **分档结构本身导致的退化**。例如某档规定 `a_i ≤ 3`，而正解的公式在该取值集合下恰好恒等于 `N`，于是"输出 N"必然拿下这一档（20 分）；又如某档规定「所有 `a_i > b_j`」，于是「去掉绝对值」的错公式在该档刚好正确（10 分）。这是**分档定义的直接后果**，不是数据没造好。
- **含 clamp 的式子**：形如 `答案 = max(0, X) + c`。clamp 一旦生效答案就恒为 `c`，于是"输出 c"必然命中这些点；而"漏写 clamp"的解在其余所有点上都对。两者得分恒为 `100` 的互补，**不可能同时压住**。默认选择压住前者（常数骗分更廉价），即让 clamp 只在小数点上生效，并在 README 写明这个取舍；如果题目显然更想考 clamp 本身，就反过来并同样写明。

### 6.5 反例（都是真实踩过的）

- 只做「输入合法 + 答案正确」两层校验，从不做防骗分 → 某题 20 个点里有 8 个点的答案等于 `sum + m`，`cout << sum + m` 直接 40 分。
- 某条规则（会员券）在 20 个点里**一次都没真正影响答案**（要么 `k = 0`，要么被 clamp 吞掉，要么 `z = 0`）→ 漏写它的错解 100 分。修法是把该规则在多数点上变成"必要"：`k = 1`、`z > 0`、且折后价严格大于 0。
- 为了压住"输出常数"而把 clamp 分支从数据里删干净 → 该分支再也没被测过，漏写 clamp 的解 100 分。
- 常数骗分表里只测了 `0` 和 `-1`，没测 `n`、`m`、`P-1`、`Σa` → 恰好被后几个命中。

## 7.（可选）转成 Hydro 导入 zip

按官方 [Hydro Problem Format](https://hydro.js.org/zh/docs/Hydro/user/problem-format) 与 [测试数据格式](https://hydro.js.org/zh/docs/Hydro/user/testdata)：

```
<题目文件夹，名任意>/
├── problem.yaml        # title / tag；pid 省略以免与目标域已有题号冲突
├── problem_zh.md       # 完整题面 md（与源题面文件逐字节一致地拷贝）
└── testdata/
    ├── config.yaml
    ├── chk.cc          # 仅 SPJ 题：自定义比较器（见 §4.2）
    └── 1.in … N.in、1.out … N.out
```

- `problem.yaml`：`title: <中文题名（含题源缩写）>`，可选 `tag` 列表。
- `config.yaml`：`type: default`、`time`/`memory` 取自题面、`checker_type: default`（忽略行末空格与文末换行；SPJ 题改成 `checker_type: testlib` + `checker: chk.cc`）；`filename` 按下面第三条的 I/O 约定决定写不写；用 `subtasks:` 显式列 20 个 case，每个 `score: 100/20`、`type: min`，`input/output` 指向 `1.in…`。
- 打包用 Python `zipfile`（条目名用 ASCII，内容 UTF-8）：单题一个 zip；需要“一键导入多题”就再做一个包含多个题目文件夹的合集 zip（Hydro 支持一次导入多文件夹）。
- 校验 zip：`namelist()` 看结构；把包内 `problem_zh.md` 与全部 `.in/.out`（SPJ 题还有 `chk.cc`）与源文件**逐字节比对**，确保无换行符被改写、无缺漏。可以做个 `verify_hydro.py`：结构、`config.yaml` 引用的每个文件都存在、`checker` 指定的文件在 zip 内、subtasks 总分 = 100 且恰好 20 个测试点。
- **I/O 约定以题面封面表为准，不要想当然**：封面表写「输入文件名 `dice.in` / 输出文件名 `dice.out`」就是**文件输入输出**，`config.yaml` 必须写 `filename: dice`——不写的话判题机不会把 `dice.in` 接到标准输入，选手程序会去读 CWD 里不存在的文件（或者以为题面写错了）。只有题面明确写「标准输入 / 标准输出」时才省略 `filename`。写错了要么整场 0 分，要么被选手当成题面笔误。
- **标程没 `freopen` ≠ 标准输入输出**：原题包里的标程常常是裸 `cin`/`cout`，而封面表照样声明 `<题目>.in`/`<题目>.out`。这两件事并不矛盾——判题机按文件名做重定向即可；`filename:` 正是让 Hydro 做同样的重定向，所以标程**不用改**，写 `freopen` 的选手程序也能跑。**不要因为标程没 `freopen` 就把题面改成标准输入输出**。
- `verify_hydro.py` 把「`filename` 必须等于题目名」写成硬检查（标准 IO 题反过来检查"不得出现 `filename`"）。
- 设计数据前先确认这套约定，因为它会连带影响：题面 PDF 封面表、四份 `problem_zh.md`、Hydro 的 `config.yaml`、下发文件里的样例说明。**任何一处改了都要全部重建并重新复核**（见 §10 最后一条）。

## 8. 交付清单与质量门

- **开工前确认已闭环**（§0.4）：确认清单已发出并获得答复（或用户明确授权按默认值先做）；默认项与所有保留意见写进 README。
- 每个题目 20 个 `.in` + 20 个 `.out`（非空、能对应）；每题至少一次冒烟样例比对通过；`validate` 模式全过。
- **标程来源已标注**：官方标程 → 记录路径；自建标程（§0.2）→ 在 README 写明"非官方、本次自建"，附依据的题面条款、所有推测点、以及与独立实现的对拍结论。
- **题解状态已记录**：用户不要 → README 注明"无题解，已询问用户"；用户要 → 交付 `<prob>-solution.md` 并注明其与自建标程的对应关系。
- **防骗分表**（见 §6）：所有纯常数策略 ≤ 10 分；每条规则至少在 60% 的测试点上改变答案；题面固有的送分项已在 README 中逐条列出理由与命中的测试点。
- 附：生成器源码（含校验）、运行/复现脚本（固定种子）、`cheese_<prob>.cpp` 与防骗分 harness、目录内 README（说明测试点↔分档对应、复现方式、防骗分结论）。
- **SPJ 题额外五步**（缺一不可）：
  1. **真实接口回归**：写脚本按评测机真实调用方式跑比较器（Lemon 六参数 / Hydro 三参数），断言**退出码 + 分数文件内容 + 报告文件内容**三者，而不只是“看起来跑过了”。正例覆盖题面样例与边界（`m`/`n`/值域的上下限、多重集重复值），反例覆盖每条约束（越界、个数不足、多余输出、空输出、非整数、溢出、覆盖失败、需要超过 k 个数、同一元素被复用）。
  2. **数据自检**：比较器的 `--validate` 模式对 20 组 `.in/.out` 全过（等价于「标程答案合法」这一层校验）。
  3. **差分测试**：两套独立实现（Lemon 版 / Hydro 版 / 另写一份暴力核验）在**必 AC 与必 WA 混合**的边界语料上判定逐点一致；语料若全 AC 或全 WA，说明强度不足，要改语料。
  4. **确认没顺手改坏数据**：交付前用 sha256 清单比对全部 `.in/.out`（只应新增 checker 文件）。
  5. 实测最坏数据的比较器耗时，写进 README。
- 向用户交付：数据路径、每题规模/特殊性质覆盖说明、两个比较器路径与配置方法、Hydro zip 路径（若导出），以及**标程/题解来源与状态**，并说明哪些档存在保留意见/不确定点（如从图片 PDF 读取的分档数字、语义歧义如“选出不超过 3 个”是否允许重复）。
- **目录就按 §11 的模板建**：根目录、`data/`、`tools/`、`hydro/`、`下发文件/` 的文件名都是固定的，脚本按名互相调用，不要临时发明新的层级或改名。

## 9. 下发文件（附加样例）

题面里能印出来的样例通常只有 1~5 组、规模是个位数，拿来**调程序、测常数、验溢出**完全不够。正式比赛都会另外**下发**几组较大的样例，和题面一起打包给选手。这一步很容易漏，交付前逐条确认：

- 每题 **3~5 组逐级放大**的样例；最大一组尽量顶到该题上限（如 $n=10^5$、$N=2\times10^5$、$n=m=3\times10^5$），中间再来一两组中等的；
- 每组成对给出 `*.in` 与 `*.ans`；`.ans` 一律由**标程**运行产生，并**用独立实现复核**（与 §8 同一套校验器）；
- 生成器要给样例准备**单独的配方表与种子**，不要复用 20 个评测点的配方；并且**断言没有任何一组样例与评测点逐字节相同**（sha256 比对），否则等于变相泄题；
- 目录建议：`下发文件/<题目>/<题目>N.in|.ans`，加一份 `README.md`（样例一览表：文件 / 输入大小 / 说明、用法、比较口径、编译选项），再加题面 PDF；最后打一个 zip 便于分发，打包脚本重开 zip **逐条目比对 sha256**；
- 样例说明里写清 I/O 约定（§7 的文件名规则）与「全文比较，忽略行末空格与文末换行」；如果题目是文件 IO，示例命令要写成"把样例拷成 `<题目>.in` 再跑程序、与 `<题目>.out` 比对"，而不是 `< sample.in`；
- 题面的【提示】一节可以加一句"另随下发文件提供若干组较大的附加样例"，免得选手不知道有这东西。

反例：只把题面样例当"下发样例" → 选手没法验常量和溢出；样例直接复用正式评测点 → 泄题；`.ans` 手算 → 错一个带偏一批人；样例文件混进 `data/` 目录 → 交付目录不再是干净的 20 组。

## 10. 题面 / 题解文档：Markdown → 无头 Chrome → PDF

这一节记录"把题面、题解排成 PDF"的可行路径与踩过的坑。

**环境现实**：这类机器上通常**没有** LaTeX（`xelatex`/`pdflatex`）、没有 pandoc、没有 LibreOffice、没有 reportlab/weasyprint。`matplotlib` 虽然能写 PDF，但它把字形嵌成 **Type 3**（纯字形轮廓、没有 `/ToUnicode`）：结果是**不能选中/搜索，文本提取器也读不出内容**——既没法复核也没法复制，不要用它交付正式文档。

**可行路径**：Markdown →（先把公式藏起来）→ `markdown` 库转 HTML → 注入 CSS + MathJax → 无头 Chrome/Edge `--print-to-pdf`。

1. **公式必须在转 Markdown 之前"藏起来"**：`$a_i$` 里的下划线会被当成强调、`\{` `\_` 之类会被转义。做法：先用正则把 `$...$` / `$$...$$` 抽成占位符（`@@MATHn@@`），转完 HTML 再替换回来（替换时做 HTML 转义，让 `&`、`<`、`>` 正常传给 MathJax）。
2. **MathJax 走 CDN + `--virtual-time-budget=30000`**：不加这个参数，Chrome 会在 MathJax 排完版之前就打印，PDF 里留下原始 LaTeX。用 `tex-svg` 配 `svg:{fontCache:'local'}` 出的是自包含 SVG，最稳（也验证过：MathJax 生效时 `$...$` 源码会从文字层消失）。
3. **公式里不要写中文**：`\text{否则}` 依赖数学字体里的 CJK 字形，很多渲染器直接缺字或报警。把 `\text{otherwise}`、`\text{for }b_j>a_i` 写成英文，或者把说明挪到公式外面。
4. **慎用"集合基数"写法**：`k=\#\{j: b_j\le a_i\}` 排出来是 `k = #{j : b_j ≤ a_i}`，紧跟的 `#{` 很容易被读者当成"公式没渲染好"。能用中文说清就用中文——"二分求出满足 $b_j\le a_i$ 的 $b_j$ 有几个，记作 $k$"。**读者看不懂的排版等于排版错误。**
5. **分页**：`page-break-before: always` 要挂在**有内容的元素**上（章节标题 `<h2>`）。挂在一个空的 `<div class="page"></div>` 上 Chrome 会直接忽略，几道题会挤在同一页，而且从文字层看不出来。
6. **表格列宽**：默认 `table-layout: auto` 按内容撑列，同一个表里"每题一列"的几列会宽窄不一。封面这种表要用 `table-layout: fixed` + 显式列宽；一行里**不要留空单元格**（要么每列都填，要么合并），否则看起来就是排版事故。封面表里的 I/O 文件名、编译选项这类"每题都一样"的项也要每列都写。
7. **先打印到临时文件，再 `os.replace` 就位**：交付物很可能正被 PDF 阅读器（WPS / Acrobat / Edge）打开着，直接往目标路径打印会 `PermissionError`，甚至留下半个文件。被占用时把新文件留在旁边并**明确提示**，别静默失败；重命名前先确认没有阅读器进程占着它（Windows 上可以用 `Get-CimInstance Win32_Process` 过滤命令行找出来是哪个程序）。
8. **打印完必须自检**：把 PDF 文字层抽出来（仓库里的 `pdftext2.py` 之类），断言里面**没有残留 `$`、`\begin{`、`\sum`、`\max`、`\#`、`\underbrace` 等 LaTeX 记号**。正文不使用裸美元号，所以**出现单个 `$` 就说明有公式没排出来**——只查 `$$` 是不够的，单行 `$...$` 失败会整片漏过去。
9. **Markdown 是唯一真源**：题面、题解都用 `.md` 维护，PDF 由它生成；分题文件 + 一个拼装脚本产出单文件，避免同一段内容多处手改。拼装时只对**代码块之外**的 ATX 标题降级——`#include <bits/stdc++.h>` 被改成 `##include` 就毁了。
10. **改一处要重建全部**：改题面措辞或 I/O 约定，会连带影响题面 PDF、题解 PDF、`problem_zh.md`、Hydro zip、下发文件 zip（里面附了题面 PDF）以及各处 README——重建后把 §8 的复核脚本再跑一遍。

## 11. 交付目录结构模板（照这个建目录，别临时发明）

下面这套目录是**按 §1–§10 全流程跑通并复核过**的落地形态（三题 J 组规模）。新任务直接照它建，
文件名不要换：脚本之间就是靠这些名字互相引用的——`statement/<题目>.md` 是 `problem_zh.md` 的真源，
`data/<题目>/<n>.in` 就是 `config.yaml` 里的 case，`verify_<题目>` 与 `cheese_<题目>` 由
`verify_all.py` / `anticheese.py` **按名**调用，改名就得同步改脚本。

```
<比赛目录>/                            # 一场比赛的根目录，用比赛名
├── problems.md                        # 【题面真源】合卷：封面表 + 各题，题与题之间用页分隔 div
├── solution.md                        # 【题解真源】合卷：每题一节（§0.3 用户要才写）
├── statement/
│   └── <题目>.md                      # 单题题面 = problems.md 的逐字节切片（Hydro problem_zh.md 的真源）
├── source/
│   ├── std/<题目>.cpp                 # 标程（§0.2；来源必须标注官方 / 自建）
│   └── checker.cpp                    # 仅 SPJ：Lemon 侧比较器，与 testlib.h 同目录（§4.1）
├── data/
│   ├── README.md                      # ★ 测试数据总说明：分档表、防骗分表、复核结果、保留意见
│   └── <题目>/
│       ├── gen_<题目>.cpp             # 生成器：默认写 1..N.in；validate 复核磁盘；samples 出附加样例
│       ├── gen.exe                    # 编译产物（可删，重编即可）
│       ├── 1.in … N.in                # ★ 恰好 N 组（默认 20），编号不补零
│       ├── 1.out … N.out              # ★ 由标程运行产生，与 .in 同号一一对应
│       ├── checker.cpp / testlib.h    # 仅 SPJ（与 §4.1 同一份，随数据目录走）
│       └── README.md                  # 可选：单题分档↔测试点说明（总说明够用时省略）
├── tools/                             # 全部脚本与复核记录（清单见下表）
├── hydro/
│   ├── <题目>/
│   │   ├── problem.yaml               # title / tag（不含原主办方信息，见 §7）
│   │   ├── problem_zh.md              # = statement/<题目>.md，逐字节一致
│   │   └── testdata/
│   │       ├── config.yaml            # type/time/memory/filename/checker_type + N 个子任务
│   │       ├── chk.cc                 # 仅 SPJ：Hydro 侧比较器（§4.2）
│   │       └── 1.in … N.out           # 与 data/<题目>/ 逐字节一致
│   └── package/
│       ├── <比赛缩写>-T<序号>-<题目>.zip   # 单题包
│       └── <比赛缩写>-all.zip              # 合集包（多题一次导入）
├── 下发文件/                            # 与题面一起发给选手（§9）
│   ├── README.md                      # 样例一览表 + 用法 + 比较口径 + 编译选项
│   ├── <比赛名>.pdf                    # 题面 PDF（§10）
│   └── <题目>/<题目>1.in|.ans …         # 每题 3~5 组逐级放大，不与评测点重复
├── <比赛名>.pdf                        # 题面合卷 PDF
├── <比赛名>-题解.pdf                   # 题解 PDF（用户要才出）
├── <比赛名>-data.zip                   # 工程包：题面 + 标程 + 生成器 + N 组数据 + README
├── <比赛名>-std.zip                    # 工程包：标程 + 全部脚本
└── <比赛名>-下发文件.zip                 # 下发文件整包（含题面 PDF）
```

`tools/` 的固定清单（文件名固定）：

| 文件 | 作用 | 章节 |
|---|---|---|
| `README.md` | 一键流程 + 脚本清单 + 环境坑（解释器、执行策略、管道限制） | — |
| `split_statement.py` | 合卷 `problems.md` → `statement/<题目>.md`；断言切片是逐字节子串、非空行数守恒 | §10.9 |
| `extract_std.py` | 从 `solution.md` 抽每题【参考代码】→ `source/std/<题目>.cpp`；`--check` 复核磁盘副本 | §0.2 |
| `statement_samples.py` | 从题面解析印出的样例 → `tools/smoke/<题目>/<n>.{in,out}` | §5 |
| `build_stds.ps1` | 编译标程（`-O2 -std=gnu++14 -static -Wl,--stack,…`） | §5 |
| `smoke.py` | 标程 vs 题面样例，逐字节比对（忽略行末空格与文末换行） | §5 |
| `run_std.py` | 逐点跑标程写 `.out`，打印退出码 / 耗时 / 输出字节数，统一行尾 | §5 |
| `verify_<题目>.cpp` | **每题一份独立实现**（换算法，不是抄标程），带 `selftest` / `all` / `run` 三个模式 | §8 |
| `verify_all.py` | 编译校验器 → `selftest` → N 点复核 → 每题数百组随机差分 → `verification_report.txt` | §8 |
| `cheese_<题目>.cpp` | 骗分策略集合，含一个正确 `std` 模式用于 harness 自检 | §6.1 |
| `anticheese.py` | 逐策略跑满 N 点算分、断言上限 → `anticheese_report.txt` | §6.3 |
| `make_hydro.py` / `verify_hydro.py` | 打 Hydro 包 / 独立重开 zip 复核结构与字节 | §7 |
| `make_samples.py` | 附加样例（另一套配方 + 种子）+ `.ans` + 独立复核 + 断言不重复 → `下发文件/` 与 zip | §9 |
| `make_archives.py` | `-data.zip` / `-std.zip`，逐条目比对字节（含排除表） | §8 |
| `check_delivery.py` | 交付目录 vs 构建目录**逐文件 sha256** 对账 → `sha256_manifest.txt` | §8 |
| `build_pdf.py` / `build_solution_pdf.py` / `pdftext2.py` | 排两份 PDF / 抽文字层自检（自带极简 Markdown 转换器，不依赖 `markdown` 包） | §10 |
| `verification_report.txt` / `anticheese_report.txt` / `smoke_report.txt` | 三步复核的完整记录，随包交付 | §8 |

命名与格式的硬约定：

- **`<题目>` 一律取封面表的「目录」列**（它同时是可执行文件名与 `<题目>.in` / `<题目>.out` 的前缀）：
  `data/<题目>/`、`statement/<题目>.md`、`hydro/<题目>/`、`source/std/<题目>.cpp`、`verify_<题目>.cpp`、
  `cheese_<题目>.cpp` 必须用同一个词。
- **数据文件编号不补零**：`1.in` … `20.in`（不是 `01.in`），`.out` 与 `.in` 同号。
- **行尾统一 LF + 恰好一个结尾换行**；`.out` 只做行尾归一，数值完全来自标程。
- 交付目录里**只放交付物**：编译产物、临时输入输出、`smoke/`、`*_build/`、`pdf_scratch/` 一律由脚本
  重建、不进包（`make_archives.py` 里维护排除表）。
- SPJ 题：`checker.cpp` 放 `data/<题目>/`（与 `testlib.h` 同目录，Lemon 按相对路径找），`chk.cc` 放
  `hydro/<题目>/testdata/`（与 `config.yaml` 同目录），两份判定规则必须一致（§4）。

交付前"必须存在"清单（配合 §8 逐条打勾）：

1. `data/<题目>/` 恰好 N 个 `.in` + N 个 `.out`，**没有多余文件**（临时文件、临时样例都要清）；
2. `tools/` 下三份报告齐全；`anticheese` 里每题的 `std` 策略都是 100 分；
3. 两份 PDF 存在，且文字层自检无 LaTeX 残留；
4. 至少一份 Hydro zip 通过 `verify_hydro.py`（含 `problem_zh.md` 与全部 case）；
5. `下发文件/` 里每组样例的 `.ans` 都由标程产生、独立复核、且与评测点零重复；
6. `check_delivery.py` 报 OK（交付目录与构建目录逐文件一致）。

**目标目录写入受限时**：另建一个**临时构建目录**，把上面这套结构完整生成并复核，最后整体拷进
交付目录，再用 `check_delivery.py` 做逐文件 sha256 对账。所有脚本都按 `__file__` 推导路径，
所以整棵树可以搬移，搬完还能在原地重跑全部复核。

## 常见坑

- PDF 文本无文本层 / 字体子集无 `/ToUnicode` → 输出乱码或缺字；务必检查并换方法（坐标还原、多字形级联），不要用“看起来像”的文本。
- 样例块左侧行号列未剥离 → 数据首列多一位数字。
- 上标还原错误（如把 10^5 读成 10）→ 生成规模超界或强度不足。
- 生成时只校验“内存值”而未校验“文件内容”，换行翻译（CRLF）或格式差异导致校验失真。
- 深递归标程不加大栈 → 栈溢出、.out 为空。
- 标程语义与题面冲突时仍按题面手写 .out → 与官方不符；一律以标程输出为准。
- 答案不唯一却用 `checker_type: default` → 合法解被判错（见 §4）。
- 比较器用退出码表达 AC/WA → Lemon 忽略退出码，非 0 反而被当评测失败；分数必须写进 `argv[5]`（用 `registerLemonChecker` 时由 testlib 代写）。
- `specialJudge` 与测试点路径**相对 `data` 目录**，填绝对路径或在别处建目录联接 → Lemon 判 `InvalidSpecialJudge`（实测 result 8），校验器根本不会启动。
- 校验器动态链接 → Lemon 启动时缺 `libstdc++-6.dll` 等，以 `0xC0000135` 退出；必须 `-static`。
- 校验器是在带 Low 完整性标记的目录里编出来的 → 它启动的进程也是 Low，写不了 `%TEMP%` 里 Lemon 的 `_score`，同样报 `InvalidSpecialJudge`（"校验器没有被正确配置"）；把编译产物编到受限目录外再同盘移入即可（详见 §4.3）。
- 只跑本地 `--validate` 就交付 SPJ → 它只证明判定逻辑，证明不了评测机启动得了它、更证明不了它能写分数文件；回归要按 Lemon 的真实调用把 `score_file` 指到 `%TEMP%` 验一次。
- 手写 argv 解析而不先确认目标评测机约定（经典 Lemon 六参数 vs LemonLime testlib vs Hydro 三参数）→ 参数错位、把 `argv[3]` 当分数文件之类。
- Lemon 报告文件里写中文 → 部分版本乱码；改成 ASCII + README 中文对照表。
- 本地 `--validate` 模式忘记 `disableFinalizeGuard()` → 正常 `return` 时被 testlib 收尾守卫 `_fail`（`Call register-function in the first line of the main`）。
- 拿 `.out` 当真值判定构造题 → 换标程/换合法构造即失效。
- 只写一份比较器而忘了另一份，或改了 `checker.cpp` 没同步 `chk.cc`、没同步打包/复核脚本里硬编码的文件名 → Hydro 包缺比较器或两边判定不一致。
- 测试语料全是 AC（或全是 WA）→ 差分测试等于没测；“我预期它会 WA”也要用独立实现验证，别凭直觉断言（构造题的覆盖范围很容易算错）。
- Windows PowerShell 5.1 读**无 BOM** 的 `.ps1` 会按 ANSI 解码 → 脚本里的中文字面量触发语法错误（含中文的目录名也一样）；测试/打包脚本保持 ASCII，路径一律用 `$PSScriptRoot` / `__file__` 推导。
- 交付前不做 sha256 比对 → 分不清“只新增了 checker”还是“顺手动了测试数据”。
- 只验证「输入合法」「答案正确」，从不验证「白送分」→ `cout << 0` / `cout << n` 之类拿走到手的一半分数（见 §6）。
- 某条规则在 20 个测试点里一次都没真正影响答案（参数恒为中性值、或被 clamp 吞掉）→ 漏写这条规则的错解拿满分。
- 为了把防骗分表压好看，删掉本该校验的分支（如 clamp、某个 `k = 0` 的档）→ 该考点彻底失去覆盖；题面固有的送分项应如实记录而不是硬压。
- 没有标程就照题面**手写 `.out`** → 违反"标程是唯一真值"；正确做法是自建标程再跑出 `.out`（§0.2）。
- 自建标程只靠"看起来对"就投产 → 标程错一次，20 个 `.out` 全部作废；必须配独立实现做对拍并跑通题面样例。
- 自建标程不标注来源 → 后来被当成官方标程，语义争议无从追溯；交付时必须写明"非官方、本次自建"及全部推测点。
- 题面有多种读法（是否允许重复、是否取模、字典序要求、`T`/Σn 上限）却不问就开工 → 整批数据语义错误，且返工要从生成器重跑到 `.out`。
- 缺题解时**擅自代写**或**默认跳过** → 前者浪费时间且可能与官方解法冲突，后者漏掉用户真实需求；一律先问（§0.3）。
- 边生成边提问 → 档位/语义/评测机一变就要重造全部 20 个 `.in/.out`（连带 sha256 复核、防骗分表全部重跑）；存疑点必须开工前一次性问清（§0.4）。
- 悬而未决的默认值不记录 → 用户后来发现数据与预期不符却查不到依据；每个默认项都要进 README 的「保留意见」一节。
- `python` 在 PATH 上可能是 **Windows 商店别名**（`…\WindowsApps\python.exe`）：它会**偶发地以 9009 失败**——同一条命令前一次能跑、下一次就"找不到程序"，于是复核脚本莫名其妙挂掉。脚本里别写死 `python`，按 `py -3.10` → `py -3` → `py` → `python` → `python3` 顺序**实际试跑**探测（`& $exe -c "print(1)"` 看退出码），取第一个能跑通的；注意 `py` 默认可能是另一个小版本（如 3.8），要显式指定版本。
- 用 `Get-Content` 看 **UTF-8 无 BOM** 的文件会按 ANSI 解码成乱码，看着像"文件编码坏了"，其实文件没问题。判断编码一律用 Python 解码验证，不要凭控制台显示下结论。
- PowerShell 变量名**大小写不敏感**：`$D = ...` 之后写 `$d = Compare-Object ...` 会把 `$D` 清掉，后面 `"$D\x.in"` 全部变成相对路径。
- PowerShell 的 `$a[1..($a.Count-1)]` 在 `Count -eq 1` 时会变成 `$a[1..0]`（倒序区间）→ 取到 `$null`、参数错位；先判断长度再切片。
- PowerShell 函数**必须先定义后使用**（脚本自上而下执行）；把初始化代码写在 `function Say` 之前，调用它就会报"无法识别"。
- `.ps1` 保持纯 ASCII 要**连注释一起**：中文场次名写在首行注释里同样会被 ANSI 解码弄乱（而且它藏在注释里，很容易一直没被发现）。交付前扫一遍每个 `.ps1` 的字节。
- 造样例/差分用的临时输入（如 `gen.exe random` 写出的 `<题目>.in`）会留在数据目录里 → 交付目录必须**恰好** 20 个 `.in` + 20 个 `.out`，跑完要对账（`Get-ChildItem *.in` 计数）并清理；harness 自己也要顺手删掉临时文件。
- **防骗分 harness 自己要先自检**：把某个 mode 写错成"就是正解"（例如漏了一个减法），表格会显示该策略 100 分，看起来像数据崩了，其实是 harness 的 bug。跑之前先自证——拿正解喂进去，该满分的 mode 必须满分、该 0 分的必须 0 分。
- **校验器的 bug 会被误判成数据 bug**：给 dice 写的"可达性检查"一开始构造出的塔不是最少的，于是 20/20 全报 MISMATCH，白查半天。校验器报错时先怀疑校验器，用一组已知答案的小数据手工核对它。
- **防骗分表是第一版配方表的验收测试**：先按"覆盖面"写配方，跑一遍防骗分，按结果回去改配方（哪条规则没被用起来就把它变成"必要"的），改完再跑——通常要来回两三轮才收敛。别一次写完就交付。
