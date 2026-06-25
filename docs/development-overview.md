# 项目功能、技术架构与二次开发指南

本文面向准备在 Daily Stock Analysis（DSA）基础上进行长期二次开发的开发者，集中说明：

- 项目目前能做什么，不能做什么。
- 前端、后端、数据、AI、桌面端使用了哪些技术。
- 开发模式与单端口部署模式之间的关系。
- 本地开发需要安装什么、配置什么、如何启动和验证。
- 如果继续产品化或 SaaS 化，哪些基础设施值得引入，哪些暂时不需要重写。

具体环境变量全集、服务商参数和部署细节仍以 [完整配置与部署指南](full-guide.md)、[LLM 配置指南](LLM_CONFIG_GUIDE.md) 和仓库根目录的 `.env.example` 为准。

## 1. 项目定位

DSA 当前是一个：

> 行情与资讯聚合平台 + AI 投研分析助手 + 自选股监控台 + 投资组合记录与后验评估工具。

系统会聚合行情、历史 K 线、技术面、基本面、新闻、公告、市场环境等信息，生成结构化分析报告、决策信号、风险提示和通知。

它目前不是券商交易终端，也没有完整的实盘下单闭环。报告中的 `buy`、`sell`、买入区间、止损位和目标价属于分析建议；持仓中的交易记录属于本地账本，不代表系统已经向券商提交订单。

## 2. 完整功能概览

### 2.1 多市场行情与数据聚合

当前覆盖：

- A 股、港股、美股和 ETF。
- 日股 `.T`、韩股 `.KS` / `.KQ` 的基础行情和日线分析。
- 指数、大盘环境、板块热点和市场宽度等市场级数据。

可聚合的数据包括：

- 实时或近实时行情。
- 日线、历史 K 线和成交量。
- 均线、MACD、RSI、KDJ、CCI 等技术指标。
- 量价结构、趋势、部分资金流和筹码信息。
- 估值、盈利、成长、财报等基本面。
- 新闻、公告、财经快讯、RSS/Atom 和搜索结果。
- 社交情绪等可选外部情报。

主要数据源适配位于 `data_provider/`。系统存在多数据源优先级、超时、重试和 fallback；不同市场和数据源的字段能力并不完全相同，缺失能力会按 `missing`、`not_supported`、`fallback`、`stale` 等状态降级。

### 2.2 AI 个股分析

用户可以从 Web、CLI、API 或 Bot 触发股票分析。典型输出包括：

- 核心结论、趋势判断和综合评分。
- 买入、加仓、持有、观望、减仓或卖出建议。
- 理想买入区间、目标价和止损位。
- 风险因素、利好催化和最新动态。
- 操作检查清单和观察条件。
- 输入数据质量、缺失项和降级说明。
- 可追踪的结构化 DecisionSignal 决策信号。

分析主流程大致为：

```text
股票代码
  → 行情与历史数据
  → 技术面、基本面、新闻和市场上下文
  → AnalysisContextPack 低敏质量摘要
  → LLM 分析
  → 结构化报告与决策信号
  → 历史记录、Web 展示和通知
```

### 2.3 大盘复盘

系统可以单独或随个股任务生成大盘复盘，覆盖：

- 主要指数表现。
- 上涨、下跌、涨停和跌停概况。
- 板块强弱和热点方向。
- 市场情绪、风险标签和大盘红绿灯。
- 宏观、市场新闻和事件。
- 对个股分析的低敏大盘环境约束。

### 2.4 Agent 策略问股

Web 对话页和相关 API 支持多轮问股。系统包含：

- Single Agent 和 Multi-Agent 两种架构模式。
- Technical、Intel、Portfolio、Risk、Decision 等 Agent 职责。
- 均线、趋势、量价突破、热点题材、事件驱动、成长质量、预期重估、缠论、波浪等内置策略。
- 行情、历史数据、搜索、市场和回测工具。
- 可选对话记忆与上下文。

Agent 工具调用目前主要通过 LiteLLM 路径运行。`codex_cli` generation backend 是显式 opt-in 的本地生成后端，不等同于 Agent 工具调用后端。

### 2.5 AlphaSift 智能选股

启用 AlphaSift 后，可进行：

- 全市场候选扫描。
- 多因子或策略筛选。
- 热点题材与概念发现。
- 候选股票评分。
- 可选 LLM 重排。
- 候选理由、风险和催化因素输出。

该能力默认关闭，需要对应配置和数据源支持。

### 2.6 资讯与情报池

系统可以接入或沉淀：

- 财经 RSS/Atom。
- NewsNow 类型数据源。
- 搜索 API 结果。
- 个股、市场和板块相关资讯证据。
- 资讯去重、保留期限和查询。

资讯可供普通分析、Agent 和大盘复盘 fail-open 复用；单一资讯源失败不应拖垮完整分析流程。

### 2.7 持仓管理

持仓模块支持：

- 创建多个投资账户。
- 手动录入买入、卖出、现金和公司行动。
- 导入 CSV/Excel 等交易记录。
- FIFO 或平均成本等成本计算。
- 持仓数量、市值、盈亏和集中度。
- 多币种和汇率相关估值。
- 回撤、止损和组合风险分析。
- 将最近 AI 决策信号关联到持仓。

这里的交易记录是本地投资账本，不会自动发送到券商。

### 2.8 实时告警

告警模块覆盖：

- 价格突破和涨跌幅。
- 放量和成交量条件。
- 均线穿越。
- RSI、MACD、KDJ、CCI 等指标。
- 持仓止损、集中度和组合回撤。
- 行情价格过期和数据质量。
- 大盘环境变化。

触发后可写入告警记录，并通过已配置的通知渠道发送。

### 2.9 DecisionSignal 决策信号

系统会从分析报告中提取并保存结构化建议，例如：

- 动作和方向。
- 置信度与有效周期。
- 入场区间、止损和目标价。
- 风险、催化因素和失效条件。
- 信号状态、过期时间和来源报告。

用户可以反馈信号是否有用，系统也可以进行后验评估和统计。

### 2.10 回测与后验验证

当前回测更接近“历史 AI 建议后验评估”，可分析：

- 方向准确率和胜率。
- 标的后续实际收益。
- 模拟执行收益。
- 止盈、止损触发情况。
- 单股和整体表现。
- 不同市场阶段下的结果。

它不是面向高频、撮合、滑点和组合优化的专业量化回测引擎。

### 2.11 自动化与通知

运行方式包括：

- CLI 手动执行。
- 本地定时任务。
- Web/API runtime scheduler。
- GitHub Actions。
- Docker。
- WebUI。
- Electron 桌面端。
- Bot 命令。

通知渠道包括企业微信、飞书、Telegram、Discord、Slack、邮件、PushPlus、Server 酱、Pushover、ntfy、Gotify、自定义 Webhook 等。

### 2.12 系统管理与可观测性

Web 管理功能还包括：

- 可选管理员登录认证。
- 系统配置管理。
- 分析任务进度和 SSE 更新。
- 历史报告。
- LLM Token 用量和调用统计。
- 运行诊断和通知诊断。
- 数据质量与分析上下文摘要。

## 3. 用户侧最终体验

典型流程是：

```text
添加自选股或导入持仓
  → 系统聚合行情、技术面、基本面、新闻和事件
  → AI 生成报告与结构化决策信号
  → 用户查看买卖区间、止损、风险和催化
  → 满足条件时收到告警或定时报告
  → 用户自行在券商 App 下单
  → 将成交记录录入或导回系统
  → 系统继续评估持仓风险和历史建议表现
```

因此当前产品形态是信息聚合和决策辅助，不是自动交易。

如果未来增加半自动交易，推荐采用：

```text
AI 决策信号
  → 生成待审批订单
  → 用户确认数量、价格和止损
  → 账户与风控检查
  → Broker Gateway 提交订单
  → 同步订单和成交状态
```

不建议直接从“AI 建议”跳到“无人审批的自动实盘下单”。

## 4. 技术栈

### 4.1 Web 前端

前端目录为 `apps/dsa-web/`。

| 类别 | 当前技术 |
| --- | --- |
| 核心框架 | React 19 |
| 开发语言 | TypeScript 5.9 |
| 构建工具 | Vite 7 |
| CSS | Tailwind CSS 4 |
| 路由 | React Router 7 |
| 状态管理 | Zustand 5 |
| HTTP | Axios，部分流式场景使用 Fetch/SSE |
| 图表 | Recharts |
| 动画 | Motion |
| Markdown | React Markdown + Remark GFM |
| 图标 | Lucide React、Remix Icon |
| 主题 | next-themes |
| 单元测试 | Vitest + Testing Library |
| E2E | Playwright |
| 代码检查 | ESLint 9 |

主要页面路由包括：

- `/`：首页、分析任务和历史报告入口。
- `/chat`：Agent 策略问股。
- `/portfolio`：持仓管理。
- `/decision-signals`：AI 决策信号。
- `/screening`：智能选股。
- `/backtest`：回测与后验。
- `/alerts`：告警中心。
- `/usage`：Token 用量。
- `/settings`：系统配置。
- `/login`：可选管理员登录。

### 4.2 Python 后端

| 类别 | 当前技术 |
| --- | --- |
| 开发语言 | Python 3.10+ |
| Web 框架 | FastAPI |
| ASGI Server | Uvicorn |
| Schema | Pydantic |
| ORM | SQLAlchemy 2 |
| 默认数据库 | SQLite |
| 数据处理 | Pandas、NumPy |
| 定时任务 | schedule + runtime scheduler |
| AI 接入 | LiteLLM、OpenAI SDK、GenerationBackend 抽象 |
| 报告模板 | Jinja2 |
| 配置 | `.env` + python-dotenv |
| API | REST，统一 `/api/v1` 前缀 |
| 认证 | 可选管理员 Cookie Session |

主要分层：

```text
API Endpoint
  → Service 业务服务
  → Repository / Storage 数据访问
  → SQLAlchemy / SQLite
```

关键目录：

| 目录 | 职责 |
| --- | --- |
| `api/v1/endpoints/` | FastAPI 控制器 |
| `api/v1/schemas/` | API 请求与响应模型 |
| `src/core/` | 分析流程、调度和核心编排 |
| `src/services/` | 业务服务 |
| `src/repositories/` | 数据访问 |
| `src/schemas/` | 内部 Schema |
| `src/agent/` | Agent、策略和工具 |
| `src/llm/` | LLM generation backend |
| `src/reports/` | 报告生成 |
| `data_provider/` | 行情和基本面数据源适配 |
| `src/notification_sender/` | 通知渠道 |
| `bot/` | Bot 接入 |
| `tests/` | pytest 测试 |

### 4.3 数据与 AI

现有行情或基本面适配包括 EFinance、AkShare、Tushare、Pytdx、Baostock、YFinance、Longbridge、TickFlow、Tencent、Finnhub 和 AlphaVantage 等。

LLM 默认通过 LiteLLM 统一路由，可配置 Gemini、Anthropic、OpenAI、DeepSeek、Ollama 和多种 OpenAI-compatible 服务。仓库还提供：

- `GenerationBackend` 抽象。
- `litellm` generation backend。
- 显式 opt-in 的 `codex_cli` 本地 CLI generation backend。
- generation fallback、超时、输出上限和并发配置。

### 4.4 桌面端

桌面端目录为 `apps/dsa-desktop/`，技术栈为 Electron + electron-builder。桌面包会携带构建后的 Web 静态资源和 Python 后端产物。

## 5. 前后端关系与端口

项目在代码上是前后端分离的，在默认部署方式上采用单服务入口。

### 5.1 前端开发模式

```text
浏览器
  → React/Vite :5173
  → /api 代理
  → FastAPI :8000
```

此时：

- Vite 提供热更新。
- React 源码由 `5173` 提供。
- `/api` 自动代理到 `http://127.0.0.1:8000`。
- FastAPI 只负责 API 和后端任务。

### 5.2 整合运行或默认部署模式

```text
浏览器
  → FastAPI :8000
      ├── /api/*：后端接口
      └── /*：React 构建后的 static/ 静态文件
```

这里不是把 `5173` 转发成 `8000`。执行 `npm run build` 后，Vite 把前端输出到仓库根目录 `static/`，FastAPI 直接托管该目录；运行时不再需要 Vite，也不会监听 `5173`。

这种方式适合当前阶段，因为部署简单、只需一个端口、默认没有跨域问题，并且不影响前后端源码独立开发。

未来需要多实例、CDN 或独立发布时，可以改为：

```text
CDN / Nginx
  ├── /：React 静态站点
  └── /api：FastAPI 多实例
```

无需因此重写 React 工作台。

## 6. 本地开发环境

### 6.1 必需软件

建议安装：

- Git。
- Python 3.10 或更高版本，推荐使用项目当前已验证的 Python 3.12。
- Node.js `>=20.19.0 <27`。
- npm 10 或更高版本。

按功能选装：

- Docker / Docker Compose：容器开发和部署。
- `wkhtmltopdf`：部分 Markdown 转图片能力。
- Ollama：运行本地模型。
- Electron 构建所需平台工具：桌面端开发和打包。

### 6.2 初始化 Python 环境

在仓库根目录执行：

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Windows PowerShell 激活方式：

```powershell
.\.venv\Scripts\Activate.ps1
```

### 6.3 初始化前端环境

```bash
cd apps/dsa-web
npm ci
```

若本地没有 lockfile 对应的完整安装条件，可使用 `npm install`，但日常验证和 CI 对齐优先使用 `npm ci`。

### 6.4 创建本地配置

```bash
cp .env.example .env
```

`.env` 包含密钥和本地配置，不应提交到 Git。

## 7. 最小配置

### 7.1 股票列表

```env
STOCK_LIST=600519,300750,002594
```

也可以在运行时使用 `--stocks` 临时覆盖。

### 7.2 至少一个 LLM

推荐从 `.env.example` 选择一种已支持的配置方式，不要同时填一堆互相冲突的模型配置。

DeepSeek 示例：

```env
DEEPSEEK_API_KEY=your-key
```

OpenAI-compatible 示例：

```env
OPENAI_API_KEY=your-key
OPENAI_BASE_URL=https://your-provider.example/v1
LITELLM_MODEL=openai/your-model
```

Ollama 示例：

```env
OLLAMA_API_BASE=http://localhost:11434
LITELLM_MODEL=ollama/qwen3:8b
```

多渠道和模型 fallback 请使用 `LLM_CHANNELS` 系列配置，详见 [LLM 配置指南](LLM_CONFIG_GUIDE.md)。

### 7.3 可选数据源

按需要配置：

- `TUSHARE_TOKEN`
- `TICKFLOW_API_KEY`
- Longbridge OAuth 或兼容旧凭证
- `FINNHUB_API_KEY`
- `ALPHAVANTAGE_API_KEY`

不配置时系统会尽量使用不需要密钥的数据源和 fallback，但字段完整度、速度和稳定性可能不同。

### 7.4 可选新闻搜索

可选配置：

- `ANSPIRE_API_KEYS`
- `SERPAPI_API_KEYS`
- `TAVILY_API_KEYS`
- `BOCHA_API_KEYS`
- `BRAVE_API_KEYS`
- `MINIMAX_API_KEYS`
- `SEARXNG_BASE_URLS`

新闻源会明显影响事件、公告、催化和风险分析质量。

### 7.5 可选通知

按需要配置企业微信、飞书、Telegram、Discord、Slack、邮件或其他渠道。具体字段见 [通知能力基线](notifications.md) 和 `.env.example`。

### 7.6 可选管理员认证

默认：

```env
ADMIN_AUTH_ENABLED=false
```

如果服务监听公网地址，建议启用认证，并同时检查反向代理、Cookie、安全来源和 CORS 配置。

## 8. 启动方式

以下命令均在仓库根目录执行，并假设 Python 虚拟环境已经激活。

### 8.1 单端口 WebUI + API

```bash
python main.py --serve-only
```

默认访问：

- WebUI：`http://127.0.0.1:8000`
- API 文档：`http://127.0.0.1:8000/docs`
- 健康检查：`http://127.0.0.1:8000/api/health`

如果 `static/` 不存在，`main.py --serve-only` 会尝试准备前端资源；也可以先手动构建：

```bash
cd apps/dsa-web
npm run build
cd ../..
python main.py --serve-only
```

### 8.2 前后端独立开发

终端一，启动后端：

```bash
python main.py --serve-only
```

终端二，启动前端：

```bash
cd apps/dsa-web
npm run dev
```

访问 `http://127.0.0.1:5173`。前端会把 `/api` 代理到后端 `8000`。

这是修改 React 页面时推荐的开发方式。

### 8.3 直接启动 Uvicorn

```bash
uvicorn server:app --reload --host 0.0.0.0 --port 8000
```

该方式适合只调试 API；如果前端静态资源未构建，根页面会显示构建提示。

### 8.4 分析任务

```bash
python main.py
python main.py --debug
python main.py --dry-run
python main.py --stocks 600519,hk00700,AAPL
python main.py --market-review
python main.py --no-market-review
python main.py --no-notify
python main.py --force-run
python main.py --workers 5
```

### 8.5 Web 服务与分析一起运行

```bash
python main.py --serve
```

`--serve` 会启动 Web/API，并按当前参数执行分析；`--serve-only` 只启动服务，不自动执行分析。

### 8.6 定时任务

```bash
python main.py --schedule
python main.py --schedule --no-run-immediately
python main.py --serve --schedule
```

Web/API 长运行模式下，runtime scheduler 会接管调度状态；详细语义见 [完整配置与部署指南](full-guide.md)。

### 8.7 桌面端

桌面端依赖后端打包产物，完整流程见 [桌面端打包说明](desktop-package.md)。仅开发 Electron 壳时可进入 `apps/dsa-desktop/` 执行：

```bash
npm install
npm run dev
```

## 9. 开发与验证

### 9.1 后端

优先执行：

```bash
./scripts/ci_gate.sh
```

常用的局部验证：

```bash
python -m pytest -m "not network"
python -m py_compile path/to/changed_file.py
```

### 9.2 Web 前端

```bash
cd apps/dsa-web
npm ci
npm run lint
npm run test
npm run build
```

需要浏览器级验证时：

```bash
npm run test:smoke
```

### 9.3 桌面端

先完成 Web 构建，再按平台执行后端和 Electron 构建。桌面端自身测试：

```bash
cd apps/dsa-desktop
npm install
npm run test
npm run build
```

### 9.4 文档与 AI 协作资产

修改 `AGENTS.md`、`.github` AI 指令或 `.claude/skills/` 时执行：

```bash
python scripts/check_ai_assets.py
```

## 10. 当前架构边界

### 10.1 数据库

当前默认数据库是 SQLite。虽然使用 SQLAlchemy，但代码包含：

- 固定生成 SQLite URL 的配置逻辑。
- `sqlite_insert`。
- WAL 和 `busy_timeout`。
- SQLite 写入重试和锁错误处理。
- SQLite 专属索引和轻量 schema migration 逻辑。

因此目前不能把连接字符串改成 PostgreSQL 就认为迁移完成。

### 10.2 Redis 与任务系统

当前仓库没有把 Redis、Celery、Dramatiq 或 RQ 作为运行依赖。任务队列、任务状态、调度和部分缓存仍以单进程或本地持久化思路为主。

### 10.3 Next.js

当前登录后工作台不依赖 SEO 或 SSR，React + Vite 更轻、更适合复用到单端口部署和 Electron。没有必要为了技术栈形式重写成 Next.js。

如果未来需要官网、公开报告分享页、内容门户或 SEO，可以新增独立 Next.js 应用：

```text
marketing-web：Next.js，官网和公开内容
dashboard-web：现有 React + Vite，登录后工作台
api：FastAPI
worker：Python Worker
```

### 10.4 交易执行

当前没有完整的：

- 券商订单创建和撤单。
- 订单状态与成交回报同步。
- 账户资金和购买力检查。
- 实盘风控。
- 订单幂等。
- 人工审批流。
- 统一 Broker Adapter / Gateway。

Longbridge 等现有接入不能据此解释为系统已支持实盘交易。

## 11. 产品化二开建议

如果目标只是个人本地使用，保留 SQLite 和单端口部署最省心。

如果目标是多人使用、云端 SaaS 或长期产品化，建议分阶段演进。

### Phase 1：保持现有产品闭环稳定

- 固化本地和测试环境。
- 梳理核心用户流程与配置。
- 补齐关键 API、任务、报告和数据源测试。
- 保留 React + Vite 和单端口部署。

### Phase 2：PostgreSQL

- 新增统一 `DATABASE_URL`。
- 引入 PostgreSQL 驱动和连接池配置。
- 引入 Alembic。
- 抽象 SQLite/PostgreSQL upsert。
- 清理 SQLite 专属 migration 与索引逻辑。
- 提供 SQLite → PostgreSQL 数据迁移和回滚脚本。

### Phase 3：Redis + Worker

- Redis 用于短时缓存、任务状态、分布式锁、限流和事件广播。
- 使用 Dramatiq、Celery 或 RQ 执行耗时分析。
- Web/API 与分析 Worker 分离。
- 解决多实例任务重复、状态丢失、SSE 跨实例不可见和定时任务重复执行。

目标结构：

```text
React + Vite
  → FastAPI API
  → Redis Queue / Event Bus
  → Analysis Worker / Alert Worker
  → PostgreSQL
```

### Phase 4：用户、组织与多租户

- 用户、组织、角色和权限。
- 自选股、报告、持仓、告警和配置的数据隔离。
- 审计日志。
- 配额、计费和模型成本边界。
- 多租户任务限流。

### Phase 5：半自动交易

- Broker Adapter / Gateway。
- 待审批订单。
- 风控和购买力检查。
- 用户二次确认。
- 订单幂等。
- 订单、撤单和成交状态同步。
- 模拟盘与实盘隔离。
- 完整审计与紧急停止机制。

优先顺序建议为：

```text
PostgreSQL
  → Redis + Worker
  → 用户/组织权限
  → 多租户数据隔离
  → 半自动订单审批
```

不要在数据库、任务一致性、权限和审计尚未稳定时直接接入自动实盘下单。

## 12. 继续阅读

- [完整配置与部署指南](full-guide.md)
- [LLM 配置指南](LLM_CONFIG_GUIDE.md)
- [LLM 服务商配置指南](llm-providers.md)
- [通知能力基线](notifications.md)
- [AnalysisContextPack 专题](analysis-context-pack.md)
- [DecisionSignal 专题](decision-signals.md)
- [实时告警中心](alerts.md)
- [贡献指南](CONTRIBUTING.md)
- [桌面端打包说明](desktop-package.md)
