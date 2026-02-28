# 开发 TODO：流日干支对 A 股涨跌的影响

> 本 TODO 由 `ProjectPlan.md` 派生，用于把研究做成 **可 Run All 的 notebooks + 可复现的输出文件**。  
> 完成条目后请把对应项改为 `[×]`，并在必要时补充运行说明/配置示例。

研究协议与预注册（方法学真值源）：
- `docs/Protocol.md`：确认性（可写结论）vs 探索性（只能诊断）分层 + Phase 2 交互 gate
- `docs/Preregistration.md`：每次扩展前冻结假设/阈值/输出清单，防止事后挑断点/挑窗口

建议执行顺序（VS Code Run All）：
- A 股版本：`01_fetch_market_data` → `02_build_ganzhi_calendar` → `03_ganzhi_effect_analysis` →（可选）`04a_controls_models`…`04f_resonance_harmonics`（索引：`04_robustness_and_modeling`）→（可选）`05_report`
- US 对照版本：`notebooks_US/01_fetch_market_data` → `notebooks_US/02_build_ganzhi_calendar` → `notebooks_US/03_ganzhi_effect_analysis` →（可选）`notebooks_US/04a`…`notebooks_US/04f`（索引：`notebooks_US/04_robustness_and_modeling`）→（可选）`notebooks_US/05_report`

---

## A. 项目重构与约束确认
- [×] 确认研究对象与口径：默认宽基指数代表 A 股（`000300.SH/000852.SH` + 可选 `000001.SH/399001.SZ`）
- [×] 确认收益口径：`ret_1d = close/prev_close - 1`，标签 `up = ret_1d > 0`
- [×] 固定“日柱干支”的日界：北京时间 00:00~24:00（避免 23:00 子时换日争议）
- [×] 修复 notebook 工作目录不一致导致输出落到 `notebooks/data/`（现统一写入 `data/`）
- [×] 建立最小输出清单并落盘到 `data/clean/`（统计表 + 检验表）

---

## B. Notebook 01：拉取与缓存（`notebooks/01_fetch_market_data.ipynb`）
目标：一次 Run All 后在 `data/cache/` 生成可复用的交易日历与指数日线。

- [×] Token 读取：仅 `TUSHARE_API_KEY` 或 `.tushare_token`/`~/.tushare_token`
- [×] `trade_cal` 缓存：`data/cache/trade_cal/trade_cal.csv.gz`
- [×] `index_daily` 缓存：`data/cache/index_daily/{ts_code}.csv.gz`
- [ ] 增量更新（可选但推荐）：只补齐缺失日期；落盘前按 `ts_code + trade_date` 去重
- [ ] 输出一个“样本覆盖摘要”（起止日期、缺失天数、停牌/空值情况，建议落盘到 `data/clean/`）

---

## C. Notebook 02：流日干支计算与校验（`notebooks/02_build_ganzhi_calendar.ipynb`）
目标：实现“公历日期 → 日柱干支（天干/地支/六十甲子）”，并提供自动校验。

- [×] 实现 `date -> jdn`（标准算法，标准库）
- [×] 实现 `jdn -> jiazi_idx(0..59) -> ganzhi_day/stem/branch`
- [×] 内置最小校验向量（Run All 自动 `assert`）
- [ ] 补充“权威校验向量”并注明来源（万年历/历书），避免口径漂移
- [×] 输出交易日干支表：`data/clean/ganzhi_trade_dates.csv.gz`

---

## D. Notebook 03：主分析（`notebooks/03_ganzhi_effect_analysis.ipynb`）
目标：合并市场数据与干支字段，输出分组统计、显著性检验与图表。

- [×] 构造 `ret_1d/up` 并落盘：`data/clean/market_{ts_code}.csv.gz`
- [×] 合并干支字段并落盘：`data/clean/market_ganzhi_{ts_code}.csv.gz`
- [×] 分组统计表：按 `stem/branch/ganzhi_day` 输出 `n/p_up/mean_ret/std_ret/q05` → `data/clean/ganzhi_stats_{ts_code}.csv`
- [×] 检验（当前已实现的最小口径）：方向=二项检验（vs 全样本上涨率），收益=Welch t-test（组内 vs 其余）
- [ ] 检验（建议升级）：方向=卡方 / Logistic（含控制变量），收益=OLS（含控制变量）/ 置换检验
- [×] 多重比较修正：输出 `p_value` 与 `q_value(FDR-BH)` → `data/clean/ganzhi_tests_{ts_code}.csv`
- [×] 图表最小集：`p_up` 柱状图 + 10×12 热力图 + 六十甲子柱状图（按样本量过滤）

---

## E. Notebook 04（系列）：稳健性与建模（可选；索引：`notebooks/04_robustness_and_modeling.ipynb`）

### E1. `notebooks/04a_controls_models.ipynb`（控制变量回归）
- [×] 加入控制变量：`weekday/month/year`
- [×] 控制回归对比检验：输出 `p_value_effect/q_value_effect`（“组 vs 全样本边际均值”）
- [×] 启用 `ganzhi_day`（60 甲子）控制变量模型：`RUN_GANZHI_DAY_MODELS=True` 并落盘到 `data/clean/robustness/`

### E2. `notebooks/04b_subsample_stability_bing.ipynb`（子样本）
- [×] 子样本分析：按年份段检查 `stem=丙`（并输出落盘）

### E3. `notebooks/04c_permutation_and_dependence.ipynb`（置换 + 序列相关稳健性）
- [×] 置换检验（permutation，全局）：打乱干支标签，估计“最大显著性”的零分布（防 data snooping）
- [×] 升级置换检验：在“先回归掉控制变量”的残差上做 permutation（更贴近因果/更保守）
- [×] 升级稳健标准误：对 controls 残差做 block bootstrap + 对 HAC `maxlags` 做敏感性（maxlags sweep）

### E4. `notebooks/04d_oos_walk_forward.ipynb`（样本外）
- [×] walk-forward：按年样本外检验（输出 Spearman 稳定性与 `丙` 的 OOS 差异）
- [×] 候选 stem 诊断底表：walk-forward 的 controls-only residual OOS（剥离 `weekday/month`；不含 year）+ `train_years=[3,5,7]` sweep 落盘到 `data/clean/robustness/`

### E5. `notebooks/04f_resonance_harmonics.ipynb`（Resonance；诊断）
- [×] Resonance（诊断）：对 `jiazi_idx(0..59)` 构造 k=5/6 谐波项（12 周期/10 周期），做 joint Wald test（HAC；含 `weekday/month/year` controls）+ 跨指数 Fisher 合并；并补充“回归掉 `stem+branch` 后再检 k=5/6”的残差诊断（输出 `data/clean/robustness/resonance_*`）

---

## F. Notebook 05：报告汇总（可选，`notebooks/05_report.ipynb`）
- [×] 报告结构：数据覆盖 → 干支日历 sanity check → 主结论（03）→ 稳健性（04 系列）→ 风险提示
- [×] 一页结论表：汇总 `stem=丙` 的证据链 + “全量扫描最小 q”表
- [×] 报告导出：生成 `data/clean/report/report_*.md` + `data/clean/report/figures/*.png`
- [×] 跨指数复现（正式结论）：Meta 合并 + 跨 stem BH-FDR + “方向一致 + OOS 稳定”筛选（并输出到 `data/clean/robustness/`）
- [×] 候选 stem 深挖（诊断）：对 `癸/丁` 输出 raw vs controls OOS 年度效应、翻符号年份表、断点（2020 前后）对比与训练窗敏感性，并写入报告 04h 小节
- [×] Resonance（诊断）：报告新增 `jiazi_idx` 谐波检验汇总表 + 诊断图（fit curve / FFT spectrum / 10×12 embedding），并写入 `data/clean/report/figures/resonance_*.png`

---

## G. 文档与可复现
- [×] 更新/补充 `Model.md`：字段字典 + 检验与模型口径 + 多重比较处理
- [×] 新增研究协议：`docs/Protocol.md`（确认性 vs 探索性分层；Phase 2 的可证伪 gate）
- [×] 新增预注册模板：`docs/Preregistration.md`（扩展前先写）
- [×] 新增立春年映射表：`docs/li_chun_year_mapping.csv`（`year_element` 真值源）
- [ ] 输出字段字典：`data/clean/feature_dictionary_ganzhi.csv`（可选）

---

## H. 候选 stem（癸/丁）进一步解释（可选，避免数据挖掘）
- [ ] 先写 `docs/Preregistration.md` 再跑任何“条件性解释”：固定分段/固定状态变量；避免搜索式选断点/窗口
- [ ] 若 `year_element` 交互 gate 不通过：暂停“年运解释”路径，把癸/丁维持为诊断对象（不升级为结论）
- [ ] 备选解释（需预注册）：加入更贴近交易日历的控制（如月末/节假日前后/长假复盘），观察癸/丁 是否被吸收

---

## I. Phase 2：立春年 / 年五行（`year_element`）交互（确认性 gate → 才看局部）

> 本阶段目标：把“时运/大环境”叙事翻译成可证伪的 `day_group × year_element` 交互检验。  
> 默认只做 `ret_1d`；`year_element` 为唯一状态变量；采用层级检验（全局 gate → 局部解释）。

### I1. 数据字段（立春年口径）
- [×] 在合并表中加入：`li_chun_year/year_stem/year_element`（真值源：`docs/li_chun_year_mapping.csv`）
- [×] 输出映射 sanity check 到 `data/clean/quality/`（覆盖率=100%；抽样检查立春前后几天的标签切换）

### I2. 交互模型（仅 `ret_1d`）
- [×] 单指数：对 `stem/branch/ganzhi_day` 分别拟合  
  `ret_1d ~ C(day_group) * C(year_element) + C(weekday) + C(month)`（HAC）
- [×] 单指数输出：`data/clean/robustness/interaction_joint_test_{ts_code}_{day_group}_ret_1d.csv`（含交互 joint test p）

### I3. 跨指数合并 + 多重比较（Phase 2 的“正式结论”只看 gate）
- [×] 跨指数合并：对每个 `day_group` 的 `p_interaction` 用 Fisher 合并得到 `p_meta_interaction`
- [×] 多重比较：对 3 个 `day_group` 的 `p_meta_interaction` 做 BH-FDR → `q_meta_interaction`
- [×] gate 汇总表：`data/clean/robustness/interaction_gate_summary_ret_1d.csv`

### I4. gate 通过后的局部解释（探索性，仍需 OOS 与 FDR）
- [×] 输出 cell-level 边际效应（热力图 + CSV），并标注“Exploratory（不得替代确认性结论）”
- [×] 在报告中强制写明：若 gate 未通过，则不展示/不解读局部显著

> 最新运行记录（2026-02-14 16:01；见 `data/clean/report/report_20260214_160139.md` 的 04i）：  
> gate 通过：`branch`、`ganzhi_day`；gate 未通过：`stem`。  
> 因此仅 `branch/ganzhi_day` 生成局部 heatmap 与 `interaction_candidates_*`（均为 Exploratory）。

---

## J. Phase 2b：把局部 cell 变成“可证伪结论”（需先预注册）

> 背景：Phase 2 的 gate 只回答“交互是否存在”。若要把某些 `branch×year_element`（或 `ganzhi_day×year_element`）写成更具体的结论，必须升级为确认性检验：统一 family + 多重比较 + OOS 稳定性。

- [ ] 在 `docs/Preregistration.md` 中新增 Phase 2b 的 prereg 条目：冻结 family（优先 `branch×year_element` 的 60 个 cell）、阈值、最小样本数规则（如 `n_cell>=__`）、OOS 评价指标
- [ ] 实现 cell-level 的 OOS 稳定性检验（walk-forward；输出 `cell×year_element×oos_year` 的长表 + 稳定性汇总）
- [ ] 实现 cell-level 的跨指数合并证据（meta / 合并 p），并在 cell family 内做 BH-FDR
- [ ] 仅对 Phase 2b 通过项在报告中展示局部结论；其余保持 Exploratory

---

## K. US 对照实验（`notebooks_US/`；Stooq；华盛顿时间口径）

> 目的：作为负控制（falsification）检验“天干地支是否为跨市场通用的自然时间因子”。US 版本复用同一套统计流程，但用美股宽基指数替代 A 股指数。

- [×] 新建 `notebooks_US/` 并镜像 `01..05` 流程（与 A 股版本一一对应）
- [×] 数据源切换为 Stooq（免 Token）：缓存到 `data/cache_us/index_daily/{symbol_id}.csv.gz`（`symbol_id ∈ {spx,ndq,ndx,dji}`）
- [×] 华盛顿时间立春年：新增真值源 `docs/li_chun_year_mapping_washington.csv`；在 `02` 中按美股收盘 `16:00` 决定立春当日归属
- [×] 输出目录切换：`data/clean_us/`、`data/clean_us/robustness/`（与 A 股版本结构一致）
- [×] 已产出并可审计的关键输出（US）：
  - `data/clean_us/ganzhi_tests_*.csv`（03；无控制分组检验）
  - `data/clean_us/robustness/controls_*_ret_1d.csv` / `controls_*_up.csv`（04a；HAC；含 controls）
  - `data/clean_us/robustness/perm_global.csv`（04c；全局置换）
  - `data/clean_us/robustness/interaction_gate_summary_ret_1d.csv`（04e；Phase 2 gate）
  - `data/clean_us/robustness/resonance_meta_k56_ret_1d.csv`（04f；Resonance meta）
- [ ] （可选）运行 `notebooks_US/05_report.ipynb`：生成 `data/clean_us/report/report_*.md`（US 版报告）
