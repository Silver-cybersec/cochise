# 验收矩阵

阅读对象：上游 https://github.com/andreashappe/cochise ，`main` @ `4db4fded894305083366b83e5cdba5fb68fcebc3`（2026-10-24 23:30 香港时间）。工作区检出与该 SHA 一致。查询与阅读发生在 2026-11-26。未运行本仓代码。

| ID | 需求 | 检查项 | 证据 | 结论 | 未覆盖 |
|---|---|---|---|---|---|
| R1 | 对用户指定的本仓、本单固定基准 SHA 做只读静态重审（09:07 原话） | 记录上游 URL、默认分支、基准 SHA、提交时间（香港时间）；只读源码、清单与锁文件 | 工作区 `HEAD` 为 `4db4fded894305083366b83e5cdba5fb68fcebc3`；作者 Andreas Happe，AuthorDate `2026-10-24T17:30:18+02:00`（即 2026-10-24 23:30 香港时间，与 09:29 盘点的 API 时间 `2026-10-24T15:30:18Z` 相同） | 已按该树阅读。PR 开在 fork `Silver-cybersec/cochise`，不合并 | 无。SHA 已检出 |
| R2 | 代码安全 | 故意漏洞与部署者／宿主机风险分开；危险执行、默认凭据、硬编码密钥、数据外传、隔离、暴露面、容器挂载 | `audit/findings.json` 的 F-C1 至 F-C5；Issue #13 仍 open | 高 2、中 2（另有供应链中 1、低 1）。本仓不是靶场关卡。未动态复现 Issue #13 | 未动态复现；未对任何目标测试；未判断示例材料是否仍有效 |
| R3 | 架构（含与学习目标相称的功能阅读） | 组件、信任边界、数据与权限流；不做完整宣称对照（对照表见 R8） | `src/cochise/cli/cochise.py`、`planner.py`、`executor.py`、`knowledge.py`、`ssh_connection.py`、`common.py`、`logger.py` 与 `templates/` | 单进程：规划器持久循环，执行器按任务新建，命令只经 SSH 出去，提示经 LiteLLM 出去 | 未读 GOAD／Kali 运行环境；模型供应商内部策略未覆盖 |
| R4 | 供应链，且必须单列「依赖是否恶意」 | 锁文件、镜像、子模块、管道脚本、版本与校验和、仿冒名、安装钩子、非官方源、OSV／GHSA | `uv.lock`、`pyproject.toml`、`Dockerfile`；OSV querybatch，2026-11-26 10:27 香港时间 | 82 个第三方锁定版本均来自 `https://pypi.org/simple` 且带 sha256。OSV 无 `MAL-*`。certifi 与 litellm 版本落在公告范围内。详见 `audit/supply-chain.md` | 未解包 sdist／安装脚本；未拉镜像层；pillow／aiohttp／filelock／multidict 的公告未逐条做可达性；`uv_build` 构建后端不在锁的 package 列表中 |
| R5 | 产出精练中文报告 | 四个交付文件；每节先结论 | `audit/RERUN-20261126.md`、`audit/matrix.md`、`audit/findings.json`、`audit/supply-chain.md` | 已写。未改产品代码，未改总报告 | 无 |
| R6 | 查是否存在可对照的旧仓库安全审计基准 | 本单说明：总报告当时将本仓列为未审计 | 本单任务说明，无仓内旧审计文件 | 无旧仓库安全审计基准。不单列差异章 | 无旧基准，不构成缺口 |
| R7 | 不新建私库、不改产品代码、不把静态阅读写成已运行验证 | 交付物范围与用词 | 本分支只新增 `audit/` | 符合。全文未写已验证可部署或已验证可用 | 无 |
| R8 | 功能与使用（2026-11-26 10:14 香港时间启用） | README／文档宣称对照实现；命令、配置、一条工作流。每条标未实测 | `README.md`、`docs/walkthrough.md`、`docs/running-cochise-vs-goat.md`、`pyproject.toml`、`src/cochise/` | 主循环与四个入口在代码中存在。walkthrough、RCR 文档和作图命令与本树不一致。详见主报告该章 | 未运行任何命令；论文中的成功率、时长和费用未复核；未用 tokei 复算行数 |
| R9 | 编译与部署要求（同次启用） | 构建方式、依赖与运行时、硬件、外网、文档与代码不符。每条标未实测 | `Dockerfile`、`pyproject.toml`、`uv.lock`、`.python-version`、`docs/running-cochise-vs-goat.md` | 当前文档主路径是 uv；Dockerfile 与 RCR 文档对不上本树。无 compose、无 Makefile。详见主报告该章 | 未执行 uv、pip 或 docker；未检出 RCR 所写旧提交 `3084bcdd`；未拉 Kali／GOAD |
