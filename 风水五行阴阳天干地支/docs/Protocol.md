# Protocol（研究协议：可复现 / 可迭代 / 可证伪）

> 目的：把“风水/玄学叙事”翻译成 **可被否定** 的统计假设与检验流程，并把“正式结论”与“探索性诊断”严格分离。
>
> 本协议是项目的**方法学真值源**：Notebook 可以迭代，但任何会改变结论口径的升级都必须先更新 `docs/Preregistration.md`。

协议版本：v0.2（更新于 2026-02-15；新增 US 负控制对照）  
适用范围：`风水五行阴阳天干地支/` 目录下全部 notebook 与文档

---

## 1. 研究分层：Confirmatory vs Exploratory

### 1.1 Confirmatory（确认性：可写成“正式结论”）
只有满足以下条件的发现，才能被写为项目“正式结论”：
- **跨指数复现**：在多个宽基指数上方向一致，并以合并证据（meta / 合并 p）+ 统一多重比较控制；
- **样本外稳定（OOS）**：使用 walk-forward（按年）检验方向稳定性；
- **多重比较控制**：在预先定义的 family 内做 BH-FDR（必要时给 Bonferroni 上界）。

> 备注：截至 `data/clean/report/report_20260214_160139.md`：
> - Phase 1（Meta+OOS）：确认性结论仅 `stem=丙` 通过；
> - Phase 2（交互 gate）：`branch`、`ganzhi_day` 的 `day_group × year_element` 交互通过 gate；`stem` 未通过（局部 cell 仍为 Exploratory）。
> - US 对照（负控制）：Phase 1 主效应整体为 null（见第 7 节）。

### 1.2 Exploratory（探索性：只能写“候选/诊断/机制假说”）
满足以下任一情况的，只能作为探索性输出：
- 单指数显著但跨指数不稳；
- Meta 显著但 OOS 不稳（例如 `癸/丁` 目前属于该类）；
- 事后挑选断点/窗口/分组才显著（未预注册）。

探索性输出必须：
- 明确标注“诊断/提示”，不得替代确认性结论；
- 若要升级为确认性检验，必须先在 `docs/Preregistration.md` 里冻结假设与阈值。

---

## 2. 数据与口径冻结点（Phase 1）

### 2.1 市场代理（默认）
默认宽基指数代表“A 股整体”：
- `000300.SH`、`000852.SH`，并用 `000001.SH/399001.SZ` 作为对照（以接口可用为准）。

### 2.2 交易日与时区
- 交易日以 Tushare `trade_cal` 为准。
- 干支“日界”固定为北京时间自然日 `00:00~24:00`（避免 23:00 子时换日争议）。

### 2.3 目标变量（Phase 1 已实现）
- 主目标：`ret_1d = close/prev_close - 1`
- 方向标签（可选）：`up = 1[ret_1d>0]`（用于描述/辅助，不作为 Phase 2 默认目标）

---

## 3. Confirmatory：当前“正式结论”的证据链（Phase 1）

本项目当前的确认性结论流程（已在 `04/05` 实现并落盘）：
1. 单指数控制回归：`ret_1d ~ C(stem) + C(weekday) + C(month) + C(year)`（HAC 标准误）
2. 以 contrast 得到“组 vs 全样本边际均值”的 `effect/se/p`（避免基准水平口径误读）
3. 跨指数合并：固定效应 Meta（逆方差权重）
4. 多重比较：对 10 个天干 `p_meta` 做 BH-FDR → `q_meta`
5. OOS：按年 walk-forward，要求年度符号相对 Meta 方向稳定（`match_ratio/p_sign`）

通过规则（冻结）见 `Model.md` 的“跨指数复现（Meta + OOS）”部分。

---

## 3.5（新增，默认诊断）Resonance：`jiazi_idx` 的谐波检验（k=5/6；仅 `ret_1d`）

> 目的：把“10×12（stem×branch）热力图条纹”的直觉回到一维 60 周期（`jiazi_idx=0..59`）上，检验是否存在可复现的周期成分。  
> 边界：该检验回答的是“周期成分是否存在”，不等价于机制/因果证明，也不等价于可交易信号。

### 3.5.1 模型族（当前实现；需在 prereg 冻结阈值才可升级为确认性）
对每个 `ts_code`：
- controls-only（基线）：
  - `ret_1d ~ C(weekday) + C(month) + C(year)`
- harmonic（主检验；周期=60）：
  - `ret_1d ~ s5 + c5 + s6 + c6 + C(weekday) + C(month) + C(year)`
  - `sK = sin(2π*K*jiazi_idx/60)`，`cK = cos(2π*K*jiazi_idx/60)`
- 标准误：HAC / Newey-West（`maxlags` 需在 `docs/Preregistration.md` 冻结）

检验口径（当前实现）：
- 单指数：对 `{s5,c5,s6,c6}` 做 joint Wald test → `p_wald_k56(ts_code)`
- 跨指数：对 4 个 `p_wald_k56` 用 Fisher 合并 → `p_meta_k56`
- 多重比较：该 family 很小（4 个指数），仍输出 BH-FDR 作为一致格式（`q_wald_k56`）

输出文件（落盘到 `data/clean/robustness/`）：
- `resonance_harmonic_k56_{ts_code}_ret_1d.csv` / `resonance_harmonic_k56_ret_1d.csv`
- `resonance_meta_k56_ret_1d.csv`

### 3.5.2 关键解释：k=5/6 与 stem/branch 的结构关系
由于 `sin(2π*5*i/60)=sin(2π*i/12)`、`sin(2π*6*i/60)=sin(2π*i/10)`：
- `k=5` 是 **12 周期（仅依赖地支）**；
- `k=6` 是 **10 周期（仅依赖天干）**；
因此：该检验更像是“stem/branch 主效应的频域表达”，不是 `stem×branch` 的非线性耦合。

### 3.5.3 诊断（Exploratory）：回归掉 `stem+branch` 后再检 k=5/6
用于判断“条纹是否只是加性主效应空间的投影”：
1) 拟合：
   - `ret_1d ~ C(stem) + C(branch) + C(weekday) + C(month) + C(year)`
2) 残差 `ret_resid_additive`
3) 再拟合：
   - `ret_resid_additive ~ s5 + c5 + s6 + c6`
4) joint Wald test → `p_wald_k56_resid_additive`

注意：由于 k=5/6 本身属于 stem/branch 的加性空间，上述残差检验 `p≈1` 往往是预期现象；它不否定“周期成分存在”，而是提示“该周期成分可由加性主效应解释”。

### 3.5.4 结论分层建议
默认将 Resonance 作为**诊断/解释性输出（Exploratory）**，用于支持或反驳“条纹=投影纹理+噪声放大”的判断。  
若要升级为确认性结论，需在 `docs/Preregistration.md` 冻结：
- 频率集合（例如仅 k=5/6 或扩展到其它 k）
- 单指数与跨指数的通过阈值（以及是否需要 OOS 稳定性）
- 多重比较 family 定义

---

## 4. Phase 2（已实现）：立春年 / 年五行 `year_element` 交互（仅 `ret_1d`）

> 背景：`Thoughts.txt` 的核心提示是“市场可能 time-varying”，因此需要把叙事翻译为可证伪的**交互项假设**。
>
> 本阶段只引入一个状态变量：`year_element`（五行 5 类），并采用层级检验防止维度爆炸与事后解释。

### 4.1 立春年口径（Year boundary）
- 采用“立春年”（不是公历年）。
- 立春时刻/日期的真值源：`docs/li_chun_year_mapping.csv`
  - 本表包含 `li_chun_begin_utc` 以及换算到 UTC+8 的 `li_chun_date_yyyymmdd`。
  - 对每个 `trade_date`，定义：
    - 用“市场收盘时刻”近似该交易日的时间戳：`trade_close_hm = 15:00`（北京时间）
    - `li_chun_year = trade_year - 1` 若 `trade_close_datetime(trade_date) < li_chun_begin_local(trade_year)`；否则 `li_chun_year = trade_year`
      - 等价实现：若 `trade_date < li_chun_date(trade_year)` → 上一年；若 `trade_date >` → 当年；若 `trade_date ==` → 取决于 `li_chun_time_hm` 是否晚于 `15:00`
    - `year_stem/year_element` 由 `li_chun_year` 决定（见表中字段）

数据来源说明（用于审计/复现）：
- `docs/li_chun_year_mapping.csv` 的 `li_chun_begin_utc` 来自 Wikipedia “Lichun” 页面 *Date and Time (UTC)* 表（该表注明来源为 JPL Horizons）
- `li_chun_begin_local/li_chun_date_yyyymmdd` 为按 UTC+8（北京时间/香港时间）换算得到

### 4.2 Phase 2 的确认性假设（可证伪）
H1（状态依赖 / 交互存在性）：
- 零假设：`day_group` 与 `year_element` **无交互**（即不同 `year_element` 下，日干支差异结构不变）。
- 可证伪标准：若交互项在跨指数 + 多重比较控制下不通过，则“年运/五行解释”在当前数据范围内缺乏支持，应暂停该解释路径。

### 4.3 模型族（仅 `ret_1d`；不再使用 `C(year)`）
对每个 `day_group ∈ {stem, branch, ganzhi_day}`，在每个指数分别拟合：
- `ret_1d ~ C(day_group) * C(year_element) + C(weekday) + C(month)`
- 标准误：HAC（Newey-West），`maxlags` 取 Phase 1 的默认值（如需改动必须预注册）

> 说明：此处 **不含 `C(year)`**，避免把年份 dummy 当作可泛化信息、并避免与 `year_element` 高度同构导致识别/解释混乱。

### 4.4 层级检验（Hierarchical testing，Phase 2 的关键）
Phase 2 不直接从 50/60/300 个 cell 里“挑显著”，而是两层：

**层 1（确认性 gate：全局交互是否存在）**
- 单指数：对 `C(day_group):C(year_element)` 的所有交互项做 joint Wald test → 得到 `p_interaction(ts_code, day_group)`
- 跨指数：合并 p（Fisher）→ `p_meta_interaction(day_group)`
- 多重比较：对 3 个 `day_group`（stem/branch/jiazi）的 `p_meta_interaction` 做 BH-FDR → `q_meta_interaction(day_group)`
- Gate 通过规则（预注册，建议默认如下）：
  - `q_meta_interaction(day_group) <= 0.1`
  - 且至少 `3/4` 个指数满足 `p_interaction <= 0.1`

**层 2（探索性：局部 cell 的解释与候选）**
- 仅当层 1 gate 通过，才允许输出 cell-level 的效应热力图与候选清单；
- 局部候选仍需：
  - 跨指数方向一致；
  - OOS（按年）方向稳定；
  - 且在对应 family 内做 BH-FDR（具体 family 在 `Preregistration.md` 冻结）。

### 4.5 当前 gate 结果（仅记录；不改变协议）
截至 `2026-02-14 16:01`（见 `data/clean/report/report_20260214_160139.md` 的 04i）：
- gate 通过：`branch`（`q_meta_interaction≈6.85e-05`；`3/4` 指数单指数 joint test `p<=0.1`）、`ganzhi_day`（`q_meta_interaction≈9.69e-26`；`4/4`）
- gate 未通过：`stem`（`q_meta_interaction≈0.265`）
- 因此仅对 `branch/ganzhi_day` 输出局部 heatmap 与 `interaction_candidates_*`（均为 Exploratory）。

---

## 5. 报告与落盘规范
- 所有实验输出落盘到 `data/`（不写入 repo 根以外位置）。
- 报告必须包含：
  - run timestamp（已有）
  - 本次协议版本 + prereg 版本（新增）
  - 明确标注“确认性结论”与“探索性诊断”

---

## 6. 变更流程（避免“跑出来才改口径”）
任何会改变以下任一项的改动，都必须先更新 `docs/Preregistration.md`：
- 新的状态变量 / 新的分段断点 / 新的窗口长度搜索范围
- 新的通过阈值（q、OOS 稳定性阈值、方向一致比例等）
- 新的多重比较 family 定义

---

## 7. 跨市场负控制（US 版本；华盛顿时间口径）

> 目的：把“干支是自然时间因子（跨市场通用）”作为一个可证伪命题。  
> 若该命题为真，则在 US 宽基指数上，用同一套分析流程应能观察到相近量级、可复现的信号；若为假，则 US 结果应接近 null。

### 7.1 US 版本如何构造（冻结点）
- Notebook：`notebooks_US/01..05`（与 A 股版本一一对应）
- 数据源：Stooq 日线（免 API key）
- 指数代理：`spx(^spx)`、`ndq(^ndq)`、`ndx(^ndx)`、`dji(^dji)`
- 输出目录：`data/cache_us/`、`data/clean_us/`、`data/clean_us/robustness/`
- 时区与口径：
  - 日柱：以 `America/New_York`（华盛顿/纽约同一时区）自然日 `00:00~24:00` 的 `trade_date` 计算
  - 立春年：真值源 `docs/li_chun_year_mapping_washington.csv`（UTC → 华盛顿本地时间）；立春当日以美股收盘 `16:00` 判定归属

### 7.2 结果记录（截至 2026-02-13）
Phase 1（主效应链条）：
- `03`/`04a`：未出现 `q<=0.1` 的稳定主效应（单指数 BH-FDR；输出：`data/clean_us/ganzhi_tests_*.csv`、`data/clean_us/robustness/controls_*`）。
- `04c` 全局置换：最小 `p_empirical≈0.216`（输出：`data/clean_us/robustness/perm_global.csv`）。
- `04f` Resonance（k=5/6）：跨指数 `p_meta_k56≈0.912`（输出：`data/clean_us/robustness/resonance_meta_k56_ret_1d.csv`）。

Phase 2（交互 gate）：
- `ganzhi_day × year_element` 在 US 也会通过 gate（`q_meta_interaction≈1.91e-12`；输出：`data/clean_us/robustness/interaction_gate_summary_ret_1d.csv`）。
- 解释边界：该 family 维度极高且 cell 样本数极小（单格 `n_cell` 约 `6~20`），且 US 宽基指数高度相关；因此这一“通过”更像是高维交互对时间非平稳的敏感性体现，不宜直接被解释为“干支机制证据”。

### 7.3 协议含义（如何使用 US 负控制）
- 对“跨市场通用规律”的判断应以 Phase 1 的主效应链条（03/04a/04c/04f）为主：US 结果整体为 null → 不支持“通用自然时间因子”叙事。
- 若未来仍要推进 Phase 2 的局部解释（Phase 2b），必须（预注册后）附加更强约束，避免“高维交互在负控制也显著”的情形：
  1) 缩小 family（优先 `branch×year_element`，而非 `ganzhi_day×year_element`）并设定最小 `n_cell`；
  2) 引入 OOS 稳定性与更强负控制（例如随机打乱 `year_element` 映射、或对齐偏移的对照映射）；
  3) 明确跨指数合并的相关性边界（US 宽基指数不应被当作独立复现）。

### 7.4 当前阶段的最终结论（收尾）
综合 A 股与 US 负控制结果，本项目更一致的解释是：所谓“天干地支效应”更可能是一种 **本土投资者信念/行为与市场结构共同作用的‘自我实现’**，而非跨市场通用规律。
