# 流日干支对 A 股涨跌的影响（研究与开发计划）

> 项目阶段目标（2026-02-13 起）：用**可复现的数据管线 + 可审计的统计检验**，回答“流日干支（日柱：天干+地支）是否对 A 股日频涨跌有稳定信息量”。
>
> 重要澄清：
> - 本项目优先解决“**统计关系是否存在、是否稳健**”，不直接承诺可交易的超额收益。
> - 六十甲子属于天然多重比较场景，所有显著性必须给出 **FDR/Bonferroni** 等修正，避免“挑出来才显著”。

---

## 0.4 研究协议与预注册（从“项目进度”升级到“可证伪研究”）

为避免“跑出来才改口径/挑断点/挑窗口”，本项目新增两份方法学真值源：
- `docs/Protocol.md`：定义 **Confirmatory（确认性）vs Exploratory（探索性）** 分层；只有通过确认性门槛的才写成“正式结论”
- `docs/Preregistration.md`：每次扩展前冻结 **假设/阈值/输出文件清单**；报告必须引用本次 prereg ID

核心原则（必须坚持）：
- **确认性结论**：统一多重比较（BH-FDR）+ 跨指数合并证据（meta/合并 p）+ OOS 稳定性
- **探索性诊断**：允许解释与定位不稳定来源，但不得降低门槛“把候选写成结论”

Phase 2 的关键升级（已在 Protocol 中冻结方向）：
- 采用“立春年”口径的 `year_element`（五行 5 类）作为唯一状态变量（真值源：`docs/li_chun_year_mapping.csv`）
- 对 `stem/branch/ganzhi_day` 分别做 `day_group × year_element` 交互检验，并采用 **层级检验 gate（全局交互通过 → 才看局部 cell）**，避免维度爆炸与事后解释

## 0.5 当前进度与初步结果（截至 2026-02-14 22:51）

**运行状态**
- 已完成并可 Run All：`notebooks/01_fetch_market_data.ipynb`、`notebooks/02_build_ganzhi_calendar.ipynb`、`notebooks/03_ganzhi_effect_analysis.ipynb`
- 已完成并可 Run All（可选增强）：`notebooks/04a_controls_models.ipynb`…`notebooks/04f_resonance_harmonics.ipynb`（索引：`notebooks/04_robustness_and_modeling.ipynb`）、`notebooks/05_report.ipynb`（一键生成 `data/clean/report/report_*.md`）
- 已新增 US 对照实验（负控制，可 Run All）：`notebooks_US/01_fetch_market_data.ipynb`…`notebooks_US/05_report.ipynb`（数据源 Stooq；华盛顿时间口径；输出 `data/clean_us/`）
- 已修复：Notebook 工作目录不一致导致的“输出落到 `notebooks/data/`”问题；现统一自动定位项目根目录并写入 `data/`
- 图表中文字体：已在 `notebooks/03` 增加字体回退逻辑（机器本身安装中文字体后即可正常显示）
- 已补充稳健性与复现：controls 残差置换、block bootstrap、HAC `maxlags` 敏感性、跨指数 Meta + OOS 稳定性筛选（输出见 `data/clean/robustness/`，报告见 `data/clean/report/report_20260214_225132.md`）
- 已新增候选 stem 深挖（诊断）：针对 `癸/丁` 输出 raw OOS 年度效应与“翻符号年份”表、controls-only residual OOS（剥离 `weekday/month`，不含 year）、预设断点（2020 前后）对比与训练窗（3/5/7 年）敏感性（输出位于 `data/clean/robustness/`，并在 `05_report` 报告中新增 04h 小节）
- 已实现 Phase 2（可证伪）：立春年 `year_element` 的 `day_group × year_element` 交互 gate（`notebooks/04e_phase2_interaction_gate.ipynb` 产出 `interaction_gate_summary_ret_1d.csv`；`05` 报告新增 04i 小节；仅 gate 通过才输出局部 heatmap/候选）
- 已新增 Resonance（诊断）：`jiazi_idx` 的 k=5/6 谐波检验（HAC；含 `weekday/month/year` controls），用于把“10×12 热力图条纹”回到 60 周期上做可审计检验（输出见 `data/clean/robustness/resonance_*`；由 `notebooks/04f_resonance_harmonics.ipynb` 产出；报告见 `data/clean/report/report_20260214_225132.md`）

**数据覆盖（本轮输出）**
- 指数：`000300.SH`、`000852.SH`、`000001.SH`、`399001.SZ`
- 区间：`2010-01-04..2026-02-13`（各指数 `n=3916` 个交易日）

**主结论（正式：跨指数复现 + OOS 稳定）**
- `stem=丙`：在“控制变量回归（04a）→ 跨指数 Meta → OOS 稳定性筛选（04d）”的证据链下通过
  - Meta（固定效应）：`effect_meta≈-0.192%/日（≈-19.2bp/日）`，`q_meta≈3e-06`（见 `data/clean/robustness/meta_replication_pass.csv`）
  - OOS（按年 walk-forward）：4 个指数均满足“负向年份占比 ≥ 0.8 且符号检验 `p<=0.1`”（见 `data/clean/robustness/oos_stability_stem_ret_1d.csv`）
- 其他 `stem`：`癸/丁` 在 Meta 的 `q_meta<=0.1` 下出现候选，但 OOS 稳定性不足（`match_ratio≈0.73/0.64`），未进入“正式结论”（见 `data/clean/robustness/meta_controls_stem_ret_1d.csv` + `oos_stability_stem_ret_1d.csv`）
  - 深挖诊断（报告 04h）：两者的 OOS 符号在不同年份存在明显翻转/阶段性，且在剥离 `weekday/month` 后（controls-only residual OOS）`match_ratio` 与 raw 基本一致，提示其不稳并非简单日历代理；预设断点（2020 前后）对比有方向性差异，但年份数少（11 年），置换 p-value 多在 `0.14~0.31`，不足以形成条件性结论。

**Phase 2（确认性 gate：`day_group × year_element`；立春年五行）**
- gate 通过（跨指数 Fisher 合并 + BH-FDR；且至少 `3/4` 指数单指数 joint test `p<=0.1`）：`branch`、`ganzhi_day`（见 `data/clean/robustness/interaction_gate_summary_ret_1d.csv`；报告 04i）
  - `branch`：`q_meta_interaction≈6.85e-05`（`3/4` 通过；`000852.SH` 的 `p_interaction≈0.151` 未过单指数门槛）
  - `ganzhi_day`：`q_meta_interaction≈9.69e-26`（`4/4` 通过；交互证据极强）
- gate 未通过：`stem`（`q_meta_interaction≈0.265`），提示“年五行”不足以解释天干层面的时变不稳定（在当前数据与口径下）。
- 解释边界：gate 仅判定“交互存在性”；局部 cell 的 heatmap/候选属于 Exploratory（尤其 `ganzhi_day×year_element` 单格样本数很小），不得升级为结论。

**解释边界**
- 当前结论是“统计关系 + 稳健性 + 样本外符号稳定”，仍不等价于可交易或因果；尤其要警惕多重比较与数据窥探。
- 收尾决定：在新增 US 对照实验后，Phase 1 的“日柱主效应”在 US 版本基本为 null，因此当前阶段不再推进 Phase 2b 的 cell-level 深挖与“机制解释”路径；如未来重启，需先在 `docs/Preregistration.md` 冻结更严格的 family/阈值，并引入更强负控制与 OOS 约束。

**Resonance（诊断：`jiazi_idx` 谐波；仅 `ret_1d`）补充结论（本轮：2026-02-14 22:51）**
- 主检验（k=5/6 joint Wald；HAC；含 controls）跨指数 Fisher 合并：`p_meta≈0.00274`（见 `data/clean/robustness/resonance_meta_k56_ret_1d.csv`）。
- 单指数：`000001.SH` 通过（`p≈0.0079`），`000300.SH` 边缘（`p≈0.0359`），`000852.SH/399001.SZ` 不显著（见 `resonance_harmonic_k56_ret_1d.csv`）。
- 样本量均衡（`jiazi_idx` 的 `n_min≈61, n_med≈65, n_max≈70`），排除“条纹主要来自某些格子样本稀疏”的担忧。
- 诊断：在回归掉 `C(stem)+C(branch)` 后再检 k=5/6，`p≈1` 为预期现象（k=5/6 属于 stem/branch 的加性主效应空间），提示本轮“条纹感”更像是**投影几何 + 加性主效应**的频域表现，而非必须诉诸“非线性共振机制”（详见 `Model.md` 的 Resonance 小节）。

**下一步（围绕候选癸/丁的可解释性）**
- `癸/丁` 的进一步追踪（若未来重启）：需按 `docs/Preregistration.md` 先冻结“时间结构假设”，避免在断点与窗口长度上做搜索式调参。
- 与“年五行解释”相关：本轮 Phase 2 的 `stem×year_element` gate 未通过；结合 US 对照结果，相关叙事暂不推进。

## 0.6 稳健性检验结果（Notebook 04 系列/05，截至 2026-02-14 22:51）

> `notebooks/04a_controls_models.ipynb`…`notebooks/04f_resonance_harmonics.ipynb`（索引：`notebooks/04_robustness_and_modeling.ipynb`）/ `notebooks/05_report.ipynb` 已 Run All，输出位于 `data/clean/robustness/`，汇总报告见 `data/clean/report/report_20260214_225132.md`。

**控制变量回归（控制 `weekday/month/year`）**
- `ret_1d`（OLS+HAC）下，使用 contrast 直接检验“组 vs 全样本边际均值”（见输出表 `p_value_effect/q_value_effect`）：`stem=丙` 在 `000300.SH`（`q≈0.0474`）与 `399001.SZ`（`q≈0.0372`）仍显著偏低；在 `000001.SH/000852.SH` 未通过 `q<=0.1`。
- `up`（GLM(Binomial)+HAC）下未见跨指数一致通过；单指数：`000001.SH` 出现 `stem=癸` 通过（`q_effect≈0.07`，上涨概率边际效应约 `+6.4pct`）。
- `branch`：在上述 contrast 口径下未见 `q<=0.1`；此前“单点显著”来自相对基准系数口径，需谨慎解读。
- `ganzhi_day`（60 甲子）：`ret_1d` 未见 `q_effect<=0.1`；`up` 仅在 `000852.SH` 出现 `壬寅` 单点通过（`q_effect≈0.07`，上涨概率边际效应约 `+17.5pct`），需进一步做跨指数复现与更严格的多重修正。

**子样本（年份段）**
- `stem=丙` 的 `mean_ret` 差异在所有指数、所有分段里方向一致为负；强度在 `2021-now` 段更大且更容易显著（详见 `data/clean/robustness/subsample_bing.csv`）。

**置换检验（全局）**
- 对 `stem × ret_1d` 的“全局最大偏离”统计量，`000300.SH`（`p≈0.034`）与 `399001.SZ`（`p≈0.025`）在本轮置换次数下通过（详见 `data/clean/robustness/perm_global.csv`）。
- `up`（上涨概率）与 `branch` 的全局检验未见稳定通过。
- 更保守（controls 残差）：对 `stem × ret_resid` 的全局置换检验在 `000300.SH`（`p≈0.024`）与 `399001.SZ`（`p≈0.015`）通过（详见 `data/clean/robustness/perm_global_controls_resid.csv`）。

**序列相关稳健性（新增）**
- HAC `maxlags` 敏感性（`[0,1,3,5,10]`）：`stem=丙` 在 `000300.SH/399001.SZ` 的显著性对 `maxlags` 不敏感（见 `data/clean/robustness/hac_sensitivity_controls_*_stem_ret_1d.csv`）。
- block bootstrap（controls 残差，`block_len=10`）：`stem=丙` 的 95% CI 在 4 个指数上均为负（见 `data/clean/robustness/block_bootstrap_controls_resid_*_stem_ret_1d.csv`）。

**样本外（按年 walk-forward）**
- `stem=丙` 的 OOS（按年）收益差异大多数年份为负（约 `10/11` 或 `11/11` 年），但 10 个天干的整体排序稳定性较弱（Spearman 相关在年与年之间波动较大；详见 `data/clean/robustness/walk_forward_*_stem.csv`）。
- 跨指数复现（Meta + OOS）：仅 `stem=丙` 通过“`q_meta<=0.1` + 方向一致 + OOS 稳定”的正式结论筛选（见 `data/clean/robustness/meta_replication_pass.csv`）。

## 0.7 US 对照实验（负控制：华盛顿时间立春/天干地支 → 美股宽基指数）

> 目的：检验“天干地支/立春年”是否具有跨市场的普遍性。若为“自然时间周期”，应在 US 宽基指数上也能复现同类信号；若主要由本土投资者信念/行为驱动，则 US 应更接近 null。

**US 版本如何构造**
- Notebook：`notebooks_US/01..05` 与 A 股版本一一对应（运行顺序同样是 `01→02→03→04*→05`）。
- 数据源（免 Token）：Stooq 日线；symbol：
  - `spx`=`^spx`（S&P 500）、`ndq`=`^ndq`（Nasdaq Composite）、`ndx`=`^ndx`（Nasdaq 100）、`dji`=`^dji`（Dow Jones Industrial Average）。
- 目录：
  - 缓存：`data/cache_us/`
  - 输出：`data/clean_us/`、`data/clean_us/robustness/`
- 华盛顿时间口径：
  - 日柱：以 `America/New_York`（华盛顿/纽约同一时区）的自然日 `00:00~24:00` 对 `trade_date` 计算。
  - 立春年：真值源 `docs/li_chun_year_mapping_washington.csv`（由 UTC 立春时刻换算为华盛顿本地时间）；在立春当日用美股收盘 `16:00` 判断“立春前/后”。

**US 结果摘要（数据截至 2026-02-13；区间 2010-01-05..2026-02-13；n≈4053）**
- `03`（无控制）与 `04a`（控制 `weekday/month/year`）对 `stem/branch/ganzhi_day × (ret_1d/up)` 均未出现 `q<=0.1` 的稳定信号（单指数 BH-FDR）。
- `04c` 全局置换检验（“是否存在任意组效应”）：最小 `p_empirical≈0.216`（见 `data/clean_us/robustness/perm_global.csv`），未支持存在“全局显著”。
- `04f` Resonance（k=5/6；HAC；含 controls）：跨指数 `p_meta_k56≈0.912`，为 null（见 `data/clean_us/robustness/resonance_meta_k56_ret_1d.csv`）。
- `04e` Phase 2 gate：`ganzhi_day × year_element` 的交互 gate 在 US 也会通过（`q_meta_interaction≈1.91e-12`），但该 family 参数维度很高且 cell 样本数极小（单格 `n_cell` 仅 `6~20`），更像“时间非平稳/模型可塑性”触发的统计显著，不建议作为干支机制证据继续深挖。

**最终结论（收尾）**
- 在 US 宽基指数的负控制下，未观察到可复现的“日柱主效应”（03/04a/04c/04f 全部为 null）。这削弱了“天干地支是一种普遍自然时间因子”的解释。
- 结合 A 股侧的“存在少数可复现信号（如 `丙`）但强依赖本土口径/样本结构”的现象，更合理的解释是：这类效应更接近 **本土投资者信念/行为与市场结构共同作用的‘自我实现’**，而非跨市场通用规律。

## 1. 研究问题（Research Questions）
1. **干支是否区分涨跌？**
   - 不同 `stem`（10）、`branch`（12）、`ganzhi_day`（60）的交易日，上涨概率 `P(up=1)`、平均收益 `E[ret_1d]` 是否有显著差异？
2. **是否跨指数一致？**
   - 在不同宽基指数（沪深300/中证1000/上证综指等）是否方向一致？差异是否来自指数成分结构不同？
3. **是否被日历效应解释？**
   - 控制 `weekday/month/year`（以及可选的趋势/波动状态）后，干支效应是否仍显著？
4. **是否样本外可复现？**
   - 使用 walk-forward（滚动/扩展窗口）重新估计效应，对未来样本仍有信息量吗？

---

## 2. 数据与口径（Data & Conventions）

### 2.1 市场代理（默认：宽基指数）
在“只做日频 + 积分 5000”约束下，默认用宽基指数代表 “A 股涨跌”：
- 沪深300：`000300.SH`
- 中证1000：`000852.SH`
- 可选对照：上证综指 `000001.SH`、深证成指 `399001.SZ`（以接口可用为准）

> 后续扩展：若要研究“全市场涨跌家数/全A截面收益”，需要按股票层面拉取日线，接口频次与积分可能成为瓶颈（不作为本期硬交付）。

### 2.2 交易日集合
以 Tushare `trade_cal` 为准：只研究 **交易日**。

### 2.3 收益与标签（默认口径）
- `ret_1d(t) = close(t)/close(t-1) - 1`
- `up(t) = 1[ret_1d(t) > 0]`（`ret_1d==0` 默认记为 0；也可在分析中剔除做敏感性）

---

## 3. 流日干支（日柱）计算（Ganzhi Day）

### 3.1 输出字段定义
- `ganzhi_day`：六十甲子（如 `甲子`）
- `stem`：天干（10）
- `branch`：地支（12）
- `jiazi_idx`：`0..59`（便于排序/回归；`0` 对应 `甲子`）

### 3.2 计算原则
- 不引入新的农历/历法三方依赖；使用标准库实现。
- 用公历日期计算儒略日数（JDN）后映射到 60 日循环。
- 固定口径：以 **北京时间自然日（00:00~24:00）** 计算日柱（避免 23:00 子时换日争议）。

### 3.3 必做校验（防止“算错了还不知道”）
- 在 `notebooks/02_build_ganzhi_calendar.ipynb` 内置若干“公历日期 ↔ 日柱”的校验向量（来自权威万年历/历书），Run All 时自动 `assert`。
- 校验通过后冻结常数/基准日，保证后续版本一致。

---

## 4. 研究设计（统计检验与模型）

### 4.1 描述性统计（必须）
按 `stem/branch/ganzhi_day` 分组输出：
- 样本数 `n`
- 上涨概率 `p_up`
- 平均/中位数收益、波动（标准差）、左尾分位数（如 `q05`）
- 可视化：条形图（`p_up`）、箱线图（收益分布）、热力图（10×12 组合）

### 4.2 显著性检验（必须）
至少两条线并行，避免“单一检验口径偏误”：
- **方向**（`up`）：卡方检验 / Logistic 回归（分类变量 one-hot）
- **收益**（`ret_1d`）：OLS 回归（分类变量） + 置换检验（推荐，抗分布假设）

多重比较（必须）：
- 对六十甲子/多指数/多子样本的同时检验，输出 `p_value` 与 `q_value(FDR-BH)`；必要时提供 Bonferroni 作为上界。

### 4.3 控制变量与稳健性（必须）
默认控制（基线）：
- `weekday`（周几）
- `month`（月份）

Phase 1（已实现）：
- 额外控制 `year`（或时间趋势项），用于吸收缓慢变化的均值/制度差异（注意：year dummy 不具备 OOS 可泛化性）

Phase 2（协议冻结的第一种“状态变量”扩展）：
- 状态变量：`year_element`（立春年五行，5 类；真值源 `docs/li_chun_year_mapping.csv`）
- 模型将用 `day_group × year_element` 的交互来表达“时变”，因此不再同时加入 `C(year)`（避免同构与不可泛化）

可选扩展（后续阶段，需预注册）：
- 趋势状态：如 `mom_20` 分组
- 波动状态：如 `rv_20` 分位
- 更贴近交易日历的控制：月末、节假日前后、长假复盘等
- 分样本：固定分段（必须预先指定断点）

### 4.4 样本外（建议）
- walk-forward 估计效应（滚动/扩展窗口）→ 用下一段样本检验稳定性
- 评价指标：命中率/校准（Brier/分桶），以及效应符号一致性

---

## 5. Notebook 交付物与运行顺序（VS Code Run All）
1. `notebooks/01_fetch_market_data.ipynb`：拉取并缓存交易日历 + 指数日线 → `data/cache/`
2. `notebooks/02_build_ganzhi_calendar.ipynb`：实现并校验“公历→日柱干支”，生成交易日干支表
3. `notebooks/03_ganzhi_effect_analysis.ipynb`：合并市场数据 + 干支，输出分组统计/检验/图表 → `data/clean/`
4. （可选）Notebook 04 系列：稳健性与建模（索引：`notebooks/04_robustness_and_modeling.ipynb`；模块：`notebooks/04a_controls_models.ipynb`…`notebooks/04f_resonance_harmonics.ipynb`）
5. （可选）`notebooks/05_report.ipynb`：汇总报告（单文件可阅读）

---

## 6. 输出文件（Outputs）
缓存（可删，可重跑）：
- `data/cache/trade_cal/trade_cal.csv.gz`
- `data/cache/index_daily/{ts_code}.csv.gz`

研究输出（可删，可重跑）：
- `data/clean/market_{ts_code}.csv.gz`（含 `ret_1d/up`）
- `data/clean/market_ganzhi_{ts_code}.csv.gz`（市场 + 干支字段）
- `data/clean/ganzhi_stats_{ts_code}.csv`（分组统计表）
- `data/clean/ganzhi_tests_{ts_code}.csv`（检验结果：effect/p/q）

---

## 7. 约束（需持续遵守）
- 不把 Token 写入任何文件或代码；Token 仅从 `TUSHARE_API_KEY` 或 `.tushare_token`/`~/.tushare_token` 读取。
- 默认不引入新三方依赖；缓存格式优先 `csv.gz`。
- 所有缓存与输出写入 `data/`（默认 `data/cache/`、`data/clean/`）。

---

## 8. 风险提示（Research Pitfalls）
- **数据窥探**：多重比较下“必然有显著”，必须修正并报告效应大小与稳定性。
- **非因果**：干支显著并不代表可交易或存在因果；可能是节假日结构/制度/样本划分造成。
- **序列相关**：收益并非独立同分布；建议用置换/区块 bootstrap 作为稳健性手段。
