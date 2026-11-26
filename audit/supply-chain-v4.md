# 供应链补查 v4

审计对象仍是上游 https://github.com/andreashappe/cochise 的 `4db4fded894305083366b83e5cdba5fb68fcebc3`（2026-10-24 23:30 香港时间）。已有结论见 `audit/supply-chain.md`，这里只按八项归类并补缺口。未安装依赖，未拉镜像层，未运行仓库脚本。结论只用「未发现问题／可疑／已确认／未核实」。源码里写明的调用是实证；没有执行导入，所以没有观察到实际网络请求。

## 总表

| 项 | 结论 | 证据 |
|---|---|---|
| 1 清单与 lockfile（含间接依赖） | 可疑 | `pyproject.toml:10-18`；`uv.lock:1-3`、`357-369`；`src/cochise/cli/cochise.py:6` |
| 2 版本钉、校验和、git／URL 直装 | 已确认 | `pyproject.toml:12,18`；`uv.lock` 各包 `hash = "sha256:…"`；`Dockerfile:8` |
| 3 仿冒包名、安装钩子、非官方源 | 未核实 | 锁内源已读；传递包的安装脚本未解包 |
| 4 镜像来源、tag、digest | 已确认 | `Dockerfile:1` |
| 5 子模块、下载脚本、运行时远程拉取 | 已确认 | 无 `.gitmodules`；`src/cochise/common.py:2`；litellm `v1.95.0` 的 `__init__.py:402-518`（见第 5 项） |
| 6 CI 工作流 | 未发现问题 | 本树 `.github/` 只有 `FUNDING.yml` |
| 7 公开漏洞库与恶意包库 | 已确认 | OSV querybatch，2026-11-26 11:34 香港时间 |
| 8 主要依赖维护状态 | 未发现问题 | PyPI 与 GitHub 仓库元数据，2026-11-26 11:32–11:33 香港时间 |

本次新的已确认项：锁定的 `litellm` 1.95.0 在导入时向未钉提交的 GitHub `main` 原始文件发 GET。见第 5 项。其余已确认项在 `audit/supply-chain.md` 已有记录，这里只归类。

## 1. 清单与 lockfile

可疑。`uv.lock` 存在，`version = 1`、`revision = 3`（`uv.lock:1-3`）。第三方包 82 个，其中直接依赖 8 个（`uv.lock:360-369`），间接依赖 74 个，都在同一锁里。没有第二种语言的清单。

缺口：`src/cochise/cli/cochise.py:6` 导入 `dotenv`，`pyproject.toml:10-18` 的直接依赖没有 `python-dotenv`。锁内 1.2.2 只作为 `litellm` 的依赖出现（`uv.lock:1042`）。构建后端 `uv_build>=0.8.16,<0.9.0`（`pyproject.toml:28`）不在 `uv.lock` 的 package 列表中，实际会解析到哪一个 0.8 分发未运行解析，未核实。

## 2. 版本钉、校验和、git／URL

已确认控制不完整。锁内第三方来源都是 `https://pypi.org/simple`，sdist 与 wheel 都有 sha256，没有 git 或 URL 直装。这是锁文件本身的状态。

清单没有钉死 `litellm` 和 `jinja2`（`pyproject.toml:12,18`）。`Dockerfile:8` 与 `docs/running-cochise-vs-goat.md:212` 的 `pip install -e .` 不使用这份锁的哈希。因此除 uv 锁路径以外，版本和校验和约束不成立。

## 3. 仿冒包名、安装钩子、非官方源

未核实。锁内索引只有 PyPI simple，直接依赖名与已知上游一致，见 `audit/supply-chain.md`「依赖是否恶意」一节，本项不重复包名表。本仓没有 `setup.py`，没有 postinstall；`pyproject.toml:21-25` 的 `[project.scripts]` 是入口点，不是安装时执行的钩子。

传递依赖的 sdist 和构建钩子没有解包，所以安装钩子这一半未核实，不能把该项写成没有钩子。

## 4. 容器镜像

已确认 tag 未固定。`Dockerfile:1` 为 `FROM python:3.13`，没有 `@sha256:`，也没有补丁版本。没有 compose。镜像层未拉取，层内系统包未核实，不能写成镜像内容安全。

## 5. 子模块、下载脚本、运行时拉取

已确认有一条导入时的远程拉取。

- 无 `.gitmodules`。仓库脚本和 `src/` 中没有 `curl|bash`。`docs/running-cochise-vs-goat.md:144` 是一条缺少下载命令的 Kali 镜像 URL，不是管道脚本。
- 本仓不下载模型权重。场景模板假定 Kali 上已经有工具，源码没有写死模型或插件 URL。
- 实证：`src/cochise/common.py:2` 与 `src/cochise/cli/cochise.py:2` 导入 `litellm`。全 `src/` 没有设置 `LITELLM_LOCAL_MODEL_COST_MAP`。锁定版本 1.95.0 对应标签 `v1.95.0`、提交 `72a4a55f43ea7266de589f005d0d33624fe5d555`。2026-11-26 11:34 香港时间读取该树：
  - [`litellm/__init__.py:402-404`](https://github.com/BerriAI/litellm/blob/72a4a55f43ea7266de589f005d0d33624fe5d555/litellm/__init__.py#L402) 把缺省 URL 设为 `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json`（`main`，不是钉死的提交）。
  - 同文件 [`516-518`](https://github.com/BerriAI/litellm/blob/72a4a55f43ea7266de589f005d0d33624fe5d555/litellm/__init__.py#L516) 在导入过程中调用 `get_model_cost_map`。
  - [`get_model_cost_map.py:153-162`](https://github.com/BerriAI/litellm/blob/72a4a55f43ea7266de589f005d0d33624fe5d555/litellm/litellm_core_utils/get_model_cost_map.py#L153) 用 `httpx.get`。同文件 `120-136` 的检查是「非空字典」和「条目数不要缩太多」，不是密码学校验和。`277` 行只有环境变量 `LITELLM_LOCAL_MODEL_COST_MAP=true` 才跳过远程拉取。
- 同文件 `406-412` 还给出博客 RSS 和 `anthropic_beta_headers` 的缺省 URL。本次只看到赋值，没有看到导入时调用。
- 推断而未观察：导入 cochise 时会发起上述 GET；失败或条数检查不通过则用包内备份。本单没有执行导入。
- `v1.95.0` 树里没有 `litellm/telemetry/`。当前默认分支上的遥测目录不能算进 1.95.0。该版本是否另有产品遥测上报，除这条 cost map 以外未逐文件读完，其余遥测未核实。

## 6. CI 工作流

未发现问题。本树 `.github/` 只有 `FUNDING.yml`（赞助用户名，无 `uses:`、无密钥、无触发条件）。没有 `workflows/`，因此没有未钉 SHA 的 Action，也没有 `pull_request_target` 或 `workflow_run`。仓库外的 Actions 权限和 Dependabot 应用设置不在本树，未核实。基准提交说明是 Dependabot 合并，但仓库内没有 `dependabot.yml`。

## 7. 公开漏洞库与恶意包库

该项结论为已确认，因为锁定版本命中已知漏洞。恶意包与已知漏洞分开写。

### 依赖是否恶意

未核实。OSV 对 82 个锁定版本的查询（`POST https://api.osv.dev/v1/querybatch`，2026-11-26 11:34 香港时间）返回的 `MAL-*` 为 0。这只说明 OSV 没有把这些版本收成恶意包记录。传递包安装脚本未解包，也没有另查 OSV 以外的恶意包库。

### 已知漏洞

已确认有命中。同一次 OSV 查询，按锁定版本过滤。编号即 OSV 认为该版本受影响的 GHSA。除下列两则外，每条的 introduced／fixed 区间本次没有逐条重读。可达性除注明者外未核实，不能写成可被本程序触发。

- `certifi` 2026.7.22：`GHSA-6q48-6c7w-9gw2`（PYSEC-2026-4320）。区间自 `2019.03.09` 至 `2026.10.18`（不含）。模型 API 的 TLS 会用到该信任库。基准时刻 PyPI 已有 `2026.10.23`（上传时间 2026-10-23 19:36 香港时间，PyPI `upload_time` `2026-10-23T11:36:14`），锁仍停在 2026.7.22。查询日最新为 2026.11.20。
- `litellm` 1.95.0：`GHSA-3cv6-jpf6-8222`（含 `1.95.0` 至 `1.95.1` 不含）、`GHSA-7hp6-4w63-5g45`（`1.91.0` 至 `1.100.4` 不含）。公告对象是 LiteLLM proxy。本仓只调用 `litellm.completion`，没有启动 proxy 的代码，可达性未确认。基准时刻 PyPI 已有 `1.106.3`（2026-10-24 17:14 香港时间上传）。
- `aiohttp` 3.14.3（17）：`GHSA-24c4-3p7v-29cm`、`GHSA-28qf-cpwq-vmc2`、`GHSA-2g87-pp6c-29x5`、`GHSA-3rp8-rxj8-mvjw`、`GHSA-494j-54xm-rwp3`、`GHSA-5pv8-vvhc-h7jv`、`GHSA-5rmc-48h2-hvcm`、`GHSA-5wr8-gcqf-gqf3`、`GHSA-6cjc-53gc-5rqm`、`GHSA-9xw5-w5jj-pvcg`、`GHSA-f9xw-jh8m-3mwq`、`GHSA-gx76-c99x-2c96`、`GHSA-h4cc-hm5h-fmm9`、`GHSA-jm3w-gw2p-xjfj`、`GHSA-pqxq-w44r-73xw`、`GHSA-pw86-h8fm-j323`、`GHSA-xgw8-8ccx-rmqh`。传递依赖。抽看为服务端问题。本仓不起服务。可达性未逐条核实。
- `pillow` 12.3.0（43）：`GHSA-22hg-r758-rrw9`、`GHSA-24rc-8hj2-v98c`、`GHSA-254f-3936-288h`、`GHSA-27h8-gf7j-v5hc`、`GHSA-332p-7hrc-92wf`、`GHSA-3j72-p6qg-6xp7`、`GHSA-4558-3h8r-v8xp`、`GHSA-462x-pgg6-55fp`、`GHSA-4rm8-3hmm-39fg`、`GHSA-4rr5-89j8-mwwm`、`GHSA-58r4-6rv3-gfh8`、`GHSA-5x8w-8jj5-q52h`、`GHSA-5xm2-6676-qcp9`、`GHSA-654x-cwwv-5j9c`、`GHSA-65qm-jvmh-6pqg`、`GHSA-74r2-7ff9-mjj2`、`GHSA-78g4-xqwv-q3g8`、`GHSA-7mwm-ghpf-9vrm`、`GHSA-8g5g-8vmr-9782`、`GHSA-9v9v-9qvf-h2hj`、`GHSA-c6hw-5grf-ff96`、`GHSA-cp9v-662q-vvx9`、`GHSA-f2g6-542v-9j58`、`GHSA-fc9g-5qg7-mf45`、`GHSA-g4qv-2jxx-769h`、`GHSA-g7cr-3p24-7wf7`、`GHSA-g842-9rc4-69pw`、`GHSA-gg2r-wvfm-q9rx`、`GHSA-j4m9-6w36-pw47`、`GHSA-jg86-v3jv-hrpm`、`GHSA-mrfp-p49p-3hf2`、`GHSA-p4wg-p73j-c528`、`GHSA-p8hx-v2p6-rp84`、`GHSA-pchr-8j38-3x4p`、`GHSA-q45j-58j2-45rg`、`GHSA-rfgq-q3w2-pmv2`、`GHSA-v9pv-5gc8-w2m6`、`GHSA-vx6w-vv9f-6gvg`、`GHSA-w889-685w-c8fh`、`GHSA-wqgw-9xfh-qf2g`、`GHSA-xcvg-w56q-r89m`、`GHSA-xgh8-jp6q-p5x8`、`GHSA-xvh5-g6gv-559v`。`matplotlib` 的依赖。分析脚本不打开外部图像。可达性未逐条核实。
- `filelock` 3.32.2（12）：`GHSA-567q-5qvw-pf92`、`GHSA-6627-m3xw-93mp`、`GHSA-69hj-f68j-2879`、`GHSA-8mx5-7m5m-r77g`、`GHSA-9c4x-gm8v-wgc4`、`GHSA-9vhv-jxww-xfm5`、`GHSA-cwqr-hh33-gh44`、`GHSA-fq56-3cx3-fjxp`、`GHSA-j8f7-rjxc-mr56`、`GHSA-m462-7rxw-wvpg`、`GHSA-rfpc-qcw5-3pc4`、`GHSA-v4x8-3wr5-3wq8`。经 `huggingface-hub` 进入。本仓不调用。可达性未核实。
- `multidict` 6.7.1：`GHSA-54p9-h82j-f925`。经 `aiohttp` 进入。可达性未核实。
- 其余锁定版本，包括 `asyncssh` 2.24.0、`jinja2` 3.1.6、`cryptography` 50.0.0、`urllib3` 2.8.0、`requests` 2.34.2，本次查询无 GHSA。无命中不是以后没有公告。

## 8. 主要依赖的维护状态

未发现问题：抽查的上游仓库都没有归档，PyPI 锁定文件没有被 yanked，分类里没有弃用标记。查询：`https://pypi.org/pypi/<name>/json`（2026-11-26 11:32 香港时间）与 `https://api.github.com/repos/<repo>`（2026-11-26 11:33 香港时间）。

| 包 | 锁 | 查询日最新 | 归档 | 说明 |
|---|---|---|---|---|
| asyncssh | 2.24.0 | 2.24.1 | 否 `ronf/asyncssh` | 2.24.1 在基准日前已上传（2026-10-04） |
| litellm | 1.95.0 | 1.111.0 | 否 `BerriAI/litellm` | 基准日已有 1.106.3；项目仍在发版 |
| jinja2 | 3.1.6 | 3.1.6 | 否 `pallets/jinja` | 最近发版 2025-03-06 04:05 香港时间；仓库 `pushed_at` 2025-06-15 04:35 香港时间。未归档，发版间隔长 |
| matplotlib | 3.11.1 | 3.11.2 | 否 | 3.11.2 在基准日前已有 |
| numpydoc | 1.10.0 | 1.11.0 | 否 `numpy/numpydoc` | 1.11.0 在基准日前已有 |
| python-dateutil | 2.9.0.post0 | 4.0.0 | 否 `dateutil/dateutil` | 基准日前已有 3.0.0（2026-10-22 07:41 香港时间上传）。项目未弃用，锁未跟上 |
| rich | 15.0.0 | 15.0.0 | 否 `Textualize/rich` | 最近发版 2026-04-12；仓库 2026-11 仍有 push |
| structlog | 26.1.0 | 26.1.0 | 否 `hynek/structlog` | 锁与查询日最新一致 |
| certifi | 2026.7.22 | 2026.11.20 | 否 | 见第 7 项，修复版在基准日前已发布 |
| urllib3 | 2.8.0 | 2.8.0 | 否 | 基准日已是当时最新 |
| cryptography | 50.0.0 | 50.0.2 | 否 `pyca/cryptography` | 50.0.2 在基准日前已有 |
| uv（构建后端所属项目） | 不在锁内 | uv-build 0.13.10 | 否 `astral-sh/uv` | 清单把 `uv_build` 限制在 `<0.9.0`。0.8 线最后一版是 0.8.24（2025-10-07 11:33 香港时间上传）。未运行解析，实际装到的 0.8 小版未核实 |

锁比当时 PyPI 旧，表示这份锁没有追新，不表示上游已归档。
