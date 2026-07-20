# multialphaV 多人协作开发规范

> 适用：**3-5 人中型团队**，每人独立开发机，每人独立 LLM/Embedding API Key，trace/实验产物走共享对象存储对照。
> 与 `CLAUDE.md`（开发行为约束，最高优先级）配合使用：本文件管"**人与人/人与资源**"的协作，CLAUDE.md 管"**Agent 与代码**"的开发约束。
> 冲突时：**CLAUDE.md > 本文件 > 通用 git 流程**。
>
> 维护约定：分支策略、环境约定、Key 流程、trace schema、review 流程任一变更，必须同步本文档。

---

## 目录

- [0. 协作全景图](#0-协作全景图)
- [1. 仓库与分支策略](#1-仓库与分支策略)
  - [1.1 双仓库分工](#11-双仓库分工)
  - [1.2 分支模型](#12-分支模型)
  - [1.3 上游同步与裁剪重放](#13-上游同步与裁剪重放)
  - [1.4 PR 与合并门槛](#14-pr-与合并门槛)
- [2. 环境一致性（每人独立机器）](#2-环境一致性每人独立机器)
  - [2.1 新成员 onboarding 清单](#21-新成员-onboarding-清单)
  - [2.2 conda env 命名约定](#22-conda-env-命名约定)
  - [2.3 Qlib 数据一致性](#23-qlib-数据一致性)
  - [2.4 Docker 镜像一致性](#24-docker-镜像一致性)
  - [2.5 共享 0.8 env 的禁忌](#25-共享-08-env-的禁忌)
- [3. LLM / Embedding Key 管理](#3-llm--embedding-key-管理)
- [4. trace 与实验产物共享](#4-trace-与实验产物共享)
  - [4.1 产物层级与命名](#41-产物层级与命名)
  - [4.2 上传约定](#42-上传约定)
  - [4.3 对照与复现流程](#43-对照与复现流程)
- [5. Code Review 与多 Agent 审查](#5-code-review-与多-agent-审查)
- [6. 共享资源变更协议](#6-共享资源变更协议)
- [7. 角色与责任分工](#7-角色与责任分工)
- [8. 常见冲突与处置](#8-常见冲突与处置)
- [附录 A：新人第一周 checklist](#附录-a新人第一周-checklist)
- [附录 B：PR 模板](#附录-bpr-模板)
- [附录 C：trace 元数据 schema](#附录-ctrace-元数据-schema)

---

## 0. 协作全景图

```
┌─────────────────────────────────────────────────────────────────┐
│  multialphaV 根仓库 (github.com:seuzxh/multialphaV, 私有)         │
│  跟踪：CLAUDE.md / docs/* / PLAN-*.md / REPOS.md / API.md / ENV.md │
│  职责：规范、文档、协作流程                                       │
└────────────────────────────┬────────────────────────────────────┘
                             │ git submodule-like 引用（实际是独立 git）
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  RD-Agent fork (github.com:seuzxh/RD-Agent, 私有 fork)            │
│  上游：github.com/microsoft/RD-Agent                              │
│  跟踪：rdagent/ 源码（裁剪后仅 Qlib 场景）                         │
│  工作分支：cut-non-qlib-scenarios                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │ 3-5 人各自 clone 到独立机器
                             ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ 成员 A 机器   │  │ 成员 B 机器   │  │ 成员 C 机器   │   ...
│ conda env    │  │ conda env    │  │ conda env    │
│ multialphav  │  │ multialphav  │  │ multialphav  │   (各自独立)
│ ~/.qlib/软链 │  │ ~/.qlib/软链 │  │ ~/.qlib/软链 │
│ local_qlib   │  │ local_qlib   │  │ local_qlib   │
│ .env (独立Key)│  │ .env (独立Key)│  │ .env (独立Key)│
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                  │                  │
       └──────────┬───────┴──────────────────┘
                  ▼
       ┌──────────────────────────┐
       │  共享对象存储（OSS/S3/内网）│  ← trace/指标/因子产物只读对照
       │  multialphav-traces/      │
       │  multialphav-results/     │
       └──────────────────────────┘
```

**三层资产归属**（CLAUDE.md §7.1）：

| 资产 | 归属 | 协作规则 |
|---|---|---|
| Qlib 底层数据（features/calendars/instruments） | **完全共享只读** | 任一方修改需全员验证 |
| 因子层（Alpha 表达式 / .bin / 因子库） | **完全独立** | 跨人复用走 §4.3 转换规则 |
| trace / 实验 / 知识库 | **各自独立产出**，元数据/指标上传共享存储 | 全量产物不互传，仅元数据对照 |

---

## 1. 仓库与分支策略

### 1.1 双仓库分工

**任何改动先分清落在哪个仓库**：

| 改动类型 | 落入仓库 | 例子 |
|---|---|---|
| rdagent 源码、场景扩展、CoSTEER、prompt 模板 | **RD-Agent fork** | 新增因子 Scenario、改 `rd_loop.py` |
| 项目规范、协作流程、文档 | **multialphaV 根** | 改 CLAUDE.md、写新 docs、更新 API.md |
| 配置模板（`.env.example`） | **RD-Agent fork** | 新增配置项模板 |
| 真实 `.env`、trace、本地产物 | **不入库** | `.env`、`git_ignore_folder/traces/`、`pickle_cache/` |

> 双仓库是平级独立 git，**不是 submodule**。multialphaV 根仓库的 `.gitignore` 忽略整个 `RD-Agent/` 子目录（见根仓库 `.gitignore`）。在两仓库间切换时 `cd` 即可，git 身份按各自仓库配置。

### 1.2 分支模型

**RD-Agent fork 采用 feature-branch + rebase 模型**：

```
main                          # fork 自上游 microsoft/RD-Agent，保持只读同步能力
 │
 └─ cut-non-qlib-scenarios    # 主工作分支（裁剪 commit + 团队成果集成）
     │
     ├─ feat/<issue>-<slug>   # 成员 A 的 feature 分支
     ├─ feat/<issue>-<slug>   # 成员 B 的 feature 分支
     ├─ fix/<issue>-<slug>    # bug 修复分支
     └─ exp/<user>-<topic>    # 实验分支（因子探索、调参，可丢弃）
```

**分支命名约定**：
- `feat/<issue号>-<简短描述>`：新功能，如 `feat/42-multi-factor-selector`
- `fix/<issue号>-<简短描述>`：bug 修复
- `exp/<用户名>-<主题>`：实验分支，如 `exp/alice-volatility-factors`（实验分支不强求合并，可长期保留或定期清理）
- `docs/<主题>`：纯文档改动

**操作纪律**：
- 任何改动**禁止直接 push 到 `cut-non-qlib-scenarios`**，必须走 feature 分支 + PR
- feature 分支从 `cut-non-qlib-scenarios` 切出，开发期间**定期 `git rebase cut-non-qlib-scenarios`** 保持最新
- 实验/调参产出**不要进 feature 分支**，留在本地或上传共享存储（§4）
- `main` 分支**永远不直接改**，仅用于同步上游

**multialphaV 根仓库**较轻：直接在 `main` 上提 PR 即可（文档变更相对低风险）。

### 1.3 上游同步与裁剪重放

multialphaV 的裁剪 commit（`8f60d6ea`，删除非 Qlib 场景）让上游同步变成**特殊操作**：

**同步流程（每月一次，由 §7 的"上游同步负责人"执行）**：

```bash
cd /home/zxh/projects/1.multialphaV/RD-Agent
git fetch upstream

# 1. 查看上游新增
git log --oneline $(git describe --tags --always)..upstream/main

# 2. 在专用同步分支上 rebase（不要直接在 cut-non-qlib-scenarios 上做）
git checkout -b sync/upstream-<YYYYMMDD> cut-non-qlib-scenarios
git rebase upstream/main

# 3. 处理裁剪 commit 的冲突（必然出现，因为上游改了被裁剪的场景）
#    冲突原则：上游对 data_science/kaggle/finetune/general_model/rl 的改动一律 drop
#              上游对 core/components/scenarios/qlib 的改动保留
#              上游对 .gitignore/docs/Makefile 的改动人工判断

# 4. 跑完整冒烟（见 §1.4 门槛）

# 5. 提 PR 到 cut-non-qlib-scenarios，至少 2 人 review（含上游同步负责人）
```

**禁忌**：
- ❌ 不要 `git merge upstream/main` 到 `cut-non-qlib-scenarios`（会让裁剪 commit 永久 entangled）
- ❌ 不要在 rebase 冲突时"全保留上游"（会让裁剪失效，data_science 等场景又混进来）
- ✅ 用独立 `sync/upstream-*` 分支隔离，rebase 失败可整支丢弃重来

### 1.4 PR 与合并门槛

**所有 PR 必须满足的硬门槛**（CLAUDE.md §9/§10）：

| 门槛 | 触发条件 | 验证方式 |
|---|---|---|
| 冒烟测试通过 | 所有 PR | `conda activate multialphav && rdagent health_check` + `rdagent fin_factor --loop_n 1`（最小循环不崩） |
| CodeGraph 影响分析 | 改 `rdagent/core/` / `components/workflow/` / `scenarios/qlib/` | PR 描述附 CodeGraph `Show Callers` / `Show Impact` 结果截图 |
| 异构实现登记 | 触发 CLAUDE.md §4 红旗 | PR 描述附 §4.2 异构清单表 |
| 多 Agent 审查（≥3 份） | 触发 CLAUDE.md §8 原则 3 | `docs/review/<task>-<agent>.md` 留档，任一否决不得合入 |
| 兼容性矩阵 | 涉及与 0.8 项目数据/API 交互 | PR 描述附 §7.2 适配矩阵 |
| 文档同步 | 行为/接口/配置变更 | 同步 API.md / ENV.md / CLAUDE.md 相应章节（§9 步骤 10） |

**review 人数**：
- 普通改动：**1 人 approve**
- 改 `rdagent/core/`、改共享资源（Qlib 数据/Docker/0.8 env）：**2 人 approve + 多 Agent 审查**
- 上游同步 PR：**2 人 approve，其中 1 人是上游同步负责人**

---

## 2. 环境一致性（每人独立机器）

### 2.1 新成员 onboarding 清单

新成员拿到独立机器后，**按顺序**执行（对应 CLAUDE.md §6 + 本文档）：

```bash
# 1. 克隆双仓库
git clone git@github.com:seuzxh/multialphaV.git ~/projects/multialphaV
cd ~/projects/multialphaV
git clone git@github.com:seuzxh/RD-Agent.git   # 已在根仓库 .gitignore

# 2. 配置 git 身份（仅 RD-Agent 仓库）
cd RD-Agent
git config user.name "<你的名字>"
git config user.email "<你的邮箱>"

# 3. 添加上游远端
git remote add upstream https://github.com/microsoft/RD-Agent.git
git checkout cut-non-qlib-scenarios

# 4. 创建 conda env（Python 3.10，CLAUDE.md §6.2）
conda create -n multialphav python=3.10 -y
conda activate multialphav
cd ~/projects/multialphaV/RD-Agent && pip install -e .

# 5. 准备 Qlib 数据（见 §2.3）
# 6. 拉取/构建 Docker 镜像（见 §2.4）
# 7. 创建 .env（见 §3，从维护者拿模板，填自己的 Key）
# 8. 跑冒烟
rdagent health_check
rdagent fin_factor --loop_n 1
```

### 2.2 conda env 命名约定

每人独立机器 → conda env 名 **统一叫 `multialphav`**（CLAUDE.md §6.2 已规定，无须加用户名后缀）。

**例外**：Factor CoSTEER 子进程有意指向 0.8 共享 env（`/home/zxh/miniconda3/envs/rdagent/bin/python`，见 CLAUDE.md §6.1）。这条规则**仅适用 zxh 的机器**（0.8 项目所在地）。其他成员机器无此 env，需要：

```bash
# 其他成员机器：创建等价 env
conda create -n rdagent4qlib python=3.10 -y   # 名字与代码默认 QlibCondaConf 一致
conda activate rdagent4qlib
pip install pyqlib==0.9.7 catboost xgboost tables torch
# 然后在 .env 显式覆盖（Factor CoSTEER 子进程实际读 CONDA_DEFAULT_ENV）
```

> 注：CLAUDE.md §6.1 的 `FACTOR_CoSTEER_PYTHON_BIN` 当前是死配置（代码实际读 `CONDA_DEFAULT_ENV`，见 ENV.md §5.2）。其他成员机器需保证 `CONDA_DEFAULT_ENV` 在跑 factor loop 时指向装好 qlib 的 env，或在 `.env` 里加 `CONDA_DEFAULT_ENV=rdagent4qlib`（注意：这是直接 os.environ 读取，不是 Settings 字段，见 ENV.md §4）。

### 2.3 Qlib 数据一致性

**目标**：所有成员机器的 Qlib 数据**严格同源同时点**，否则因子对照失真。

**数据来源**：
- 官方数据：`https://github.com/microsoft/qlib/tree/main/scripts/data_collector/yahoo`（自动抓取，国内 A 股用 `collector.py` 拉）
- 本项目当前数据：`/home/zxh/qlib_data`，已更新到 2026-07-17（CLAUDE.md §6.1）

**同步策略（按频率从高到低）**：

| 频率 | 内容 | 方式 |
|---|---|---|
| 每周一 | 增量数据（上周交易日） | 由 §7 "数据维护者" 跑 `collector.py`，打包成 `qlib_data_<YYYYMMDD>.tar.gz` 上传共享存储；其他成员 `wget` + 解压覆盖 |
| 季度 | 全量数据重抓 | 修正历史数据漂移，全员重新对齐 |
| 临时 | 紧急修复（数据源 bug） | 数据维护者发公告，全员约定时点重新拉取 |

**每成员机器的数据布局**（CLAUDE.md §6.1 约定）：

```bash
# 数据本体放在用户目录外，避免误删
mkdir -p ~/qlib_data                  # 或 /data/qlib_data，看磁盘
tar xzf qlib_data_<YYYYMMDD>.tar.gz -C ~/qlib_data

# 软链到代码硬编码路径
mkdir -p ~/.qlib/qlib_data
ln -s ~/qlib_data ~/.qlib/qlib_data/cn_data
```

**验证一致性**（所有成员机器跑同样命令，比对输出）：

```bash
python -c "import qlib; from qlib.constant import REG_CN; qlib.init(provider_uri='~/.qlib/qlib_data/cn_data', region=REG_CN); from qlib.data import D; cal=D.calendar(freq='day'); print(f'days={len(cal)} first={cal[0].date()} last={cal[-1].date()}')"
# 预期全队一致：days=6430 first=2000-01-04 last=2026-07-17（随更新变化）
```

> 全队 first/last 日期必须一致；days 数若不一致，说明有人数据版本不对，需重新对齐。

### 2.4 Docker 镜像一致性

**目标**：所有成员跑 Model 训练时用同一份镜像（`local_qlib:v2.1`，qlib 0.9.7）。

**镜像来源**：
- Dockerfile 在 `rdagent/scenarios/qlib/docker/`（已从 0.8 项目同步，CLAUDE.md §6.4）
- 镜像 10.7GB，**不入库**；但构建依赖 `qlib-src.tar.gz`（4.7MB）**已入库**，新成员无需额外下载

**新成员获取镜像**（两种路径，详见 `rdagent/scenarios/qlib/docker/README.md`）：

```bash
cd ~/projects/multialphaV/RD-Agent/rdagent/scenarios/qlib/docker

# 路径 1（最快）：加载镜像 tar（需先从维护者处获取 tar 文件）
docker load < local_qlib-v2.1.tar.gz

# 路径 2：从源码构建（qlib-src.tar.gz 已在仓库内，约 20-40 分钟）
docker build -t local_qlib:v2.1 .

# 验证
docker run --rm local_qlib:v2.1 python -c "import qlib;print(qlib.__version__)"   # 预期 0.9.7
```

**版本对齐**：每季度或上游 Qlib 发布新版时，全队约定升级镜像版本号（如 `v2.1`），`.env` 同步改 `QLIB_DOCKER_IMAGE=local_qlib:v2.1`。版本变更走 §6 共享资源变更协议。

### 2.5 共享 0.8 env 的禁忌

`/home/zxh/miniconda3/envs/rdagent`（0.8 项目 env）**只在 zxh 机器存在**。其他成员：

- ❌ 不要尝试复刻此 env 名（会与代码默认值混淆）
- ❌ 不要在 `.env` 写 `FACTOR_CoSTEER_PYTHON_BIN=/home/zxh/miniconda3/envs/rdagent/bin/python`（这是 zxh 机器的绝对路径）
- ✅ 用本地 `rdagent4qlib` env 替代（§2.2）

---

## 3. LLM / Embedding Key 管理

**原则**：每人独立 Key，`.env` 不入库，分级管理。

### 3.1 Key 类型与获取

| Key 类型 | 用途 | 默认提供商（CLAUDE.md §6.3） | 获取方式 |
|---|---|---|---|
| Chat LLM Key | 因子/模型假设生成、CoSTEER 编码、反馈 | 火山方舟 Coding Plan（多 model：glm-5.2 / minimax-m3 / kimi-k2.7-code / deepseek-v4-flash） | 各成员去 https://www.volcengine.com/product/ark 开通 Coding Plan |
| Embedding Key | 知识库向量化、相似因子检索 | 火山方舟豆包（与 chat 同账号同 key） | 同上，Coding Plan 含 embedding 额度 |

### 3.2 `.env` 分发流程

`.env` 在 `.gitignore`（根仓库 + RD-Agent 都忽略了），不入库。新成员的 `.env` 流程：

```bash
# 1. 从 RD-Agent fork 拿模板
cp ~/projects/multialphaV/RD-Agent/.env.example ~/projects/multialphaV/RD-Agent/.env

# 2. 按模板填自己的 Key（参考 ENV.md §7 最小模板）
#    Key 值通过团队密码管理器分发（见 §3.4），禁止 IM/邮件明文

# 3. 验证
cd ~/projects/multialphaV/RD-Agent
rdagent health_check   # 应该 test_chat / test_embedding 全 PASS
```

### 3.3 Key 轮换与分级

**分级**（建议团队约定）：

| 级别 | 用途 | 额度 | 轮换周期 |
|---|---|---|---|
| 生产 Key | 跑正式 loop（fin_factor/fin_model/fin_quant） | 高 | 季度轮换，泄露立即换 |
| 实验 Key | 短期实验、调参 | 中 | 按需 |
| 调试 Key | 开发调试、health_check | 低 | 长期 |

**轮换流程**：
1. 在提供商后台禁用旧 Key（不要先删，先禁用便于回滚）
2. 生成新 Key
3. 通过密码管理器（§3.4）分发新值
4. 各成员改本地 `.env`，跑 `rdagent health_check` 验证
5. 观察 1 周，确认无回退后彻底删除旧 Key

### 3.4 密码管理器推荐

**禁止**用微信/邮件/Slack 明文传 Key。建议任选其一：
- **1Password / Bitwarden**（共享 vault）：团队 vault 存 `.env` 模板 + 各成员 Key
- **GitHub Secrets + 自动化**（进阶）：在 fork 仓库设 GitHub Secrets，CI 自动注入；本机用 `gh secret` 拉取
- **内网 Vault**（重型）：团队自建 HashiCorp Vault，按角色分发

> 本文档不绑定具体工具，但**必须用某种加密分发机制**，PR review 时检查提交者是否声明了 Key 来源。

### 3.5 费用与遥测

LiteLLM 后端默认累积 `ACC_COST`（`rdagent/oai/backend/litellm.py`）。建议：
- 每月各成员上报自己的 `ACC_COST`（trace 末尾会打印）
- 设单次 loop 成本上限（环境变量 `CHAT_TOKEN_LIMIT=100000` 已限单次对话，可额外约定 loop 级上限）
- 异常高额（> 团队均值 3x）触发告警 review

---

## 4. trace 与实验产物共享

**目标**：trace/指标/因子产物上传共享对象存储，全员可只读拉取任意历史实验全量产物做对照与复现。

### 4.1 产物层级与命名

```
<共享存储根>/
├── multialphav-traces/                    # 全量 trace（含 pickle、stdout、h5）
│   └── <YYYY>/<MM>/<DD>/<user>-<scenario>-<trace_name>/
│       ├── meta.json                       # 必需，见附录 C
│       ├── stdout.log
│       ├── __session__/                    # rdagent session pickle
│       └── ...其他产物
├── multialphav-results/                   # 提取后的指标（轻量，便于 diff）
│   └── <YYYY>/<MM>/<DD>/<user>-<scenario>-<trace_name>.json
└── multialphav-factors/                   # 因子表达式库（跨人复用）
    └── <factor_set_name>.json              # {factor_name: expression, ...}
```

**命名约定**：
- `<user>`：成员用户名（与 git config user.name 一致）
- `<scenario>`：`factor` / `model` / `quant` / `factor_report`
- `<trace_name>`：rdagent 自动生成的 randomname（如 `adoring-einstein`）
- 完整 trace id：`<user>-<scenario>-<trace_name>`（全局唯一）

### 4.2 上传约定

**上传触发**：每次完整 loop 结束后**自动上传**（或人工触发）。建议在 rdagent 跑完后执行：

```bash
# 假设共享存储是 OSS（命令可替换为 S3/aws/minio client）
TRACE_ID="<user>-<scenario>-$(basename $TRACE_PATH)"
REMOTE="oss://multialphav-bucket/multialphav-traces/$(date +%Y/%m/%d)/$TRACE_ID/"

# 1. 上传全量 trace
ossutil cp -r $TRACE_PATH $REMOTE

# 2. 提取指标 JSON（从 trace 末尾的 feedback 节点抽取 IC/IR/Sharpe）
python scripts/extract_metrics.py $TRACE_PATH > /tmp/$TRACE_ID.json
ossutil cp /tmp/$TRACE_ID.json oss://multialphav-bucket/multialphav-results/$(date +%Y/%m/%d)/

# 3. 上传 meta.json（见附录 C，必需）
# meta.json 含：提交者、git commit、数据版本、环境版本、配置摘要
```

**强制要求**（CLAUDE.md §11 交付规范的延伸）：
- **meta.json 必填**：没有 meta.json 的 trace 视为不可复现，review 时不计入对照
- **git commit 必填**：meta.json 必须记录跑该 trace 时的 `git describe --tags --always` 值
- **数据版本必填**：meta.json 必须记录 Qlib 数据的 first/last 日期（§2.3 验证命令输出）

### 4.3 对照与复现流程

**对照流程**（跨人/跨实验比较指标）：

```bash
# 1. 拉取某天的所有 results JSON
ossutil cp -r oss://multialphav-bucket/multialphav-results/2026/07/19/ /tmp/results/

# 2. 用 jq 对照 IC/IR/Sharpe
jq -s '.[] | {trace_id, ic: .feedback.IC, sharpe: .feedback.sharpe}' /tmp/results/*.json | column -t

# 3. 找最优基线
jq -s 'sort_by(-.feedback.sharpe) | .[0]' /tmp/results/*.json
```

**复现流程**（拉取他人 trace 在本机重跑验证）：

```bash
# 1. 拉取全量 trace
ossutil cp -r oss://multialphav-bucket/multialphav-traces/2026/07/19/alice-factor-xxx/ /tmp/alice-trace/

# 2. 读 meta.json，对齐环境
cat /tmp/alice-trace/meta.json
#   → 检查 git commit，若自己机器未到此 commit：git checkout <commit>
#   → 检查数据版本（first/last），不一致则按 §2.3 重新拉对应版本
#   → 检查镜像版本，不一致则按 §2.4 重新构建

# 3. 用 rdagent 的 session 恢复机制重跑
cd ~/projects/multialphaV/RD-Agent
rdagent fin_factor --path /tmp/alice-trace/__session__/<最新session>/ --step_n 1
```

**因子跨人复用规则**（CLAUDE.md §7.4）：
- 因子表达式（字符串形式）：可直接复用（如 `Mean($close, 5)`）
- 因子 .bin 文件：跨人复用**必须验证 `start_index < len(calendar)`**（CLAUDE.md §7.4 第 2 条）
- 因子库 JSON：从 `multialphav-factors/` 拉取，导入自己的 loop

---

## 5. Code Review 与多 Agent 审查

### 5.1 Review 维度

每个 PR 至少覆盖以下维度之一（CLAUDE.md §8 原则 3 的多 Agent 审查对应到人工 review）：

| 维度 | 必查项 | 适合的 reviewer |
|---|---|---|
| 框架一致性 | 是否触发 §4 红旗？是否填 §3 表？是否复用 rdagent 基类？ | 熟悉 rdagent core 的人 |
| 健壮性/可靠性 | 异常处理、并发、回滚、边界（空数据、全 NaN、start_index 溢出） | 任何人 |
| 跨项目兼容性 | 是否填 §7.2 矩阵？数据格式转换是否完备？ | 熟悉 0.8 项目的人 |
| 性能 | LLM 调用次数、Docker 重复构建、pickle 缓存命中率 | 任何人 |
| 文档同步 | API.md / ENV.md / CLAUDE.md 是否同步？ | 任何人 |

### 5.2 多 Agent 审查触发与留档

CLAUDE.md §8 原则 3 触发时（>3 模块协作 / 改 core / 涉及 0.8 兼容 / 触共享资源），PR 必须**附 ≥3 份独立审查报告**到：

```
docs/review/<PR号>-<task名>-<agent角色>.md
```

例如 `docs/review/42-multi-factor-selector-framework.md`、`-robustness.md`、`-compatibility.md`。

人工 review 与 Agent 审查关系：
- **Agent 审查**：深度、可并行、覆盖机械性检查（红旗、矩阵、影响面）
- **人工 review**：判断性、业务正确性、设计哲学契合度
- **任一 Agent 否决不得合入**（CLAUDE.md §8 原则 3），人工 review 1-2 人 approve 视改动范围定（§1.4）

### 5.3 AI Agent 作为协作者

本项目专为 Claude Code / ZCode 设计，Agent 也是"协作者"。协作纪律：
- Agent 的所有改动必须能追溯到用户请求（CLAUDE.md §8 原则 4）
- 派发子 Agent 遵循科斯定理（§8 原则 6）：核心模块禁派，边界清晰才派
- Agent 产出的代码 review 标准与人工一致（不降标）
- 多人协作时，**明确哪个 Agent 属于哪个成员**（trace 上传时 meta.json 记 user，不记 agent name）

---

## 6. 共享资源变更协议

**共享资源**（CLAUDE.md §6.1）：Qlib 数据、Docker 镜像、0.8 env、上游代码同步。

**变更流程**（任何一项变更必须走）：

```
1. 提案：发起人在 multialphaV 根仓库提 issue，标题 [INFRA] <资源名> <变更内容>
2. 评估：至少 2 人评估影响面（CodeGraph Show Impact + 数据/镜像一致性核查）
3. 公告：全队通告变更时点（建议周一早晨，避开实验高峰）
4. 执行：发起人在自己机器执行，记录 before/after 版本号
5. 验证：全员按 §2.3 / §2.4 验证命令核对一致性
6. 文档：同步 CLAUDE.md §6.1 / ENV.md 对应章节
7. 窗口期：变更后 48 小时内，旧实验产物标记为"基于旧版本"，不参与新对照
```

**禁止**：
- ❌ 未经公告私自改 Qlib 数据（会让他人正在跑的实验结果失真）
- ❌ 在工作日实验高峰期（9:00-22:00）拉新版镜像
- ❌ 跳过验证步骤直接合入

---

## 7. 角色与责任分工

3-5 人团队建议的角色分工（一人可兼多角色，角色轮换周期建议季度）：

| 角色 | 职责 | 权限 |
|---|---|---|
| **上游同步负责人** | §1.3 上游同步、裁剪重放、rebase 冲突处理 | 唯一有 `sync/upstream-*` 分支 push 权的人 |
| **数据维护者** | §2.3 Qlib 数据更新、增量打包、版本公告 | 唯一有权改共享 Qlib 数据源 |
| **镜像维护者** | §2.4 Docker 镜像构建、版本升级 | 唯一有权改 Dockerfile + 推共享镜像 |
| **基础设施负责人** | §3 Key 管理、§4 共享存储运维、密码管理器 | 持 admin Key，分发成员 Key |
| **核心代码 owner** | review `rdagent/core/` 改动、把关框架一致性 | 核心模块 PR 的强制 reviewer |

**默认分工建议**（3-5 人）：
- 3 人：A=上游同步+核心 owner，B=数据+镜像，C=基础设施
- 5 人：A=上游同步，B=核心 owner，C=数据，D=镜像，E=基础设施+文档

> 角色分工不是僵化的，目的是**让每类高风险操作有明确责任人**，避免"谁都以为别人会改"的真空。

---

## 8. 常见冲突与处置

| 场景 | 处置 |
|---|---|
| 两人 feature 分支都改了 `rdagent/core/scenario.py` | rebase 时冲突；核心 owner 介入裁决，原则上保留更贴近 rdagent 设计哲学的一方（CLAUDE.md §4.3） |
| 一人改了 Qlib 数据，另一人正在跑 loop | 跑 loop 的人有权要求回滚（§6 公告机制缺失时）；事后补公告流程 |
| 上游同步 rebase 冲突太多（>50 文件） | 暂停同步，先在 issue 里拆分冲突，分批处理；避免一次性大规模 rebase |
| Agent 审查与人工 review 结论冲突 | 以人工 review 为准（CLAUDE.md §8 原则 3，Agent 审查是辅助） |
| 因子 .bin 跨人复用时 start_index 溢出 | 立即停止使用，按 CLAUDE.md §7.4 重新生成；记录到 issue 防再犯 |
| 成员机器数据版本不一致（§2.3 验证输出不同） | 全员停跑实验，数据维护者统一拉版本，全员重新对齐后再恢复 |
| trace 上传到共享存储失败 | 保留本地 trace，issue 报告；不可丢 trace（影响可复现性） |
| Key 泄露怀疑 | 立即按 §3.3 轮换流程禁用可疑 Key，审计近期 ACC_COST 异常 |

---

## 附录 A：新人第一周 checklist

```
Day 1：环境搭建
  [ ] 克隆双仓库，配置 git 身份
  [ ] 创建 conda env multialphav，pip install -e .
  [ ] 读 CLAUDE.md 全文 + 本文档 + API.md + ENV.md
  [ ] 拿到密码管理器访问权，获取 .env 模板

Day 2：数据与镜像
  [ ] 按 §2.3 拉取最新 Qlib 数据，建软链
  [ ] 按 §2.4 构建 local_qlib:v2.1 镜像
  [ ] 跑 §2.3 验证命令，与团队 first/last 日期核对

Day 3：Key 与冒烟
  [ ] 注册智谱 + 火山方舟账号，申请自己的 Key
  [ ] 填 .env，跑 rdagent health_check（全 PASS）
  [ ] 跑 rdagent fin_factor --loop_n 1（最小 loop 不崩）

Day 4：产物上传
  [ ] 配置共享存储客户端（ossutil / aws s3）
  [ ] 把 Day 3 的 trace 按 §4.2 上传
  [ ] 验证其他人能拉取你的 trace

Day 5：实战
  [ ] 从 issue 列表挑一个 starter 任务
  [ ] 切 feature 分支，开发，提 PR
  [ ] 经历完整 review 流程，合入 cut-non-qlib-scenarios
```

## 附录 B：PR 模板

```markdown
## 改动落在哪个仓库
- [ ] multialphaV（文档/规范）
- [ ] RD-Agent fork（代码）

## 改动摘要
<一句话说清改了什么、为什么>

## CLAUDE.md 自检
- [ ] 已读 CLAUDE.md 相关章节
- [ ] 已填 §3 核心组件表（如涉及框架扩展）
- [ ] 已登记 §4.2 异构实现清单（如触发红旗）
- [ ] 已填 §7.2 兼容性矩阵（如涉及 0.8 数据/API）
- [ ] 已用 CodeGraph Show Callers / Impact（如改 core/workflow/qlib）

## Review 触发
- [ ] 触发 §8 原则 3 多 Agent 审查（附 docs/review/ 报告路径）
- [ ] 触发 §6 共享资源变更协议（附 issue 链接）

## 验证证据
- [ ] rdagent health_check 通过
- [ ] rdagent fin_factor --loop_n 1 冒烟通过
- [ ] 单元测试 / 集成测试结果

## 文档同步（§9 步骤 10）
- 关联影响：无 / 已处理 X 处
- 文档同步：无 / 已更新 <文件> §<章节>

## trace / 实验（如涉及）
- 共享存储路径：oss://multialphav-bucket/multialphav-traces/...
- meta.json 已上传：是 / 不适用
```

## 附录 C：trace 元数据 schema

```json
{
  "trace_id": "alice-factor-adoring-einstein-20260719",
  "scenario": "fin_factor",
  "user": "alice",
  "timestamp": "2026-07-19T14:30:00+08:00",
  "git": {
    "commit": "v0.8.0-35-g<hash>",
    "branch": "feat/42-multi-factor-selector",
    "dirty": false
  },
  "data": {
    "qlib_first": "2000-01-04",
    "qlib_last": "2026-07-17",
    "qlib_days": 6430,
    "market": "csi300"
  },
  "docker": {
    "image": "local_qlib:v2.1",
    "qlib_version": "0.9.7"
  },
  "config_digest": "<.env 关键项的 sha256，不含 Key>",
  "metrics": {
    "ic": 0.053,
    "ir": 0.41,
    "sharpe": 1.27
  },
  "cost": {
    "llm_usd": 2.34,
    "loop_n": 10
  },
  "notes": "测试多因子选择器，对照 baseline 提升 Sharpe 0.15"
}
```

**必填字段**：`trace_id` / `scenario` / `user` / `git.commit` / `data.qlib_first/last` / `docker.image`。其余可选但建议填写。

---

**版本**：v1.0（2026-07-19）
**适用目录**：`/home/zxh/projects/1.multialphaV`（根仓库，本文档所在地）+ `/home/zxh/projects/1.multialphaV/RD-Agent`（代码仓库）
**配套文档**：[CLAUDE.md](file:///home/zxh/projects/1.multialphaV/CLAUDE.md)（开发行为约束）、[API.md](file:///home/zxh/projects/1.multialphaV/API.md)（接口参考）、[ENV.md](file:///home/zxh/projects/1.multialphaV/ENV.md)（配置参考）、[REPOS.md](file:///home/zxh/projects/1.multialphaV/REPOS.md)（仓库结构）
