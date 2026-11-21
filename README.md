# DataWeaver 🧵

> 把散落在 SaaS、CSV、数据库里的数据织成一张网，然后用大白话问它。

**DataWeaver = 个人 / 小团队数据中枢 + 自然语言查询引擎。**
多源接入 → 本地数仓 → 语义层定义口径 → 自然语言问答 → 表格与图表。

[![CI](https://img.shields.io/badge/CI-coming%20soon-lightgrey)](./.github/workflows)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/downloads/)
[![Status: MVP](https://img.shields.io/badge/status-v0.1.0--MVP-orange)](./SPEC.md)

---

## 它能做什么

> 用户：**"这个月我在吃饭上花了多少，跟上个月比怎么样？"**
>
> DataWeaver：返回对比表格 + 柱状图，并附上
> *"口径：`category in ('餐饮','外卖','咖啡')`，不含转账；SQL 见下方可复核。"*

**核心能力**

- **多源接入**——CSV / Excel / GitHub / Notion / 数据库直连，可插拔连接器架构。
- **本地数仓**——DuckDB 单文件存储，零运维，数据留在你自己的机器上。
- **语义层即代码**——指标口径用 YAML 明文定义，可 review、可 diff、可版本化。
- **自然语言问答**——基于 LangGraph 的多轮状态机，支持追问澄清与失败自愈。
- **五层安全护栏**——sqlglot AST 白名单 + 只读连接 + 成本预估 + dry-run + 行级权限，**默认拒绝一切写操作**。
- **模型可热切换**——OpenAI / 腾讯混元 / Ollama 本地模型，改 4 行 `.env` 即可切换，零代码变更。
- **完全可解释**——每次回答都给出生成的 SQL、命中的指标、护栏报告。

## 架构

```text
                     ┌──────────────────────────────────┐
                     │        Streamlit Frontend        │
                     │   chat / connectors / metrics    │
                     └────────────────┬─────────────────┘
                                      │ HTTP + JSON
                     ┌────────────────▼─────────────────┐
                     │            FastAPI                │
                     │  chat · connectors · datasets ·   │
                     │  metrics · health                 │
                     └───┬───────────┬───────────┬──────┘
                         │           │           │
      ┌──────────────────▼┐  ┌───────▼────────┐  │
      │  LangGraph Agent   │  │ Semantic Layer │  │
      │  intent→retrieve→  │  │ YAML → Pydantic│  │
      │  generate→validate │  │  → SQL Compiler│  │
      │  →execute→format   │  └───────┬────────┘  │
      └─┬────────────┬─────┘          │           │
        │            │                │           │
 ┌──────▼─────┐ ┌────▼───────┐ ┌──────▼─────────┐ │
 │ Retrieval  │ │ Guardrails │ │    Storage      │ │
 │  Qdrant    │ │  sqlglot   │ │ warehouse(DuckDB)│ │
 │ 召回/重排   │ │ 成本/权限   │ │  vector_store    │ │
 └──────┬─────┘ └────────────┘ └──────┬──────────┘ │
        │                             │            │
 ┌──────▼─────┐              ┌────────▼─────────┐  │
 │   Qdrant   │              │   Meta + DuckDB  │◄─┘
 └────────────┘              └────────▲─────────┘
                                      │
                  ┌───────────────────┴────────────────────┐
                  │  Connectors: BaseConnector + registry  │
                  │  CSV / GitHub / Notion / SQL (read-only)│
                  └───────────────────┬────────────────────┘
                                      │
                  CSV · GitHub API · Notion API · PostgreSQL/MySQL
```

## 快速开始

### 方式一：Docker Compose（推荐）

```bash
git clone https://github.com/your-org/dataweaver.git
cd dataweaver
cp .env.example .env          # 按需填写 LLM / Embedding 配置
docker compose up -d --build
```

- 前端：http://localhost:8501
- API 文档：http://localhost:8000/docs
- Qdrant 控制台：http://localhost:6333/dashboard

### 方式二：本地开发

```bash
python -m venv .venv && . .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -e ".[dev,frontend]"
docker compose up -d qdrant                    # 本地只起向量库

cp .env.example .env
uvicorn dataweaver.main:app --reload           # 终端 1：后端
streamlit run frontend/app.py                  # 终端 2：前端
```

### 5 分钟跑通第一条查询

1. 打开前端 → **Connectors** 页 → 上传一个记账 CSV（字段需含 `occurred_at, amount, category, direction`）。
   > 没有现成文件？`semantic/examples/` 与 `eval/golden_queries.json` 描述了期望的数据结构，自行造 20 行即可。
2. 切到 **Metrics** 页 → 确认已加载 `personal_finance` 语义模型（示例在 `src/dataweaver/semantic/examples/personal_finance.yaml`）。
3. 切到 **Chat** 页 → 输入：**"这个月我在吃饭上花了多少？"**
4. 展开 *SQL* 折叠区查看生成语句，查看护栏报告。

> 用 Ollama 本地模型时，`DW_LLM_PROVIDER=ollama`、`DW_LLM_BASE_URL=http://localhost:11434/v1`、`DW_LLM_MODEL=qwen2.5:7b`。

## 项目结构

```text
dataweaver/
├── SPEC.md                  # 产品规格（唯一事实来源，改动前先读它）
├── TASKS.md                 # 任务拆解（P0/P1/P2）
├── docs/adr/                # 架构决策记录
├── src/dataweaver/
│   ├── api/                 # FastAPI 路由层
│   ├── agent/               # LangGraph 状态机与节点
│   ├── connectors/          # 数据接入（BaseConnector + registry）
│   ├── semantic/            # 语义层：模型 / 解析 / 编译器 / 示例
│   ├── guardrails/          # 安全护栏：AST 校验 / 成本 / 权限
│   ├── retrieval/           # Schema Linking：向量化 / 索引 / 召回
│   ├── llm/                 # LLM 抽象层（OpenAI / 混元 / Ollama）
│   ├── storage/             # DuckDB 数仓 + Qdrant 向量库封装
│   └── utils/               # 日志与异常体系
├── frontend/                # Streamlit 前端
├── tests/                   # pytest 单测
└── eval/                    # 评测集与 runner
```

## 文档

| 文档 | 内容 |
| --- | --- |
| [SPEC.md](./SPEC.md) | 产品规格：用户场景、数据模型、API、安全模型、非功能需求 |
| [TASKS.md](./TASKS.md) | 任务清单：P0 核心链路 → P1 体验 → P2 扩展 |
| [docs/adr/](./docs/adr/README.md) | 架构决策记录：为什么是 LangGraph / 语义层 / 五层护栏 / 连接器抽象 / LLM 抽象 |

## 开发

```bash
ruff check .           # lint
ruff format .          # 格式化
mypy src               # 类型检查
pytest -q              # 单测
pytest --cov=dataweaver
```

## 安全说明

- DataWeaver **只执行 `SELECT` / `WITH`**，任何写操作（`INSERT/UPDATE/DELETE/DDL/COPY/ATTACH`）在执行前即被拒绝。
- 查询连接以**只读模式**打开；接入外部数据库请务必使用数据库侧的只读账号。
- 所有 SQL 在真正执行前经过 sqlglot AST 白名单、表列白名单、成本预估与 `EXPLAIN` dry-run 检查。
- 凭据不落库：`Connector.config` 只存非敏感项，密钥通过 `secret_ref` 指向环境变量。
- 发现安全问题请直接开私有 issue 联系维护者（独立的安全响应流程规划中）。

## 贡献指南

1. **先读 [SPEC.md](./SPEC.md)**——它是唯一事实来源。要改行为，先改规格。
2. 在 [TASKS.md](./TASKS.md) 找到或新增一条任务，把 `[ ]` 改成 `[~]` 并署上你的 ID。
3. 涉及架构层面的改动，必须新增一条 ADR 并说明备选方案与取舍。
4. PR 要求：类型注解 + docstring（Google 风格）+ 至少 1 个 pytest 用例；`ruff check` 与 `mypy src` 必须通过。
5. 分支命名 `feat/xxx` / `fix/xxx`，提交信息遵循 Conventional Commits。
6. 安全相关代码（`guardrails/`、`semantic/compiler.py`、连接器路径处理）需要额外一位维护者 review。

## 路线图

- **v0.1.0（当前）**：CSV 接入 → 语义层 → 自然语言问数 → 表格/图表，Docker Compose 一键起。
- **v0.2**：更多连接器、定时同步、结果缓存、评测闭环。
- **v0.3**：RBAC、多租户、语义层 Git 工作流、可观测性。
- **v1.0**：插件生态、MCP Server 模式、执行器池、本地模型深度适配。

详见 [SPEC.md §9](./SPEC.md#9-演进路线)。

## License

[MIT](./LICENSE) © 2026 DataWeaver Contributors
