# Model.md（当前主题：流日干支 → A 股日频涨跌）

本文件说明本项目的核心字段、标签口径与统计/建模方式，避免 notebook 之间“各说各话”。

研究分层与扩展规则（方法学真值源）：
- `docs/Protocol.md`：Confirmatory（可写正式结论）vs Exploratory（只能诊断）分层；Phase 2 交互的层级 gate
- `docs/Preregistration.md`：扩展前冻结假设/阈值/输出清单，防止事后挑断点/挑窗口

---

## 1. 核心数据表（建议落盘）

### 1.1 市场数据（指数日线）
文件：`data/clean/market_{ts_code}.csv.gz`

必备字段（示例）：
- `trade_date`：交易日（`YYYYMMDD`）
- `close`：收盘价
- `prev_close`：昨收（如接口不提供则由 `close.shift(1)` 构造）
- `ret_1d`：日收益（`close/prev_close-1`）
- `up`：涨跌标签（`1[ret_1d>0]`）

### 1.2 干支字段（交易日）
文件：`data/clean/ganzhi_trade_dates.csv.gz`

字段：
- `trade_date`
- `ganzhi_day`：六十甲子（如 `甲子`）
- `stem`：天干（`甲乙丙丁戊己庚辛壬癸`）
- `branch`：地支（`子丑寅卯辰巳午未申酉戌亥`）
- `jiazi_idx`：`0..59`（`0=甲子`）

### 1.3 合并表（市场 + 干支）
文件：`data/clean/market_ganzhi_{ts_code}.csv.gz`

字段 = `market_{ts_code}` ∪ `ganzhi_trade_dates`（按 `trade_date` 左连接）。

### 1.4（Phase 2：已实现）立春年 / 年五行字段
> 用于把“时间不变”假设放松为可证伪的交互项：`day_group × year_element`。

真值源：
- `docs/li_chun_year_mapping.csv`：立春时刻（UTC）与换算到 UTC+8 的日期；以及对应的 `year_stem/year_element`

建议在 `market_ganzhi_{ts_code}.csv.gz` 中新增字段（由 `trade_date` 派生）：
- `li_chun_year`：立春年标签（整数）
- `year_stem`：年天干（10）
- `year_element`：年五行（5；由 `year_stem` 映射：甲乙木、丙丁火、戊己土、庚辛金、壬癸水）

定义（口径冻结，见 `docs/Protocol.md`）：
- 用“市场收盘时刻”近似该交易日时间戳：`trade_close_hm = 15:00`（北京时间）  
  若 `trade_close_datetime(trade_date) < li_chun_begin_local(trade_year)`，则 `li_chun_year = trade_year - 1`；否则 `li_chun_year = trade_year`

---

## 2. 干支计算口径（必须固定）

### 2.1 日界与时区
默认以北京时间自然日（00:00~24:00）计算日柱干支。  
说明：传统命理存在“子时换日（23:00）”等口径差异；本项目选择与交易日日期一致的口径，避免混淆。

### 2.2 校验向量
在 `notebooks/02_build_ganzhi_calendar.ipynb` 中内置校验用日期，Run All 必须自动 `assert` 通过。  
（校验向量一旦确定，后续版本禁止随意改常数/基准日。）

---

## 3. 统计输出（必须）

### 3.1 分组统计
输出：`data/clean/ganzhi_stats_{ts_code}.csv`

分组维度：
- `stem`（10 组）
- `branch`（12 组）
- `ganzhi_day`（60 组，样本更少）

建议指标：
- `n`：样本数
- `p_up`：上涨概率
- `mean_ret` / `median_ret`
- `std_ret`
- `q05_ret`：5% 分位（左尾）

### 3.2 显著性与多重比较
输出：`data/clean/ganzhi_tests_{ts_code}.csv`

至少包含：
- `group_type`：`stem/branch/ganzhi_day`
- `group_value`：具体取值
- `effect`：效应大小（当前实现：`p_up - p_up_all` 或 `mean_ret - mean_ret_all`）
- `p_value`
- `q_value`：FDR（Benjamini–Hochberg）

当前实现说明（便于复现实验口径）：
- `p_up`：对每个组做二项检验（`binomtest`），检验该组上涨次数是否显著偏离全样本上涨率
- `mean_ret`：对每个组做 Welch t-test（组内 vs 其余样本，`ttest_ind(..., equal_var=False)`）
- `q_value`：对每个 `group_type × metric` 单独做 BH（例如：`stem/p_up` 只在 10 个天干内做修正；`ganzhi_day/mean_ret` 在 60 甲子内做修正）

---

## 4. 已实现的检验与模型（Notebooks 03/04）

本节写清楚：**谁对谁检验/回归**、用什么统计口径、输出字段如何解释。

### 4.1 Notebook 03：无控制变量（样本内）分组检验
输入：`data/clean/market_ganzhi_{ts_code}.csv.gz`

对每个 `group_type ∈ {stem, branch, ganzhi_day}`、每个 `group_value=v`：
- `up`（方向）：
  - 检验：二项检验（two-sided）`binomtest(k, n, p=overall_p)`
  - 原假设：`P(up=1 | g=v) = P(up=1)`（以全样本上涨率为基准）
  - 输出 `effect = p_up(v) - p_up(all)`
- `ret_1d`（收益）：
  - 检验：Welch t-test（`ttest_ind(sub, other, equal_var=False)`，`sub`=组内，`other`=其余样本）
  - 输出 `effect = mean_ret(v) - mean_ret(all)`（注意：p-value 来自“组内 vs 其余样本”，但 effect 报的是“组内 vs 全样本均值”）

多重比较：
- `q_value`：对每个 `group_type × metric` 单独做 BH-FDR（10/12/60 维各自修正）
  - 注意：当前 `q_value` 是**每个指数单独**修正，尚未对“多指数同时检验”的二次多重比较做统一校正；因此单点显著更需要跨指数复现。

重要提示：
- `03` 的二项检验 / t 检验**不使用 HAC**，默认把日度样本当作 i.i.d.；因此它更适合作为**筛选/描述**，稳健结论以 `04` 为准。

### 4.2 Notebook 04：控制变量回归（HAC / Newey-West）
输入：同上（并从 `trade_date` 派生控制变量）

控制变量（全部做成分类变量）：
- `weekday`：`0..6`（周一..周日）
- `month`：`1..12`
- `year`：如 `2010..2026`

对每个 `ts_code`、每个 `group_col ∈ {stem, branch, ganzhi_day}` 分别拟合两类模型：

#### 4.2.1 `ret_1d`：OLS + HAC
回归式（示例，以 `stem` 为例）：
- `ret_1d ~ C(stem) + C(weekday) + C(month) + C(year)`

估计：
- OLS
- 协方差：HAC / Newey-West（`maxlags=5`），用于日度序列相关的稳健标准误

#### 4.2.2 `up`：GLM(Binomial, logit) + HAC
回归式（示例）：
- `up ~ C(stem) + C(weekday) + C(month) + C(year)`

估计：
- GLM（Binomial, logit link）
- 协方差：HAC / Newey-West（`maxlags=5`）

#### 4.2.3 `controls_*.csv` 输出字段解释
文件：`data/clean/robustness/controls_{ts_code}_{group_col}_{target}.csv`

1) **边际均值（建议主要看这一套）**
- `marginal_mean`：把 `group_col` 强制设为某个取值 `v`，对每个交易日保留其控制变量取值，计算预测值并取平均  
  直观上是 `E[ y_hat | do(g=v) ]` 的样本平均
- `marginal_mean_all`：按原始样本 `g` 分配直接预测并平均（全样本预测均值）
- `marginal_effect_vs_all = marginal_mean - marginal_mean_all`

2) **边际效应的显著性（建议用这一套做显著性结论）**
- `se_effect / z_effect / p_value_effect / q_value_effect`：
  - 对原假设 `H0: marginal_effect_vs_all(v) = 0` 的 Wald 检验
  - `ret_1d`：用 OLS 的 `t_test(contrast)`（同一套 HAC 协方差）做 contrast
  - `up`：对“平均预测概率差”用 delta-method 近似标准误（同一套 HAC 协方差）
  - `q_value_effect`：对同一模型下所有组（10/12/60 个）做 BH-FDR
  - 注意：同样是**每个指数单独**修正；跨指数汇总结论应以“方向一致 + 多处复现”为主。

3) **相对基准水平系数（不建议用来做核心结论）**
- `coef / p_value / q_value`：
  - 这是 `C(group_col)` 的哑变量系数（相对“基准水平”的差）
  - `q_value` 只对“出现的组系数项”（通常是 `k-1` 个）做 BH-FDR
  - 对 `ganzhi_day` 而言“基准水平”是字符串排序后的第一个类别，语义弱；因此更建议用 `p/q_effect`。

### 4.3 Notebook 04：稳健性模块（非回归）
同一 notebook 还包含（当前实现口径）：
- 子样本：按年份段（`2010-2015/2016-2020/2021-now`）比较 `stem=丙` 的 `ret_1d` 均值差异（Welch t-test）
- 全局置换检验：打乱标签，检验“是否存在任意组效应”（统计量=最大绝对组均值偏离）
- 样本外：按年 walk-forward（训练窗口默认 5 年），观察 OOS 年度差异与排序稳定性（Spearman）

---

## 5. 解释与边界（必须写进报告）
- 多重比较下，显著性不等于可交易；优先报告效应大小、稳定性与样本外表现。
- 若发现“仅在某些年份显著”，更可能是制度/宏观/样本结构导致的伪相关，需要用控制变量与置换检验兜底。

---

## 6. 当前运行结果摘要（数据截至 2026-02-13；报告生成 2026-02-14 16:01）

参考报告：`data/clean/report/report_20260214_160139.md`

数据覆盖：
- 指数：`000300.SH`、`000852.SH`、`000001.SH`、`399001.SZ`
- 区间：`2010-01-04..2026-02-13`（各指数 `n=3916`）

主结论（`q<=0.1`，FDR-BH）：
- `mean_ret`：仅发现 `stem=丙` 在部分指数上显著更低
  - `000300.SH`：`effect≈-0.20%/日`，`q≈0.0456`
  - `399001.SZ`：`effect≈-0.24%/日`，`q≈0.0317`
- 其余：`p_up`、`branch`、`ganzhi_day` 在本轮阈值下均未通过筛选

注意事项：
- 上述为“无控制变量”的样本内结果；更严格口径与跨指数复现结论见第 7/8 节与 `data/clean/robustness/meta_replication_pass.csv`。

---

## 7. 稳健性结果摘要（数据截至 2026-02-13；报告生成 2026-02-14 16:01）

输出目录：`data/clean/robustness/`

### 7.1 控制变量回归（`weekday/month/year`）
- `ret_1d`：OLS + HAC（Newey-West，`maxlags=5`），输出 `controls_{ts_code}_{group_col}_ret_1d.csv`
- `up`：GLM(Binomial) + HAC（Newey-West，`maxlags=5`），输出 `controls_{ts_code}_{group_col}_up.csv`

字段说明（建议以输出表为准）：
- `p_value/q_value`：组系数相对“基准水平”的显著性（不等价于“组 vs 全样本均值”）
- `p_value_effect/q_value_effect`：对比检验“组 vs 全样本边际均值”的显著性（`ret_1d` 为 contrast；`up` 为 delta-method 近似）

观察到的现象（需谨慎解释）：
- `stem=丙` 的 `ret_1d` 边际均值（marginal mean）在四个指数上均偏低（效应量约 `-0.16%~-0.23%/日`），说明该差异**不容易被 `weekday/month/year` 直接吸收**。
- 在 `p/q_effect` 口径下，`stem=丙` 在 `000300.SH`（`q≈0.0474`）与 `399001.SZ`（`q≈0.0372`）通过；在 `000001.SH/000852.SH` 未通过（`q>0.1`）。
- `branch` 在本轮 `p/q_effect` 口径下未见 `q<=0.1` 的稳定信号。
- `up`：未见跨指数一致通过；但 `000001.SH` 出现 `stem=癸` 单点通过（`q_effect≈0.07`，上涨概率边际效应约 `+6.4pct`）。
- `ganzhi_day`（60 甲子）：
  - `ret_1d`：未见 `q_effect<=0.1`；`丙午` 在四个指数上均为显著负向候选（边际效应约 `-0.44%~-0.54%/日`），但多重比较后 `q_effect` 仍在 `0.22~0.50`。
  - `up`：仅 `000852.SH` 出现 `壬寅` 单点通过（`q_effect≈0.07`，上涨概率边际效应约 `+17.5pct`）；其余指数未通过。

### 7.2 子样本（年份段）
文件：`data/clean/robustness/subsample_bing.csv`
- `stem=丙` 的 `mean_ret` 差异在所有指数、所有年份段里方向一致为负；在 `2021-now` 分段更强、也更容易显著。

### 7.3 置换检验（全局）
文件：`data/clean/robustness/perm_global.csv`
- `stem × ret_1d` 的全局置换检验在 `000300.SH`（`p≈0.034`）与 `399001.SZ`（`p≈0.025`）通过；其余组合未见稳定通过。
- 注意：这是对原始序列做的全局检验，并未先回归掉控制变量；更保守版本见 7.5（controls 残差置换）。

### 7.4 样本外（walk-forward，按年）
文件：`data/clean/robustness/walk_forward_{ts_code}_stem.csv`
- `stem=丙` 的 OOS 年度差异多数年份为负（约 `10/11` 或 `11/11` 年），但整体天干排序的跨期相关较弱（Spearman 年际波动大）。

### 7.5 控制变量残差置换（全局）
文件：`data/clean/robustness/perm_global_controls_resid.csv`
- `stem × ret_resid`（`ret_resid`=先回归掉 `weekday/month/year` 的残差）的全局置换检验：
  - `399001.SZ`：`p≈0.015`
  - `000300.SH`：`p≈0.024`
  - `000001.SH`：`p≈0.060`
  - `000852.SH`：`p≈0.168`
- 解读：这是“是否存在任意 stem 残差均值偏离”的全局检验；不直接给出某个 stem 的单点 p-value。

### 7.6 HAC `maxlags` 敏感性（控制变量回归，stem×ret_1d）
文件：`data/clean/robustness/hac_sensitivity_controls_{ts_code}_stem_ret_1d.csv`
- `stem=丙`：在 `maxlags ∈ [0,1,3,5,10]` 下
  - `000300.SH/399001.SZ` 的 `q_value_effect` 均在 `≈0.03~0.05`，对 `maxlags` 不敏感
  - `000001.SH/000852.SH` 的 `q_value_effect` 均大于 `0.1`

### 7.7 block bootstrap（controls 残差，stem=丙）
文件：`data/clean/robustness/block_bootstrap_controls_resid_{ts_code}_stem_ret_1d.csv`
- `stem=丙` 的 95% CI 在 4 个指数上均为负（约 `-28~-3 bp/日` 至 `-38~-8 bp/日`），对应 `p_boot≈0.002~0.03`。

### 7.8 跨指数复现（Meta + OOS）
文件：`data/clean/robustness/meta_controls_stem_ret_1d.csv` / `oos_stability_stem_ret_1d.csv` / `meta_replication_pass.csv`
- 正式通过（`q_meta<=0.1` + 方向一致 + OOS 稳定）：`stem=丙`
  - `effect_meta≈-0.192%/日（≈-19.2bp/日）`，`q_meta≈3e-06`（4 指数一致为负）
  - OOS：4 指数均满足 `match_ratio≈0.91~1.00` 且符号检验 `p<=0.1`
- 候选但未通过 OOS：`癸`（`q_meta≈0.008`）、`丁`（`q_meta≈0.099`）的 OOS `match_ratio` 分别约 `0.73/0.64`，未进入正式结论；详细诊断见 7.9 与报告 04h 小节。

### 7.9 候选 stem（癸/丁）深挖结论（诊断：时间稳定性）
诊断输出：`data/clean/robustness/` 的 `candidate_*`、`walk_forward_*_controls*` 与 `oos_window_sensitivity_*`；图见 `data/clean/report/figures/`（`candidate_*png`），汇总见报告 04h。

结论要点（仅解释“为何没通过 OOS”，不改变正式筛选规则）：
- `stem=丁`：raw OOS（按年）符号序列在四个指数上一致为 `+-++++---++`，翻符号年份集中在 `2016` 与 `2021-2023`（相对 Meta 的正方向），因此 `match_ratio≈0.636`（符号检验 `p≈0.55`）。
- `stem=癸`：raw OOS 符号序列大致为 `-+-+-++++++`（`000852.SH` 略有差异），翻符号主要发生在早期年份（如 `2015/2017/2019`），整体 `match_ratio≈0.727`（符号检验 `p≈0.23`）。
- controls-only residual OOS（训练窗内回归掉 `weekday/month`，不含 `year`）后，两者 `match_ratio` 与 raw 基本一致，说明“候选不稳”**不是**简单的 `weekday/month` 代理效应造成。
- 预设断点（2020 前后）对比：`癸` 的年度均值差在 2020 后更偏正、`丁` 在 2020 后更接近 0；但年份层面置换 p-value 多在 `0.14~0.31`，由于 OOS 年份数仅 `11`，更适合作为提示而非结论。
- 训练窗敏感性（`train_years=3/5/7`，controls-only residual）：`癸` 的 median `match_ratio` 随窗口增大上升到 `~0.78` 但仍未达到 `0.8` 阈值；`丁` 在 `~0.64~0.67` 徘徊；因此“调参训练窗”不能把候选变成稳定结论。

---

## 8. 新增稳健性与跨指数复现（2026-02-14 起）

> 说明：本节描述新增的**方法口径与输出文件**；具体数值以重新 Run All
> `notebooks/04a_controls_models.ipynb`…`notebooks/04f_resonance_harmonics.ipynb`（索引：`notebooks/04_robustness_and_modeling.ipynb`）
> / `notebooks/05_report.ipynb` 后生成的表为准。

输出目录：`data/clean/robustness/`

### 8.1 控制变量残差置换（全局）
目的：把 `weekday/month/year` 先回归掉，再检验 `stem` 是否仍对“残差收益”有全局信息量。

口径：
- 残差：先拟合 `ret_1d ~ C(weekday) + C(month) + C(year)`，得到 `ret_resid`
- 统计量：`T = max_g | mean(ret_resid|stem=g) - mean(ret_resid) |`
- 置换：随机打乱 `stem` 标签 `N_PERM_RESID` 次得到零分布

输出：
- `perm_global_controls_resid.csv`

### 8.2 HAC `maxlags` 敏感性（控制变量回归）
目的：检查 Newey-West `maxlags` 取值变化时，`stem×ret_1d` 的边际效应显著性是否稳定。

口径：
- 对 `maxlags ∈ [0, 1, 3, 5, 10]` 重复拟合 `ret_1d ~ C(stem) + C(weekday) + C(month) + C(year)`
- 对每个 `maxlags` 单独做 10 个天干的 BH-FDR，输出 `q_value_effect`

输出：
- `hac_sensitivity_controls_{ts_code}_stem_ret_1d.csv`

### 8.3 block bootstrap（controls 残差）
目的：用对序列相关更友好的 bootstrap 估计不确定性，给出每个天干效应的 CI 与“过零”稳定性。

口径（默认参数）：
- 数据：`ret_resid`（见 8.1）
- bootstrap：circular block bootstrap（`block_len=10`，`n_boot=1000`）
- 统计量：每个 stem 的 `effect = mean(ret_resid|stem) - mean(ret_resid)`（与置换检验口径一致）

输出：
- `block_bootstrap_controls_resid_{ts_code}_stem_ret_1d.csv`

### 8.4 walk-forward OOS effect 长表（全 stem）
目的：为跨指数的 OOS 稳定性筛选提供可复用的底表。

输出：
- `walk_forward_{ts_code}_stem_oos_effects.csv`（列：`ts_code/oos_year/stem/oos_effect`）

### 8.5 跨指数复现（Meta + OOS 稳定性筛选）
目的：把“跨指数复现”做成正式结论：先合并证据、再做统一多重比较、并要求样本外方向稳定。

口径（固定效应 Meta）：
- 输入：`controls_{ts_code}_stem_ret_1d.csv` 的 `marginal_effect_vs_all` 与 `se_effect`
- 合并：按 `w=1/se^2` 做固定效应合并得到 `p_meta`
- 多重比较：对 10 个天干的 `p_meta` 做 BH-FDR 得到 `q_meta`

通过规则（用于“正式结论”）：
- `q_meta <= 0.1`
- 方向一致：至少 `ceil(0.75*k)` 个指数与 `effect_meta` 同号（`k` 为参与合并的指数数）
- OOS 稳定：单指数满足 `match_ratio>=0.8 且 p_sign<=0.1` 的指数个数 ≥ `ceil(0.75*k)`

输出：
- `meta_controls_stem_ret_1d.csv`（Meta 全量）
- `oos_stability_stem_ret_1d.csv`（按 `stem×ts_code` 的 OOS 稳定性）
- `meta_replication_pass.csv`（通过项）

### 8.6 候选 stem 深挖（癸/丁：时间稳定性诊断）
目的：当 `stem=癸/丁` 出现 “Meta 显著但 OOS 不稳” 时，用更可解释的方式定位**哪些年份/阶段在翻符号**，以及“剥离常见日历效应后”是否更稳定。  
说明：本节输出用于诊断与解释，不改变 8.5 的“正式结论”筛选规则。

1) **raw OOS（无控制变量）年度效应**
- 输入：`walk_forward_{ts_code}_stem_oos_effects.csv`
- 输出（候选汇总）：  
  - `candidate_oos_yearly_raw_stem_ret_1d.csv`（列：`ts_code/oos_year/stem/oos_effect/meta_sign/sign_match`）  
  - `candidate_oos_flip_years_stem_ret_1d.csv`（列：`stem/ts_code/match_ratio_raw/flip_years_raw/sign_seq_raw/...`）
- 解读：把 OOS 年度 `oos_effect` 逐年画出来，并把“不符合 Meta 方向”的年份列出来，便于解释 `match_ratio` 为什么卡在 `0.64/0.73`。

2) **controls-only residual OOS（剥离 weekday/month）**
目的：检验候选 stem 是否只是 `weekday/month` 日历效应的代理变量。
- 口径（walk-forward 训练窗内拟合，避免泄露）：  
  - 在训练窗拟合 `ret_1d ~ C(weekday) + C(month)`（**不含 year**，避免把年份 dummy 当作可泛化信息）  
  - 在 OOS 年计算残差 `ret_resid_controls = ret_1d - y_hat_controls`  
  - 再按 stem 做年度差异：`oos_effect = mean(ret_resid_controls|stem) - mean(ret_resid_controls)`（同 raw 口径）
- 输出：  
  - `walk_forward_{ts_code}_stem_oos_effects_controls.csv`（列：`ts_code/oos_year/stem/oos_effect`）  
  - `walk_forward_{ts_code}_stem_oos_effects_controls_sweep.csv`（列同上 + `train_years`，用于窗口敏感性）  
  - `oos_stability_stem_ret_1d_controls.csv`（按 `stem×ts_code` 的稳定性，match_ratio 相对 Meta 方向）
  - `candidate_oos_yearly_controls_stem_ret_1d.csv`（候选 stem 的 controls-only OOS 年度长表）

3) **断点对比（预设 2020 前后；年份层面置换）**
- 输出：`candidate_break_test_stem_ret_1d.csv`
- 含义：对候选 stem 的年度 `oos_effect` 做 `2015-2019` vs `2020-2025` 的均值差，并在“年份层面”随机重排分组做经验 p-value（仅作提示；年份数很少）。

4) **训练窗敏感性（train_years=3/5/7；controls-only residual）**
- 输出：  
  - `oos_window_sensitivity_controls_{ts_code}_stem_ret_1d.csv`（按 `train_years×stem` 给出 `match_ratio/p_sign`）  
  - `oos_window_sensitivity_summary_controls_stem_ret_1d.csv`（仅汇总 `丙/癸/丁` 的跨指数中位数 match_ratio 与通过数）

---

## 9. Phase 2（已实现）：`day_group × year_element` 交互（仅 `ret_1d`；确认性 gate → 才看局部）

> 目标：把 `Thoughts.txt` 的“时变/环境依赖”叙事转化为可证伪的交互检验。  
> 关键：先做全局交互的确认性 gate，再决定是否展示局部 cell（避免维度爆炸与 p-hacking）。

### 9.1 模型族（不含 `C(year)`）
对每个 `ts_code`、每个 `day_group ∈ {stem, branch, ganzhi_day}`：
- `ret_1d ~ C(day_group) * C(year_element) + C(weekday) + C(month)`
- 协方差：HAC / Newey-West（`maxlags` 必须在 `docs/Preregistration.md` 中冻结）

说明：
- 该阶段的“时间结构”由 `year_element` 表达，因此不再把 `C(year)` 当作控制项加入（避免同构与不可泛化）。

### 9.2 层级检验（Hierarchical testing）

**层 1（Confirmatory gate：全局交互是否存在）**
- 单指数：对交互项整体做 joint Wald test → `p_interaction(ts_code, day_group)`
- 跨指数：对同一 `day_group` 的 `p_interaction` 用 Fisher 合并 → `p_meta_interaction(day_group)`
- 多重比较：对 3 个 `day_group` 的 `p_meta_interaction` 做 BH-FDR → `q_meta_interaction(day_group)`

建议的 gate 通过规则（需在 prereg 冻结）：
- `q_meta_interaction(day_group) <= 0.1`
- 且至少 `3/4` 个指数满足 `p_interaction <= 0.1`

若 gate 不通过（可证伪标准）：
- 不输出或不解读局部 cell 显著性；“年运/五行解释”在当前数据范围内缺乏统计支持，应暂停该解释路径（除非预注册新的 state）。

**层 2（Exploratory：局部 cell 的解释与候选）**
- 仅当 gate 通过，才允许输出局部 cell 的边际效应表/热力图；
- 局部 cell 必须：
  - 标注为 Exploratory（不得替代确认性结论）
  - 给出跨指数方向一致性与 OOS 稳定性
  - 并在对应 family 内做 BH-FDR（family 定义在 prereg 冻结）

### 9.3 计划输出（文件接口，供 notebook 落盘）
确认性（最小集）：
- `data/clean/robustness/interaction_joint_test_{ts_code}_{day_group}_ret_1d.csv`
- `data/clean/robustness/interaction_joint_tests_ret_1d.csv`（长表汇总，便于审计/复盘）
- `data/clean/robustness/meta_interaction_{day_group}_ret_1d.csv`
- `data/clean/robustness/interaction_gate_summary_ret_1d.csv`

探索性（仅 gate 通过后）：
- `data/clean/robustness/interaction_cell_effects_{ts_code}_{day_group}_ret_1d.csv`
- `data/clean/robustness/interaction_candidates_{day_group}_ret_1d.csv`（候选清单；Exploratory；由 `05_report` 生成）
- `data/clean/report/figures/phase2_interaction_heatmap_{day_group}_*.png`

### 9.4 最新 gate 结果（截至 2026-02-14 16:01）
参考：
- gate 汇总：`data/clean/robustness/interaction_gate_summary_ret_1d.csv`
- 一键报告：`data/clean/report/report_20260214_160139.md`（04i）

| day_group | q_meta_interaction | pass_gate | n_pass_p_interaction / 4 |
| --- | --- | --- | --- |
| stem | 0.264882 | False | 0 |
| branch | 6.84798e-05 | True | 3 |
| ganzhi_day | 9.68508e-26 | True | 4 |

解释边界（再次强调）：
- 上表属于确认性 gate（结论=“交互存在性”是否成立）；不等价于“某个具体 cell 可交易”。
- gate 通过后生成的 `interaction_candidates_*` 与 heatmap 属于 Exploratory；尤其 `ganzhi_day×year_element` 单格样本数很小，应优先作为假设生成而非结论。

---

## 10. Resonance（新增）：`jiazi_idx` 的谐波检验（k=5/6；仅 `ret_1d`）

> 目的：把“10×12（stem×branch）热力图的条纹感”回到一维 60 周期（`jiazi_idx=0..59`）上，检验是否存在可复现的周期成分。  
> 重要边界：显著性只代表“周期成分存在”，不等价于机制/因果证明。

### 10.1 谐波特征定义（周期 = 60）
对每个交易日，令 `i = jiazi_idx ∈ {0..59}`，定义：
- `sK = sin(2π*K*i/60)`
- `cK = cos(2π*K*i/60)`

本阶段默认只检验：
- `k=5`（对应 12 周期）
- `k=6`（对应 10 周期）

### 10.1.1 关键恒等式：`k=5` 只依赖地支、`k=6` 只依赖天干
由于 `60/5=12`、`60/6=10`，有：
- `sin(2π*5*i/60) = sin(2π*i/12)`，`cos(2π*5*i/60)=cos(2π*i/12)`  
  ⇒ 仅依赖 `i mod 12`，而 `branch_idx = i mod 12`，所以 `k=5` 是**纯地支（12 周期）**成分。
- `sin(2π*6*i/60) = sin(2π*i/10)`，`cos(2π*6*i/60)=cos(2π*i/10)`  
  ⇒ 仅依赖 `i mod 10`，而 `stem_idx = i mod 10`，所以 `k=6` 是**纯天干（10 周期）**成分。

含义：
- 本检验的 `k=5/6` 不是“stem×branch 的非线性耦合”，而是对 **branch-only + stem-only** 的“正弦近似/频域表达”。
- 若要检验“超越 `stem+branch` 加性主效应”的 60 周期结构，应检验 **不会退化为 10/12 周期**的频率（见 10.6）。

### 10.2 主检验（含控制变量；HAC）
对每个 `ts_code`：
- controls-only（基线）：
  - `ret_1d ~ C(weekday) + C(month) + C(year)`
- harmonic（主检验）：
  - `ret_1d ~ s5 + c5 + s6 + c6 + C(weekday) + C(month) + C(year)`
- 协方差：HAC / Newey-West（`maxlags` 需在 `docs/Preregistration.md` 冻结）

检验口径：
- 单指数：对 `{s5,c5,s6,c6}` 做 joint Wald test → `p_wald_k56(ts_code)`
- 跨指数：对 4 个 `p_wald_k56` 用 Fisher 合并 → `p_meta_k56`（可选 BH-FDR，family 很小）

补充输出（用于解释，不作为通过判据）：
- 幅值/相位：
  - `beta_sK * sin(wt) + beta_cK * cos(wt) = A_K * sin(wt + phi_K)`
  - `A_K = sqrt(beta_sK^2 + beta_cK^2)`，`phi_K = atan2(beta_cK, beta_sK)`
- `delta_r2`：相对 controls-only 的 `R^2` 增量（描述性）

### 10.3 诊断：是否“超越加性主效应”（Exploratory）
用于判断周期成分是否只是 stem/branch 主效应叠加的投影结果：
1) 先拟合：
   - `ret_1d ~ C(stem) + C(branch) + C(weekday) + C(month) + C(year)`
2) 得到残差 `ret_resid_additive`
3) 再拟合：
   - `ret_resid_additive ~ s5 + c5 + s6 + c6`
4) 对 `{s5,c5,s6,c6}` 做 joint Wald test，得到 `p_wald_k56_resid_additive`

说明：
- 结合 10.1.1（`k=5`=branch-only，`k=6`=stem-only），在回归中显式加入 `C(stem)+C(branch)` 后，残差对 `s5/c5/s6/c6` **应近似正交**，因此 `p_wald_k56_resid_additive` 往往接近 1 是**预期现象**。
- 因此：`p_wald_k56_resid_additive≈1` 不是“否定共振”，而更像是“k=5/6 的周期成分可以被 `stem+branch` 的加性结构解释”。  
  若要检验“超越加性”的结构，请看 10.6 的建议路线。

### 10.4 输出文件接口（由 notebooks/04a–04f 与 notebooks/05 落盘）
统计表（`data/clean/robustness/`）：
- `resonance_harmonic_k56_{ts_code}_ret_1d.csv`
- `resonance_harmonic_k56_ret_1d.csv`（长表汇总）
- `resonance_after_additive_{ts_code}_ret_1d.csv`
- `resonance_after_additive_ret_1d.csv`（长表汇总）
- `resonance_meta_k56_ret_1d.csv`（跨指数 Fisher 合并）

图（`data/clean/report/figures/`；诊断用途）：
- `resonance_fitcurve_{ts_code}_ret_1d_*.png`
- `resonance_spectrum_{ts_code}_ret_1d_*.png`
- `resonance_embed_heatmap_{ts_code}_ret_1d_*.png`

### 10.5 本轮运行结果摘要（数据截至 2026-02-13；报告生成 2026-02-14 22:51）
参考：
- 报告：`data/clean/report/report_20260214_225132.md`
- 主表：`data/clean/robustness/resonance_harmonic_k56_ret_1d.csv`
- 诊断：`data/clean/robustness/resonance_after_additive_ret_1d.csv`

结论（就“条纹是否只是噪声/色标放大”而言）：
- **样本量均衡**：60 个 `jiazi_idx` 的 `n` 分布为 `min=61, median=65, max=70`，且 `n_missing=0`（排除“某些格子样本太少”的主要担忧）。
- **主检验（k=5/6 joint）跨指数显著**：Fisher 合并 `p_meta_k56≈0.00274`。单指数里 `000001.SH` 最强（`p≈0.0079`），`000300.SH` 边缘（`p≈0.0359`），其余两只不显著。
- **效应大小很小但可检出**：`amp_5≈7.5~8.6 bp/日`、`amp_6≈3.4~4.7 bp/日`，`delta_r2≈0.0013~0.0029`。
- **“回归掉 stem+branch 后再检 k=5/6”几乎全为 1**：`p_wald_k56_resid_additive≈0.99`（与 10.1.1 的结构性解释一致：k=5/6 属于加性主效应空间）。

补充分解（用于解释，不作为当前落盘主输出）：
- 把 joint Wald test 拆成两个子检验（`s5=c5=0` vs `s6=c6=0`）会发现，本轮显著性主要来自 **k=5（12 周期/地支）**；而 **k=6（10 周期/天干）**在跨指数合并下不显著。  
  （这与 `amp_5 > amp_6` 的数量级关系一致。）

### 10.6 若要检验“超越 stem+branch 加性”的 60 周期结构（下一步建议）
若你的“共振”定义是 **60 甲子层面的耦合/非线性**（而非 stem-only/branch-only），建议（需预注册）：
1) **扩展频率集合**：检验不退化为 10/12 周期的 `k`（例如 gcd(`k`,60)=1 的频率会保留 60 周期），并在该 family 内做 BH-FDR；避免事后挑 k。
2) **嵌套模型检验**：比较 `C(ganzhi_day)+controls` 与 `C(stem)+C(branch)+controls` 的拟合增益（HAC/Wald joint test），判断是否存在“超出加性主效应”的系统性结构。

---

## 11. US 对照实验（负控制：华盛顿时间口径；`notebooks_US/`）

> 目的：用同一套统计流程，在“非中国市场/非本土叙事环境”的 US 宽基指数上做对照；用于检验干支效应是否具有跨市场普遍性。

### 11.1 US 版本如何构造（与 A 股版本的差异）
Notebook：
- US 全流程位于 `notebooks_US/`，文件名与 A 股版本一一对应（运行顺序同样是 `01→02→03→04*→05`）。

数据源（免 Token）：
- Stooq 日线（`notebooks_US/01_fetch_market_data.ipynb`），缓存到 `data/cache_us/index_daily/`：
  - `spx`=`^spx`（S&P 500）
  - `ndq`=`^ndq`（Nasdaq Composite）
  - `ndx`=`^ndx`（Nasdaq 100）
  - `dji`=`^dji`（Dow Jones Industrial Average）

输出目录（与 A 股版本平行）：
- `data/clean_us/market_{symbol_id}.csv.gz`
- `data/clean_us/ganzhi_trade_dates.csv.gz`
- `data/clean_us/market_ganzhi_{symbol_id}.csv.gz`
- `data/clean_us/robustness/*`

华盛顿时间口径（关键冻结点）：
- 日柱：以 `America/New_York`（华盛顿/纽约同一时区）的自然日 `00:00~24:00` 对 `trade_date` 计算。
- 立春年/年五行：真值源为 `docs/li_chun_year_mapping_washington.csv`（由 UTC 立春时刻换算为华盛顿本地时间）。
  - 立春当日的归属用美股收盘 `trade_close_hm = 16:00` 判断（见 `notebooks_US/02_build_ganzhi_calendar.ipynb` 的实现）。

### 11.2 US 结果摘要（数据截至 2026-02-13；区间 2010-01-05..2026-02-13）
主效应（Phase 1）：
- `03`（无控制）与 `04a`（控制 `weekday/month/year`）对 `stem/branch/ganzhi_day × (ret_1d/up)` 均未出现 `q<=0.1` 的稳定信号（输出：`data/clean_us/ganzhi_tests_*.csv`、`data/clean_us/robustness/controls_*`）。
- `04c` 全局置换检验（“是否存在任意组效应”）：最小 `p_empirical≈0.216`（输出：`data/clean_us/robustness/perm_global.csv`）。
- `04f` Resonance（k=5/6；HAC；含 controls）：跨指数 `p_meta_k56≈0.912`（输出：`data/clean_us/robustness/resonance_meta_k56_ret_1d.csv`）。

Phase 2（`day_group × year_element` 交互 gate）：
- `ganzhi_day × year_element` 在 US 也会通过 gate（`q_meta_interaction≈1.91e-12`；输出：`data/clean_us/robustness/interaction_gate_summary_ret_1d.csv`）。
- 解释边界：该 family 参数维度极高，且 `ganzhi_day×year_element` 单格样本数很小（US 版单格 `n_cell` 约 `6~20`）；因此更像是“时间非平稳 + 高维交互可塑性”触发的统计显著，不建议把其当作干支机制证据继续深挖。

### 11.3 对比结论（CN vs US）
- US 负控制下，“日柱主效应”的核心证据链（03/04a/04c/04f）整体为 null，未支持“跨市场通用的自然时间因子”解释。
- 结合 A 股侧的少数可复现现象（例如 `stem=丙` 的跨指数 + OOS 稳定）与 US null，对本项目更一致的结论是：所谓“天干地支效应”更可能是一种 **中国本土投资者信念/行为与市场结构共同作用的‘自我实现’**，而非跨市场通用规律。
