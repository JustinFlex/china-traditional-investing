# 流日干支对 A 股涨跌的影响：可复现的统计研究

> 用**可复现的数据管线 + 可审计的统计检验**，回答"流日干支（日柱：天干+地支）是否对 A 股日频涨跌有稳定信息量"。

---

## 目录

- [项目简介](#项目简介)
- [核心发现](#核心发现)
- [方法论亮点](#方法论亮点)
- [项目结构](#项目结构)
- [快速开始](#快速开始)
- [Notebook 运行顺序](#notebook-运行顺序)
- [数据说明](#数据说明)
- [研究设计](#研究设计)
- [详细结果](#详细结果)
- [US 对照实验（负控制）](#us-对照实验负控制)
- [文档索引](#文档索引)
- [局限性与风险提示](#局限性与风险提示)
- [环境与依赖](#环境与依赖)
- [许可与声明](#许可与声明)

---

## 项目简介

本项目将中国传统文化中"天干地支影响市场涨跌"的民间信念翻译为**可证伪的统计假设**，并以现代量化金融的方法论严格检验。

**研究问题**：
1. 不同天干（10 组）、地支（12 组）、六十甲子日柱（60 组）的交易日，A 股涨跌是否有显著差异？
2. 这种差异是否跨指数一致、控制日历效应后仍存在、且样本外可复现？
3. 该效应是"自然时间周期"还是"本土投资者行为的自我实现"？

**研究对象**：A 股四大宽基指数（沪深300、中证1000、上证综指、深证成指），2010—2026 年共 3916 个交易日。

---

## 核心发现

### 确认性结论（通过全部稳健性检验）

**天干"丙"日，A 股收益显著偏低。**

| 证据环节 | 结果 |
|---|---|
| 效应大小（Meta 合并） | **-19.2 bp/日**（约 -0.192%/日） |
| 跨指数 Meta（BH-FDR） | q ≈ 3×10⁻⁶，4 个指数方向一致为负 |
| 控制变量回归（weekday/month/year + HAC） | 沪深300 q ≈ 0.047，深证成指 q ≈ 0.037 |
| 置换检验（controls 残差） | 沪深300 p ≈ 0.024，深证成指 p ≈ 0.015 |
| Block bootstrap 95% CI | 4 个指数均不含零 |
| 样本外 walk-forward（按年） | 11 年中 10—11 年方向一致为负 |
| HAC maxlags 敏感性 | 结果对 maxlags ∈ [0,1,3,5,10] 不敏感 |

### 探索性发现（候选，未成为正式结论）

- **天干"癸/丁"**：Meta 合并显著（癸 q ≈ 0.008），但样本外稳定性不足（match_ratio 仅 0.64—0.73），多年符号翻转
- **Phase 2 交互（立春年五行 × 日柱）**：`branch` 和 `ganzhi_day` 的交互 gate 通过，`stem` 未通过
- **Resonance 谐波**：60 甲子 k=5/6 谐波跨指数 p ≈ 0.003，但被证实为 stem/branch 加性主效应的频域表现

### US 对照实验（负控制）

**美股完全没有类似信号**（S&P 500、Nasdaq、Dow Jones），所有检验均为 null。

**最终解释**：天干地支效应更可能是一种**本土投资者信念/行为与市场结构共同作用的"自我实现"**，而非跨市场通用的自然时间规律。

---

## 方法论亮点

本项目并非简单地"用玄学炒股"，而是建立了一套对抗数据挖掘的完整防线：

| 方法论措施 | 说明 |
|---|---|
| **多重比较修正** | 60 甲子属天然多重比较场景，全部输出 BH-FDR / Bonferroni |
| **预注册（Preregistration）** | 在跑实验前冻结假设、阈值、输出文件清单，防止事后挑显著 |
| **确认性 vs 探索性分层** | Protocol 明确定义哪些发现可写结论、哪些只能作诊断 |
| **跨指数 Meta 分析** | 固定效应逆方差加权合并，要求多个指数方向一致 |
| **多套稳健性检验** | 置换检验、block bootstrap、HAC 敏感性、子样本分段 |
| **样本外 walk-forward** | 按年滚动估计，检验方向稳定性 |
| **US 负控制** | 同一套流程检验美股，排除"通用自然周期"假说 |
| **层级检验 gate** | Phase 2 交互先通过全局 gate，才允许看局部 cell |

---

## 项目结构

```
风水五行阴阳天干地支/
├── notebooks/                          # A 股主分析（按顺序 Run All）
│   ├── 01_fetch_market_data.ipynb      #   拉取并缓存交易日历 + 指数日线
│   ├── 02_build_ganzhi_calendar.ipynb  #   公历 → 日柱干支（含校验向量）
│   ├── 03_ganzhi_effect_analysis.ipynb #   分组统计 + 显著性检验 + 图表
│   ├── 04_robustness_and_modeling.ipynb #   稳健性模块索引
│   ├── 04a_controls_models.ipynb       #   控制变量回归（weekday/month/year + HAC）
│   ├── 04b_subsample_stability_bing.ipynb  # 子样本（年份段）分析
│   ├── 04c_permutation_and_dependence.ipynb # 置换检验 + 序列相关稳健性
│   ├── 04d_oos_walk_forward.ipynb      #   样本外 walk-forward + Meta 合并
│   ├── 04e_phase2_interaction_gate.ipynb #  Phase 2：立春年五行交互 gate
│   ├── 04f_resonance_harmonics.ipynb   #   jiazi_idx 谐波检验（诊断）
│   └── 05_report.ipynb                 #   一键生成汇总报告
│
├── notebooks_US/                       # US 对照实验（负控制，结构同上）
│   ├── 01_fetch_market_data.ipynb      #   Stooq 数据源（免 Token）
│   ├── 02_build_ganzhi_calendar.ipynb  #   华盛顿时间口径
│   ├── 03_ganzhi_effect_analysis.ipynb
│   ├── 04a ~ 04f ...                   #   同 A 股版一一对应
│   └── 05_report.ipynb
│
├── data/                               # 所有数据（缓存 + 研究输出）
│   ├── cache/                          #   A 股原始数据缓存（Tushare）
│   ├── cache_us/                       #   US 原始数据缓存（Stooq）
│   ├── clean/                          #   A 股研究输出
│   │   ├── market_ganzhi_*.csv.gz      #     合并表（市场 + 干支字段）
│   │   ├── ganzhi_stats_*.csv          #     分组统计表
│   │   ├── ganzhi_tests_*.csv          #     检验结果（effect/p/q）
│   │   ├── robustness/                 #     全部稳健性输出（~90 个文件）
│   │   ├── report/                     #     汇总报告（Markdown + 图表）
│   │   └── quality/                    #     数据质量检查
│   └── clean_us/                       #   US 研究输出（结构同 clean/）
│
├── docs/                               # 方法学文档
│   ├── Protocol.md                     #   研究协议（确认性 vs 探索性分层）
│   ├── Preregistration.md              #   预注册模板
│   ├── li_chun_year_mapping.csv        #   立春年映射（北京时间；真值源）
│   └── li_chun_year_mapping_washington.csv # 立春年映射（华盛顿时间）
│
├── ProjectPlan.md                      # 研究计划（含完整结果摘要）
├── Model.md                            # 字段字典 + 检验口径 + 模型说明
├── TODO.md                             # 开发进度追踪
├── AGENTS.md                           # Agent / 协作者的约定
└── .gitignore
```

---

## 快速开始

### 前置条件

- Python 3.12+
- Tushare 账号（积分 ≥ 5000）— 仅 A 股数据需要；US 对照使用 Stooq（免费、免 Token）
- VS Code + Jupyter 插件（推荐）

### 安装依赖

项目优先使用标准库，核心第三方依赖：

```
tushare          # A 股数据接口（仅 A 股版本需要）
pandas
numpy
scipy
statsmodels
matplotlib
```

### 配置 Tushare Token

三选一（优先级从高到低）：

```bash
# 方式 1：环境变量（推荐）
export TUSHARE_API_KEY='你的Token'

# 方式 2：仓库根目录文件
echo '你的Token' > .tushare_token
chmod 600 .tushare_token

# 方式 3：家目录文件
echo '你的Token' > ~/.tushare_token
chmod 600 ~/.tushare_token
```

> **安全提示**：Token 不得写入任何代码或提交到 Git。`.tushare_token` 已在 `.gitignore` 中排除。

### WSL 用户注意

如需代理，获取宿主机 IP 并设置：

```bash
host_ip=$(ip route | awk '/default/ {print $3; exit}')
export TUSHARE_PROXY="http://$host_ip:10808"  # 端口按实际修改
```

详见 `tushare_api_docs/TUSHARE_CONFIG.md`。

---

## Notebook 运行顺序

所有 Notebook 设计为在 VS Code 中 **Run All** 即可，无需命令行操作。

### A 股版本（主实验）

```
notebooks/01_fetch_market_data.ipynb          ← 拉取数据（首次需联网）
    ↓
notebooks/02_build_ganzhi_calendar.ipynb      ← 计算干支（含自动校验）
    ↓
notebooks/03_ganzhi_effect_analysis.ipynb     ← 主分析（统计 + 图表）
    ↓
notebooks/04_robustness_and_modeling.ipynb     ← 稳健性索引（指向 04a—04f）
    ├── 04a_controls_models.ipynb             ← 控制变量回归
    ├── 04b_subsample_stability_bing.ipynb    ← 子样本分段
    ├── 04c_permutation_and_dependence.ipynb  ← 置换检验 + block bootstrap
    ├── 04d_oos_walk_forward.ipynb            ← 样本外 + Meta + OOS 筛选
    ├── 04e_phase2_interaction_gate.ipynb     ← Phase 2 交互 gate
    └── 04f_resonance_harmonics.ipynb         ← 谐波诊断
    ↓
notebooks/05_report.ipynb                     ← 一键生成报告
```

### US 对照版本（负控制）

```
notebooks_US/01 → 02 → 03 → 04a~04f → 05    （结构与 A 股版完全对应）
```

US 版本使用 Stooq 免费数据源，无需 Tushare Token。检验美股 S&P 500、Nasdaq Composite、Nasdaq 100、Dow Jones 四个指数。

---

## 数据说明

### 市场数据

| 版本 | 指数 | 数据源 | 区间 | 交易日数 |
|---|---|---|---|---|
| A 股 | 沪深300 `000300.SH`、中证1000 `000852.SH`、上证综指 `000001.SH`、深证成指 `399001.SZ` | Tushare | 2010-01-04 ~ 2026-02-13 | 3916 |
| US | S&P 500 `^spx`、Nasdaq Composite `^ndq`、Nasdaq 100 `^ndx`、DJIA `^dji` | Stooq | 2010-01-05 ~ 2026-02-13 | ~4053 |

### 核心字段

| 字段 | 说明 |
|---|---|
| `ret_1d` | 日收益率 = `close / prev_close - 1` |
| `up` | 涨跌标签 = `1[ret_1d > 0]` |
| `stem` | 天干（甲乙丙丁戊己庚辛壬癸，10 组） |
| `branch` | 地支（子丑寅卯辰巳午未申酉戌亥，12 组） |
| `ganzhi_day` | 六十甲子日柱（如"甲子""乙丑"，60 组） |
| `jiazi_idx` | 甲子序号 0—59（便于排序与回归） |
| `year_element` | 立春年五行（木火土金水，5 类；Phase 2 使用） |

### 干支计算口径

- 日界：**北京时间自然日 00:00—24:00**（避免 23:00 子时换日争议）
- 算法：公历日期 → 儒略日数（JDN） → 模 60 映射到六十甲子
- 校验：`notebooks/02` 内置权威万年历校验向量，Run All 时自动 `assert`

### 立春年口径

- 以"立春"为年分界（非公历 1 月 1 日），贴合传统历法
- 真值源：`docs/li_chun_year_mapping.csv`（立春时刻来自 JPL Horizons / Wikipedia）
- 在立春当日以市场收盘时刻（A 股 15:00、美股 16:00）判断归属前一年或当年

---

## 研究设计

### Phase 1：日柱主效应

对每个分组维度（stem/branch/ganzhi_day），检验其对 `ret_1d` 和 `up` 是否有显著差异。

**统计检验链**：

```
无控制检验（03）→ 控制变量回归 + HAC（04a）→ 置换检验（04c）
    → block bootstrap（04c）→ 子样本分段（04b）
    → 样本外 walk-forward（04d）→ 跨指数 Meta 合并（04d）
    → "正式结论"筛选：q_meta ≤ 0.1 + 方向一致 + OOS 稳定
```

### Phase 2：立春年五行交互

检验"不同年运（五行）下，日柱效应是否变化"，即 `day_group × year_element` 交互。

**层级检验 gate**：
1. **全局 gate**（确认性）：交互项 joint Wald test → Fisher 合并 → BH-FDR，且至少 3/4 指数单指数通过
2. **局部 cell**（探索性）：仅 gate 通过后才输出 cell-level 候选，并标注"不得替代确认性结论"

### 负控制：US 对照

同一套流程检验美股，若"天干地支是自然时间因子"，美股也应出现类似信号。若为 null，则支持"本土投资者行为"解释。

---

## 详细结果

### 主分析（无控制变量）

通过 BH-FDR q ≤ 0.1 筛选的项：

| 指数 | 天干 | 效应（vs 全样本均值） | q 值 |
|---|---|---|---|
| 沪深300 | 丙 | -0.198%/日 | 0.046 |
| 深证成指 | 丙 | -0.236%/日 | 0.032 |

### 控制变量回归（04a）

控制 weekday/month/year 后，OLS + HAC（Newey-West）：

| 指数 | 天干 | 边际效应 | q_effect |
|---|---|---|---|
| 沪深300 | 丙 | -0.197%/日 | 0.047 |
| 深证成指 | 丙 | -0.234%/日 | 0.037 |

> 上证综指与中证1000 方向一致为负，但 q > 0.1。

### 置换检验（04c）

| 口径 | 沪深300 | 深证成指 | 上证综指 | 中证1000 |
|---|---|---|---|---|
| 原始（stem × ret_1d） | p ≈ 0.034 | p ≈ 0.025 | p ≈ 0.068 | p ≈ 0.171 |
| Controls 残差 | p ≈ 0.024 | p ≈ 0.015 | p ≈ 0.060 | p ≈ 0.168 |

### Block bootstrap（04c）

stem=丙，controls 残差，block_len=10：

| 指数 | 效应 | 95% CI | p_boot |
|---|---|---|---|
| 上证综指 | -16.1 bp | [-27.8, -2.9] | 0.018 |
| 沪深300 | -19.7 bp | [-32.7, -5.4] | 0.002 |
| 中证1000 | -18.8 bp | [-36.5, -2.2] | 0.030 |
| 深证成指 | -23.4 bp | [-38.5, -7.9] | 0.006 |

### 样本外 walk-forward（04d）

stem=丙，按年 OOS 方向一致性：

| 指数 | OOS 年数 | 负向年数 | 负向比例 | p (符号检验) |
|---|---|---|---|---|
| 上证综指 | 11 | 11 | 100% | 0.001 |
| 沪深300 | 11 | 10 | 91% | 0.012 |
| 中证1000 | 11 | 10 | 91% | 0.012 |
| 深证成指 | 11 | 11 | 100% | 0.001 |

### 跨指数 Meta 合并（04d）

10 天干 Meta（固定效应），按 q_meta 排序（前 3）：

| 天干 | effect_meta | q_meta | OOS 通过？ | 正式结论？ |
|---|---|---|---|---|
| **丙** | **-19.2 bp/日** | **2.6×10⁻⁶** | **4/4 通过** | **是** |
| 癸 | +10.5 bp/日 | 0.008 | 0/4 通过 | 否（OOS 不稳） |
| 丁 | +8.0 bp/日 | 0.099 | 0/4 通过 | 否（OOS 不稳） |

### Phase 2 交互 gate（04e）

| day_group | q_meta_interaction | 指数通过数 | gate 结果 |
|---|---|---|---|
| stem | 0.265 | 0/4 | 未通过 |
| branch | 6.85×10⁻⁵ | 3/4 | **通过** |
| ganzhi_day | 9.69×10⁻²⁶ | 4/4 | **通过** |

> gate 通过仅说明"交互存在"；局部 cell 候选为探索性，单格样本数较小，不作为结论。

### Resonance 谐波（04f）

| 检验 | A 股 p_meta | US p_meta |
|---|---|---|
| k=5/6 joint Wald（HAC + controls） | 0.003 | 0.912 |
| 回归掉 stem+branch 后 | ≈ 1（预期） | — |

> 诊断结论：谐波信号为 stem/branch 加性主效应的频域表现，非独立的"非线性共振"。

---

## US 对照实验（负控制）

| 检验环节 | US 结果 |
|---|---|
| 无控制分组检验（03） | 无 q ≤ 0.1 |
| 控制变量回归（04a） | 无 q ≤ 0.1 |
| 全局置换检验（04c） | 最小 p ≈ 0.216 |
| Resonance k=5/6（04f） | p_meta ≈ 0.912 |
| Phase 2 交互 gate（04e） | ganzhi_day gate 通过，但 cell 样本极小（n ≈ 6—20），不可信 |

**结论**：在美股上使用同一套方法论，未观察到任何可复现的天干地支效应。

**解释含义**：
- 排除了"天干地支是普遍自然时间因子"的可能性
- A 股侧的信号更可能来自本土投资者信念、行为模式与市场微观结构

---

## 文档索引

| 文件 | 用途 |
|---|---|
| [`ProjectPlan.md`](ProjectPlan.md) | 研究计划、完整结果摘要、下一步方向 |
| [`Model.md`](Model.md) | 字段字典、检验与模型口径、输出字段解释 |
| [`TODO.md`](TODO.md) | 开发进度追踪（已完成 / 待做 / 搁置） |
| [`docs/Protocol.md`](docs/Protocol.md) | 研究协议：确认性 vs 探索性分层、Phase 2 gate 规则、US 负控制 |
| [`docs/Preregistration.md`](docs/Preregistration.md) | 预注册模板：每次扩展前冻结假设/阈值/输出清单 |
| [`docs/li_chun_year_mapping.csv`](docs/li_chun_year_mapping.csv) | 立春年映射真值源（北京时间；来源：JPL Horizons） |
| [`docs/li_chun_year_mapping_washington.csv`](docs/li_chun_year_mapping_washington.csv) | 立春年映射（华盛顿时间；US 版本使用） |
| [`AGENTS.md`](AGENTS.md) | Agent / 协作者的代码与结构约定 |

### 报告文件

`data/clean/report/` 下按时间戳生成的 Markdown 报告，最新版本为 `report_20260214_225132.md`，包含：
- 数据覆盖摘要
- 干支日历 sanity check
- 一页结论表（stem=丙证据链）
- 全量扫描最小 q 表
- 稳健性全套结果（04a—04f）
- 候选 stem 诊断（癸/丁）
- Phase 2 交互 gate 结果
- Resonance 谐波结果

---

## 局限性与风险提示

1. **非因果**：统计关系不等于因果关系。天干"丙"日偏低可能有未观测到的混淆因素。
2. **非可交易**：约 -19 bp/日的效应在扣除交易成本后能否产生正收益，本项目不做承诺。
3. **数据窥探**：尽管已做多重比较修正与预注册，60 甲子 × 多指数 × 多口径的搜索空间仍存在残余风险。
4. **序列相关**：日度收益非 i.i.d.，已通过 HAC、block bootstrap、置换检验缓解，但不能完全消除。
5. **样本期依赖**：效应在 2021—至今分段更强，可能受市场结构变化影响。
6. **"自我实现"的脆弱性**：若该效应确由投资者信念驱动，一旦信念改变或被套利者利用，效应可能消失。

---

## 环境与依赖

| 组件 | 版本 / 说明 |
|---|---|
| Python | 3.12+ |
| tushare | 1.4.24（仅 A 股数据需要；积分 ≥ 5000） |
| pandas | 数据处理 |
| numpy | 数值计算 |
| scipy | 统计检验（binomtest、t-test、Fisher 合并等） |
| statsmodels | OLS / GLM + HAC、Wald test、contrast |
| matplotlib | 可视化 |

> 项目不使用 `pyarrow` / `fastparquet`，数据缓存统一使用 `csv.gz` 格式。

---

## 许可与声明

- 本项目为**学术研究性质**，旨在检验传统文化信念与市场数据之间的统计关系，不构成任何投资建议。
- Tushare 数据受其服务条款约束。
- 立春时刻数据引用自 JPL Horizons（经 Wikipedia 整理）。
