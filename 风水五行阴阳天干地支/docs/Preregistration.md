# Preregistration（预注册清单：每次扩展前填写）

> 用途：把“我要验证什么”写在跑之前，避免事后挑断点/挑窗口/挑显著。
>
> 规则：只要会影响结论口径，就必须先在这里写清楚，并在报告中引用本文件版本。

当前模板版本：v0.2（2026-02-15；新增 US 对照版本字段）

---

## 0. 本次预注册元信息（必填）
- 预注册 ID：`prereg_YYYYMMDD_HHMM`（例如 `prereg_20260214_1600`）
- 负责人：
- 预计运行的时间区间（数据截止到哪天）：
- 将生成的报告文件名（`{CLEAN_DIR}/report/report_*.md`）：
- 市场版本（冻结）：`CN_Ashares` / `US_broad`（或自定义，但需写清楚数据源）
- 指数列表（冻结）：（CN：`000300.SH/...`；US：`spx/ndq/ndx/dji`）
- 输出根目录 `CLEAN_DIR`（冻结）：`data/clean/` 或 `data/clean_us/`
- 时区口径（冻结）：CN=`Asia/Shanghai`（北京时间）；US=`America/New_York`（华盛顿时间）
- 立春映射表（冻结）：CN=`docs/li_chun_year_mapping.csv`；US=`docs/li_chun_year_mapping_washington.csv`
- 市场收盘时刻 `trade_close_hm`（冻结）：CN=`15:00`；US=`16:00`

---

## 1. 本次要回答的确认性问题（Confirmatory）

> 说明：确认性问题必须给出“通过/失败”判据；失败同样是结果。

### 1.1 Phase 2：`day_group × year_element` 交互（仅 `ret_1d`）
本次启用的 `day_group`（勾选）：
- [ ] `stem`
- [ ] `branch`
- [ ] `ganzhi_day`

状态变量（冻结）：
- `year_element`（由立春映射表定义；CN=`docs/li_chun_year_mapping.csv`；US=`docs/li_chun_year_mapping_washington.csv`）

模型族（冻结）：
- `ret_1d ~ C(day_group) * C(year_element) + C(weekday) + C(month)`
- 标准误：HAC / Newey-West（`maxlags = ____`）

层级检验 gate（冻结）：
- 单指数交互 joint test 阈值：`p_interaction <= ____`
- 跨指数合并：Fisher 合并 p（写清参与指数列表）
- 多重比较：对 `stem/branch/ganzhi_day` 三个 `p_meta_interaction` 做 BH-FDR
- gate 通过规则：
  - `q_meta_interaction(day_group) <= ____`
  - 且 `>= ____ / ____` 个指数满足 `p_interaction <= ____`

通过后的解释边界（冻结）：
- [ ] gate 不通过：不输出 cell-level 候选（或只放“不可解读”的附录）
- [ ] gate 通过：允许输出 cell-level 候选，但必须标注 Exploratory，并附带 OOS 与 FDR

---

### 1.2 Resonance：`jiazi_idx` 的谐波检验（k=5/6；仅 `ret_1d`）
> 目的：把“10×12 热力图条纹”回到 60 周期（`jiazi_idx=0..59`）上，检验是否存在可复现的周期成分（不等价于机制/因果）。

本次启用（勾选）：
- [ ] `k=5&6`（默认；`k=5` 对应 12 周期，`k=6` 对应 10 周期）

模型族（冻结）：
- controls-only（基线）：`ret_1d ~ C(weekday) + C(month) + C(year)`
- harmonic（主检验）：`ret_1d ~ s5 + c5 + s6 + c6 + C(weekday) + C(month) + C(year)`
  - 其中：`sK = sin(2π*K*jiazi_idx/60)`，`cK = cos(2π*K*jiazi_idx/60)`
- 标准误：HAC / Newey-West（`maxlags = ____`）

检验口径（冻结）：
- 单指数：对 `{s5,c5,s6,c6}` 做 joint Wald test → `p_wald_k56(ts_code)`
- 跨指数：对 4 个 `p_wald_k56` 用 Fisher 合并 → `p_meta_k56`
- 通过/失败判据（冻结）：`p_meta_k56 <= ____`（可选：在本 family 内做 BH-FDR 并给出 `q_meta_k56`）

诊断（可选，冻结为 Exploratory）：
- [ ] 先回归掉 `stem+branch` 主效应后，检验残差：`ret_resid_additive ~ s5 + c5 + s6 + c6`
- 说明：该诊断用于判断“周期成分是否超出 stem/branch 加性主效应”，不得替代主检验。

---

## 2. 本次探索性/诊断性问题（Exploratory，可选）
> 说明：探索性输出不得升级为结论；若要升级，必须单独写新的 prereg。

- [ ] 候选 stem（如 `癸/丁`）机制诊断（写清只做“定位不稳定来源”，不改结论门槛）
- [ ] 断点对比（必须预先指定断点，不得搜索）
- [ ] 训练窗敏感性（必须预先给出候选窗口集合，不得搜索最优）

---

## 3. 输出文件清单（必须落盘，可审计）

> 每一项都要写清楚“由哪个 notebook 产出”，并落在 `{CLEAN_DIR}/` 或其子目录。

确认性（建议最小集）：
- `{CLEAN_DIR}/robustness/interaction_joint_test_{ts_code}_{day_group}_ret_1d.csv`
- `{CLEAN_DIR}/robustness/interaction_joint_tests_ret_1d.csv`（长表汇总，便于审计）
- `{CLEAN_DIR}/robustness/meta_interaction_{day_group}_ret_1d.csv`
- `{CLEAN_DIR}/robustness/interaction_gate_summary_ret_1d.csv`

确认性（若启用 Resonance 1.2）：
- `{CLEAN_DIR}/robustness/resonance_harmonic_k56_{ts_code}_ret_1d.csv`
- `{CLEAN_DIR}/robustness/resonance_harmonic_k56_ret_1d.csv`（长表汇总）
- `{CLEAN_DIR}/robustness/resonance_meta_k56_ret_1d.csv`

探索性（仅 gate 通过后）：
- `{CLEAN_DIR}/robustness/interaction_cell_effects_{ts_code}_{day_group}_ret_1d.csv`
- `{CLEAN_DIR}/robustness/interaction_candidates_{day_group}_ret_1d.csv`
- `{CLEAN_DIR}/report/figures/phase2_interaction_heatmap_{day_group}_*.png`

诊断图（Resonance；仅作解释，不作为通过判据）：
- `{CLEAN_DIR}/report/figures/resonance_fitcurve_{ts_code}_ret_1d_*.png`
- `{CLEAN_DIR}/report/figures/resonance_spectrum_{ts_code}_ret_1d_*.png`
- `{CLEAN_DIR}/report/figures/resonance_embed_heatmap_{ts_code}_ret_1d_*.png`

---

## 4. 禁止事项（写清楚，防止“无意中 p-hack”）
- 不得新增/替换断点与状态变量（除非先更新 prereg）。
- 不得在全局 gate 失败后仍挑选局部显著当“发现”。
- 不得在看到结果后临时改 `maxlags`、改多重比较 family、改通过阈值。
