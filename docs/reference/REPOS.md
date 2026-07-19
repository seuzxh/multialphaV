# multialphaV GitHub 仓库记录

> 本文档记录 multialphaV 项目推送到 GitHub 的两个仓库的完整信息，便于后续同步、协作与故障排查。
> 创建时间：2026-07-18（ssh key 配置后首次成功推送）

---

## 仓库概览

本项目包含 **两个独立 GitHub 仓库**，分别承载不同职责，互不嵌套（multialphaV 根仓库的 `.gitignore` 忽略整个 `RD-Agent/` 子目录）：

| 仓库 | URL | 可见性 | 职责 | 默认分支 |
|---|---|---|---|---|
| **multialphaV** | `git@github.com:seuzxh/multialphaV.git` | 私有 | 项目级文档与规范（CLAUDE.md / PLAN-*.md / .gitignore） | `main` |
| **RD-Agent** | `git@github.com:seuzxh/RD-Agent.git` | 私有（fork） | RD-Agent 代码（裁剪后的 Qlib-only 分支） | `main`（已合并裁剪与修复，含 PR #1 + 直接 merge） |

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
| `cut-non-qlib-scenarios` | `d027d9ca` | **本项目工作分支**：裁剪 + docker 同步 + 一系列 bug 修复（已全部 merge 到 main） |
| `main` | `8e6413a9` | 项目主线：fork 自上游，**已通过 PR #1 + 直接 merge 合入 cut 全部 8 个 commit** |

> **main 跟踪关系**：本地 main 已修正为跟踪 `origin/main`（之前曾错位跟踪 `upstream/main`，已于 2026-07-19 修正）。同步上游走 `git fetch upstream && git rebase upstream/main`（在 cut 分支上做，不在 main 上做）。

### `cut-non-qlib-scenarios` 分支 commit 历史

本分支在 fork 点（`4f9ecb00`）基础上有 **8 个本项目 commit**：

| commit | 时间 | 说明 | 改动量 |
|---|---|---|---|
| `d027d9ca` | 2026-07-19 11:46 | fix(qlib): generate.py debug segment time range (2018-2019 → 2024-2025) | 1 文件 |
| `7a3e98b1` | 2026-07-19 | revert: generate.py time slice back to upstream original (2008-12-29) | 1 文件 |
| `889d26f1` | 2026-07-19 11:39 | fix(model): get_model_env extra_volumes override bug + cleanup .env.example | 2 文件 |
| `6987d33d` | 2026-07-19 | fix(qlib): adapt generate.py time slices to 2020-2026 data range | 1 文件 |
| `ebdc5c9b` | 2026-07-19 | revert: d542dcc3 Docker symlink loop fix (data moved to default path) | — |
| `d542dcc3` | 2026-07-19 | fix(qlib): resolve Docker symlink loop in data mount | — |
| `7cedd780` | 2026-07-18 16:11 | docker(qlib): sync 0.8 project's Dockerfile + lock + README | +268 / -8（4 文件） |
| `8f60d6ea` | 2026-07-18 15:32 | cut: remove non-Qlib scenarios per CLAUDE.md §0/§10 | +26 / -61139（539 文件） |

### main 分支合并历史

main 在 fork 点之后有 **2 个 merge commit**：

| merge commit | 时间 | 说明 |
|---|---|---|
| `8e6413a9` | 2026-07-19 17:47 | 直接 merge cut-non-qlib-scenarios 到 main（`--no-ff`，补齐剩余 6 个 commit） |
| `87050683` | 2026-07-18 16:29 | PR #1：首次 merge cut 到 main（当时只含 2 个 commit：裁剪 + docker 同步） |

**`git describe --tags --always` 当前值**：
- cut 分支：`v0.8.0-36-gd027d9ca`
- main 分支：`v0.8.0-38-g8e6413a9`（比 cut 多 2 个 merge commit）
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

# RD-Agent 仓库 —— 注意：main 现在是项目主线，cut 已全部合入
cd /home/zxh/projects/1.multialphaV/RD-Agent
git push origin main                    # 主线推送
git push origin cut-non-qlib-scenarios  # 工作分支（可与 main 保持同步或独立继续开发）
```

### 同步上游更新（RD-Agent）

> main 已含裁剪 commit，**禁止** `git merge upstream/main` 到 main（会让裁剪 entangled）。上游同步必须走 cut 分支 rebase：

```bash
cd /home/zxh/projects/1.multialphaV/RD-Agent
git fetch upstream
git log --oneline $(git describe --tags --always)..upstream/main   # 看上游新增了什么

# 在 cut 分支上 rebase（不在 main 上）
git checkout cut-non-qlib-scenarios
git rebase upstream/main
# 处理裁剪冲突：上游对 data_science/kaggle/finetune/general_model/rl 的改动一律 drop；
#               对 core/components/scenarios/qlib 的改动保留
# rebase 成功后再 merge 回 main：
git checkout main
git merge --no-ff cut-non-qlib-scenarios
git push origin main
```

### 查看远端同步状态

```bash
# 任一仓库内
git rev-list --count origin/<branch>..HEAD   # 本地领先远端（应为 0）
git rev-list --count HEAD..origin/<branch>   # 远端领先本地（应为 0）
```

### 回滚方案

- **multialphaV 文档误改**：`git checkout <file>` 或 `git reset --hard <最新 commit>`
- **RD-Agent main 出问题**：
  - 撤销本次 merge（回到 PR #1 后状态）：`git reset --hard 87050683`（force push 需团队通告）
  - 撤销所有 multialphaV 改动（回到 fork 点）：`git reset --hard 4f9ecb00`
- **RD-Agent cut 分支出问题**：`git checkout cut-non-qlib-scenarios`，可回退到 `8f60d6ea`（裁剪）或 `4f9ecb00`（fork 点，撤销所有裁剪）

---

## 后续待办（与推送无关，备忘）

- **qlib 数据软链**（PLAN-env-setup 阶段 3，用户暂缓）：需 sudo 执行 `sudo mv /home/zxh/.qlib/qlib_data/cn_data{,.bak.2020} && sudo ln -s /home/zxh/qlib_data /home/zxh/.qlib/qlib_data/cn_data`，详见 PLAN-env-setup.md 阶段 3 说明

---

**版本**：v1.1（2026-07-19 更新：main 合入 cut 全部 8 commit，修正 main 跟踪错位）
**维护策略**：每次有新的推送/分支/远程变更时更新本文档
