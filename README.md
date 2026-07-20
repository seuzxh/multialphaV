# multialphaV

> 基于 RD-Agent（最新 main 分支）+ Qlib 的量化金融因子挖掘平台。
> 裁剪非 Qlib 场景，专注 fin_quant / fin_factor / fin_model / fin_factor_report 四个场景。

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10-green.svg)](https://www.python.org/)
[![rdagent](https://img.shields.io/badge/rdagent-0.8.1.dev29-orange.svg)](https://github.com/microsoft/RD-Agent)
[![qlib](https://img.shields.io/badge/qlib-0.9.7-red.svg)](https://github.com/microsoft/qlib)

---

## 快速开始

```bash
# 1. 创建 conda env
conda create -n multialphav python=3.10 -y && conda activate multialphav
cd RD-Agent && pip install -e .

# 2. 配置 .env（参考 docs/reference/ENV.md）
cp .env.example .env  # 编辑填入 API key

# 3. 运行因子挖掘
rdagent fin_factor --step-n 1

# 4. 运行模型挖掘
rdagent fin_model --step-n 1

# 5. 运行因子+模型联合
rdagent fin_quant --step-n 1

# 6. 研报复现
rdagent fin_factor_report --report-folder <PDF目录>
```

## 项目结构

```
multialphaV/
├── CLAUDE.md                    # Agent 行为约束（强制规范，优先级最高）
├── README.md                    # 本文件（项目入口）
├── RD-Agent/                    # RD-Agent 代码（独立 git 仓库，fork of microsoft/RD-Agent）
│   ├── rdagent/                 # 主代码（裁剪为 Qlib-only）
│   ├── rdagent/app/qlib_rd_loop/# 四个场景 CLI 入口
│   └── rdagent/scenarios/qlib/  # Qlib 场景实现
├── docs/                        # 项目文档（本目录）
│   ├── guide/                   # 指南（环境搭建、验证记录）
│   ├── reference/              # 参考资料（API、环境配置、仓库结构）
│   ├── architecture/           # 架构设计（数据流、执行流程）
│   ├── faq/                    # 常见问题
│   └── COLLABORATION.md        # 多人协作规范
└── .env                         # 环境配置（gitignored，含 API key）
```

## 文档导航

### 指南（Guide）
- [环境搭建计划](docs/guide/PLAN-env-setup.md) — conda env / .env / qlib 数据 / Docker / git remote 完整搭建步骤
- [四场景验证记录](docs/guide/VERIFICATION-log.md) — factor / model / quant / factor_from_report 真实执行验证

### 参考资料（Reference）
- [API 参考](docs/reference/API.md) — CLI 命令 / HTTP API / Python 库 API / 配置接口
- [环境配置说明](docs/reference/ENV.md) — .env 全量字段、加载机制、死配置清单
- [双仓库结构](docs/reference/REPOS.md) — multialphaV 根仓库 + RD-Agent fork 的关系、SSH 推送通道
- [多人协作规范](docs/COLLABORATION.md) — 分支策略 / 环境一致性 / Key 管理 / review 流程

### 架构设计（Architecture）
- [数据流与执行架构](docs/architecture/data-flow.md) — generate.py 数据预处理 / Docker 执行 / CoSTEER 迭代 / LLM 调用

### 常见问题（FAQ）
- [FAQ](docs/faq/faq.md) — 版本获取 / Python 选择 / market 配置 / embedding bug / Docker 死循环等

### 核心规范
- [CLAUDE.md](CLAUDE.md) — **Agent 行为约束**（开发前必读，优先级最高）

## 仓库关系

| 仓库 | 职责 | 远程 |
|---|---|---|
| multialphaV（根） | 项目文档 + 规范 | `git@github.com:seuzxh/multialphaV.git` |
| RD-Agent（fork） | 代码（裁剪为 Qlib-only） | `git@github.com:seuzxh/RD-Agent.git` |

详见 [REPOS.md](docs/reference/REPOS.md)。

## 技术栈

| 组件 | 版本 | 说明 |
|---|---|---|
| Python | 3.10 | 与官方 CI / 0.8 项目 / constraints 一致 |
| rdagent | 0.8.1.dev29 | fork 自 microsoft/RD-Agent main |
| qlib | 0.9.7 | Docker 容器内运行 |
| LLM | glm-5.2 | 智谱 AI |
| Embedding | doubao-embedding-vision | 火山方舟 |
| Docker | local_qlib:v2.1 | 10.7GB，8 卡 H20 GPU |
| pydantic | 2.x | rdagent 依赖 |

## 开发约束

所有开发工作必须遵守 [CLAUDE.md](CLAUDE.md) 的行为约束（§0-§12），包括：
- 场景禁区（只做 Qlib）
- 异构实现识别
- CodeGraph 红线
- 改后审查清单（关联影响 + 文档同步）
- 多 Agent 对抗性审查触发条件

## License

MIT（继承自 RD-Agent 上游）
