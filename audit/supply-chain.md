# 供应链

基准：https://github.com/andreashappe/cochise `4db4fded894305083366b83e5cdba5fb68fcebc3`（2026-10-24 23:30 香港时间）。未安装依赖，未拉取镜像。

## 结论

锁文件把 82 个第三方包固定在 PyPI，并带 sha256。没有 git 依赖、没有非官方索引、没有本仓安装脚本或管道下载。OSV 对这 82 个版本没有返回 `MAL-*`。抽查的包名指向已知上游，锁定文件没有被 yanked。不能据此写成「依赖无恶意」：传递包的 sdist 和安装脚本没有解包，镜像层没有拉取。

与本仓数据流直接相关的公告有两条。`certifi` 2026.7.22 落在 [GHSA-6q48-6c7w-9gw2](https://osv.dev/vulnerability/GHSA-6q48-6c7w-9gw2) 范围内，模型 API 的 TLS 会用到它（F-S1，中）。`litellm` 1.95.0 落在两则 **proxy** 公告范围内，本仓只调用 `litellm.completion`，可达性未确认（F-S2，低）。

## 锁定结果

- 清单：`pyproject.toml`。锁：`uv.lock`（`version = 1`，`revision = 3`，`requires-python >=3.12`）。包计数 83，其中本包 `cochise` 0.3.0 为 `editable = "."`。
- 全部第三方 `source.registry` 都是 `https://pypi.org/simple`。每个 sdist 和 wheel 都有 `sha256`。没有缺哈希，没有 git／http 源。
- 直接依赖与锁内版本：

| 包 | pyproject | uv.lock |
|---|---|---|
| asyncssh | `>=2.22.0` | 2.24.0 |
| litellm | 无上界 | 1.95.0 |
| jinja2 | 无上界 | 3.1.6 |
| matplotlib | `>=3.10.8` | 3.11.1 |
| numpydoc | `>=1.10.0` | 1.10.0 |
| python-dateutil | `>=2.9.0.post0` | 2.9.0.post0 |
| rich | `>=14.3.3` | 15.0.0 |
| structlog | `>=25.5.0` | 26.1.0 |

- `litellm` 与 `jinja2` 在清单里未钉死。`uv sync` 会跟锁文件；`docs/running-cochise-vs-goat.md` 和 `Dockerfile` 的 `pip install -e .` 不使用这份锁，解析结果可以漂。未实测。
- 本仓 `cli/cochise.py` 第 6 行 `import dotenv`，但 `python-dotenv` 不在直接依赖里。锁内 1.2.2 是 `litellm` 的依赖。PyPI 上该版本未被 yanked，主页指向 `theskumar/python-dotenv`。
- `numpydoc` 1.10.0 依赖 `sphinx`，因此锁进一整组 sphinxcontrib。本仓源码没有 `import numpydoc`。
- 传递链：`litellm` → `aiohttp`、`httpx`、`openai`、`tiktoken`、`tokenizers` → `huggingface-hub` → `hf-xet`（仅部分 CPU 架构）和 `filelock`。`matplotlib` → `pillow`。`httpx` 与 `requests` → `certifi`。本仓源码不导入 huggingface、aiohttp 或 pillow。
- HEAD 提交说明是 Dependabot 将 `urllib3` 从 2.7.0 升到 2.8.0。锁内 `urllib3` 2.8.0 在本次 OSV 查询中无命中。
- 构建后端 `uv_build>=0.8.16,<0.9.0` 写在 `pyproject.toml`，不在 `uv.lock` 的 `package` 列表里。它的分发哈希未由这份锁覆盖。未核实。

## 依赖是否恶意

结论：这次只读检查没有看到恶意包的正面证据，也没有做完能排除恶意包的检查。

- 包名：直接依赖都是长期公开项目的名字（asyncssh、litellm、jinja2、matplotlib、numpydoc、python-dateutil、rich、structlog）。锁内没有与这些名字差一个字符的条目。
- 来源：只有 PyPI simple。没有额外 index、没有 VCS、没有本仓里的 `curl`／`wget` 或管道执行脚本。`docs/running-cochise-vs-goat.md` 第 144 行是一条缺少下载命令的 Kali 镜像 URL，不是管道脚本。
- 校验和：锁内分发包都有 sha256。清单本身不强制这些哈希；只有走 uv 锁的安装才会用到。
- 安装钩子：本仓没有 `setup.py`，没有自定义 `cmdclass`。`[project.scripts]` 只是入口点。传递依赖的 `setup.py` 和构建钩子没有解包，未核实。
- 抽查 PyPI JSON（与 OSV 同一次，2026-11-26 10:27 香港时间起）：`litellm` 1.95.0 指向 `BerriAI/litellm`（当时最新 1.111.0）；`asyncssh` 2.24.0 指向 `ronf/asyncssh`（最新 2.24.1）；`fastuuid` 0.14.0 指向 `thedrow/fastuuid`；`hf-xet` 1.5.2 指向 `huggingface/xet-core`；`jinja2` 3.1.6 指向 `pallets/jinja`；`urllib3` 2.8.0 指向 `urllib3/urllib3`。上述锁定版本的文件均未被 yanked。
- OSV 恶意包编号：82 个版本的 `querybatch` 结果里 `MAL-*` 数量为 0。没有 `MAL-*` 只说明这批版本没有被 OSV 收成恶意包记录。

## OSV／GHSA

查询：`POST https://api.osv.dev/v1/querybatch`，生态 `PyPI`，版本为锁内版本。时间 2026-11-26 10:27 香港时间，同日复点计数一致。82 个版本里 6 个有命中：

| 锁定版本 | GHSA 条数 | 与本仓路径的关系 |
|---|---|---|
| certifi 2026.7.22 | 1 | 模型 API 的 TLS 信任库会用到。见 F-S1 |
| litellm 1.95.0 | 2 | 公告对象是 proxy。本仓无启动 proxy 的代码。见 F-S2 |
| aiohttp 3.14.3 | 17 | `litellm` 的传递依赖。抽看的条目是服务端解析与资源限制。本仓不起 aiohttp 服务。可达性未逐条核实 |
| pillow 12.3.0 | 43 | `matplotlib` 的依赖。分析脚本只 `savefig` PDF，不打开外部图像。可达性未逐条核实 |
| filelock 3.32.2 | 12 | `huggingface-hub` 的依赖。抽看为本地锁文件问题。本仓不调用该库。可达性未逐条核实 |
| multidict 6.7.1 | 1 | `aiohttp` 的依赖。公告描述服务端头解析上的引用泄漏。可达性未核实 |

其余锁定版本，包括 `asyncssh` 2.24.0、`jinja2` 3.1.6、`cryptography` 50.0.0、`requests` 2.34.2、`urllib3` 2.8.0，本次查询无命中。无命中不等于以后无公告。

`litellm` 两则公告的版本范围已核对：

- [GHSA-3cv6-jpf6-8222](https://osv.dev/vulnerability/GHSA-3cv6-jpf6-8222) / CVE-2026-84377：含区间 `1.95.0`–`1.95.1`（不含）。已认证 proxy 用户可改路由并带出上游凭据。
- [GHSA-7hp6-4w63-5g45](https://osv.dev/vulnerability/GHSA-7hp6-4w63-5g45)：`1.91.0` 起，修复于 `1.100.4`。proxy 内部用户可提升为 proxy 管理员。

## Docker、子模块、脚本

- `Dockerfile` 第一行 `FROM python:3.13`。没有 digest，标签也不是补丁级。未拉取，层内容未确认，不能写成镜像安全。
- 没有 `docker-compose` 文件，没有 `.gitmodules`，没有 GitHub Actions 工作流（`.github/` 只有 `FUNDING.yml`）。
- 镜像构建还 `sed` 一个不存在的 `src/kalissh.py`，并 `pip3 install -e .`。该路径不消费 `uv.lock` 的哈希。详见主报告「编译与部署要求」。

## 未覆盖

- 未解包任何 sdist，未读传递依赖的安装脚本和入口代码。
- 未拉 `python:3.13` 的层，未核对其系统包。
- pillow、aiohttp、filelock、multidict 的公告没有逐条做可达性。
- 未查询 OSV 以外的另一套恶意包库；结论只覆盖本次 OSV `MAL-*` 与 PyPI 抽查。
- `uv_build` 的实际分发未在锁内核对。
- `litellm` 1.95.0 是否在 import 时访问遥测域名，未读该包源码。
