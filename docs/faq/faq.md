# 常见问题（FAQ）

> 整理 multialphaV 项目开发过程中遇到的常见问题和解答。

---

## Q1: rdagent 版本号怎么获取？

`rdagent.__version__` 不存在（setuptools-scm 动态生成不注入包级属性）。正确方式：

```bash
# 方式 1：git describe（推荐，含 fork 点信息）
cd RD-Agent && git describe --tags --always
# 输出示例：v0.8.0-30-g7cedd780

# 方式 2：pip 元数据
python -c "from importlib.metadata import version; print(version('rdagent'))"
# 输出示例：0.8.1.dev29
```

详见 [CLAUDE.md §6.4](../../CLAUDE.md)。

## Q2: Python 版本选 3.10 还是 3.12？

**选 3.10**。官方 CI 只测 3.10/3.11，constraints 只有这两个版本的锁定文件，0.8 项目用 3.10.20。选 3.10 实现三方一致。

## Q3: RDAGENT_MARKET 环境变量能改股票池吗？

**不能**。`RDAGENT_MARKET` 是孤儿变量（代码零引用）。market 值（csi300）硬编码在 5 个 yaml 模板的 YAML 锚点 `&market csi300`。要换股票池只能改 yaml。

详见 [ENV.md](../reference/ENV.md) §5.1。

## Q4: 回测日期怎么配置？

通过 `.env` 的 18 个 `QLIB_*` 变量（`QLIB_FACTOR_TRAIN_START` 等），三个场景各自 6 个。默认值假设 2008-2020 数据，需按实际数据范围调整。

当前配置（数据 2020-2026）：
```
train: 2020-01-01 ~ 2022-12-31
valid: 2023-01-01 ~ 2024-06-30
test:  2024-07-01 ~ 2026-07-17
```

详见 [ENV.md](../reference/ENV.md) §2 回测日期分段。

## Q5: generate.py 的调试样本可以配置吗？

**不能**。股票数（100）和时间段（2024-2025）硬编码在 generate.py 里，不读环境变量。官方设计如此。

详见 [数据流架构](../architecture/data-flow.md) §2 调试样本设计。

## Q6: 为什么 embedding 报"模型不存在"？

根因:litellm 按 **model 前缀**路由凭证。`openai/xxx` 前缀会让它用 `OPENAI_API_KEY`/`OPENAI_API_BASE`,若这套凭证指向的 provider 没有该 embedding 模型(比如 chat 凭证指 OpenAI,embedding 想调 doubao),就报"模型不存在"。

两种修复方式:
- **方式 A(chat + embedding 同 provider,推荐)**:chat 和 embedding 用同一个 provider 的凭证(本项目现状:均在火山方舟),直接 `EMBEDDING_MODEL=openai/doubao-embedding-vision` 复用 `OPENAI_API_KEY/BASE` 即可。
- **方式 B(chat + embedding 跨 provider)**:改 `litellm_proxy/` 前缀 + 单独设 `LITELLM_PROXY_API_KEY/BASE` 指向 embedding 的 provider。

详见 [数据流架构](../architecture/data-flow.md) §4 LLM 调用架构的"历史教训"段。

## Q7: Docker 报 "Symlink loop" 怎么解决？

根因：宿主软链 + Dockerfile 软链 + extra_volumes 挂载三者叠加。解决方案：数据直接复制到 `~/.qlib/qlib_data/cn_data`（真实目录），不用软链。

详见 [数据流架构](../architecture/data-flow.md) §3 历史教训。

## Q8: qlib 数据应该放在哪里？

官方默认路径 `~/.qlib/qlib_data/cn_data`（三个核心子目录：calendars/instruments/features）。不要放其他路径——rdagent 代码全部硬编码读这个路径。

数据维护项目在 `/home/zxh/qlib_data`（含 scripts/.git/.ifind_token），用于下载/更新数据；更新后复制三个核心子目录到 `~/.qlib/qlib_data/cn_data`。

## Q9: FACTOR_CoSTEER_PYTHON_BIN 配了但没用？

`get_factor_env()` 当前实际改读 `CONDA_DEFAULT_ENV`，`FACTOR_CoSTEER_PYTHON_BIN` 未被消费。保留此行作为意图标注。详见 [ENV.md](../reference/ENV.md) §5.2。

## Q10: git 推送 https 报 TLS 错误怎么办？

github.com 的 https 在某些网络环境下 TLS 握手被干扰（`gnutls_handshake() failed`）。改用 ssh 协议（22 端口不受影响）：

```bash
git remote set-url origin git@github.com:seuzxh/xxx.git
```

需要先在 GitHub 添加 ssh 公钥。详见 [REPOS.md](../reference/REPOS.md) SSH 配置章节。
