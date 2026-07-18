# multialphaV 环境准备计划

## 目标与边界

依据 [CLAUDE.md](file:///home/zxh/projects/1.multialphaV/CLAUDE.md) §6（环境配置）+ §5（CodeGraph）+ §7（0.8 兼容），为裁剪后的 RD-Agent 代码搭建项目级独立运行环境，使其能在 conda env `multialphav` 中正常加载 Qlib 场景。

**适用代码基线**：commit `8f60d6e`（分支 `cut-non-qlib-scenarios`，已推送 fork），裁剪后仅保留 Qlib 运行时 + 共享底座（230 文件）。

**前置事实（裁剪后 review 已核实，2026-07-18）**：
- 裁剪**未破坏任何环境依赖假设**：`FACTOR_CoSTEER_PYTHON_BIN` / `MODEL_CoSTEER_ENV_TYPE` / `QLIB_DOCKER_*` 等配置读取点全部保留（见 review 核对 1/2）
- `LiteLLMAPIBackend`（`rdagent/oai/backend/litellm.py:48`）保留
- context7 原生集成（`rdagent/components/agent/context7/`，3 个文件）保留
- `provider_uri` 在裁剪后**仍全部硬编码** `~/.qlib/qlib_data/cn_data`（6 处 yaml + 1 处 generate.py）—— 软链是唯一零改动方案
- CLI 装配已在 0.8 rdagent env 验证通过：`fin_quant --help` 正常显示 5 个选项

**CLAUDE.md §6.3 注释勘误**：原文"provider_uri 在代码中显式传 /home/zxh/qlib_data（见 qlib_data_format.md §5）"与代码实际不符（代码全是 `~/.qlib/qlib_data/cn_data`）。本计划通过软链实现等价效果，CLAUDE.md 该注释应在执行后修正为"通过软链 `~/.qlib/qlib_data/cn_data → /home/zxh/qlib_data` 实现"。

---

## 执行进度（6 阶段）

| 阶段 | 内容 | 状态 |
|---|---|---|
| 1 | conda env `multialphav`（Python 3.10）+ `pip install -e .` | ⬜ 待执行 |
| 2 | 创建 `.env`（以 CLAUDE.md §6.3 为准） | ⬜ 待执行 |
| 3 | qlib 数据软链 `~/.qlib/qlib_data/cn_data → /home/zxh/qlib_data` | ⬜ 待执行 |
| 4a | CodeGraph 索引重建 | ✅ **已完成**（230 文件/3635 节点） |
| 4b | `.gitignore` 加 `.codegraph/` | ⬜ 待执行 |
| 5 | git fork remote 配置 | ✅ **已完成**（origin→fork, upstream→microsoft） |
| 6 | 验收（multialphav env 跑导入链 + CLI） | ⬜ 待执行（依赖 1-3） |

---

## 阶段 1：conda env multialphav（Python 3.10）

> CLAUDE.md §6.2（已更新为 conda + Python 3.10 方案）。

**Python 版本选择依据**：官方 CI 仅测 3.10/3.11（`ci.yml` 矩阵），constraints 仅有 3.10/3.11 锁定文件，0.8 项目用 3.10.20。选 3.10 实现三方一致：
- 与官方 CI 测试版本一致（有官方 constraints 锁定依赖组合兜底）
- 与 0.8 项目（`FACTOR_CoSTEER_PYTHON_BIN` 指向的 env）同版本 → 主子进程无 pickle 跨版本风险

```bash
cd /home/zxh/projects/1.multialphaV/RD-Agent
/home/zxh/miniconda3/bin/conda create -n multialphav python=3.10 -y
source /home/zxh/miniconda3/bin/activate multialphav
pip install -e .  # 安装最新 rdagent 本体（editable）
```

**说明**：
- `pip install -e .` 会通过 setuptools-scm 从 git tag 动态生成版本号（预期 `0.8.1.dev28` 左右）
- pydantic 系列未 pin 版本，会拉最新兼容版；若遇兼容问题，可参考 `constraints/3.10.txt`（官方锁定）或临时 pin `pydantic<2.11` 重装
- qlib **不装进主环境**（走 Docker，见阶段 3 说明）

**版本基线核对（CLAUDE.md §6.4）**：
```bash
python -c "import sys, rdagent, pydantic; print(sys.version.split()[0], rdagent.__version__, pydantic.VERSION)"
# 预期: Python 3.10.x / rdagent 0.8.x / pydantic 2.x
```

**回滚**：`conda env remove -n multialphav` 后重建。

---

## 阶段 2：创建 `.env` 配置文件

> CLAUDE.md §6.3。**以 CLAUDE.md §6.3 为准，不要复制 `.env.example`**（后者是上游 gpt-4o 通用模板）。

**路径**：`/home/zxh/projects/1.multialphaV/RD-Agent/.env`

**密钥来源**：从 `/home/zxh/quant_projects/rdagent/.env`（0.8 项目）继承 3 个 key 的真实值：
- `CHAT_OPENAI_API_KEY` / `OPENAI_API_KEY`（同值，指向智谱 AI，49 字符）
- `EMBEDDING_OPENAI_API_KEY`（火山方舟/豆包，36 字符）

**完整内容**（非密钥项见 CLAUDE.md §6.3 模板）：
```dotenv
BACKEND=rdagent.oai.backend.LiteLLMAPIBackend

CHAT_OPENAI_BASE_URL=https://open.bigmodel.cn/api/coding/paas/v4
OPENAI_API_BASE=https://open.bigmodel.cn/api/coding/paas/v4
CHAT_MODEL=openai/glm-5.2          # 保留 openai/ LiteLLM 前缀，勿去
CHAT_TEMPERATURE=0.5
CHAT_MAX_TOKENS=16384
CHAT_OPENAI_API_KEY=<从 0.8 .env 继承>
OPENAI_API_KEY=<同上>

EMBEDDING_MODEL=openai/doubao-embedding-vision
EMBEDDING_OPENAI_BASE_URL=https://ark.cn-beijing.volces.com/api/coding/v3
EMBEDDING_OPENAI_API_KEY=<从 0.8 .env 继承>

RDAGENT_MARKET=csi300

FACTOR_CoSTEER_PYTHON_BIN=/home/zxh/miniconda3/envs/rdagent/bin/python  # 有意指向 0.8 env（§6.1 红线，勿改）

MODEL_CoSTEER_ENV_TYPE=docker
QLIB_DOCKER_IMAGE=local_qlib:v2.0
QLIB_DOCKER_BUILD_FROM_DOCKERFILE=False
```

**安全**：
- `.gitignore` 第 116-117 行已忽略 `.env*` / `*.env`，不会入库
- 创建后立即用 `git status .env` 确认其为 untracked 且被忽略

**验证 `.env` 加载**：
```bash
conda activate multialphav
python -c "from rdagent.oai.llm_conf import LLM_SETTINGS; print(LLM_SETTINGS.backend, LLM_SETTINGS.chat_model, LLM_SETTINGS.chat_openai_base_url)"
# 预期: rdagent.oai.backend.LiteLLMAPIBackend openai/glm-5.2 https://open.bigmodel.cn/api/coding/paas/v4
```

**回滚**：`rm .env`。

---

## 阶段 3：Qlib 数据归并（软链）

> CLAUDE.md §6.1 + §6.3。用户已选"软链（推荐）"方案。

**背景**：
- `~/.qlib/qlib_data/cn_data`（即 `/home/zxh/.qlib/qlib_data/cn_data`）当前是 **owner=root 的真实目录**，数据停留在 **2020-09-25（过期 6 年）**
- `/home/zxh/qlib_data` 是最新数据（到 **2026-07-17**），结构对齐（顶层即 calendars/features/instruments）
- 裁剪后代码里 `provider_uri` **仍全部硬编码** `~/.qlib/qlib_data/cn_data`（6 处 yaml + generate.py），改 yaml 会触发 §3 扩展表登记，软链是零代码改动方案

**操作**：
```bash
# 旧目录 owner=root，需 sudo；先改名备份，再建软链
sudo mv /home/zxh/.qlib/qlib_data/cn_data /home/zxh/.qlib/qlib_data/cn_data.bak.2020
sudo ln -s /home/zxh/qlib_data /home/zxh/.qlib/qlib_data/cn_data
ls -ld /home/zxh/.qlib/qlib_data/cn_data  # 应显示 -> /home/zxh/qlib_data
```

**⚠️ 负外部性提示**（§8 原则 6）：此软链是系统级共享路径，会影响其他读 `~/.qlib/qlib_data/cn_data` 的项目（如 conda base 默认 qlib 行为）。备份已保留（`cn_data.bak.2020`），可随时还原。**执行前需向用户确认**。

**验证（需 qlib，用 0.8 rdagent env）**：
```bash
/home/zxh/miniconda3/envs/rdagent/bin/python -c "
import qlib
from qlib.constant import REG_CN
qlib.init(provider_uri='~/.qlib/qlib_data/cn_data', region=REG_CN)
from qlib.data import D
cal = D.calendar(freq='day')
print('calendar days:', len(cal), '| first:', cal[0], '| last:', cal[-1])
"
# 预期: 6430 天 / 2000-01-04 / 2026-07-17
```

**回滚**：
```bash
sudo rm /home/zxh/.qlib/qlib_data/cn_data
sudo mv /home/zxh/.qlib/qlib_data/cn_data.bak.2020 /home/zxh/.qlib/qlib_data/cn_data
```

---

## 阶段 4a：CodeGraph 索引（✅ 已完成）

**已完成内容**（2026-07-18）：
- 全量重建（`codegraph index`）：673 文件/8909 节点 → **230 文件/3635 节点**
- 准确反映裁剪后真实结构，无幽灵节点（KGDockerEnv/KAGGLE_IMPLEMENT_SETTING 已查无）
- 索引位置：`/home/zxh/projects/1.multialphaV/.codegraph/`（项目根）
- daemon v1.4.1，idle timeout 300s

**后续维护**（依赖大变更后）：`cd /home/zxh/projects/1.multialphaV && codegraph index`（全量）或 `codegraph sync`（增量）。

---

## 阶段 4b：`.gitignore` 加 `.codegraph/`

> CLAUDE.md §5.1 硬要求"必须加入 `.gitignore`"。

**现状**：`.gitignore` 未显式忽略 `.codegraph/`（仅内部 `*.db`/`*.log` 被通用规则覆盖，目录本身可被 `git add .` 误加入）。

**操作**：在 `/home/zxh/projects/1.multialphaV/RD-Agent/.gitignore` 适当位置追加一行 `.codegraph/`。

**说明**：`.codegraph/` 实际在 `multialphaV/` 根而非 `RD-Agent/` 内，但 `RD-Agent/.gitignore` 是本项目唯一 git 仓库的忽略文件；若未来 `.codegraph/` 位置变动，此条目无害。

---

## 阶段 5：git fork remote（✅ 已完成）

**已完成内容**（2026-07-18）：
- `origin` → `seuzxh/RD-Agent`（fork，https + gh token 认证）
- `upstream` → `microsoft/RD-Agent`（上游）
- 当前分支 `cut-non-qlib-scenarios` 已推送 fork
- 本仓库 git 身份：`seuzxh <seuzxh@gmail.com>`（仅本仓库）

**验证**：
```bash
cd /home/zxh/projects/1.multialphaV/RD-Agent
git remote -v
# origin    https://github.com/seuzxh/RD-Agent.git
# upstream  https://github.com/microsoft/RD-Agent.git
```

---

## 阶段 6：验收

> 建成 multialphav env 后执行（依赖阶段 1-3 完成）。

```bash
cd /home/zxh/projects/1.multialphaV/RD-Agent
source /home/zxh/miniconda3/bin/activate multialphav

# 1. 版本基线（CLAUDE.md §12）
python --version                    # 3.10.x
python -c "import rdagent, pydantic; print(rdagent.__version__, pydantic.VERSION)"  # 0.8.x / 2.x

# 2. .env 加载
python -c "from rdagent.oai.llm_conf import LLM_SETTINGS; print(LLM_SETTINGS.chat_model)"  # openai/glm-5.2

# 3. Qlib 导入链（裁剪后核心验证）
python -c "import rdagent.app.cli; print('cli OK')"
python -c "from rdagent.app.qlib_rd_loop.quant import main; print('quant OK')"
python -c "from rdagent.app.qlib_rd_loop.factor import main; print('factor OK')"
python -c "from rdagent.app.qlib_rd_loop.model import main; print('model OK')"
python -c "from rdagent.app.qlib_rd_loop.factor_from_report import main; print('factor_from_report OK')"
python -c "from rdagent.scenarios.qlib.experiment.quant_experiment import QlibQuantScenario; print('scenario OK')"
python -c "from rdagent.components.coder.factor_coder.factor import FactorFBWorkspace, FactorTask; print('factor_coder OK')"
python -c "from rdagent.components.coder.model_coder.model import ModelFBWorkspace, ModelTask; print('model_coder OK')"

# 4. CLI 装配
python -m rdagent.app.cli fin_quant --help   # 应显示 path/step-n/loop-n/all-duration/checkout 5 选项

# 5. log/ui + server
python -c "import rdagent.log.ui.app; print('log/ui OK')"
python -c "import rdagent.log.server.app; print('log/server OK')"

# 6. qlib 数据（用 0.8 env，qlib 不在 multialphav）
# 见阶段 3 验证命令

# 7. Docker 镜像
docker images | grep local_qlib   # local_qlib:v2.0
```

每项如实报告 ✅/❌。任何 ❌ 给出根因和修复方案。

---

## 不做的事（§8 原则 4 范围限定）

- **不修改 `rdagent/` 源码**（数据归并走软链而非改 yaml，避免 §3 表登记；本计划仅改 `.gitignore`）
- **不装 qlib 到 multialphav env**（设计上走 Docker，§6.1 已确认 `local_qlib:v2.0` 镜像就绪）
- **不配置 context7 MCP server 进程**（§2.1 已是原生集成，`CONTEXT7_*` 用默认值即可，server 进程管理属运行期事务，非准备工作）
- **不跑 test/qlib 测试套件**（依赖 qlib 数据 + Docker，属运行期验证；静态导入链验证已足够）
- **不创建 README / 其他 docs**（§10 红线）
- **不 commit / push**（§10 红线；本计划产生的 `.gitignore` 改动属裁剪分支后续，由用户决定是否单独提交）
- **不触碰 `rdagent/core/`、`rdagent/components/workflow/`、`rdagent/scenarios/qlib/`**（§5.2 红线目录，本计划全程零源码修改）

---

## 风险与回滚

| 风险点 | 缓解 / 回滚 |
|---|---|
| multialphav env 装包失败（pydantic 未 pin 版本冲突） | `conda env remove -n multialphav` 重建；必要时 pin `pydantic<2.11` |
| `.env` 密钥泄露 | `.gitignore` 已忽略；创建后 `git status .env` 确认未跟踪 |
| 软链影响其他读 `~/.qlib/qlib_data/cn_data` 的项目 | 备份 `cn_data.bak.2020` 保留，可随时还原（见阶段 3 回滚） |
| `.gitignore` 改动未提交导致 `git add .` 误加 `.codegraph/` | 阶段 4b 执行后立即 `git status` 确认 |

---

## 与代码裁剪计划的关系

本计划是 [代码裁剪计划](file:///home/zxh/projects/1.multialphaV/.zcode/plans/plan-sess_f11a5a33-68cf-4214-af93-2a6d663f9e78.md) 的后续。裁剪（commit `8f60d6e`）已完成并推送 fork，本计划在其成果上搭建运行环境。两份计划共享同一份 [CLAUDE.md](file:///home/zxh/projects/1.multialphaV/CLAUDE.md) 权威规范。

---

**版本**：v1.0（2026-07-18 整理）
**适用代码基线**：`cut-non-qlib-scenarios` 分支 @ `8f60d6e`
**权威规范**：[CLAUDE.md](file:///home/zxh/projects/1.multialphaV/CLAUDE.md) §5/§6/§7/§10
