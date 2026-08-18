# 09 - 服务层详解：Agent 服务

Agent 服务层为 Alpha 引入"自主诊断与优化闭环"：Hermes Runtime 负责调度与执行，Hermes Memory 负责持久化，MCP Server 把 Alpha 自身 API 暴露为工具供 Agent 调用。

---

## 1. HermesRuntime

**文件位置**：[app/services/hermes_runtime.py](file:///workspace/app/services/hermes_runtime.py)

Hermes 任务调度的核心执行器，支持"Agent 模式 / 降级模式"双模式自动探测，并提供完整的并发 / 熔断 / 超时保护。

### 1.1 双模式自动探测

每次任务执行时（`_do_daily_review / _do_notice_review / _do_full_diagnosis`），先调用 `_check_hermes_agent()` [L184-L192](file:///workspace/app/services/hermes_runtime.py#L184-L192)：

```python
health_url = self._HERMES_AGENT_BASE.replace("/v1", "/health")
# 对 HERMES_AGENT_URL /health 发 GET，返回 200 → Agent 可用
```

| 模式 | 触发条件 | 执行方式 |
|------|---------|---------|
| **Agent 模式** | `_check_hermes_agent() → 200` | 发任务描述给 Hermes Agent → Agent 自主通过 MCP 工具调用 Alpha API → 收集结果 |
| **降级模式** | Agent 不可用（连接失败 / 非 200） | 内部并行 `_collect_observations()` 采 5 类数据 → 单次 `_call_llm()` 让 LLM 直接分析 → 返回结构化建议 |

两种模式对外接口一致（返回 success + task_id + summary + observations），上层无感。

### 1.2 三重保护

#### ① 并发控制：Semaphore(2)

`self._sem = asyncio.Semaphore(2)` [L36](file:///workspace/app/services/hermes_runtime.py#L36)。

最多同时执行 2 个任务（通常是 15:30 daily_review + 21:00 notice_review，刚好错开 1 小时），避免 LLM 请求打爆。

#### ② 熔断：Circuit Breaker

模块常量 [L20-L21](file:///workspace/app/services/hermes_runtime.py#L20-L21)：
```python
_CIRCUIT_BREAKER_THRESHOLD = 3   # 连续失败 3 次
_CIRCUIT_BREAKER_COOLDOWN = 3600 # 冷却 1 小时
```

`_is_circuit_open(task_type)` [L66-L76](file:///workspace/app/services/hermes_runtime.py#L66-L76)：
- 同一 task_type 最近失败时间戳列表
- 失败次数 ≥ 3 且距离最后失败 < 3600s → 熔断打开，新任务直接返回"任务处于熔断状态"
- 成功一次立即 `_clear_failures(task_type)` 清空记录

#### ③ 超时：wait_for(timeout=180s)

`run_task()` [L94-L97](file:///workspace/app/services/hermes_runtime.py#L94-L97)：
```python
result = await asyncio.wait_for(
    self._dispatch(...),
    timeout=_TASK_TIMEOUT,   # 180s
)
```

Agent 模式需要包含"自主工具链调用链"的长超时（MCP 工具 + LLM 思考可能需要几分钟），180s 是保守值。超时后任务状态记为 `timeout`，计入熔断失败计数。

### 1.3 任务类型

| task_type | 默认调度 | 诊断对象 |
|-----------|---------|---------|
| `daily_review` | 15:30 每日 | 漏斗状态 + 策略参数 + K 线同步质量 + 热门概念 → 盘后复盘报告 |
| `notice_review` | 21:00 每日 | 公告选股漏斗 + 关键词命中分布 + 风险词过滤 → 公告复盘建议 |
| `full_diagnosis` | 手动触发 | 以上全部 + 手动指定参数，全量深度诊断 |

调度入口：`async def run_task(task_type, trigger, params)` [L83-L127](file:///workspace/app/services/hermes_runtime.py#L83-L127)，统一走"熔断检查 → 信号量 → wait_for → 状态记录"管线。

### 1.4 数据采集层：_collect_observations()

降级模式使用的数据采集器 [L139-L178](file:///workspace/app/services/hermes_runtime.py#L139-L178)，并行调用 5 个工具（`asyncio.to_thread` 逐个包裹，全并发）：

| 工具名 | 采集内容 |
|--------|---------|
| `get_funnel_snapshot` | 三池数量 + 每只股票 score + stats |
| `get_notice_snapshot` | 公告漏斗三池 + LLM/规则来源标识 |
| `get_strategy_profile` | 当前 StrategyConfig 所有参数当前值 |
| `get_kline_sync_status` | 同步状态 status + 覆盖率 count |
| `get_hot_concepts` | 热门概念 Top10（含涨跌幅 + 热度） |

采集完成后自动聚合 `metrics` 指标卡（字段名简洁，方便 LLM 理解）：
```python
metrics = {
  "funnel_candidate_count": ...,
  "funnel_focus_count": ...,
  "funnel_buy_count": ...,
  "notice_candidate_count": ...,
  "hot_concepts_top3": [...],
  ...
}
```

### 1.5 LLM/Agent JSON 容错

由于 LLM 偶尔输出"解释文字 + JSON"的混合格式或 Markdown 代码块，`_parse_llm_json()` 采取分级兜底：
1. 优先 `json.loads(text)`
2. 失败 → `find("{")` + `rfind("}")` 截取最大花括号片段 → 再 json.loads
3. 仍失败 → 返回 {"raw_summary": text[:500]}，至少保留原始摘要不丢失

### 1.6 System Prompts

#### _AGENT_SYSTEM_PROMPT（Agent 模式用）
见 [L308-L339](file:///workspace/app/services/hermes_runtime.py#L308-L339)，强调：
- **只能建议不能修改**：严禁调用变更接口，只输出建议
- **严禁编造股价**：预测数据必须通过 `predict_kronos` 工具获取
- **参数建议每次 < 20%，最多调 2 个参数**：避免一次性大幅改动导致系统失控
- **主动调用工具**：不要等待数据，自己取数据

同时给出诊断参考（候选池健康范围 5-30 只、buy_score_threshold 默认 78 等），Agent 可以对比判断。

#### _FALLBACK_SYSTEM_PROMPT（降级模式用）
见 [L341-L353](file:///workspace/app/services/hermes_runtime.py#L341-L353)，精简版：
- 同样强调"只能建议不能改"
- 约束"输出必须是严格的 JSON 格式"（降级模式是单轮 LLM 调用，严格 JSON 便于机器解析）

---

## 2. HermesMemory

**文件位置**：[app/services/hermes_memory.py](file:///workspace/app/services/hermes_memory.py)

Hermes 的持久化层，所有表建立在 `funnel_state.db`（共享 KV 存储同一个 SQLite 文件，WAL 模式天然多表安全）。

### 2.1 任务记忆：agent_tasks 表

建表 DDL [L28-L44](file:///workspace/app/services/hermes_memory.py#L28-L44)：

```sql
CREATE TABLE IF NOT EXISTS agent_tasks (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    task_type     TEXT NOT NULL,          -- daily_review / notice_review / full_diagnosis
    trigger       TEXT NOT NULL,          -- scheduled / manual
    status        TEXT NOT NULL,          -- running / success / timeout / failed
    input_summary TEXT,                   -- params JSON
    output_summary TEXT,                  -- LLM 返回 summary JSON
    observations  TEXT,                   -- 采集 observations JSON
    tool_calls    TEXT,                   -- 工具调用链 JSON
    error_message TEXT,                   -- timeout/异常信息
    started_at    TEXT NOT NULL,
    finished_at   TEXT,
    elapsed_ms    INTEGER,
    created_at    TEXT NOT NULL DEFAULT (datetime('now','localtime'))
);
```

核心 CRUD：
- `create_task(task_type, trigger, input_summary) -> int` [L66-L75](file:///workspace/app/services/hermes_memory.py#L66-L75)：插入 running 记录，返回 task_id
- `finish_task(task_id, status, output_summary, observations, tool_calls, error_message, elapsed_ms)` [L77-L98](file:///workspace/app/services/hermes_memory.py#L77-L98)：完成后更新全部字段
- `get_recent_tasks(limit=10) -> list[dict]` [L100-L105](file:///workspace/app/services/hermes_memory.py#L100-L105)：倒序取最近 N 条历史（前端历史面板）
- `get_last_task(task_type=None) -> dict | None` [L107-L118](file:///workspace/app/services/hermes_memory.py#L107-L118)：取最近一次某类型（判断是否当天已跑过）

### 2.2 监控配置：agent_monitor_config 表

智能监控（每 N 分钟跑一次盘中诊断）的配置存储 [L46-L52](file:///workspace/app/services/hermes_memory.py#L46-L52)：

```sql
CREATE TABLE IF NOT EXISTS agent_monitor_config (
    id                INTEGER PRIMARY KEY CHECK (id = 1),  -- 单行配置
    system_prompt     TEXT NOT NULL,                       -- 监控专用 Prompt（覆盖默认）
    interval_minutes  INTEGER NOT NULL DEFAULT 10,         -- 监控周期
    enabled           INTEGER NOT NULL DEFAULT 0,          -- 0=关 1=开
    updated_at        TEXT NOT NULL DEFAULT (datetime('now','localtime'))
);
```

默认 Prompt 在 `_DEFAULT_MONITOR_PROMPT` [L122-L148](file:///workspace/app/services/hermes_memory.py#L122-L148)，输出"最多 3 条主线"的简洁机会雷达格式，每条主线包含【等级】+ 逻辑链 + 关注个股（主板）+ 失效条件。

CRUD：`get_monitor_config()` / `update_monitor_config(system_prompt, interval_minutes, enabled)` — id=1 单行 upsert。

### 2.3 监控消息流：agent_monitor_messages 表

```sql
CREATE TABLE IF NOT EXISTS agent_monitor_messages (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    content     TEXT NOT NULL,                          -- LLM 输出的完整监控消息
    trigger     TEXT NOT NULL DEFAULT 'scheduled',      -- scheduled / manual
    created_at  TEXT NOT NULL DEFAULT (datetime('now','localtime'))
);
```
见 [L54-L60](file:///workspace/app/services/hermes_memory.py#L54-L60)。

前端 `/api/agent/monitor/messages` 拉取最近消息，通过 RealtimeHub `monitor_update` 事件广播到所有在线 WS 客户端。

---

## 3. MCP Server

**文件位置**：[app/mcp_server.py](file:///workspace/app/mcp_server.py)

基于 `FastMCP + stdio` 协议的 MCP 工具服务器。Hermes Agent（或任何 MCP 客户端）启动此脚本后，即可通过标准化 MCP 协议调用 Alpha 的所有 API。

**启动方式**：
```bash
python app/mcp_server.py              # 从 stdin/stdout 读写 MCP 消息
```
`ALPHA_API_BASE` 环境变量控制 Alpha 后端地址（默认 `http://127.0.0.1:18890`）[L16](file:///workspace/app/mcp_server.py#L16)。

### 3.1 16 个工具分类

| 分类 | 工具名 | 对应 Alpha API |
|------|--------|---------------|
| **漏斗** | `get_funnel_snapshot(trade_date)` | `GET /api/funnel` |
| **漏斗** | `get_strategy_profile()` | `GET /api/strategy/profile` |
| **行情** | `get_hot_concepts()` | `GET /api/market/hot-concepts` |
| **行情** | `get_hot_stocks()` | `GET /api/market/hot-stocks` |
| **行情** | `get_stock_detail(symbol, kline_days=30)` | `GET /api/stock/{symbol}/detail` |
| **行情** | `get_stock_realtime(symbol)` | `GET /api/stock/{symbol}/realtime` |
| **K 线** | `get_kline(symbol, days=30)` | `GET /api/kline/{symbol}` |
| **K 线** | `get_kline_sync_status()` | `GET /api/jobs/kline-cache/status` |
| **K 线** | `get_kline_cache_stats()` | `GET /api/jobs/kline-cache/stats` |
| **预测** | `predict_kronos(symbol, lookback=180, horizon=3)` | `GET /api/predict/{symbol}/kronos` |
| **公告** | `get_notice_funnel(trade_date)` | `GET /api/notice/funnel` |
| **公告** | `get_notice_keywords()` | `GET /api/notice/keywords` |
| **公告** | `get_notice_detail(symbol, days=30)` | `GET /api/notice/{symbol}/detail` |
| **执行** | `get_agent_status()` | `GET /api/agent/status` |
| **执行** | `list_agent_tasks(limit=10)` | `GET /api/agent/tasks` |
| **执行** | `trigger_notice_screen(notice_date, limit, keywords)` | `POST /api/notice/screen` |

所有工具通过 `@mcp.tool()` 装饰器注册，完整工具清单见 [FastMCP 文档](https://modelcontextprotocol.io/) + 源码。

### 3.2 内部实现：同源 REST API 调用

所有工具的内部实现完全通过 `httpx.AsyncClient` 调 Alpha 自己的 REST API（同源数据），**不直接访问内部服务实例**：

```python
async def _get(path: str, params: dict | None = None) -> dict:
    async with httpx.AsyncClient(base_url=ALPHA_API_BASE, timeout=30) as client:
        ...

async def _post(path: str, body: dict | None = None) -> dict:
    async with httpx.AsyncClient(base_url=ALPHA_API_BASE, timeout=60) as client:
        ...
```
见 [L27-L38](file:///workspace/app/mcp_server.py#L27-L38)。

设计好处：
- **数据一致性**：MCP 返回值 = HTTP API 返回值，没有第二套逻辑
- **部署灵活**：MCP Server 可以独立部署在任何机器（只要网络能通 Alpha API）
- **安全隔离**：MCP Server 不持有 SQLite 连接，不直接写 DB

### 3.3 超时配置

| 方法 | 超时 | 原因 |
|------|------|------|
| `_get`（GET 请求） | 30s | 大多数数据 API 秒级返回，30s 足够 |
| `_post`（POST 请求） | 60s | 公告筛选 / Kronos 预测 / 诊断触发可能需要长执行 |

`predict_kronos` 工具 [L159-L190](file:///workspace/app/mcp_server.py#L159-L190) 额外做了错误兜底：
- `HTTPStatusError` → 返回 `{error: "预测失败(500): ..."}` 而非抛异常
- 通用 Exception → 返回 `{error: "预测异常: ..."}`
- 保证 Agent 永远能拿到结构化 JSON，不会因单只股票中断整体诊断

`predict_kronos` 返回时附带"人类可读摘要" `prediction_summary` 字段（每行 `YYYY-MM-DD: 开X 高Y 低Z 收W (+/-N%)`），让 Agent 在没有专门图表渲染能力时也能直接看懂预测结果。
