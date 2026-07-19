# multialphaV GitHub 仓库记录

> 本文档记录 multialphaV 项目推送到 GitHub 的两个仓库的完整信息，便于后续同步、协作与故障排查。
> 创建时间：2026-07-18（ssh key 配置后首次成功推送）

---

## 仓库概览

本项目包含 **两个独立 GitHub 仓库**，分别承载不同职责，互不嵌套（multialphaV 根仓库的 `.gitignore` 忽略整个 `RD-Agent/` 子目录）：

| 仓库 | URL | 可见性 | 职责 | 默认分支 |
|---|---|---|---|---|
| **multialphaV** | `git@github.com:seuzxh/multialphaV.git` | 私有 | 项目级文档与规范（CLAUDE.md / PLAN-*.md / .gitignore） | `main` |
| **RD-Agent** | `git@github.com:seuzxh/RD-Agent.git` | 私有（fork） | RD-Agent 代码（裁剪后的 Qlib-only 分支） | `main`（fork 自上游，未改） |

**关系图**：
```
multialphaV（根仓库，私有）        github.com:seuzxh/multialphaV
├── CLAUDE.md                      └─ 仅跟踪项目规范/计划文档
├── PLAN-env-setup.md
├── REPOS.md（本文档）
├── .gitignore                     └─ 忽略 RD-Agent/、.codegraph/、.zcode/、.env*
└── RD-Agent/（被忽略，独立仓库）──→ RD-Agent（fork，私有）    github.com:seuzxh/RD-Agent
                                        └─ origin: fork（seuzxh）
                                        └─ upstream: microsoft/RD-Agent（上游）
```

---

## 仓库 1：multialphaV（根仓库）

### 基本信息

| 项 | 值 |
|---|---|
| SSH URL | `git@github.com:seuzxh/multialphaV.git` |
| HTTPS URL | `https://github.com/seuzxh/multialphaV.git`（当前网络环境 https 不稳，推荐 ssh） |
| 可见性 | 私有 |
| 默认分支 | `main` |
| 创建方式 | `gh repo create seuzxh/multialphaV --private`（2026-07-18） |
| 跟踪内容 | 项目级文档（CLAUDE.md / PLAN-*.md / REPOS.md / .gitignore），**不含代码** |
| 本地路径 | `/home/zxh/projects/1.multialphaV` |

### 分支与 commit 历史

**分支：`main`**（唯一分支，HEAD = `ecc4dba`，与远端完全同步）

| commit | 时间 | 说明 |
|---|---|---|
| `ecc4dba` | 2026-07-18 16:14 | env(stage6): full acceptance passed + stage3 deferred |
| `f58a4d4` | 2026-07-18 16:08 | docs(§6.4/§12): fix rdagent version check commands |
| `9ebc7f2` | 2026-07-18 16:01 | env(stage4b): .gitignore .codegraph/ verified |
| `4b38eb5` | 2026-07-18 16:00 | env(stage2): RD-Agent/.env created with inherited keys |
| `e1a9885` | 2026-07-18 15:59 | env(stage1): conda env multialphav created |
| `7155ef4` | 2026-07-18 15:56 | init: multialphaV root repo with project specs |

### .gitignore 策略

```
/RD-Agent/          # 子仓库独立管理，避免外层重复跟踪
.codegraph/         # CodeGraph 索引（CLAUDE.md §5.1 硬要求）
.zcode/             # ZCode harness 会话缓存
.env*  *.key  *secret*  *token*  *credential*   # 防敏感信息泄露
```

---

## 仓库 2：RD-Agent（fork）

### 基本信息

| 项 | 值 |
|---|---|
| origin（fork） | `git@github.com:seuzxh/RD-Agent.git` |
| upstream（上游） | `https://github.com/microsoft/RD-Agent.git` |
| 可见性 | 私有（fork 自公开的 microsoft/RD-Agent） |
| Fork 源 | microsoft/RD-Agent @ `4f9ecb00`（上游 main，2026-05-06） |
| 本地路径 | `/home/zxh/projects/1.multialphaV/RD-Agent` |
| git 身份 | `seuzxh <seuzxh@gmail.com>`（仅本仓库，非全局） |

### 分支

| 分支 | HEAD | 说明 |
|---|---|---|
| `cut-non-qlib-scenarios` | `7cedd780` | **本项目工作分支**：裁剪非 Qlib 场景 + docker 同步 |
| `main` | `4f9ecb00` | fork 自上游 microsoft/RD-Agent，**未修改**（保持与上游同步能力） |

### `cut-non-qlib-scenarios` 分支 commit 历史

本分支在 fork 点（`4f9ecb00`）基础上有 **2 个本项目 commit**：

| commit | 时间 | 说明 | 改动量 |
|---|---|---|---|
| `7cedd780` | 2026-07-18 16:11 | docker(qlib): sync 0.8 project's Dockerfile + lock + README | +268 / -8（4 文件） |
| `8f60d6ea` | 2026-07-18 15:32 | cut: remove non-Qlib scenarios per CLAUDE.md §0/§10 | +26 / -61139（539 文件） |

**`git describe --tags --always` 当前值**：`v0.8.0-30-g7cedd780`（fork 点 tag v0.8.0 + 30 个 commit + hash）
- 用于同步升级对比：`git fetch upstream && git log --oneline $(git describe --tags --always)..upstream/main`

### 裁剪改动摘要（commit `8f60d6ea`）

- **删除** 529 文件 / ~61k 行：`scenarios/{data_science,kaggle,finetune,general_model,rl}/`、对应 `app/`、`components/coder/{data_science,finetune,rl}/`、`log/ui/ds_*.py`、`log/mle_summary.py`、场景专用 test/docs
- **外科编辑** 10 个共享文件：cli.py、factor.py、model.py、log/ui/app.py、log/ui/storage.py、log/server/app.py、log/utils/folder.py、utils/env.py（14 个场景 Env 类）、docs/scens/catalog.rst、Makefile
- **验证**：Qlib 导入链全绿（cli/quant/factor/model/factor_from_report/scenario/factor_coder/model_coder）

### docker 配置同步（commit `7cedd780`）

从 0.8 项目（`/home/zxh/quant_projects/RD-Agent`）复制 4 个文件到 `rdagent/scenarios/qlib/docker/`：
- `Dockerfile`（阿里云镜像源 + requirements.lock 锁依赖 + 本地 qlib-src.tar.gz）
- `requirements.lock.txt`（161 包锁定）
- `README.md`（构建/使用/数据挂载文档）
- `qlib-src.tar.gz`（4.7MB，**gitignore 不入库**，按 README 用 wget 准备）

运行时镜像 `local_qlib:v2.0`（qlib 0.9.7，10.7GB）已系统级共享，`.env` 配 `QLIB_DOCKER_BUILD_FROM_DOCKERFILE=False` 不触发重建。

---

## SSH 配置（推送通道）

> 2026-07-18 配置。此前 https 推送因 443 端口 TLS 流量被干扰持续失败（`gnutls_handshake() failed`），改用 ssh 22 端口后畅通。

| 项 | 值 |
|---|---|
| key 文件 | `~/.ssh/id_ed25519`（ed25519，2026-05-23 生成） |
| 公钥指纹 | `SHA256:emfVVCClGItAfEJcG+WoE6SvpVO+mWBcyne5+RvG/hI` |
| 公钥注释 | `suezh@163.com`（GitHub 不依赖注释匹配账号） |
| GitHub 账号 | `seuzxh`（`ssh -T` 验证返回 `Hi seuzxh!`） |
| 添加方式 | 手动通过 https://github.com/settings/ssh/new 添加（gh api 当时也被 https 干扰） |
| 22 端口连通性 | ✅ 可达（ssh 不受 https 443 干扰） |

---

## 常用操作速查

### 推送（ssh，已配置）

```bash
# multialphaV 根仓库
cd /home/zxh/projects/1.multialphaV
git push origin main

# RD-Agent 仓库
cd /home/zxh/projects/1.multialphaV/RD-Agent
git push origin cut-non-qlib-scenarios
```

### 同步上游更新（RD-Agent）

```bash
cd /home/zxh/projects/1.multialphaV/RD-Agent
git fetch upstream
git log --oneline $(git describe --tags --always)..upstream/main   # 看上游新增了什么
# 决定是否 merge / rebase：
# git merge upstream/main          # 保留裁剪 commit 历史
# git rebase upstream/main          # 把裁剪 commit 重放到上游最新（历史更线性，但需 force push）
```

### 查看远端同步状态

```bash
# 任一仓库内
git rev-list --count origin/<branch>..HEAD   # 本地领先远端（应为 0）
git rev-list --count HEAD..origin/<branch>   # 远端领先本地（应为 0）
```

### 回滚方案

- **multialphaV 文档误改**：`git checkout <file>` 或 `git reset --hard ecc4dba`（最新 commit）
- **RD-Agent 裁剪出问题**：`git checkout cut-non-qlib-scenarios`，可回退到 `8f60d6ea`（裁剪）或 `4f9ecb00`（fork 点，撤销所有裁剪）

---

## 后续待办（与推送无关，备忘）

- **qlib 数据软链**（PLAN-env-setup 阶段 3，用户暂缓）：需 sudo 执行 `sudo mv /home/zxh/.qlib/qlib_data/cn_data{,.bak.2020} && sudo ln -s /home/zxh/qlib_data /home/zxh/.qlib/qlib_data/cn_data`，详见 PLAN-env-setup.md 阶段 3 说明

---

**版本**：v1.0（2026-07-18 初次记录，ssh 推送成功后）
**维护策略**：每次有新的推送/分支/远程变更时更新本文档
