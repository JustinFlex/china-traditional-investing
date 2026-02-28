# Agent Instructions (本目录生效)

## 项目目标
- 把中国风水因子投资做成一个可研究、可迭代、可复现的工程。

## 文件夹背景
- 该文件夹脱胎于一个指数风险管理模型，现在想要去研究中国风水因子。

## 代码与结构约定
- 语言：文档优先中文；代码标识符/模块名使用英文、`snake_case`。
- Python：基于 Python 3.12；优先使用标准库（配置建议用 `tomllib` 读取 TOML）。
- 依赖：默认不要引入新三方依赖（当前环境无 `pyarrow/fastparquet`），数据缓存优先用 `csv.gz`。
- 数据目录：所有本地缓存与实验输出写入 `data/`（默认 `data/cache/`），不要把 Token 写入任何文件或代码。
- 可复现：脚本应尽量幂等/可断点续跑；落盘前对关键唯一键（通常 `ts_code + trade_date`）去重。
- AGENTS运行方式：优先写成 `python3 -m ...` 可执行模块（从仓库根目录运行）。
- Notebook：按“阶段/模块”拆分为独立的 `.ipynb` 文件存储（例如：数据拉取/缓存一个 notebook、回测引擎一个 notebook），便于用户逐段运行。
- 测试：只需生成/修改代码与文档即可，不需要运行测试/回测（由用户执行）。
- 用户运行：所有的代码均以notebook的格式呈现，用户不会Terminal调用，只会在VS code环境中对 notebook进行run all

## Tushare 安全与网络
- Token：只从环境变量 `TUSHARE_API_KEY` 或 `.tushare_token`/`~/.tushare_token` 读取；不得提交/打印完整 Token。
- 代理/端点：遵循 `tushare_api_docs/TUSHARE_CONFIG.md` 的约定（如 `TUSHARE_PROXY` / `TUSHARE_BASE_URL`）。
- 网络/Whitelist：仅作为“可选的网络排查/配置手册”，不作为本项目运行前置条件；用户本地运行 notebooks 时可不配置 Whitelist/代理。
- 积分：用户现在的tushare积分为5000积分。高频数据，分钟级数据往往无法调用。只能用日频。

## TODO 维护
- 完成 `TODO.md` 中的条目后，同步把对应项标记为 `[×]`，并在必要时补充运行说明或配置示例。

## Git/GitHub（重要）
- 除非用户明确要求（例如“请上传/推送到 GitHub”），否则不要执行任何会修改远程仓库状态的操作：包括但不限于 `git push`、`gh repo create`、创建/修改 remote、开 PR、发布 release 等。

## Notebook 报错处理约定
- 用户每次运行 notebook 后都会保存文件，报错 traceback/outputs 也会一并写入对应的 `.ipynb`。
- 当用户反馈“某个 notebook 报错”时，优先直接重新读取该 `.ipynb` 文件定位报错 cell 与 traceback，并据此修复；一般不需要再让用户额外粘贴“哪里错了/报错信息”。除非 notebook 未保存输出或错误信息缺失，才向用户补充询问。
