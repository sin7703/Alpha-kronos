# Alpha Code Wiki 综合文档

> 本文件为 16 篇分章合并版，分章版见同目录 01-架构总览.md ~ 16-测试体系.md。
>
> 本 Wiki 面向**开发者与维护者**，深度讲解 Alpha 量化选股系统的代码实现原理、模块关系与扩展方式。若你想了解产品功能与使用方法，请先阅读项目根目录的 [README.md](file:///workspace/README.md)。

---

## 阅读建议路径

推荐阅读顺序（从宏观到微观）：

```
第1章 架构总览 → 第2章 入口与生命周期 → 第5章 数据层设计
                        ↓
              第3章 配置系统 · 第4章 数据模型
                        ↓
         第6/7/8/9/10章 服务层详解（按需选读）
                        ↓
              第11章 路由层 · 第12章 前端架构
                        ↓
         第14章 扩展开发指南 · 第15章 调试与运维 · 第16章 测试体系
```

---

# 第一部分 架构与基础

---

## 第1章 架构总览

**本节导读**：本章从最高视角俯瞰 Alpha 系统全貌——明确项目定位是什么、整体分为几层、有哪些核心服务、后台有多少个调度循环在跑、以及一次用户操作的数据流转路径。读完本章后，你将能在脑中建立完整的系统全景图，为后续深入各章节打下基础。

### 1.1 项目定位

**Alpha = A股自进化量化选股系统（规则 + AI预测 + Agent闭环）**

Alpha 不只是一个选股筛选工具——通过集成 **Kronos 金融 K 线基础模型** 和 **Hermes Agent 自进化智能体**，Alpha 能够自主观察市场、分析主线、输出结构化诊断与交易建议，形成 **观察 → 思考 → 校验 → 进化** 的持续优化闭环。

---

### 1.2 分层架构图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            Alpha 分层架构                                │
├──────────────────────────────────────────────────────────────────────────┤
│  前端层                                                                   │
│    原生HTML/CSS/JS + ECharts 5 · 8 Tab 单页应用 · Glassmorphism 暗色主题 │
│    10s 轮询 + WebSocket 双通道推送                                       │
├──────────────────────────────────────────────────────────────────────────┤
│  接口层（REST / WS / MCP）                                               │
│    FastAPI REST API（~50 endpoints）                                     │
│    WebSocket /ws/realtime（snapshot + monitor_update）                  │
│    MCP Server（stdio，20+ tools → Hermes Agent）                        │
├──────────────────────────────────────────────────────────────────────────┤
│  策略 / AI / Agent 服务层                                                │
│    ┌──────────────┐ ┌───────────────┐ ┌──────────────┐                  │
│    │ FunnelService│ │ NoticeService │ │StrategyEngine│                  │
│    │ 三池漏斗管理  │ │ 公告抓取+打分  │ │ 盘中实时评分  │                  │
│    └──────────────┘ └───────────────┘ └──────────────┘                  │
│    ┌──────────────────┐ ┌───────────────────┐ ┌──────────────────────┐   │
│    │KronosPredictService│ │PredictFunnelService│ │  HotStockAIService  │   │
│    │  Kronos三日预测   │ │  概念×预测选股     │ │  热门股智能分析      │   │
│    └──────────────────┘ └───────────────────┘ └──────────────────────┘   │
│    ┌──────────────────┐ ┌───────────────────┐ ┌──────────────────────┐   │
│    │  HermesRuntime   │ │   HermesMemory    │ │ PaperTradingService  │   │
│    │ Agent调度+LLM推理 │ │ 任务/监控记忆持久化│ │   模拟交易引擎       │   │
│    └──────────────────┘ └───────────────────┘ └──────────────────────┘   │
│    ┌──────────────────┐ ┌───────────────────┐ ┌──────────────────────┐   │
│    │FirstLimitAlphaSvc│ │QuietBreakoutScanner│ │CustomStrategyScanner│   │
│    │ 首板Alpha图形选股 │ │ 缩量启动形态扫描   │ │ 自定义规则组合扫描   │   │
│    └──────────────────┘ └───────────────────┘ └──────────────────────┘   │
│    ┌──────────────────┐ ┌───────────────────┐                            │
│    │   BacktestLab    │ │TradingAgentsAdapter│                            │
│    │   策略回测实验室  │ │ TradingAgents多代理│                            │
│    └──────────────────┘ └───────────────────┘                            │
├──────────────────────────────────────────────────────────────────────────┤
│  数据服务层                                                               │
│    ┌────────────────┐ ┌─────────────────────┐ ┌───────────────────────┐  │
│    │ AkshareDataProvider │ │EastmoneyMarketDataClient│ │  KlineCacheService   │  │
│    │  AkShare统一数据网关 │ │  httpx东财HTTP客户端    │ │  K线并发同步/缓存    │  │
│    └────────────────┘ └─────────────────────┘ └───────────────────────┘  │
│    ┌────────────────┐ ┌─────────────────────┐                            │
│    │ ConceptEngine  │ │    RealtimeHub      │                            │
│    │ 概念热度评分引擎 │ │  WebSocket广播中心   │                            │
│    └────────────────┘ └─────────────────────┘                            │
├──────────────────────────────────────────────────────────────────────────┤
│  数据源层                                                                 │
│    东财 HTTP API · 新浪财经 · 同花顺 · AkShare（akshare Python库）       │
│    Kronos HuggingFace 模型（NeoQuasar/Kronos-base）                     │
│    SQLite ×2（market_kline.db + funnel_state.db）                       │
│    飞书 Webhook · LLM API（OpenAI兼容 / Hermes-Agent / DeepSeek）       │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### 1.3 核心服务一览

#### 数据层服务

| 类名 | 文件路径 | 职责 |
|------|---------|------|
| `KlineSQLiteStore` | [app/services/kline_store.py#L10-L820](file:///workspace/app/services/kline_store.py#L10-L820) | `market_kline.db` K线读写主表、同步状态、股票名称映射 |
| `SQLiteStateStore` | [app/services/sqlite_store.py#L9-L371](file:///workspace/app/services/sqlite_store.py#L9-L371) | `funnel_state.db` 漏斗状态/策略配置/公告状态/KV/自定义策略 |
| `AkshareDataProvider` | [app/services/data_provider.py#L22-L80](file:///workspace/app/services/data_provider.py#L22-L80) | 统一数据网关：实时快照、概念板块、成分股、股票名称、缓存管理 |
| `EastmoneyMarketDataClient` | [app/services/market_data_client.py#L48-L80](file:///workspace/app/services/market_data_client.py#L48-L80) | K线同步专用 httpx 客户端：超时/重试/双域名切换，避免 AkShare 线程阻塞 |
| `KlineCacheService` | [app/services/kline_cache_service.py#L21-L80](file:///workspace/app/services/kline_cache_service.py#L21-L80) | K线并发同步编排：全量补缺、增量同步、批量范围、完整性检查 |

#### 策略引擎服务

| 类名 | 文件路径 | 职责 |
|------|---------|------|
| `FunnelService` | [app/services/funnel_service.py#L32-L80](file:///workspace/app/services/funnel_service.py#L32-L80) | 核心三池漏斗：评分、迁池、状态持久化、热门概念/热门个股构建 |
| `StrategyEngine`（模块函数） | [app/services/strategy_engine.py#L28-L60](file:///workspace/app/services/strategy_engine.py#L28-L60) | 盘中评分核心：`compute_intraday_score` 5项+3减分、迁池规则 |
| `ConceptEngine`（模块函数） | [app/services/concept_engine.py#L17-L80](file:///workspace/app/services/concept_engine.py#L17-L80) | 概念热度：`build_concept_heat` 热度评分、`map_stock_concepts` 个股概念标签映射 |
| `NoticeService` | [app/services/notice_service.py#L13-L79](file:///workspace/app/services/notice_service.py#L13-L79) | 公告漏斗：抓取、关键词规则打分（7条利好+12条利空）、LLM二次评分、迁池 |
| `QuietBreakoutScanner` | [app/services/quiet_breakout_scanner.py#L57-L60](file:///workspace/app/services/quiet_breakout_scanner.py#L57-L60) | 缩量启动形态：横盘N日+首板放量涨停，兼容旧接口 |
| `CustomStrategyScanner` | [app/services/custom_strategy.py#L53-L60](file:///workspace/app/services/custom_strategy.py#L53-L60) | 自定义策略中心：12条原子规则 AND 组合，全库扫描+回测 |
| `CustomStrategy`（数据类） | [app/services/custom_strategy.py#L53-L60](file:///workspace/app/services/custom_strategy.py#L53-L60) | 自定义策略数据模型：id/name/description/rules(StrategyRuleRef列表) |
| `FirstLimitAlphaService` | [app/services/first_limit_alpha_service.py#L40-L80](file:///workspace/app/services/first_limit_alpha_service.py#L40-L80) | 首板Alpha图形选股：基于 `strategy/first_limit_alpha/` 子模块的基线/序列模型推理 |

#### AI / Agent 服务

| 类名 | 文件路径 | 职责 |
|------|---------|------|
| `KronosPredictService` | [app/services/kronos_predict_service.py#L19-L80](file:///workspace/app/services/kronos_predict_service.py#L19-L80) | Kronos-base 三日K线预测：惰性加载模型、串行推理、交易日推算 |
| `PredictFunnelService` | [app/services/predict_funnel_service.py#L50-L80](file:///workspace/app/services/predict_funnel_service.py#L50-L80) | 预测选股：热门板块TopK×成分股TopM → Kronos三日预测 → 三池分级（≥2%/≥4%/≥8%） |
| `HotStockAIService` | [app/services/hot_stock_ai_service.py#L63-L80](file:///workspace/app/services/hot_stock_ai_service.py#L63-L80) | 热门股智能：TopN热门股→Kronos预测→TradingAgents多代理讨论→三池分级（≥8%/≥11.5%/≥14.5%） |
| `HermesRuntime` | [app/services/hermes_runtime.py#L24-L127](file:///workspace/app/services/hermes_runtime.py#L24-L127) | Agent调度：盘后复盘/公告复盘/全面诊断、智能监控tick、熔断限流、Agent模式与降级模式切换 |
| `HermesMemory` | [app/services/hermes_memory.py#L12-L100](file:///workspace/app/services/hermes_memory.py#L12-L100) | Agent记忆：`agent_tasks`任务历史、`agent_monitor_config`监控配置、`agent_monitor_messages`监控消息 |
| `TradingAgentsAdapter` | [app/services/tradingagents_adapter.py#L21-L60](file:///workspace/app/services/tradingagents_adapter.py#L21-L60) | TradingAgents多代理适配器：代码→交易所代码映射、BUY/OVERWEIGHT决策加权、命令行调用外部repo |

#### 基础支撑服务

| 类名 | 文件路径 | 职责 |
|------|---------|------|
| `RealtimeHub` | [app/services/realtime.py#L9-L33](file:///workspace/app/services/realtime.py#L9-L33) | WebSocket 连接管理与事件广播（snapshot / monitor_update） |
| `PaperTradingService` | [app/services/paper_trading.py#L47-L80](file:///workspace/app/services/paper_trading.py#L47-L80) | 模拟盘：开仓/平仓/持仓、佣金/印花税/滑点计算、盈亏计算、交易记录 |
| `BacktestLab` | [app/services/backtest_lab.py#L44-L60](file:///workspace/app/services/backtest_lab.py#L44-L60) | 回测实验室：自定义策略历史回测、胜率/累计收益/最大回撤/平均持仓天数 |
| `FeishuNotify`（模块函数） | [app/services/feishu_notify.py#L45-L60](file:///workspace/app/services/feishu_notify.py#L45-L60) | 飞书 Webhook 推送：纯文本 + 交互式卡片（CardBuilder） |

---

### 1.4 5 个后台调度循环

| 循环名称 | 循环函数 | 所在文件 | 频率 | 职责 |
|---------|---------|---------|------|------|
| **Ticker Loop** | `_ticker_loop` | [app/main.py#L151-L170](file:///workspace/app/main.py#L151-L170) | **60s** | 调用 `FunnelService.tick()` 刷新漏斗盘中评分与迁池；若状态变更则 WebSocket 广播快照；同时更新模拟盘持仓最新价（复用实时快照缓存） |
| **K线同步 Loop** | `kline_cache_loop` | [app/routers/kline.py#L162-L181](file:///workspace/app/routers/kline.py#L162-L181) | **600s（10分钟）** | 调用 `KlineCacheService.run_if_due()` 检测是否到达每日 15:20 自动同步窗口；若触发同步则随后执行 30 日数据完整性检查；默认跳过飞书完成通知 |
| **Hermes 调度 Loop** | `hermes_scheduler_loop` | [app/services/hermes_runtime.py#L910-L935](file:///workspace/app/services/hermes_runtime.py#L910-L935) | **300s（5分钟）** | 15:30-15:40 自动触发 `daily_review` 盘后复盘（每日一次）；21:00-21:10 自动触发 `notice_review` 公告复盘（每日一次）；通过 `get_last_task` 去重 |
| **智能监控 Loop** | `monitor_loop` | [app/services/hermes_runtime.py#L938-L973](file:///workspace/app/services/hermes_runtime.py#L938-L973) | **30s** | 仅在配置 `enabled=True` 且盘中（9:30-11:30 / 13:00-15:00）执行；按 `interval_minutes` 间隔（默认10min）调用 `HermesRuntime.run_monitor_tick()` 生成LLM分析报告；结果通过 WebSocket 推送 `monitor_update` 事件 |
| **热门智能 Loop** | `_hot_stock_ai_scheduler_loop` | [app/main.py#L200-L218](file:///workspace/app/main.py#L200-L218) | **60s 检查** | 检查配置 `auto_refresh_enabled` 且 `is_stale()` 过期时，后台 `create_task` 触发 `HotStockAIService.run(trigger="auto")`；默认 `refresh_interval_minutes=5` 分钟执行一次 Top20 热门股智能分析 + TradingAgents 讨论 |

---

### 1.5 数据流总图：一次点击策略扫描

以用户在前端点击 **「自定义策略扫描」** 按钮为例，端到端数据流转如下：

```
┌──────────────┐
│  前端浏览器  │
│  策略中心Tab │
└──────┬───────┘
       │ 1. POST /api/strategy/custom/{id}/scan
       ▼
┌───────────────────────────────────────────────────┐
│  FastAPI 路由层 scan_custom_strategy              │
│  [app/main.py#L559-L569]                          │
│   · 校验策略是否正在运行                          │
│   · 通过 _load_custom_strategy 加载 CustomStrategy│
│   · 委托给 CustomStrategyScanner.scan()           │
└───────────────────────┬───────────────────────────┘
                        │ 2. asyncio 协程调用
                        ▼
┌───────────────────────────────────────────────────┐
│  CustomStrategyScanner.scan()                     │
│  [app/services/custom_strategy.py]                │
│   · 从 KlineSQLiteStore.get_all_symbols()         │
│     获取全库 ~5000 只股票                          │
│   · 并发（semaphore=8）拉取每只 180 日 K 线        │
│   · 对每只股票：逐条 RULE_REGISTRY 规则 AND 评估  │
│   · 计算 composite_score = 命中规则数×10 + 辅助分 │
│   · 排序、截取 limit → 返回 hits[]                │
└───────────────────────┬───────────────────────────┘
                        │ 3. 批量 SELECT K线
                        ▼
┌───────────────────────────────────────────────────┐
│  KlineSQLiteStore (market_kline.db)               │
│  [app/services/kline_store.py#L209-L237]          │
│   · SELECT trade_date, open, high, low, close,    │
│     volume, amount FROM kline_daily               │
│     WHERE symbol = ? ORDER BY trade_date DESC     │
│     LIMIT 180                                     │
│   · 返回按日期升序排列的 list[dict]                │
└───────────────────────┬───────────────────────────┘
                        │ 4. 命中结果返回
                        ▼
┌───────────────────────────────────────────────────┐
│  CustomStrategyScanner 结果处理                   │
│   · 将扫描快照写入 SQLiteStateStore.kv_store      │
│     key = "custom_scan:{strategy_id}"             │
│   · 返回 {strategy_id, generated_at, hits[],      │
│     total_hits, total_scanned}                    │
└───────────────────────┬───────────────────────────┘
                        │ 5. HTTP 200 JSON 响应
                        ▼
┌──────────────┐
│  前端浏览器  │
│  · 渲染命中列表                              │
│  · 点击命中卡片 → 弹出 Kronos 预测 Modal     │
│    → GET /api/predict/{symbol}/kronos        │
│    → KronosPredictService.predict()         │
│    → 返回未来3日预测K线 + ECharts叠加渲染    │
└──────────────┘
```

**补充说明**：
- 如果用户选择「一键回测」，路由 `backtest_custom_strategy` 会将同一策略传入 `BacktestLab.run_custom_strategy()`，对 180 日历史每日锚点评估买入信号，计算胜率/累计收益/最大回撤。
- 模拟盘买入（POST `/api/paper/buy`）的数据流：`_get_realtime_price` → `provider.get_realtime_snapshot()`（超时2.5s → 降级db_fallback 1.0s）→ `PaperTradingService.open_position()` 写入 `paper_positions` + `paper_trades` 两张表。

---

## 第2章 入口与生命周期

**本节导读**：上一章看完了系统全景图，本章深入「启动时到底发生了什么」和「关闭时如何安全退出」。你将理解 FastAPI lifespan 的完整生命周期钩子、7 个后台任务的创建顺序、shutdown 为什么必须先做 SQLite checkpoint（这直接关系到 macOS UEs 卡死防护），以及 12 层全局单例依赖链是如何构建的。本章末尾还会讲到路由注册方式和 CORS 配置。

### 2.1 FastAPI lifespan 机制

[main.py#L242-L270](file:///workspace/app/main.py#L242-L270)

Alpha 使用 FastAPI 的 `@asynccontextmanager` lifespan 钩子管理启动与关闭。整个应用生命周期如下：

```
uvicorn app.main:app
      │
      ▼
┌────────────────────────────────────────────────────┐
│  模块顶层代码加载（import 时同步执行）               │
│  · 16 个全局单例 Service 初始化（见下节）            │
│  · ensure_builtin_custom_strategies 写入内置策略    │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────┐
│  lifespan startup（yield 之前）                     │
│  create_task × 7 个后台循环                         │
│    1. backfill_task      → _startup_backfill()     │
│    2. ticker_task        → _ticker_loop()          │
│    3. kline_cache_task   → kline_cache_loop()      │
│    4. hermes_task        → hermes_scheduler_loop() │
│    5. monitor_task       → monitor_loop()          │
│    6. predict_funnel_task→ _predict_funnel...()    │
│    7. hot_stock_ai_task  → _hot_stock_ai...()      │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────┐
│  yield  → 开始接受 HTTP/WebSocket 请求              │
│  · FastAPI 路由匹配 ~50 端点                        │
│  · 7 个后台循环并行运行                              │
└──────────────────────┬─────────────────────────────┘
                       │ 收到 SIGTERM / uvicorn 关闭
                       ▼
┌────────────────────────────────────────────────────┐
│  lifespan shutdown（yield 之后）                    │
│  步骤 1: await asyncio.to_thread(_checkpoint_      │
│          all_sqlite)                                │
│          对 data/funnel_state.db + market_kline.db  │
│          执行 PRAGMA wal_checkpoint(TRUNCATE)       │
│  步骤 2: 遍历 app.state 7 个 task → task.cancel()   │
│          with suppress(CancelledError): await task  │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
                  进程退出
```

#### startup：7 个 create_task

| task 名称 | 绑定变量 | 初始 sleep | 说明 |
|----------|---------|-----------|------|
| backfill_task | `app.state.backfill_task` | 0s | 启动时调用 `FunnelService.backfill_names()`，补全 symbol→name 映射（从实时快照或 DB） |
| ticker_task | `app.state.ticker_task` | 5s | `_ticker_loop()` 每 60s 刷新漏斗评分（详见架构总览） |
| kline_cache_task | `app.state.kline_cache_task` | 10s（内部） | `kline_cache_loop()` 每 600s 检测并触发 K 线自动同步 |
| hermes_task | `app.state.hermes_task` | 30s（内部） | `hermes_scheduler_loop()` 每 300s，15:30 盘后复盘/21:00 公告复盘 |
| monitor_task | `app.state.monitor_task` | 10s（内部） | `monitor_loop()` 每 30s 检测智能监控配置并执行 LLM tick |
| predict_funnel_task | `app.state.predict_funnel_task` | 60s | `_predict_funnel_scheduler_loop()` 每 300s，16:15 自动预测选股（每日一次） |
| hot_stock_ai_task | `app.state.hot_stock_ai_task` | 90s | `_hot_stock_ai_scheduler_loop()` 每 60s 检查热门智能是否需自动刷新 |

#### shutdown：先 checkpoint 再 cancel

[main.py#L221-L240](file:///workspace/app/main.py#L221-L240) + [main.py#L260-L270](file:///workspace/app/main.py#L260-L270)

**顺序：先 `_checkpoint_all_sqlite()` → 再逐个 cancel task**

**原因**：macOS 不可中断睡眠（UEs, Uninterruptible Sleep）问题。

若在 SQLite 连接正在执行 WAL fsync（checkpoint 写入主数据库文件）时取消任务，macOS 内核会将该进程标记为 `STAT=U`，进入不可中断睡眠。此时：
- `SIGTERM` 无效
- `SIGKILL`（kill -9）也无效
- 只能通过重启 Mac 清理

解决方案：
1. 在 yield 返回后立即通过 `asyncio.to_thread` 在独立线程中对两个数据库执行 `PRAGMA wal_checkpoint(TRUNCATE)`，把 WAL 日志合并进主 db
2. checkpoint 完成后，才开始 cancel 各个 asyncio Task
3. restart.sh 脚本中先调 `/api/admin/shutdown-prepare`，再通过 Python 脚本再次 checkpoint，最后才 stop.sh 发 SIGTERM

> **UEs 问题的完整根因链、现象描述、三层防护机制以及预防措施**，请继续阅读第5章 5.5节「WAL checkpoint 机制」。此外，第15章调试与运维也有「UEs 进程卡死定位」的专题讲解，包含唯一解法和预防清单。

---

### 2.2 全局单例初始化

[main.py#L50-L134](file:///workspace/app/main.py#L50-L134)

Alpha 采用**模块顶层单例**模式：所有 Service 在 import `app.main` 时按依赖顺序同步构造，通过 `_name` / `name` 等模块级变量暴露给路由函数。

#### 初始化顺序（依赖链）

```
第 1 层：纯存储（无依赖）
  └── _kline_store = KlineSQLiteStore()           [L50]

第 2 层：数据访问（依赖 _kline_store）
  ├── provider = AkshareDataProvider(kline_store=_kline_store)  [L51]
  └── market_data_client = EastmoneyMarketDataClient(store=_kline_store)  [L52]

第 3 层：K 线缓存（依赖 provider + store + client）
  └── kline_cache_service = KlineCacheService(provider, _kline_store, market_data_client)  [L53]

第 4 层：核心漏斗（依赖 provider + kline_cache_service）
  ├── service = FunnelService(provider, kline_cache_service)  [L54]
  │     内部构造：SQLiteStateStore(funnel_state.db)
  │     内部构造：加载 strategy_profile 与 funnel_state
  └── notice_service = NoticeService(state_store, kline_cache_service, provider)  [L55]

第 5 层：通信基础设施（无依赖）
  └── hub = RealtimeHub()  [L56]

第 6 层：AI 预测（依赖 kline_store + provider）
  └── kronos_service = KronosPredictService(kline_store, provider)  [L58-L61]
        注意：模型惰性加载，首次 predict() 时才 download/load

第 7 层：扩展服务（依赖前面的服务）
  ├── paper_trading = PaperTradingService()  [L63]
  │     内部：funnel_state.db → paper_positions / paper_trades
  ├── predict_funnel_service = PredictFunnelService(provider, kronos, state_store)  [L64-L68]
  ├── tradingagents_adapter = TradingAgentsAdapter()  [L69]
  └── hot_stock_ai_service = HotStockAIService(provider, _kline_store, kronos, state_store, tradingagents_adapter)  [L70-L76]
        注意：配置 STATE_KEY="hot_stock_ai"，存储于 kv_store

第 8 层：首板图形选股（依赖 _kline_store + provider + state_store）
  └── first_limit_alpha_service = FirstLimitAlphaService(_kline_store, provider, state_store)  [L77-L81]

第 9 层：辅助扫描器（依赖 _kline_store + name_lookup 闭包）
  ├── _qb_name_lookup() 闭包函数（缓存优先 → DB fallback） [L84-L96]
  ├── quiet_breakout_scanner = QuietBreakoutScanner(_kline_store, name_lookup=_qb_name_lookup)  [L99-L102]
  └── custom_strategy_scanner = CustomStrategyScanner(_kline_store, name_lookup=_qb_name_lookup)  [L104-L107]

第 10 层：ensure_builtin_custom_strategies（写入内置策略） [L109-L114]
  ├── builtin_quiet_breakout（默认策略）
  ├── builtin_adjustment_box
  └── builtin_breakout_volume

第 11 层：Agent 系统（依赖 memory + service + notice_service + kline_cache_service）
  ├── hermes_memory = HermesMemory()  [L123]
  │     内部：funnel_state.db → agent_tasks / agent_monitor_config / agent_monitor_messages
  └── hermes_runtime = HermesRuntime(memory, service, notice_service, kline_cache_service)  [L124-L129]

第 12 层：回测实验室（依赖 _kline_store + _qb_name_lookup）
  └── backtest_lab = BacktestLab(_kline_store, name_lookup=_qb_name_lookup)  [L131-L134]
```

#### 为什么用模块顶层单例

- **service 间互相引用简单**：例如 `FunnelService` 内部的 `state_store` 直接注入到 `NoticeService` / `PredictFunnelService` / `HotStockAIService` / `FirstLimitAlphaService`，所有服务共享同一个 SQLiteStateStore 实例（同一个 DB 连接池语义）。
- **跨路由共享状态**：main.py 中的 ~40 个直接装饰器路由（`@app.get`/`@app.post`）不需要 FastAPI Depends，直接读模块变量 `service` / `kronos_service` / `paper_trading` 等即可访问共享状态（如漏斗 entries、Kronos 已加载模型、模拟盘持仓）。

#### 缺点

- **测试难隔离**：pytest 中需要 mock 模块变量或在 conftest.py 中 monkeypatch 替换单例，无法通过构造函数注入不同依赖实例做并行测试。
- **import 副作用**：`import app.main` 就会触发全部初始化（包括目录创建、SQLite schema 执行、内置策略写入），测试/脚本环境下可能有意外写入。

---

### 2.3 路由注册

#### include_router 子路由（2 个）

[main.py#L281-L283](file:///workspace/app/main.py#L281-L283)

| Router | 初始化函数 | prefix | 主要端点 |
|--------|-----------|--------|---------|
| K线同步路由 | `init_kline_router(provider, kline_cache_service)` | `/api` | `/jobs/kline-cache/*`（sync/status/progress/logs/stats/check/report）、`/kline/{symbol}`、`/admin/shutdown-prepare` |
| 首板Alpha路由 | `init_first_limit_alpha_router(first_limit_alpha_service)` | `/api` | 首板图形选股相关端点（训练/推理/回测/快照） |

#### main.py 直接装饰器路由（~40 个端点）

[main.py#L286-L893](file:///workspace/app/main.py#L286-L893)

按功能分组：

| 分组 | 路由 | 说明 |
|-----|------|------|
| **静态页面** | `GET /` → index.html | 主 SPA 入口 |
| | `GET /notice` → 重定向 `/?tab=notice` | 兼容旧链接 |
| **策略漏斗** | `GET /api/funnel` | 获取三池漏斗快照（可指定 trade_date） |
| | `GET /api/stock/{symbol}/detail` | 个股评分明细 + K 线 + 触发日志 |
| | `POST /api/pool/move` | 手动迁池（MovePoolRequest） |
| | `POST /api/score/recompute` | 重新计算单股评分 |
| **市场概览** | `GET /api/market/hot-concepts` | 热门概念板块 |
| | `GET /api/market/hot-stocks` | 热门个股 Top30 |
| | `GET /api/stock/{symbol}/realtime` | 盘中实时行情（从快照过滤） |
| **策略参数** | `GET /api/strategy/profile` | 获取当前活跃策略配置 |
| **Kronos 预测** | `GET /api/predict/{symbol}/kronos` | Kronos 三日预测（lookback/horizon 参数） |
| **预测选股** | `GET/POST/配置` × `/api/predict-funnel*` | 预测选股快照/手动触发/配置读写 |
| **热门智能** | `GET/POST/配置/迁池` × `/api/strategy/hot-stock-ai*` | 热门股智能分析快照/手动触发/迁池/配置 |
| **缩量启动（旧）** | `GET/POST` × `/api/strategy/quiet-breakout*` | 兼容旧接口的缩量启动扫描 |
| **自定义策略中心** | `GET/POST/DELETE/默认/扫描/回测` × `/api/strategy/custom*` | 12 条原子规则 + 3 内置策略 CRUD + 扫描 + 180 日回测 |
| | `GET /api/strategy/rules` | 规则目录（前端动态渲染表单） |
| **公告漏斗** | `GET/POST` × `/api/notice/*` | 公告漏斗/关键词/手动筛选/迁池/个股详情 |
| **Hermes Agent** | `GET/POST` × `/api/agent/*` | 状态/运行任务(daily_review/notice_review/full_diagnosis)/任务历史 |
| **智能监控** | `GET/POST/配置/消息` × `/api/agent/monitor/*` | 监控配置/手动触发/停止/消息列表 |
| **模拟盘** | `POST /api/paper/buy` | 模拟开仓（需交易时段） |
| | `POST /api/paper/sell` | 模拟平仓（需交易时段） |
| | `GET /api/paper/positions / history / summary / trades` | 持仓/历史/汇总/交易流水 |
| | `GET/POST /api/paper/settings` | 佣金/印花税/滑点配置 |
| **WebSocket** | `WS /ws/realtime` | 连接后立即发 snapshot；后续接受 monitor_update 等事件推送 |

---

### 2.4 CORS 配置

[main.py#L274-L280](file:///workspace/app/main.py#L274-L280)

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],          # ← 允许所有来源
    allow_credentials=True,
    allow_methods=["*"],          # GET/POST/PUT/DELETE/OPTIONS...
    allow_headers=["*"],          # 所有 Header，含 Authorization
)
```

**说明**：`allow_origins=["*"]` 适合本地开发场景（前端跑在不同端口，或用 Trae 的 HTML Share 预览访问）。如果部署到公网，应将 `allow_origins` 收紧为具体域名列表（如 `["http://localhost:18888", "https://yourdomain.com"]`），避免 CSRF 风险。

---

## 第3章 配置系统

**本节导读**：了解完系统启动流程后，本章讲解 Alpha 系统"所有可调参数从哪里来、优先级如何"。你将掌握四层配置来源（API覆盖 > SQLite > dataclass默认值 > 环境变量）的合并逻辑，核心策略评分与迁池的 19 项参数详解（权重、阈值、分钟窗口、池容量），以及 11 个环境变量清单（含 LLM 三选一配置模式）。此外，热门智能、预测选股、智能监控、公告四大模块的 KV 存储配置格式也会一一列出。

### 3.1 四层配置来源及优先级

Alpha 的配置系统采用**四层覆盖**设计，优先级由高到低：

```
┌──────────────────────────────────────────────────┐
│  优先级 1（最高）：运行时 API 覆盖                │
│  · POST /api/strategy/profile 保存到 SQLite      │
│  · POST /api/predict-funnel/config → kv_store    │
│  · POST /api/strategy/hot-stock-ai/config        │
│  · 热更新，无需重启                               │
├──────────────────────────────────────────────────┤
│  优先级 2：SQLite strategy_profiles 表           │
│  [app/services/sqlite_store.py#L36-L44]          │
│  · 每条记录一个"策略版本"                         │
│  · is_active = 1 标记当前生效配置                │
│  · 支持历史版本回溯（保留全部 upsert 记录）       │
├──────────────────────────────────────────────────┤
│  优先级 3：Python dataclass 默认值               │
│  [app/config.py#L9-L33] StrategyConfig           │
│  · 编译时确定的硬编码默认值                       │
│  · 开发者修改源码后重启生效                       │
├──────────────────────────────────────────────────┤
│  优先级 4（最低）：环境变量（.env 文件加载）      │
│  · uvicorn 启动时由 python-dotenv 自动加载 .env  │
│  · 影响：LLM API 凭据 / 服务端口 / 飞书 webhook  │
│  · 修改后需重启进程                               │
└──────────────────────────────────────────────────┘
```

**合并示例（StrategyConfig）**：
1. 构造 `StrategyConfig()` → 得到 dataclass 默认值
2. 从 `strategy_profiles WHERE is_active=1` 读出 `config_json` → 调用 `config.merge(overrides)` 覆盖
3. 如果用户在前端修改参数（通过 `FunnelService.update_strategy_profile()`），新配置写回 `strategy_profiles`（旧记录 is_active=0，新记录 is_active=1），后续请求走优先级 2

---

### 3.2 StrategyConfig 详解

[app/config.py#L9-L33](file:///workspace/app/config.py#L9-L33)

`StrategyConfig` 是**核心策略评分与迁池参数**的 dataclass，当前仅保留三池盘中评分与迁池相关字段。

#### 评分权重（5 项加分 + 3 项减分）

盘中 `compute_intraday_score()` 计算总分（满分 ≈ 80 + 26 结构分 = 106，但 clamp 到 0-100）：

| 配置字段 | 默认值 | 含义 | 计分公式概要 |
|---------|--------|------|-------------|
| `score_weight_breakout` | **35.0** | 突破强度权重 | `clamp((price/breakout_level - 1) / 0.03, 0, 1) × 35` |
| `score_weight_volume` | **25.0** | 量能权重 | `clamp((amount / (avg_amount20 × elapsed_ratio)) / 2, 0, 1) × 25` |
| `score_weight_above_vwap` | **8.0** | 站上 VWAP 加分 | `price ≥ VWAP ? +8 : 0` |
| `score_weight_close_ge_open` | **6.0** | 收高于开 加分 | `price ≥ open ? +6 : 0` |
| `score_weight_drawdown` | **6.0** | 回撤控制加分 | `clamp((0.03 - drawdown_from_high) / 0.03, 0, 1) × 6` |
| `penalty_gap_up` | **8.0** | 高开惩罚（减分） | `open / prev_close > 1.05 ? clamp(..., 0, 1) × 8` 从总分扣 |
| `penalty_drawdown` | **6.0** | 冲高回落惩罚（减分） | `drawdown > 0.03 ? clamp(..., 0, 1) × 6` 从总分扣 |
| `penalty_near_limit` | **6.0** | 近涨停不追惩罚（减分） | `pct_change > near_limit_pct ? clamp(..., 0, 1) × 6` 从总分扣 |
| `near_limit_pct` | **9.2** | 近涨停阈值（%） | `pct_change ≥ 9.2` 触发 penalty_near_limit |

#### 池迁移规则

| 配置字段 | 默认值 | 含义 |
|---------|--------|------|
| `focus_score_threshold` | **60.0** | 候选→重点：评分 ≥60 且持续 `focus_consecutive_minutes` 分钟 |
| `focus_score_immediate` | **70.0** | 候选→重点（立即）：评分 ≥70 直接晋级，不需要连续分钟 |
| `focus_consecutive_minutes` | **3** | 升重点需要的连续达标分钟数 |
| `buy_score_threshold` | **78.0** | 重点→买入：评分 ≥78（核心阈值） |
| `buy_volume_ratio_threshold` | **1.3** | 重点→买入：量能比 ≥1.3（20日均额/已过时间比例 归一化后） |
| `buy_breakout_price_buffer` | **1.003** | 重点→买入：价格 ≥ breakout_level × 1.003（突破确认，+0.3% 缓冲防假突破） |
| `buy_breakout_consecutive_minutes` | **2** | 升买入需要的连续达标分钟数 |
| `downgrade_score_threshold` | **65.0** | 降级（buy→focus / focus→candidate）：评分连续 N 分钟 < 65 |
| `downgrade_consecutive_minutes` | **5** | 降级需要的连续不达标分钟数 |
| `buy_pool_max_size` | **5** | 买入池最大容量；超过后新标的需等待空位或抢占（按分数） |

**迁池逻辑速记**：
- `candidate → focus`：score ≥ 70 立即，或 score ≥ 60 连续 3 分钟
- `focus → buy`：score ≥ 78 **且** volume_ratio ≥ 1.3 **且** price ≥ breakout × 1.003，连续 2 分钟
- `buy/focus → candidate`：score < 65 连续 5 分钟（逐级降级）
- `buy_pool_max_size = 5`：买入池满了就不再进，除非有票降级出空位

#### 辅助参数

| 配置字段 | 默认值 | 含义 |
|---------|--------|------|
| `near_limit_pct` | **9.2** | 距涨停 ≤0.8% 视为"近涨停"，触发减分（避免追高接盘） |
| `buy_pool_max_size` | **5** | 买入池最大 5 只，防止过度分散 |

---

### 3.3 环境变量清单

来自 `.env.example` + README + restart.sh：

| 环境变量 | 默认值（示例） | 说明 | 影响范围 |
|---------|---------------|------|---------|
| **PORT** | `18888` | 服务监听端口 | uvicorn 启动 / restart.sh / stop.sh（pid 文件匹配） |
| **HOST** | `0.0.0.0` | 服务绑定地址 | uvicorn 启动 |
| **RELOAD** | `0` | 是否开启代码热重载（开发用） | uvicorn `--reload` 标志 |
| **PYTHON_BIN** | （自动探测） | Python 解释器路径 | restart.sh：优先 `~/arm-python/python/bin/python3.11` → `/opt/homebrew/bin/python3.11` → `/opt/homebrew/bin/python3` → `python3` |
| **OPENAI_API_KEY** | `ollama` | LLM API Key | HermesRuntime / Monitor LLM / Predict LLM；三选一（见 .env.example 方案 A/B/C） |
| **OPENAI_BASE_URL** | `http://127.0.0.1:11434/v1` | LLM Base URL（OpenAI 兼容端点） | Ollama / OpenAI / 自建代理 |
| **HERMES_MODEL** | `qwen2.5:7b` | 降级模式（非 Agent）时使用的具体模型名 | 盘后复盘/公告复盘/智能监控 的 LLM 调用 |
| **HERMES_AGENT_URL** | `http://127.0.0.1:8642/v1` | 独立 Hermes Agent API Server 地址 | Agent 模式下 `_delegate_to_hermes_agent()` 请求的 base URL |
| **API_SERVER_KEY** | （空） | Hermes Agent Bearer Token | 与 HERMES_AGENT_URL 配套使用 |
| **FEISHU_WEBHOOK_URL** | （内置默认） | 飞书机器人 Webhook 链接 | 飞书通知：K线同步完成/预测选股 Top10 摘要 等 |
| **ALPHA_API_BASE** | （保留） | Alpha 自身对外 API Base URL | TradingAgents / MCP 外部接入 Alpha 时使用 |

**LLM 三选一配置模式**（来自 `.env.example` 注释）：
- **方案 A（本地 Ollama）**：`OPENAI_API_KEY=ollama` + `OPENAI_BASE_URL=http://127.0.0.1:11434/v1` + `HERMES_MODEL=qwen2.5:7b`
- **方案 B（OpenAI 官方）**：注释掉 A，启用 `OPENAI_API_KEY=sk-xxx` + `OPENAI_BASE_URL=https://api.openai.com/v1` + `HERMES_MODEL=gpt-4o-mini`
- **方案 C（独立 hermes-agent）**：`HERMES_AGENT_URL=http://127.0.0.1:8642/v1` + 可选 `API_SERVER_KEY=xxx`（Agent 模式优先，不设 OPENAI_API_KEY 时走 HTTP fallback）

---

### 3.4 热门智能 / 预测选股 / 公告的配置存储

除了 `StrategyConfig`（策略评分与迁池），Alpha 中还有三类模块级配置，全部使用 **`SQLiteStateStore.kv_store` 通用 KV 表**或专用表持久化：

#### 1. 预测选股配置（PredictFunnelService）

- **STATE_KEY**：`"predict_funnel"`（kv_store 键）
- **配置项**：[app/services/predict_funnel_service.py#L35-L45](file:///workspace/app/services/predict_funnel_service.py#L35-L45)

| 配置键 | 默认值 | 说明 |
|-------|--------|------|
| `top_k_boards` | 10 | 热门板块 Top K |
| `top_m_stocks` | 10 | 每板块成分股 Top M |
| `threshold_candidate` | 2.0 | 候选池阈值：预测 max_high_pct ≥ 2% |
| `threshold_focus` | 4.0 | 重点池阈值：≥ 4% |
| `threshold_buy` | 8.0 | 买入池阈值：≥ 8% |
| `horizon` | 3 | Kronos 预测窗口天数 |
| `lookback` | 180 | Kronos 回看天数 |
| `feishu_enabled` | True | Top10 摘要推送飞书 |
| `auto_after_close` | True | 16:15 每日自动触发 |

- **API**：`GET /api/predict-funnel/config` 读，`POST /api/predict-funnel/config` 写（merge 语义）

#### 2. 热门智能配置（HotStockAIService）

- **STATE_KEY**：`"hot_stock_ai"`（kv_store 键）
- **配置项**：[app/services/hot_stock_ai_service.py#L31-L48](file:///workspace/app/services/hot_stock_ai_service.py#L31-L48)

| 配置键 | 默认值 | 说明 |
|-------|--------|------|
| `top_n` | 20 | 热门股 Top N 数量 |
| `lookback` | 90 | Kronos 回看天数 |
| `horizon` | 3 | Kronos 预测窗口 |
| `threshold_candidate` | 8.0 | 候选池：预测 max_high_pct ≥ 8% |
| `threshold_focus` | 11.5 | 重点池：≥ 11.5% |
| `threshold_buy` | 14.5 | 买入池：≥ 14.5% |
| `max_buy_pool_size` | 5 | 买入池上限 |
| `auto_refresh_enabled` | True | 自动刷新开关 |
| `refresh_interval_minutes` | 5 | 自动刷新间隔（调度 loop 60s 检查） |
| `use_kronos` | True | 是否调用 Kronos 预测 |
| `tradingagents_enabled` | True | 是否启用 TradingAgents 多代理讨论 |
| `tradingagents_top_n` | 20 | TradingAgents 处理 Top 多少只 |
| `tradingagents_timeout_seconds` | 240 | TradingAgents CLI 超时 |
| `tradingagents_provider` | `"deepseek"` | LLM provider |
| `tradingagents_quick_model` | `"deepseek-chat"` | 快速模型（排序/筛选） |
| `tradingagents_deep_model` | `"deepseek-reasoner"` | 深度模型（讨论/推理） |

- **API**：`GET /api/strategy/hot-stock-ai/config` 读，`POST /api/strategy/hot-stock-ai/config` 写

#### 3. 智能监控配置（HermesMemory）

- **表**：`agent_monitor_config`（专用单列表 `CHECK(id=1)`）[app/services/hermes_memory.py#L46-L52](file:///workspace/app/services/hermes_memory.py#L46-L52)
- **字段**：
  - `system_prompt TEXT NOT NULL`：监控自定义系统提示（缺省走 `_DEFAULT_MONITOR_PROMPT`）
  - `interval_minutes INTEGER NOT NULL DEFAULT 10`：监控间隔分钟数
  - `enabled INTEGER NOT NULL DEFAULT 0`：是否启用
  - `updated_at`：更新时间
- **API**：`GET /api/agent/monitor/config` 读，`POST /api/agent/monitor/config` 写

#### 4. 公告配置

- **利好关键词权重**：代码中硬编码 `BULLISH_RULES` 列表 [app/services/notice_service.py#L15-L23](file:///workspace/app/services/notice_service.py#L15-L23)，通过 `GET /api/notice/keywords` 暴露给前端
- **运行时筛选关键词**：`POST /api/jobs/notice-screen?keywords=业绩预增,股份回购` 传 `keywords` 参数，仅启用指定类别打分
- **LLM 二次评分开关**：存于 `notice_state.llm_enabled`（单列表 `CHECK(id=1)`）[app/services/sqlite_store.py#L48-L55](file:///workspace/app/services/sqlite_store.py#L48-L55)

---

## 第4章 数据模型

**本节导读**：配置系统讲完了"参数从哪来"，本章讲系统中流动的"数据长什么样"。所有 API 请求/响应的 Pydantic 模型集中在 `app/models.py`，共 18 个分类。本章将详细讲解 StockCard 的 15 个核心字段、FunnelResponse 的结构、PoolName 的 Literal 定义，以及最值得深究的 KlinePoint `amount` 字段三层架构取舍（为什么 API 层不带 amount，而存储层和 Kronos 推理层必须带）。

### 4.1 Pydantic 模型位置与分类

全部 API 请求/响应 Pydantic 模型集中在 [models.py](file:///workspace/app/models.py)，共 18 个。

```
app/models.py
├── PoolName (Literal 类型别名)
├── ConceptTag            概念标签
├── StockCard             漏斗个股卡片（核心）
├── FunnelResponse        三池漏斗快照响应
├── HotConceptItem        热门概念条目
├── HotConceptResponse    热门概念列表响应
├── HotStockItem          热门个股条目
├── HotStocksResponse     热门个股列表响应
├── MovePoolRequest       手动迁池请求
├── MovePoolResponse      手动迁池响应
├── RecomputeRequest      重算评分请求
├── KlinePoint            K线单根（OHLCV）
├── StockDetailResponse   个股详情响应（含评分分解+K线）
├── NoticeItem            公告漏斗条目
├── NoticeFunnelResponse  公告漏斗快照响应
├── NoticeDetailResponse  公告个股详情响应
└── DEFAULT_EMPTY_FUNNEL  常量：空漏斗 FunnelResponse 实例
```

---

#### 概念 / 热度

| 模型名 | 位置 | 用途 |
|-------|-----|------|
| `ConceptTag` | [models.py#L12-L22](file:///workspace/app/models.py#L12-L22) | 个股身上的概念标签（每只股票最多带 3 个，TAG_COLORS 配色） |
| `HotConceptItem` | [models.py#L47-L56](file:///workspace/app/models.py#L47-L56) | 热门概念列表的单条（热度、涨跌、涨停数、领涨股、漏斗入选数） |
| `HotConceptResponse` | [models.py#L58-L63](file:///workspace/app/models.py#L58-L63) | 热门概念整体响应（含 trade_date、frozen 非交易时段标记） |
| `HotStockItem` | [models.py#L65-L73](file:///workspace/app/models.py#L65-L73) | 热门个股单条（rank、代码、名称、最新价、当日涨幅、10日累计涨幅） |
| `HotStocksResponse` | [models.py#L75-L80](file:///workspace/app/models.py#L75-L80) | 热门个股整体响应 |

#### 漏斗核心

| 模型名 | 位置 | 用途 |
|-------|-----|------|
| `StockCard` | [models.py#L24-L37](file:///workspace/app/models.py#L24-L37) | 三池漏斗的个股卡片（前端主渲染单元） |
| `FunnelResponse` | [models.py#L40-L45](file:///workspace/app/models.py#L40-L45) | `GET /api/funnel` 响应，三池 + 统计 |
| `MovePoolRequest` | [models.py#L82-L87](file:///workspace/app/models.py#L82-L87) | 手动迁池请求体（symbol + target_pool + 可选 note） |
| `MovePoolResponse` | [models.py#L89-L94](file:///workspace/app/models.py#L89-L94) | 手动迁池响应 |
| `RecomputeRequest` | [models.py#L96-L98](file:///workspace/app/models.py#L96-L98) | 重新计算评分请求（symbol 为 None 表示全量重算） |

#### 个股详情

| 模型名 | 位置 | 用途 |
|-------|-----|------|
| `KlinePoint` | [models.py#L100-L107](file:///workspace/app/models.py#L100-L107) | K 线单根数据点（date/open/high/low/close/volume） |
| `StockDetailResponse` | [models.py#L109-L121](file:///workspace/app/models.py#L109-L121) | 个股详情：评分分解 + 指标 + 概念标签 + 候选概念 + trigger_log + K线 |

#### 公告

| 模型名 | 位置 | 用途 |
|-------|-----|------|
| `NoticeItem` | [models.py#L123-L135](file:///workspace/app/models.py#L123-L135) | 公告漏斗单条（标题、类型、日期、URL、打分、池分配、原因、风险） |
| `NoticeFunnelResponse` | [models.py#L137-L144](file:///workspace/app/models.py#L137-L144) | 公告漏斗整体响应（三池 + llm_enabled 标记 + source 数据源） |
| `NoticeDetailResponse` | [models.py#L146-L155](file:///workspace/app/models.py#L146-L155) | 单股公告详情（原因、风险、历史公告列表 + K 线） |

#### PoolName Literal

[models.py#L9](file:///workspace/app/models.py#L9)

```python
PoolName = Literal["candidate", "focus", "buy"]
```

与 `app/config.py` 中的常量对应：
- `POOL_CANDIDATE = "candidate"` — 候选池（初步命中，持续观察）
- `POOL_FOCUS = "focus"` — 重点关注池（评分达标或有题材）
- `POOL_BUY = "buy"` — 买入池（高分 + 量价齐升确认，上限 5 只）

---

### 4.2 重点模型字段说明

#### StockCard

[models.py#L24-L37](file:///workspace/app/models.py#L24-L37)

共 **15 个字段**，是整个策略漏斗的核心渲染单元：

| 字段 | 类型 | 说明 |
|-----|------|------|
| `symbol` | `str` | 6 位股票代码（纯数字，不带 .SH/.SZ） |
| `name` | `str` | 股票名称（中文） |
| `pool` | `PoolName` | 当前所在池：`candidate` / `focus` / `buy` |
| `score` | `float` | 当前综合评分（0-100，可能超 100，但前端 clamp 显示） |
| `score_delta` | `float` | 相对上次 tick 的评分变化（正=上升，负=下降，前端画△标记） |
| `recommended_pool` | `PoolName \| None` | 评分规则建议的目标池；`None` 表示与 `pool` 一致无需迁移 |
| `breakout_level` | `float` | 突破基准价（箱体顶/前高），用来计算突破强度 |
| `volume_ratio` | `float` | 当前量比（归一化：当前成交额 / (20日均额 × 已过交易时间比例)） |
| `pct_change` | `float` | 当日涨跌幅（%），来自实时快照 |
| `concept_tags` | `list[ConceptTag]` | 匹配到的热门概念标签列表（最多 3 个，按热度排序） |
| `reasons` | `list[str]` | 加分原因：人类可读文本列表，如"放量 2.1x 达标"、"站上 VWAP" |
| `warnings` | `list[str]` | 风险提示：减分/警告原因，如"高开 4.2%"、"距涨停 0.5%"、"冲高回落 3.1%" |
| `updated_at` | `str` | ISO 时间戳：本条目最后更新时间 |

> **补充说明**：用户需求提到 `trigger_log` 字段——此字段存在于 `StockDetailResponse.trigger_log`（个股详情页面中，展示历史迁池事件列表：`[{timestamp, from_pool, to_pool, reason, score}]`），而轻量级的 `StockCard` 不携带 trigger_log，避免快照过大。

#### FunnelResponse

[models.py#L40-L45](file:///workspace/app/models.py#L40-L45)

| 字段 | 类型 | 说明 |
|-----|------|------|
| `trade_date` | `str` | 对应交易日（YYYY-MM-DD）；非交易时段但数据已冻结时显示上个交易日 |
| `updated_at` | `str` | 快照整体更新时间（ISO） |
| `pools` | `dict[str, list[StockCard]]` | 三池内容字典：key = `"candidate"` / `"focus"` / `"buy"`，value = StockCard 数组 |
| `stats` | `dict[str, int]` | 各池数量统计：`{candidate: N, focus: N, buy: N}`，前端顶部计数卡片直接用 |

#### KlinePoint

[models.py#L100-L107](file:///workspace/app/models.py#L100-L107)

| 字段 | 类型 | 说明 |
|-----|------|------|
| `date` | `str` | 交易日（YYYY-MM-DD） |
| `open` | `float` | 开盘价 |
| `high` | `float` | 最高价 |
| `low` | `float` | 最低价 |
| `close` | `float` | 收盘价 |
| `volume` | `float` | 成交量（股数） |

##### 为什么 KlinePoint 不包含 amount（成交额），而 Kronos 与 KlineSQLiteStore 用 OHLCV + Amount？

**原因分层说明**：

1. **API 响应精简（KlinePoint）**：`StockDetailResponse.kline` 和 `GET /api/kline/{symbol}` 主要给前端 ECharts 画蜡烛图用。蜡烛图**只需要 O/H/L/C/V**，不需要 amount——amount 通常作为副图指标单独展示，不携带在 Pydantic 模型里避免 payload 膨胀。

2. **内部存储（KlineSQLiteStore.kline_daily）**：[app/services/kline_store.py#L26-L37](file:///workspace/app/services/kline_store.py#L26-L37) 主表字段是 `symbol/trade_date/open/high/low/close/volume/amount`，**完整存储 OHLCV + Amount**。`get_kline()` 方法虽然返回的 dict 中包含 `amount`，但被 `StockDetailResponse.kline: list[KlinePoint]` 通过 Pydantic 丢弃了（KlinePoint 没定义 amount 字段，Pydantic 默认 `extra="ignore"`）。

3. **Kronos 模型（OHLCV + Amount 缺一不可）**：Kronos-base 是 HuggingFace 金融时序基础模型（`NeoQuasar/Kronos-base`），它的 tokenizer 需要 **OHLCV + Amount 六列** 联合编码。`KronosPredictService._get_history()` 调用 `kline_store.get_kline()` 时直接使用返回 dict 中的 `amount` 字段（[app/services/kline_store.py#L209-L237](file:///workspace/app/services/kline_store.py#L209-L237)），这部分不经过 Pydantic KlinePoint，所以 amount 是通路完整的。

**架构权衡总结**：
```
存储层（最完整）     →   模型推理层（完整）    →   API/前端（精简）
kline_daily.amount       KronosPredictService      KlinePoint 不含 amount
  │  8 个字段全部有        │  读 dict.amount           │  仅 6 字段画蜡烛
  └────────────────────────┴──────────────────────────┘
         SQLite 返回 dict（不是 Pydantic）时 amount 可用
                         只有 Pydantic 序列化到 HTTP 响应时才丢 amount
```

---

## 第5章 数据层设计

**本节导读**：前面章节讲了系统启动、参数配置和数据模型结构，本章深入最底层的持久化设计。Alpha 用两个独立 SQLite 文件隔离 K 线大数据和状态小数据，并统一启用 WAL + synchronous=NORMAL 兼顾并发与性能。本章会逐字段列出 funnel_state.db 的 10 张表和 market_kline.db 的 6 张表用途，然后讲解 WAL checkpoint 的三层机制（注意 macOS UEs 问题在本章只做引用指向，完整版本请回溯第 2 章 2.1 节 shutdown 部分）。最后用 10 维度对比表总结 KlineSQLiteStore 和 SQLiteStateStore 的职责划分。

### 5.1 双 SQLite 数据库隔离设计

Alpha 采用**两个独立 SQLite 文件**物理隔离读写负载，避免单一数据库的 WAL 争用和锁冲突：

```
data/
├── market_kline.db      ← K 线只读为主，写操作在同步任务
│   ├── kline_daily（主表，~5000 股 × ~180 天 ≈ 90 万行）
│   ├── symbol_names（~5000 行）
│   ├── kline_sync_state（单例 CHECK(id=1)）
│   ├── kline_sync_tasks + kline_sync_task_details（同步日志）
│   └── kline_check_reports（完整性检查报告）
│
└── funnel_state.db      ← 状态/配置/持仓/Agent记忆，频繁写入
    ├── funnel_state（单例 CHECK(id=1)，漏斗快照 JSON）
    ├── strategy_profiles（历史策略配置）
    ├── notice_state（单例 CHECK(id=1)，公告漏斗状态）
    ├── kv_store（通用 KV）
    ├── custom_strategies（自定义策略 CRUD）
    ├── paper_positions + paper_trades（模拟盘）
    └── agent_tasks / agent_monitor_config / agent_monitor_messages（Agent记忆）
```

#### 为什么分两个库？

| 维度 | `market_kline.db` | `funnel_state.db` |
|-----|------------------|------------------|
| **访问模式** | 90% 读（扫描/回测/Kronos 推理），10% 写（每日同步批次） | 70% 写（60s ticker 落盘/模拟盘下单/监控消息），30% 读 |
| **写入频率** | 每日 1~3 批次（每批次 ~5000 条 upsert） | 盘中持续：ticker 60s/次、监控 30s/次、下单随时 |
| **数据量** | 大（~500 MB ~ 几 GB） | 小（通常 < 50 MB） |
| **并发读** | 高（策略扫描全库 → 5000 次 SELECT；Kronos 推理 → 每只 1 次） | 低（路由单条查询） |
| **WAL 增长** | 同步批次时短时间暴涨 | 小而频繁（每次 checkpoint 后回到 0） |

如果合为一个库：
1. 每日 K 线同步的大事务写入 WAL 时，会阻塞盘中 ticker/监控的小写入（SQLite 虽然 WAL 读写不阻塞写写互斥，但大事务 checkpoint 会占时间）
2. 单个 WAL 文件膨胀到几百 MB → checkpoint 耗时更长 → macOS UEs 风险加大
3. 备份策略不同：K线数据可以从东财重拉，丢失可恢复；漏斗状态含用户手动迁池/自定义策略/模拟盘记录，不可丢失

---

### 5.2 SQLite 关键 PRAGMA

两个库在每次 `_connect()` 时都会设置：[sqlite_store.py#L14-L19](file:///workspace/app/services/sqlite_store.py#L14-L19)（KlineSQLiteStore 在 [kline_store.py#L15-L21](file:///workspace/app/services/kline_store.py#L15-L21)）

```python
conn.execute("PRAGMA journal_mode=WAL")       # ← 核心
conn.execute("PRAGMA synchronous=NORMAL")     # ← 性能 vs 安全折衷
# market_kline.db 额外有：
#   conn.execute("PRAGMA busy_timeout=1000")  # 锁等待超时 1s
```

#### journal_mode = WAL

**作用**：写前日志模式（Write-Ahead Logging）。传统 DELETE 模式下写入会直接覆盖主文件旧页并加排他锁；WAL 模式下：

1. **写入**：新数据追加到 `*.db-wal` 文件，不修改主 db 文件
2. **读取**：读者从主 db + WAL 合并视图读取，不需要阻塞写者
3. **效果**：读不阻塞写，写不阻塞读（只有写写互斥）
4. **副作用**：产生 `*.db-wal`（日志）和 `*.db-shm`（共享内存索引）两个文件；WAL 增长到一定阈值时需要 checkpoint 合并回主 db

对 Alpha 的意义：
- 策略扫描全库 SELECT 5000 次时，Ticker 的 60s 一次写漏斗状态**不会被阻塞**
- K 线同步大批次 upsert 时，前端的 `GET /api/funnel` 查询**不会卡住**

#### synchronous = NORMAL

**WAL 模式下的 fsync 频率控制**：

| 模式 | fsync 时机 | 安全性 | 性能 |
|-----|-----------|--------|------|
| `FULL`（默认）| 每次提交事务 + checkpoint 关键节点多次 fsync | 掉电也不丢（ACID 完整） | 慢：同步 ~5000 股可能慢 2~3 倍 |
| **`NORMAL`（Alpha 用）** | 仅在 checkpoint 时 fsync WAL → 主 db；提交事务时只写 WAL，不强 fsync | 应用崩溃安全（OS 崩溃/掉电可能丢最近几条 WAL）；对 Alpha 足够（K线可重拉、状态 60s 前的可接受损失） | 快：同步速度显著提升 |
| `OFF` | 完全不 fsync | 危险：OS 崩溃就可能损坏 db | 最快 |

**Alpha 的折衷**：WAL + NORMAL = 兼顾并发与性能。安全底线是：系统崩溃后最多丢失上一次 checkpoint 之后的写入，但**数据库文件不会损坏**（WAL 的幂等重放语义保证）。对 K线来说可重拉；对漏斗状态来说最多回退几十分钟（可接受）。

---

### 5.3 funnel_state.db 表一览

> **关于表用途清单的交叉引用说明**：PaperTradingService 的 `paper_positions` / `paper_trades` 两张表在本节给出完整字段；第10章业务服务讲解 PaperTradingService 时会以"对应表结构详见第5章 5.3节"方式引用，避免重复叙述。Agent 记忆系统三表（agent_tasks / agent_monitor_config / agent_monitor_messages）同理，第9章 Agent服务将做同样引用。

#### funnel_state（CHECK(id=1) 单例表）

[sqlite_store.py#L24-L33](file:///workspace/app/services/sqlite_store.py#L24-L33)

**用途**：存储策略漏斗最新快照（三池 + 热门概念 + 热门个股）

| 字段 | 类型 | 说明 |
|-----|------|------|
| `id` | INTEGER PK | `CHECK(id=1)`，只有一行 |
| `trade_date` | TEXT | 对应交易日 YYYY-MM-DD；跨日期则忽略旧快照，从头初始化 |
| `entries_json` | TEXT | 三池全量 entries：`{symbol: {pool, score, breakout_level, ...}}` JSON |
| `hot_concepts_json` | TEXT | 热门概念 Top120 列表 JSON |
| `hot_stocks_json` | TEXT | 热门个股 Top30 列表 JSON |
| `updated_at` | TEXT | ISO 时间戳 |
| `frozen` | INTEGER | `0/1`，非交易时段标记（盘后/周末/节假日冻结，不刷新） |

#### strategy_profiles（历史策略配置表）

[sqlite_store.py#L37-L44](file:///workspace/app/services/sqlite_store.py#L37-L44)

**用途**：保留全部历史策略配置版本，`is_active` 标记当前生效

| 字段 | 类型 | 说明 |
|-----|------|------|
| `id` | INTEGER PK AUTOINCREMENT | 版本号，每次 upsert 新增一行 |
| `name` | TEXT | 策略版本名称，如"默认策略-20260819" |
| `config_json` | TEXT | `StrategyConfig.to_dict()` 的 JSON 序列化 |
| `is_active` | INTEGER | `0/1`，仅一条 =1 生效；upsert 时先全部 UPDATE =0，再 INSERT 新的 =1 |
| `updated_at` | TEXT | ISO 时间戳 |

#### notice_state（CHECK(id=1) 单例表）

[sqlite_store.py#L48-L55](file:///workspace/app/services/sqlite_store.py#L48-L55)

**用途**：公告漏斗状态快照

| 字段 | 类型 | 说明 |
|-----|------|------|
| `id` | INTEGER PK | `CHECK(id=1)` |
| `trade_date` | TEXT | 公告日期 |
| `entries_json` | TEXT | 公告漏斗三池 entries JSON |
| `updated_at` | TEXT | ISO 时间戳 |
| `llm_enabled` | INTEGER | `0/1`，是否启用 LLM 二次评分 |
| `source` | TEXT | 数据源标记，如"akshare/em" |

#### kv_store（通用 KV 表）

[sqlite_store.py#L60-L65](file:///workspace/app/services/sqlite_store.py#L60-L65)

**用途**：通用 key-value 存储，各模块独立 state key 命名空间

| 字段 | 类型 | 说明 |
|-----|------|------|
| `key` | TEXT PK | 如 `"predict_funnel"` / `"hot_stock_ai"` / `"custom_scan:builtin_quiet_breakout"` / `"first_limit_alpha_graphic"` |
| `value_json` | TEXT | 任意 JSON 序列化值 |
| `updated_at` | TEXT | ISO 时间戳 |

**已用 key 清单**：
- `predict_funnel` → PredictFunnelService 快照 + 配置 [predict_funnel_service.py#L47]
- `hot_stock_ai` → HotStockAIService 快照 + 配置 [hot_stock_ai_service.py#L28]
- `hot_stock_ai_tradingagents_discussions` → TradingAgents 讨论缓存
- `custom_scan:{strategy_id}` → 自定义策略扫描结果快照
- `first_limit_alpha_graphic` → FirstLimitAlpha 图形选股快照

#### custom_strategies（自定义策略表）

[sqlite_store.py#L69-L79](file:///workspace/app/services/sqlite_store.py#L69-L79)

**用途**：策略中心的自定义策略 CRUD

| 字段 | 类型 | 说明 |
|-----|------|------|
| `id` | TEXT PK | 策略 ID：hex uuid / 内置如 `"builtin_quiet_breakout"` |
| `name` | TEXT | 策略名称 |
| `description` | TEXT | 描述说明 |
| `rules_json` | TEXT | `[{rule_code, enabled, params}]` 规则引用 JSON |
| `is_builtin` | INTEGER | `0/1`，是否内置策略（不可删除） |
| `is_default` | INTEGER | `0/1`，是否默认选中策略 |
| `created_at` | TEXT | ISO 创建时间 |
| `updated_at` | TEXT | ISO 更新时间 |

#### paper_positions（模拟盘持仓表）

[paper_trading.py#L104-L120](file:///workspace/app/services/paper_trading.py#L104-L120)

**用途**：当前 + 历史持仓（含已平仓）

| 字段 | 类型 | 说明 |
|-----|------|------|
| `id` | TEXT PK | 持仓 ID（hex uuid 前 12 位） |
| `symbol` | TEXT | 6 位代码 |
| `name` | TEXT | 股票名称 |
| `direction` | TEXT | 固定 `"long"`（仅支持做多） |
| `qty` | INTEGER | 持仓股数 |
| `cost_price` | REAL | 成本价（已分摊买入费用） |
| `current_price` | REAL | 最新价（实时快照更新） |
| `opened_at` | TEXT | ISO 开仓时间 |
| `status` | TEXT | `"open"` 持仓中 / `"closed"` 已平仓 |
| `closed_at` | TEXT \| NULL | ISO 平仓时间 |
| `close_price` | REAL \| NULL | 平仓成交价 |
| `buy_fee` | REAL | 买入总费用（佣金） |
| `sell_fee` | REAL | 卖出总费用（佣金+印花税） |
| `note` | TEXT | 用户备注 |

#### paper_trades（模拟盘交易流水表）

[paper_trading.py#L122-L134](file:///workspace/app/services/paper_trading.py#L122-L134)

**用途**：每次开仓/平仓各写一条流水（审计用，对应订单）

| 字段 | 类型 | 说明 |
|-----|------|------|
| `id` | TEXT PK | 交易流水 ID |
| `position_id` | TEXT | 关联 paper_positions.id |
| `symbol` | TEXT | 代码 |
| `name` | TEXT | 名称 |
| `action` | TEXT | `"buy"` 开仓 / `"sell"` 平仓 |
| `qty` | INTEGER | 成交股数 |
| `price` | REAL | 成交价（含滑点） |
| `fee` | REAL | 本次交易总费用（佣金/印花税） |
| `created_at` | TEXT | ISO 成交时间 |
| `note` | TEXT | 用户备注 |

#### Agent 记忆系统（3 张表）

定义在 [hermes_memory.py#L28-L61](file:///workspace/app/services/hermes_memory.py#L28-L61)，但物理存储在 `funnel_state.db`（共用 SQLite 文件）。

| 表名 | 用途 | 关键字段 |
|-----|------|---------|
| `agent_tasks` | Hermes 任务历史（盘后复盘/公告复盘/全面诊断） | `task_type`、`trigger`(manual/scheduled)、`status`(running/success/timeout/failed)、`input_summary/output_summary/observations/tool_calls/error_message` JSON、`started_at/finished_at/elapsed_ms` |
| `agent_monitor_config` | CHECK(id=1) 智能监控单例配置 | `system_prompt`、`interval_minutes`(默认10)、`enabled`(0/1) |
| `agent_monitor_messages` | 历史智能监控报告（LLM 输出） | `content`（文本）、`trigger`(scheduled/manual)、`created_at` |

---

### 5.4 market_kline.db 表结构

根据 [kline_store.py#L23-L119](file:///workspace/app/services/kline_store.py#L23-L119) 推断：

#### kline_daily（K线主表）

[kline_store.py#L26-L38](file:///workspace/app/services/kline_store.py#L26-L38)

| 字段 | 类型 | 说明 |
|-----|------|------|
| `symbol` | TEXT | PK(1)，6 位股票代码 |
| `trade_date` | TEXT | PK(2)，YYYY-MM-DD 或 YYYYMMDD |
| `open` / `high` / `low` / `close` | REAL | OHLC 四价（不复权） |
| `volume` | REAL | 成交量（股数） |
| `amount` | REAL | 成交额（元） |
| `updated_at` | TEXT | ISO 写入/更新时间 |
| **复合主键** | | `PRIMARY KEY (symbol, trade_date)` |
| **索引** | | `idx_kline_symbol_date` ON kline_daily(symbol, trade_date DESC) |

#### symbol_names（股票代码名称映射表）

[kline_store.py#L112-L118](file:///workspace/app/services/kline_store.py#L112-L118)

| 字段 | 类型 | 说明 |
|-----|------|------|
| `symbol` | TEXT PK | 6 位代码 |
| `name` | TEXT | 中文名称 |
| `updated_at` | TEXT | ISO 更新时间 |

> 用途：实时快照不可用时（盘后/网络故障），前端仍可显示股票名称。写入路径：`AkshareDataProvider._ensure_names_from_snapshot()` 或 `FunnelService.backfill_names()`。

#### 同步任务日志（3 张表）

**kline_sync_state（单例 CHECK(id=1)）**：[kline_store.py#L42-L56](file:///workspace/app/services/kline_store.py#L42-L56)
- 保存当前同步状态：last_attempt/success_trade_date、status(idle/running/success/failed)、symbol_count、total_symbols/synced_symbols/success_symbols/failed_symbols、task_id、trigger_mode(auto/manual)、message

**kline_sync_tasks**：[kline_store.py#L67-L80](file:///workspace/app/services/kline_store.py#L67-L80)
- 每次同步任务一行：task_id（UUID 十六进制）、trigger_mode、trade_date、status、total_symbols/synced/success/failed、started_at/finished_at、message

**kline_sync_task_details**：[kline_store.py#L84-L93](file:///workspace/app/services/kline_store.py#L84-L93)
- 每只股票的同步明细：task_id + symbol、status(success/fail/skipped)、elapsed_ms、error_message
- 批量写入：`add_sync_task_details()` executemany 一次提交几百条

#### kline_check_reports（完整性检查报告表）

[kline_store.py#L102-L108](file:///workspace/app/services/kline_store.py#L102-L108)
- 保存 `check_data_integrity()` 的历史报告
- 报告 JSON 包含：覆盖率百分比、按日期缺失分布、最差个股 Top N
- 自动保留最近 50 份：`DELETE WHERE id NOT IN (SELECT id ORDER BY id DESC LIMIT 50)`

---

### 5.5 WAL checkpoint 机制

> **关于 macOS UEs 问题的交叉引用**：本节仅列出 checkpoint 三层机制的结构说明，涉及 UEs 的现象、根因链和唯一解法等完整讨论，**详见第 2 章 2.1 节 shutdown：先 checkpoint 再 cancel 与 macOS UEs 防护**。第15章调试与运维也有「UEs 进程卡死定位」专题。

#### 自动：lifespan shutdown 中

[main.py#L221-L270](file:///workspace/app/main.py#L221-L270)

uvicorn 收到 SIGTERM → lifespan yield 返回 → **先 checkpoint 再 cancel tasks**：
```python
await asyncio.to_thread(_checkpoint_all_sqlite)  # ← 步骤 1
# 对 data/funnel_state.db + market_kline.db 分别：
#   conn = sqlite3.connect(path)
#   conn.execute("PRAGMA wal_checkpoint(TRUNCATE)")
#   conn.close()
# 之后才遍历 7 个 task → cancel() → await（suppress CancelledError）
```

使用 `wal_checkpoint(TRUNCATE)` 而非默认 `PASSIVE`：
- `PASSIVE`：只把已就绪 WAL 页合并，不阻塞，可能不完全
- `TRUNCATE`：强制合并全部 WAL 到主 db，然后把 WAL 文件截短为 0 → 下次启动干净

#### 手动：重启脚本

[restart.sh#L18-L49](file:///workspace/restart.sh#L18-L49)

三步防 UEs：
1. **`prepare_app_shutdown()`**：`curl -X POST http://127.0.0.1:PORT/api/admin/shutdown-prepare`
   → 路由 `admin_shutdown_prepare` 在线程里 `PRAGMA wal_checkpoint(TRUNCATE)` 两个库
   → 应用进程主动清理连接
2. **`checkpoint_sqlite()`**：Python 内联脚本再次对两个库做 `wal_checkpoint(TRUNCATE)`
   → 防止第一步因 HTTP 超时或网络失败遗漏
3. **`stop.sh`**：发 SIGTERM → uvicorn 优雅退出 → lifespan shutdown 再做一遍 checkpoint + cancel

---

### 5.6 KlineSQLiteStore vs SQLiteStateStore

两个 Store 类职责严格单一、物理库隔离，**互不引用**：

| 维度 | KlineSQLiteStore | SQLiteStateStore |
|-----|-----------------|-----------------|
| **类定义** | [kline_store.py#L10-L820](file:///workspace/app/services/kline_store.py#L10-L820) | [sqlite_store.py#L9-L371](file:///workspace/app/services/sqlite_store.py#L9-L371) |
| **负责数据库** | `data/market_kline.db` | `data/funnel_state.db` |
| **主要写入者** | KlineCacheService 同步任务（每日批次） | FunnelService.tick / PaperTrading / HermesMemory / 各模块 KV 写入（盘中持续） |
| **主要读取者** | 策略扫描（QuietBreakout、CustomStrategyScanner）、Kronos 推理、BacktestLab、get_kline 路由 | FunnelService 启动 load_state、所有 /api 路由读快照、模拟盘查询、Agent 任务历史 |
| **核心方法（写）** | `upsert_symbol_klines` / `upsert_many_klines` / `record_sync_batch` / `set_sync_state` / `start_sync_task` / `upsert_symbol_names` | `save_state` / `upsert_single_active_strategy_profile` / `save_notice_state` / `set_kv` / `upsert_custom_strategy` |
| **核心方法（读）** | `get_kline` / `get_latest_snapshot` / `get_all_symbols` / `get_stats` / `load_symbol_names` / `get_sync_state` / `list_sync_tasks` / `get_existing_pairs` | `load_state` / `get_active_strategy_profile` / `load_notice_state` / `get_kv` / `list_custom_strategies` / `get_custom_strategy` / `get_default_custom_strategy` |
| **busy_timeout** | 有 `PRAGMA busy_timeout=1000`（因为写入批次大，锁等待容忍更长） | 无（默认 SQLite busy_timeout=0，获取不到锁就 SQLITE_BUSY 错误） |
| **_init_schema 表数** | 6 张：kline_daily / kline_sync_state / kline_sync_tasks / kline_sync_task_details / kline_check_reports / symbol_names | 5 张：funnel_state / strategy_profiles / notice_state / kv_store / custom_strategies（+ HermesMemory 和 PaperTrading 各自 _init_schema 再加 5 张到同一库） |

> **注意**：PaperTradingService 和 HermesMemory 虽然都有自己的 `_connect()` / `_init_schema()` 方法，但 **db_path 都指向 `data/funnel_state.db`**，所以实际 3 个 Store 类（SQLiteStateStore + PaperTradingService + HermesMemory）共享同一个物理 funnel_state.db 的 3 个连接池。通过 WAL 模式 + `journal_mode=WAL` 保证并发不冲突。

---

# 第二部分 服务层详解

---

## 第6章 数据服务

**本节导读**：第一部分完成了架构、启动、配置、模型、存储的"底座五件套"。从本章开始进入第二部分——服务层详解。数据服务是 Alpha 系统的基础数据源入口：AkshareDataProvider 负责统一封装多数据源和缓存降级、EastmoneyMarketDataClient 负责 async 硬超时和重试、KlineCacheService 是 K 线并发同步的调度大脑、KlineSQLiteStore 负责 SQLite 持久化读写。读完本章你将了解"实时行情从哪里来、K 线如何同步进本地、失败了怎么降级兜底"。

数据服务层是 Alpha 系统的基础数据源入口，负责统一封装多数据源（AkShare / 东方财富 / 新浪 / 同花顺）、提供实时行情缓存、管理交易日历、K 线数据的持久化存储与增量同步。

---

### 6.1 AkshareDataProvider

**文件位置**：[app/services/data_provider.py](file:///workspace/app/services/data_provider.py)

`AkshareDataProvider` 是一个 dataclass 风格的 provider，无状态（除了缓存字段），为上层服务提供"单一入口"的数据访问。

#### 多数据源适配

系统在不同场景下灵活切换数据源：

| 数据类型 | 主源 | 降级源 | 说明 |
|--------|------|--------|------|
| 实时行情快照 | 东方财富 `ak.stock_zh_a_spot_em` | 新浪 `ak.stock_zh_a_spot` | [_fetch_spot_em](file:///workspace/app/services/data_provider.py#L68-L87) |
| 概念板块 | 东方财富 HTTP 接口 | 同花顺概念名 | PredictFunnel 中使用 |
| 个股历史 K 线 | EastmoneyMarketDataClient（直连 push2） | AkShare `ak.stock_zh_a_hist` | KlineCacheService 中使用 |

#### 实时行情缓存

核心缓存字段：`realtime_snapshot_cache: tuple[datetime, pd.DataFrame]`（见 [L26](file:///workspace/app/services/data_provider.py#L26)），配合 `realtime_snapshot_source` 记录数据来源（`em_live` / `db_fallback` / `stale_cache` / `none`）。

`get_realtime_snapshot()` [L112-L150](file:///workspace/app/services/data_provider.py#L112-L150) 的决策链：

1. **缓存命中**：`cache_ttl_seconds`（默认 300s）内直接返回
2. **实时抓取**：盘中自动判断 `prefer_live=True`，重试东财 spot → 失败降级新浪 spot
3. **DB 兜底**：盘后或实时接口失败，调用 `_snapshot_from_db()` 从本地 K 线数据库构建类快照
4. **stale cache**：以上均失败，返回过期缓存或空 DataFrame

数据库兜底的字段映射见 [_snapshot_from_db](file:///workspace/app/services/data_provider.py#L32-L66)，会从最新一条 K 线计算 `涨跌幅 / 涨跌额`。

#### 交易日历

`async def get_trade_days(self, min_days: int = 0) -> pd.DataFrame` [L243](file:///workspace/app/services/data_provider.py#L243)

优先调用 AkShare 的 `tool_trade_date_hist_sina()`（新浪交易日历），缓存到实例变量中。返回 DataFrame 包含 `trade_date` 列（YYYY-MM-DD 字符串格式）。

如果交易日历获取失败，上层服务会退化到"weekday < 5"的启发式判断（例如 KronosPredictService）。

#### 数据字段统一

所有 snapshot 的列名统一为东财原始格式，确保上层 `market_row.get("最新价")` 等调用无需适配：

| 统一列名 | 含义 |
|---------|------|
| 代码 | 股票代码（6 位字符串） |
| 名称 | 股票简称 |
| 最新价 | 当前价 |
| 今开 | 今日开盘价 |
| 昨收 | 昨日收盘价 |
| 最高 | 今日最高价 |
| 最低 | 今日最低价 |
| 成交额 | 成交额（元） |
| 成交量 | 成交量（手） |
| 涨跌幅 | 涨跌幅（%） |

归一化入口：`_normalize_snapshot(df)` 负责把新浪 / 东财的不同列名重命名到上述标准。

#### Symbol 归一化

`def normalize_symbol(value: Any) -> str` [L630-L634](file:///workspace/app/services/data_provider.py#L630-L634)

```python
def normalize_symbol(value: Any) -> str:
    raw = str(value or "").strip().upper()
    if raw.startswith(("SZ", "SH", "BJ")) and len(raw) > 2:
        raw = raw[2:]
    return raw
```

- 去掉交易所前缀 `SZ/SH/BJ`
- 统一大写
- 不自动补零（由各数据源入口自行 `zfill(6)`）

---

### 6.2 EastmoneyMarketDataClient

**文件位置**：[app/services/market_data_client.py](file:///workspace/app/services/market_data_client.py)

专门为 K 线缓存同步设计的 async HTTP 客户端，避开 AkShare 阻塞请求路径。

#### Async HTTP 客户端

基于 `httpx.AsyncClient`，所有网络调用通过 `httpx` 严格配置 connect/read/write/pool 超时：

```python
self.timeout = timeout or httpx.Timeout(connect=2.0, read=8.0, write=2.0, pool=2.0)
```

见 [__init__](file:///workspace/app/services/market_data_client.py#L67-L85)，`trust_env=False` 避免代理干扰，`follow_redirects=True` 跟随重定向。

#### 硬超时 + 有限重试（指数退避）

`_get_json()` [L96-L109](file:///workspace/app/services/market_data_client.py#L96-L109) 和 `_get_text()` [L111-L123](file:///workspace/app/services/market_data_client.py#L111-L123) 均实现：

- 重试次数：`retries`（默认 2 次）
- 退避策略：`0.25 * (attempt + 1)` 秒等待
- 失败处理：不抛异常，返回 `{}` 或 `""`，由上层统计失败明细

#### 交易日历 / 实时行情专用接口

| 方法 | 用途 | URL 配置 |
|------|------|---------|
| `fetch_hist(symbol, start, end, adjust)` | 拉取个股历史 K 线 | `push2his.eastmoney.com/api/qt/stock/kline/get`（双域名容灾）[L57-L59](file:///workspace/app/services/market_data_client.py#L57-L59) |
| `fetch_spot(page, page_size)` | 拉取全市场 spot 分页 | `push2.eastmoney.com/api/qt/clist/get`（双域名容灾）[L61-L64](file:///workspace/app/services/market_data_client.py#L61-L64) |
| `fetch_trade_days()` | 新浪交易日历 txt | `finance.sina.com.cn/realstock/company/klc_td_sh.txt` [L65](file:///workspace/app/services/market_data_client.py#L65) |

历史 K 线的市场代码自动判断：`6/68` 开头 → `market_code=1`（沪市），否则 → `market_code=0`（深市），见 [fetch_hist L135](file:///workspace/app/services/market_data_client.py#L135)。

---

### 6.3 KlineCacheService

**文件位置**：[app/services/kline_cache_service.py](file:///workspace/app/services/kline_cache_service.py)

K 线同步调度服务，采用"任务队列 + 单 consumer"架构，避免多并发调度导致数据库冲突。

#### 架构：任务队列 + 单 Consumer Task

```python
self._queue: list[tuple[str, dict[str, Any]]] = []    # 任务队列
self._queue_task: asyncio.Task | None = None          # 消费者 Task
```

见 [L36-L37](file:///workspace/app/services/kline_cache_service.py#L36-L37)。

消费者逻辑在 `_run_queue()` [L113-L123](file:///workspace/app/services/kline_cache_service.py#L113-L123)：
- 循环 pop 队首任务
- 根据 `job_type` 分发到 `incremental_sync` 或 `sync_trade_date`
- 单任务异常不影响后续，写入 `_fail_stale_running_tasks` 记录

#### 并发控制

| 常量 | 值 | 含义 |
|------|---|------|
| `_CONCURRENCY` | 8 | 抓数据时的 asyncio.Semaphore 并发数 |
| `_FETCH_BATCH_SIZE` | 200 | 收集 200 只后批量写库一次 |

见 [L17-L18](file:///workspace/app/services/kline_cache_service.py#L17-L18)。

同步流程：`Semaphore(8)` 并发调用 EastmoneyMarketDataClient → 收集 results → 满 200 条触发 `store.upsert_many_klines()` → 最后 flush 剩余。

#### 三种同步模式

**(1) 全量补缺：enqueue_sync_trade_date** [L39-L54](file:///workspace/app/services/kline_cache_service.py#L39-L54)

对目标交易日 × 所有股票做笛卡尔补缺：把 `window_days`（默认 180）窗口内的所有 K 线缺口补齐。适用于：
- 首次初始化数据库
- 长时间停机后追数据

**(2) 增量同步：enqueue_incremental_sync** [L56-L67](file:///workspace/app/services/kline_cache_service.py#L56-L67)

只补最新一个交易日的缺口。适用于每日收盘后的自动调度（默认 15:20 触发，由 `run_if_due()` 检查）。

**(3) 日期范围批量入队：enqueue_incremental_range** [L69-L98](file:///workspace/app/services/kline_cache_service.py#L69-L98)

把 `start_date` ~ `end_date` 范围内的所有工作日拆成多个 `incremental` 任务逐个入队。适用于手动指定回补范围。

#### 失败处理

单只股票抓取失败不会阻塞批次：
- 每只股票的状态（success/failed）写入 `kline_sync_task_details` 表
- error_message 记录异常摘要（最多截断）
- 最终统计 success_symbols / failed_symbols 写入 `kline_sync_state` 和 `kline_sync_tasks`

#### 进度跟踪

- `get_sync_state()` [L740](file:///workspace/app/services/kline_cache_service.py#L740)：返回最近一次同步的整体状态（status / trigger_mode / 各 count）
- `get_sync_progress()` [L788](file:///workspace/app/services/kline_cache_service.py#L788)：返回运行中任务的实时进度（已同步 N/总数 M）

---

### 6.4 KlineStore / KlineSQLiteStore

**文件位置**：[app/services/kline_store.py](file:///workspace/app/services/kline_store.py)

SQLite 存储层，使用 WAL 模式保证读写并发。所有写操作走事务，读操作走只读连接。

> **关于表结构的交叉引用**：本节仅列出核心读写方法。`kline_daily` / `kline_sync_state` / `kline_sync_tasks` / `kline_sync_task_details` / `kline_check_reports` / `symbol_names` 六张表的完整字段定义**对应表结构详见第5章 5.4节 market_kline.db 表结构**。

#### get_kline(symbol, days)

`def get_kline(self, symbol: str, days: int = 30) -> list[dict[str, Any]]` [L209](file:///workspace/app/services/kline_store.py#L209)

按日查询指定股票的最近 N 根 K 线，按日期升序返回。每一行包含：

```python
{
  "date": "YYYY-MM-DD",
  "open": float, "high": float, "low": float, "close": float,
  "volume": float, "amount": float
}
```

#### get_all_symbols()

`def get_all_symbols(self) -> list[str]` [L696](file:///workspace/app/services/kline_store.py#L696)

从 `kline_daily` 表 `SELECT DISTINCT symbol` 获取全部股票代码。供策略扫描器（QuietBreakoutScanner / CustomStrategyScanner）遍历全 A 股。

#### 批量写库

`def upsert_many_klines(self, items, updated_at)` [L167](file:///workspace/app/services/kline_store.py#L167)

一次事务批量写入多只股票的多行 K 线，使用 `INSERT ... ON CONFLICT DO UPDATE` 语义保证 upsert：

```sql
INSERT INTO kline_daily(symbol, trade_date, open, high, low, close, volume, amount, updated_at)
VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
ON CONFLICT(symbol, trade_date) DO UPDATE SET
    open=excluded.open, high=excluded.high, ...
```

#### load_symbol_names()

`def load_symbol_names(self) -> dict[str, str]` [L775](file:///workspace/app/services/kline_store.py#L775)

从 `symbol_names` 表加载 `symbol → name` 映射。数据来源是每次实时 snapshot 抓取时 `_ensure_names_from_snapshot()` 自动写入。

---

## 第7章 策略服务

**本节导读**：上一章讲了数据从哪来、怎么存，本章讲"选股大脑"怎么运转——StrategyEngine 负责盘中评分和迁池规则，FunnelService 是三池漏斗的调度外壳，自定义策略中心提供 12 条原子规则 + AND 组合能力，BacktestLab 用于验证策略在历史上的表现。本章是整个系统"选股决策逻辑"的核心。

策略服务层是 Alpha 的"选股大脑"，包含三层能力：
1. **核心引擎**：`compute_intraday_score` 盘中实时评分 + `apply_transition_rules` 迁池
2. **漏斗服务**：`FunnelService` 调度 + 状态持久化 + 对外 API
3. **自定义策略中心**：12 条原子规则 + AND 组合 + 回测

---

### 7.1 StrategyEngine

**文件位置**：[app/services/strategy_engine.py](file:///workspace/app/services/strategy_engine.py)

纯函数模块（无副作用），便于单测与热切换。

#### compute_intraday_score — 核心盘中评分

`def compute_intraday_score(entry, market_row, elapsed_ratio, config) -> tuple[score, breakdown, metrics, warnings]` [L28-L103](file:///workspace/app/services/strategy_engine.py#L28-L103)

**入参**：
- `entry`：股票在漏斗中的当前配置（含 `breakout_level` 突破基准价、`avg_amount20` 20 日均额）
- `market_row`：从 snapshot 获取的当前行情（`最新价 / 今开 / 昨收 / 最高 / 成交额 / 成交量 / 涨跌幅`）
- `elapsed_ratio`：时间进度（0~1），9:30=0, 15:00=1，用于量能归一化
- `config`：StrategyConfig，权重与阈值在此配置

**返回**：
- `score`：最终分 0~100
- `breakdown`：四个子维度拆解 {breakout_strength, volume_quality, intraday_structure, risk_penalty}
- `metrics`：原始指标明细（供前端展示与迁池判断）
- `warnings`：风险提示列表（高开过大 / 冲高回落 / 近涨停）

##### 评分算法拆解

总分满分 100 = 突破强度35 + 量能25 + 结构分20 - 减分项20（动态）。

① **突破强度（权重 35）**
```python
breakout_ratio = (price / max(breakout_level, 0.01)) - 1
breakout_score = clamp(breakout_ratio / 0.03, 0, 1) * config.score_weight_breakout
```
见 [L45-L46](file:///workspace/app/services/strategy_engine.py#L45-L46)。

含义：相对突破基准价上涨 3% 即拿满突破分。如果 price 低于 breakout_level，breakout_score=0。

② **量能质量（权重 25）**
```python
expected_amount = avg_amount20 * max(elapsed_ratio, 0.01)
volume_ratio = amount / max(expected_amount, 1)
volume_score = clamp(volume_ratio / 2, 0, 1) * config.score_weight_volume
```
见 [L48-L50](file:///workspace/app/services/strategy_engine.py#L48-L50)。

先按时间进度"年化"今日成交额（10:30 只开市 1h，expected_amount = 20日均额 × (1h / 4h)），再和 2 倍比，超过 2 倍拿满分。

③ **日内结构分（合计权重 20）**
三项相加，不设 clamp，每项独立：
- VWAP 之上：`score_weight_above_vwap`（默认 6）
- 收盘价 ≥ 开盘价：`score_weight_close_ge_open`（默认 6）
- 回撤控制：`clamp((0.03 - drawdown_from_high) / 0.03, 0, 1) * score_weight_drawdown`（默认 8）
见 [L52-L60](file:///workspace/app/services/strategy_engine.py#L52-L60)。

④ **风险减分项**
- 高开过大：`clamp((gap_up - 0.03) / 0.05, 0, 1) * penalty_gap_up`（gap_up > 3% 开始扣分，8% 扣满）
- 冲高回落：`clamp((drawdown_from_high - 0.03) / 0.04, 0, 1) * penalty_drawdown`（回撤 > 3% 开始扣，>7% 扣满）
- 近涨停：`pct_change >= near_limit_pct` → `penalty_near_limit`（涨停板流动性差，打板风险）
见 [L62-L67](file:///workspace/app/services/strategy_engine.py#L62-L67)。

#### apply_transition_rules — 迁池规则引擎

`def apply_transition_rules(entry, config) -> dict` [L106](file:///workspace/app/services/strategy_engine.py#L106)

基于"连续分钟窗口"判断，避免单次评分抖动导致误迁池：

| 计数器 | 触发条件 | 用途 |
|--------|---------|------|
| `above60_count` | score ≥ focus_score_threshold | candidate → focus 的连续判定 |
| `breakout_confirm_count` | 同时满足：price ≥ breakout_level×buffer + 量比达标 + VWAP 之上 | candidate/focus → buy 的连续判定 |
| `below65_count` | 在 buy 池但 score < downgrade_score_threshold | buy → 降级的连续判定 |

**迁池路径**：
- **candidate → focus**：score 立即达 `focus_score_immediate`，或 `above60_count` 连续 ≥ `focus_consecutive_minutes` 分钟
- **candidate/focus → buy**：`breakout_confirm_count` 连续 ≥ `buy_consecutive_minutes` 分钟
- **buy → focus/candidate**：`below65_count` 连续 ≥ 阈值（触发降级回重点池）

每次推荐迁池时会写入 `trigger_log`（FIFO 120 条），供诊断面板回放。

#### get_last_n_trade_window

`def get_last_n_trade_window(trade_days_df, base_date, n) -> tuple[str, str]` [L16-L25](file:///workspace/app/services/strategy_engine.py#L16-L25)

从交易日历 DataFrame 中取 base_date 前推 n 个交易日，返回 `(start_yyyymmdd, end_yyyymmdd)`。用于计算 20 日均额等需要滚动窗口的统计量。

---

### 7.2 FunnelService

**文件位置**：[app/services/funnel_service.py](file:///workspace/app/services/funnel_service.py)

策略漏斗的对外服务封装：持有 entries 状态、调度 recompute、对接 RealtimeHub 广播。

#### 状态模型

```python
self.entries: dict[str, dict[str, Any]] = {}      # 核心：symbol → entry
self.hot_concepts: list[dict[str, Any]] = []       # 热门概念（TTL 300s）
self.hot_stocks: list[dict[str, Any]] = []         # 热门个股（TTL 300s）
self.trade_date: str                               # 当前交易日（跨日触发重置）
```

entry 典型结构：
```python
{
  "symbol": "600519",
  "name": "贵州茅台",
  "pool": "candidate",     # candidate / focus / buy
  "score": 72.3,
  "breakout_level": 1680.0,
  "avg_amount20": 8_500_000_000,
  "breakdown": {...},
  "metrics": {...},
  "transitions": {"above60_count": 3, ...},
  "trigger_log": [...]
}
```

#### tick() 主循环入口

`async def tick(self) -> bool` [L562-L597](file:///workspace/app/services/funnel_service.py#L562-L597)

由全局 scheduler 每 60 秒调用一次，流程：

1. **ensure_trade_date 跨日重置**：今天日期 != trade_date → `_reset_state()` 清空 entries，触发一次全新盘初扫描
2. **获取全市场 snapshot**：cache 优先 → 实时源 → DB 兜底（由 `_pick_screen_snapshot()` 封装）
3. **遍历 entries 评分**：对每个 entry 调用 `compute_intraday_score` 更新 score / breakdown / metrics
4. **apply_transition_rules 迁池**：更新 transitions 计数器，推荐池变化时写入 trigger_log 并生效
5. **刷新热门概念/个股**：`_is_hot_stocks_stale(ttl_seconds=300)` 判定，TTL 默认 5 分钟
6. **_save_state() 持久化**：通过 SQLiteStateStore 把整个 entries + hot_* 序列化写入 funnel_state.db
7. **返回 changed bool**：True 表示有迁池或有数据更新，上游 RealtimeHub 触发 WS 广播

盘后（`is_after_close()` 为 True）执行一次 `recompute()` 后 `frozen=True`，避免盘后伪数据。

#### 对外 API

| 方法 | 行号 | 说明 |
|------|------|------|
| `get_funnel(trade_date)` | [L362](file:///workspace/app/services/funnel_service.py#L362) | 返回三池列表 + 统计 + 策略配置 |
| `get_stock_detail(symbol, kline_days)` | [L471](file:///workspace/app/services/funnel_service.py#L471) | 单只股票的 entry + K 线 + 概念标签 |
| `move_pool(symbol, target_pool, note)` | [L332](file:///workspace/app/services/funnel_service.py#L332) | 手动迁池，写入 manual_moved_at 标记 |
| `recompute(symbol=None)` | [L357](file:///workspace/app/services/funnel_service.py#L357) | 单只或全部重新评分 + 迁池判定 |

#### SQLiteStateStore 依赖

状态通过 `_save_state()` [L104-L114](file:///workspace/app/services/funnel_service.py#L104-L114) 和 `_load_state()` [L68-L88](file:///workspace/app/services/funnel_service.py#L68-L88) 序列化：

- 保存：把 `{trade_date, entries, hot_concepts, hot_stocks, updated_at, ...}` 整体 JSON 化 → SQLite KV 表
- 加载：校验 trade_date 匹配，否则丢弃（每日重置）
- 兼容：支持从旧版 `data/funnel_state.json` 迁移导入

---

### 7.3 自定义策略中心

**文件位置**：[app/services/strategy_rules.py](file:///workspace/app/services/strategy_rules.py) + [app/services/custom_strategy.py](file:///workspace/app/services/custom_strategy.py)

#### 四个基础 Dataclass

全部定义在 [strategy_rules.py L22-L104](file:///workspace/app/services/strategy_rules.py#L22-L104)：

| 类 | 用途 | 关键字段 |
|----|------|---------|
| `RuleContext` | 评估单只股票时的上下文 | symbol, name, kline（升序）, trade_date |
| `RuleResult` | 规则评估结果 | passed, label, metric, detail |
| `RuleParam` | 规则参数定义（前端表单渲染） | key, label, type, default, min/max, step |
| `RuleSpec` | 一条规则的完整定义 | code, title, category, params, evaluator, min_kline_days |

`RuleSpec.evaluate(ctx, params)` 会自动：
1. 检查 K 线长度是否满足 `min_kline_days`
2. 合并 default_params 与用户覆盖
3. try/except 捕获 evaluator 异常 → 返回 passed=False

#### RULE_REGISTRY：12 条原子规则

`RULE_REGISTRY: dict[str, RuleSpec] = _build_registry()` [L554](file:///workspace/app/services/strategy_rules.py#L554)

按 5 类组织：

| 类别 | 规则 code | 说明 |
|------|-----------|------|
| **price** | `price_range` | 股价区间过滤（默认 3~60 元） |
| **price** | `change_pct_today` | 当日涨跌幅范围 |
| **volume** | `volume_spike_today` | 今日放量倍数（vs N 日均量） |
| **volume** | `volume_shrink` | 缩量横盘：N 日量变异系数 CV ≤ 阈值 |
| **pattern** | `limit_up_today` | 今日涨停（±tolerance 容差） |
| **pattern** | `box_consolidation` | N 日箱体振幅 ≤ max_amp_pct |
| **pattern** | `ma_above` | 均线多头排列（短/中/长） |
| **pattern** | `no_near_limit_down` | 近期未出现跌停 |
| **trend** | `above_ma` | 收盘价 > N日均线 |
| **trend** | `rsi_range` | RSI 区间（避免超买） |
| **filter** | `exclude_boards` | 排除 ST / 创业板 / 科创板 / 北交所 |
| **filter** | `market_cap_filter` | 流通市值范围 |

#### 规则组合策略：AND 组合

`CustomStrategy` 数据模型在 [custom_strategy.py L53-L92](file:///workspace/app/services/custom_strategy.py#L53-L92)，包含：
- `rules: list[StrategyRuleRef]`：每条 rule_ref 含 `rule_code / enabled / params`
- `enabled_rules()`：过滤出 enabled=True 且在 REGISTRY 中存在的规则

**执行语义（AND 模式）**：所有启用规则必须全部 passed=True，股票才入选。任何一条失败即跳过。

综合分 `composite_score`：
```
命中规则数 × 10 + 辅助分（放量倍数 + 箱体紧密度 + 涨停加分）
```

#### CustomStrategyScanner.scan()

`class CustomStrategyScanner` [L217](file:///workspace/app/services/custom_strategy.py#L217)，`async def scan()` [L233](file:///workspace/app/services/custom_strategy.py#L233)。

扫描流程（全 A 股 5000 只 ~30 秒）：
1. `store.get_all_symbols()` 获取全部股票
2. 用 `ThreadPoolExecutor`（`asyncio.to_thread` + 内部分片）并发读 K 线
3. 对每只股票：**顺序跑所有启用规则**，失败则 early-return
4. 命中后计算 composite_score，收集所有 rule_hits（前端展示每条规则如何判定）
5. 按综合分排序 Top N 返回

#### 3 条内置策略

在 `_builtin_strategies_seed()` [custom_strategy.py L125-L150](file:///workspace/app/services/custom_strategy.py#L125-L150) 定义（id 固定，便于幂等写入 DB）：

| id | 名称 | 核心规则组合 |
|----|------|-------------|
| `builtin_quiet_breakout` | 缩量启动（内置） | 排除ST + 价格3~60 + 箱体20%/25日 + 缩量CV≤0.40/25日 + 放量≥3x + 涨停 |
| `builtin_adjustment_box` | 调整期横盘（内置） | 缩量 + 箱体窄幅 + 未破前高 → 发现调整末期 |
| `builtin_breakout_volume` | 突破放量（内置） | 箱体收敛 + 今日量比≥2 + 突破N日高点 |

---

### 7.4 BacktestLab

**文件位置**：[app/services/backtest_lab.py](file:///workspace/app/services/backtest_lab.py)

策略回测实验室：对自定义策略做"历史 N 日滚动窗口 + 锚点买入信号 + hold_days 持有期"的批量回测。

#### run_custom_strategy()

`async def run_custom_strategy(...)` [L233](file:///workspace/app/services/backtest_lab.py#L233)

典型配置：`lookback=180` 天，对每个交易日作为锚点，跑一次 `CustomStrategyScanner.scan()`，记录：
- 每日 TopK 命中信号
- 每只信号股的后续收益（hold_days 期间）

#### 输出指标

`BacktestResult` dataclass [L12-L41](file:///workspace/app/services/backtest_lab.py#L12-L41) 字段：

| 字段 | 含义 |
|------|------|
| `total_signals` | 信号总数 |
| `wins / losses` | 胜/负次数 |
| `hit_rate` | 胜率 = wins / total |
| `total_pnl_pct` | 累计收益（按等权分配的简单平均） |
| `max_drawdown_pct` | 最大回撤（累计收益曲线的历史高点回撤） |
| `avg_hold_days` | 平均持仓天数 |
| `samples[:50]` | 样本明细（前端只展示 50 条） |

#### 交易约束

回测引擎统一的退出条件（按优先级）：

| 条件 | 参数默认值 | 行为 |
|------|-----------|------|
| 持有到期 | hold_days = 3 | 第 3 日收盘卖出（exit_reason="hold_end"） |
| 止盈 | tp_pct = 8.0% | 日内最高价 ≥ entry_price × (1+8%)，以 tp 价成交（"tp"） |
| 止损 | sl_pct = -5.0% | 日内最低价 ≤ entry_price × (1-5%)，以 sl 价成交（"sl"） |

进场价严格取"信号日次日开盘价"（`rows[anchor+1]["open"]`），模拟真实"选股后第二日才能买入"的约束，避免未来函数。

---

## 第8章 AI服务

**本节导读**：策略服务是规则驱动的"硬逻辑"，本章引入 AI 增强能力。KronosPredictService 封装金融时序模型做 K 线预测（用惰性加载、设备自适应、串行锁、GIL 隔离四件套解决工程化问题），HotStockAIService 叠加 Kronos 预测和 TradingAgents 多代理讨论给热门股做智能分层，FirstLimitAlphaService 是首板连板的离线研究模块（7 步研发管线完整封装）。

AI 服务层为 Alpha 提供三个核心预测与智能选股能力：
1. **Kronos 三日预测**：时序生成模型预测个股未来 K 线
2. **热门股 AI 评分**：热度 + Kronos + TradingAgents 多维度打分
3. **首板连板 Alpha**：收盘后决策模型（离线研究模块）

---

### 8.1 KronosPredictService

**文件位置**：[app/services/kronos_predict_service.py](file:///workspace/app/services/kronos_predict_service.py)

封装 `Kronos-base` 时序模型的推理服务，解决"模型加载时机、设备选择、并发安全、不阻塞事件循环"四大工程问题。

#### 设计四件套

① **惰性加载（Lazy Loading）**

`async def _ensure_model()` [L57-L69](file:///workspace/app/services/kronos_predict_service.py#L57-L69)

```python
async def _ensure_model(self):
    if self._predictor is not None:
        return self._predictor
    self._loading = True
    try:
        predictor, device = await asyncio.to_thread(self._load_model)
        self._predictor = predictor
        self._device = device
        return predictor
    finally:
        self._loading = False
```

首次 `predict()` 请求才真正下载权重 + 初始化模型，服务启动时不占用 GPU/内存。模型 ID 配置在模块顶部：
```python
_MODEL_ID = "NeoQuasar/Kronos-base"
_TOKENIZER_ID = "NeoQuasar/Kronos-Tokenizer-base"
```
见 [L14-L16](file:///workspace/app/services/kronos_predict_service.py#L14-L16)。

② **设备自适应**

`_load_model()` [L71-L80](file:///workspace/app/services/kronos_predict_service.py#L71-L80) 中：
```python
predictor = KronosPredictor(model, tokenizer, device=None, max_context=_MAX_CONTEXT)
return predictor, predictor.device
```

`device=None` 让 KronosPredictor 内部按优先级自选：
- 有 CUDA → CUDA
- 有 Apple MPS → MPS
- 否则 → CPU

实际设备字符串保存在 `self._device`，返回到响应 JSON 便于前端诊断。

③ **串行推理锁**

`self._lock = asyncio.Lock()` [L26](file:///workspace/app/services/kronos_predict_service.py#L26)

GPU 不擅长并发推理，同时多请求会导致显存 OOM。所有 `predict()` 调用先获取 Lock，串行排队执行。

④ **GIL 隔离**

推理过程是 CPU/GPU 密集的纯 Python + Torch 代码，直接跑在 asyncio 线程里会阻塞事件循环。通过 `asyncio.to_thread()` 解耦：

```python
pred_df = await asyncio.to_thread(
    self._run_inference, predictor, history, future_dates, horizon
)
```
见 [L44-L46](file:///workspace/app/services/kronos_predict_service.py#L44-L46)。

#### 推理流程

完整流程在 `async def predict(symbol, lookback=180, horizon=3)` [L31-L47](file:///workspace/app/services/kronos_predict_service.py#L31-L47)：

**Step 1 — 入参校验**
```python
lookback = max(10, min(lookback, 240))   # 10 ≤ lookback ≤ 240
horizon = max(1, min(horizon, 10))       # 1 ≤ horizon ≤ 10
```

**Step 2 — 读历史 K 线**
```python
history = self._get_history(symbol, lookback)
# → self._kline_store.get_kline(symbol, days=lookback)
```
历史长度不足会抛出明确异常，提示先同步 K 线。

**Step 3 — 交易日推算**
`async def _get_future_trade_days(history, horizon)` [L89-L106](file:///workspace/app/services/kronos_predict_service.py#L89-L106)：
- 优先：AkShare trade_days_df（精准，包含节假日）
- 退化：`weekday() < 5` 启发式（周一到周五），保证即使日历接口挂了也能返回结果

**Step 4 — 模型推理**
`_run_inference()` [L110-L130](file:///workspace/app/services/kronos_predict_service.py#L110-L130)：
```python
predictor.predict(
    df=x_df.reset_index(drop=True),
    x_timestamp=x_timestamp,
    y_timestamp=y_timestamp,
    pred_len=horizon,
    T=1.0,            # 温度系数
    top_k=0,
    top_p=0.9,        # nucleus sampling
    sample_count=100, # 采样 100 次取统计
    verbose=False,
)
```

**Step 5 — 构建响应**
`_build_response()` 输出四个核心字段：
- `history_kline`：用户请求的 lookback 根历史 K 线（四舍五入 4 位）
- `predicted_kline`：horizon 根预测 K 线（含 open/high/low/close/volume/amount/date）
- `merged_kline`：history + predicted 合并序列（前端图表直接用）
- `prediction_start_index`：预测开始在 merged 中的下标（便于前端画分隔线）
- `device`：`self._device`，诊断信息

---

### 8.2 HotStockAIService

**文件位置**：[app/services/hot_stock_ai_service.py](file:///workspace/app/services/hot_stock_ai_service.py)

面向"实时热门股"的 AI 评分服务：把 `/api/market/hot-stocks` Top20 热门股，逐股叠加多维度 AI 评分，输出三池分层（候选/重点/买入）。

#### 逐股评分维度

每只股票的总分由以下因子加权累加：

| 因子 | 权重方向 | 说明 |
|------|---------|------|
| **热度** | + | 在热门榜单的名次（Top1 加分多） |
| **涨幅** | + | 今日涨跌幅（适中加分，过高则触发冲高回落扣分） |
| **趋势** | + | 短中长均线方向（多头发散加分） |
| **量额比** | + | 成交额 / 20日均额（放量加分） |
| **Kronos 三日预测** | + | pred_max_high_pct（三日最高涨幅）线性加分 |
| **TradingAgents 评级** | ± | BUY/OVERWEIGHT/HOLD/UNDERWEIGHT/SELL 映射 ±分 |

#### TradingAgents 深度讨论

实际调用封装在 [app/services/tradingagents_adapter.py](file:///workspace/app/services/tradingagents_adapter.py)。

`TradingAgentsAdapter.analyze(symbol, trade_date, ...)` [L79-L100](file:///workspace/app/services/tradingagents_adapter.py#L79-L100)：

**执行方式**：命令行 `uv run` 启动 TradingAgents 子进程
```bash
uv run python -m cli.main analyze \
  --ticker 600519.SS \
  --date 2026-04-15 \
  --provider deepseek \
  --quick-model deepseek-chat \
  --deep-model deepseek-reasoner \
  --analysts market,news,fundamentals \
  --output {tempfile.json}
```

**模型配置**：
```python
DEEPSEEK_PROVIDER = "deepseek"
DEFAULT_QUICK_MODEL = "deepseek-chat"       # 快速模型（市场/资讯分析）
DEFAULT_DEEP_MODEL  = "deepseek-reasoner"   # 深度推理模型（基本面估值）
```
见 [L11-L14](file:///workspace/app/services/tradingagents_adapter.py#L11-L14)。

**决策映射加减分**：`_decision_bonus()` [L57-L65](file:///workspace/app/services/tradingagents_adapter.py#L57-L65)

| decision | bonus |
|----------|-------|
| BUY | +2.0 |
| OVERWEIGHT | +1.0 |
| HOLD | 0.0 |
| UNDERWEIGHT | -1.0 |
| SELL | -2.0 |

同时 `_decision_action()` [L68-L77](file:///workspace/app/services/tradingagents_adapter.py#L68-L77) 映射到中文标签（买入/观望/卖出），供前端展示。

#### 三池拆分阈值

默认配置在 `DEFAULT_CONFIG` [L31-L48](file:///workspace/app/services/hot_stock_ai_service.py#L31-L48)：

| 池 | 阈值 | 默认值 |
|----|------|--------|
| candidate | ≥ threshold_candidate | 8.0 分 |
| focus | ≥ threshold_focus | 11.5 分 |
| buy | ≥ threshold_buy | 14.5 分 |

`threshold_candidate < threshold_focus < threshold_buy`，`update_config()` 中强制校验单调。

#### 自动刷新的轻量模式

`AUTO_LIGHT_TOP_N = 12` [L50](file:///workspace/app/services/hot_stock_ai_service.py#L50)。

当 `auto` 模式（非手动触发）运行时：
- TopN 从默认 20 缩减到 12
- 跳过 Kronos 预测（省 GPU 推理时间）
- 跳过 TradingAgents 深度讨论（省 LLM 成本和 240s 超时）

保证站点每 5 分钟刷新能在合理时间内完成，不影响整体响应。

#### 进度跟踪

统一 progress dict 结构 [L79-L86](file:///workspace/app/services/hot_stock_ai_service.py#L79-L86)：

```python
self.progress: dict[str, Any] = {
    "phase": "idle",        # init / boards / constituents / predict / persisted / done / error
    "current": 0,           # 当前处理第几只
    "total": 0,             # 总股票数
    "detail": "",           # 当前股票名 + 状态描述
    "started_at": None,     # ISO datetime
    "finished_at": None,    # ISO datetime
}
```

前端通过轮询 `/api/hot-stock-ai/snapshot` 实时显示进度条。

---

### 8.3 FirstLimitAlphaService

**文件位置**：[app/services/first_limit_alpha_service.py](file:///workspace/app/services/first_limit_alpha_service.py)

**模块定位**：首板介入连板预测（收盘后决策，次日开盘介入）。
研究链路完整，但当前是**离线研究模块**，未接入盘中评分流水线。

#### 七个编排步骤

典型流程 `async def run_pipeline(...)` 会依次执行：

| Step | 模块 | 产物 |
|------|------|------|
| 1. dataset build | `FirstLimitDataBuilder` | 从 KlineStore 抽取全部"首板日"样本（T 日涨停 + T-1 日未涨停） |
| 2. features build | `FirstLimitFeatureBuilder` | 特征工程：量比/振幅/封板时间/封单强度/换手/龙虎榜/概念热度 等 30+ 特征 |
| 3. baseline train | `FirstLimitBaselineTrainer` | 训练 LightGBM/XGBoost 基线模型，输出 feature importance |
| 4. sequence train | `FirstLimitSequenceTrainer` | 训练 Transformer/LSTM 序列模型（输入前 N 日 K 线序列） |
| 5. inference run | `FirstLimitInferenceEngine` | 对当日所有首板股打分，输出 p_continue_limit / p_strong_3d / p_fail |
| 6. backtest | `FirstLimitBacktester` | 回测历史：假设"首板次日开盘买，连板打开卖"，统计胜率/收益/回撤 |
| 7. persist | `self._save_graphic_state()` | 把三池结果写入 KV 表，供前端图形化分析页展示 |

所有模块从 `strategy.first_limit_alpha` 包导入 [L14-L28](file:///workspace/app/services/first_limit_alpha_service.py#L14-L28)，确保研究代码与运行时代码同一套实现，避免数据偏移。

#### 打分产物

每只首板股返回四个核心分数：

| 指标 | 含义 | 用途 |
|------|------|------|
| `p_continue_limit` | 次日继续涨停概率 | 决定是否介入次日集合竞价 |
| `p_strong_3d` | 未来 3 日累计涨幅 ≥ 15% 概率 | 重点关注池筛选 |
| `p_fail` | 次日低开或断板亏损概率 | 风险过滤 |
| `first_limit_score` | 综合分 = 加权线性组合 | 总分排序 / 三池分层 |

三池阈值（`DEFAULT_GRAPHIC_CONFIG` [L30-L36](file:///workspace/app/services/first_limit_alpha_service.py#L30-L36)）：
- candidate ≥ 6.0
- focus ≥ 10.0
- buy ≥ 14.0

#### 离线研究说明

当前该服务主要用于：
- 收盘后手动触发 `POST /api/first-limit-alpha/run` 生成本日分析报告
- `data/first_limit_alpha/` 目录下保存 versioned 的数据集、模型权重、回测报告
- 前端"首板研究"页展示历史首板池收益曲线、命中率、混淆矩阵

**暂未接入盘中评分的原因**：
- 首板信号只在收盘后才能确认（盘中无法确定最终封板）
- 特征工程需要当日完整龙虎榜、封板时长等盘后数据
- 介入时机是"次日开盘"，盘后决策即可，不需要实时

---

## 第9章 Agent服务

**本节导读**：前两章的策略和 AI 服务输出了选股结果，但系统还需要一个"自主进化"的智能体来诊断问题、给出建议——这就是本章的 Hermes Agent 体系。HermesRuntime 是执行器（双模式自动探测 + 三重保护 + 三任务类型），HermesMemory 是持久化层（三张表），MCP Server 通过标准协议把 Alpha 的 REST API 暴露为 16 个工具，供 Agent 调用。

> **关于表用途的交叉引用**：HermesMemory 的 `agent_tasks` / `agent_monitor_config` / `agent_monitor_messages` 三张表，完整字段定义**对应表结构详见第5章 5.3节 Agent 记忆系统（3张表）**，本节只讲建表 DDL 摘要和 CRUD 用法。

Agent 服务层为 Alpha 引入"自主诊断与优化闭环"：Hermes Runtime 负责调度与执行，Hermes Memory 负责持久化，MCP Server 把 Alpha 自身 API 暴露为工具供 Agent 调用。

---

### 9.1 HermesRuntime

**文件位置**：[app/services/hermes_runtime.py](file:///workspace/app/services/hermes_runtime.py)

Hermes 任务调度的核心执行器，支持"Agent 模式 / 降级模式"双模式自动探测，并提供完整的并发 / 熔断 / 超时保护。

#### 双模式自动探测

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

#### 三重保护

① **并发控制：Semaphore(2)**

`self._sem = asyncio.Semaphore(2)` [L36](file:///workspace/app/services/hermes_runtime.py#L36)。

最多同时执行 2 个任务（通常是 15:30 daily_review + 21:00 notice_review，刚好错开 1 小时），避免 LLM 请求打爆。

② **熔断：Circuit Breaker**

模块常量 [L20-L21](file:///workspace/app/services/hermes_runtime.py#L20-L21)：
```python
_CIRCUIT_BREAKER_THRESHOLD = 3   # 连续失败 3 次
_CIRCUIT_BREAKER_COOLDOWN = 3600 # 冷却 1 小时
```

`_is_circuit_open(task_type)` [L66-L76](file:///workspace/app/services/hermes_runtime.py#L66-L76)：
- 同一 task_type 最近失败时间戳列表
- 失败次数 ≥ 3 且距离最后失败 < 3600s → 熔断打开，新任务直接返回"任务处于熔断状态"
- 成功一次立即 `_clear_failures(task_type)` 清空记录

③ **超时：wait_for(timeout=180s)**

`run_task()` [L94-L97](file:///workspace/app/services/hermes_runtime.py#L94-L97)：
```python
result = await asyncio.wait_for(
    self._dispatch(...),
    timeout=_TASK_TIMEOUT,   # 180s
)
```

Agent 模式需要包含"自主工具链调用链"的长超时（MCP 工具 + LLM 思考可能需要几分钟），180s 是保守值。超时后任务状态记为 `timeout`，计入熔断失败计数。

#### 任务类型

| task_type | 默认调度 | 诊断对象 |
|-----------|---------|---------|
| `daily_review` | 15:30 每日 | 漏斗状态 + 策略参数 + K 线同步质量 + 热门概念 → 盘后复盘报告 |
| `notice_review` | 21:00 每日 | 公告选股漏斗 + 关键词命中分布 + 风险词过滤 → 公告复盘建议 |
| `full_diagnosis` | 手动触发 | 以上全部 + 手动指定参数，全量深度诊断 |

调度入口：`async def run_task(task_type, trigger, params)` [L83-L127](file:///workspace/app/services/hermes_runtime.py#L83-L127)，统一走"熔断检查 → 信号量 → wait_for → 状态记录"管线。

#### 数据采集层：_collect_observations()

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

#### LLM/Agent JSON 容错

由于 LLM 偶尔输出"解释文字 + JSON"的混合格式或 Markdown 代码块，`_parse_llm_json()` 采取分级兜底：
1. 优先 `json.loads(text)`
2. 失败 → `find("{")` + `rfind("}")` 截取最大花括号片段 → 再 json.loads
3. 仍失败 → 返回 {"raw_summary": text[:500]}，至少保留原始摘要不丢失

#### System Prompts

① **_AGENT_SYSTEM_PROMPT（Agent 模式用）**
见 [L308-L339](file:///workspace/app/services/hermes_runtime.py#L308-L339)，强调：
- **只能建议不能修改**：严禁调用变更接口，只输出建议
- **严禁编造股价**：预测数据必须通过 `predict_kronos` 工具获取
- **参数建议每次 < 20%，最多调 2 个参数**：避免一次性大幅改动导致系统失控
- **主动调用工具**：不要等待数据，自己取数据

同时给出诊断参考（候选池健康范围 5-30 只、buy_score_threshold 默认 78 等），Agent 可以对比判断。

② **_FALLBACK_SYSTEM_PROMPT（降级模式用）**
见 [L341-L353](file:///workspace/app/services/hermes_runtime.py#L341-L353)，精简版：
- 同样强调"只能建议不能改"
- 约束"输出必须是严格的 JSON 格式"（降级模式是单轮 LLM 调用，严格 JSON 便于机器解析）

---

### 9.2 HermesMemory

**文件位置**：[app/services/hermes_memory.py](file:///workspace/app/services/hermes_memory.py)

Hermes 的持久化层，所有表建立在 `funnel_state.db`（共享 KV 存储同一个 SQLite 文件，WAL 模式天然多表安全）。

#### 任务记忆：agent_tasks 表

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

#### 监控配置：agent_monitor_config 表

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

#### 监控消息流：agent_monitor_messages 表

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

### 9.3 MCP Server

**文件位置**：[app/mcp_server.py](file:///workspace/app/mcp_server.py)

基于 `FastMCP + stdio` 协议的 MCP 工具服务器。Hermes Agent（或任何 MCP 客户端）启动此脚本后，即可通过标准化 MCP 协议调用 Alpha 的所有 API。

**启动方式**：
```bash
python app/mcp_server.py              # 从 stdin/stdout 读写 MCP 消息
```
`ALPHA_API_BASE` 环境变量控制 Alpha 后端地址（默认 `http://127.0.0.1:18890`）[L16](file:///workspace/app/mcp_server.py#L16)。

#### 16 个工具分类

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

#### 内部实现：同源 REST API 调用

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

#### 超时配置

| 方法 | 超时 | 原因 |
|------|------|------|
| `_get`（GET 请求） | 30s | 大多数数据 API 秒级返回，30s 足够 |
| `_post`（POST 请求） | 60s | 公告筛选 / Kronos 预测 / 诊断触发可能需要长执行 |

`predict_kronos` 工具 [L159-L190](file:///workspace/app/mcp_server.py#L159-L190) 额外做了错误兜底：
- `HTTPStatusError` → 返回 `{error: "预测失败(500): ..."}` 而非抛异常
- 通用 Exception → 返回 `{error: "预测异常: ..."}`
- 保证 Agent 永远能拿到结构化 JSON，不会因单只股票中断整体诊断

`predict_kronos` 返回时附带"人类可读摘要" `prediction_summary` 字段（每行 `YYYY-MM-DD: 开X 高Y 低Z 收W (+/-N%)`），让 Agent 在没有专门图表渲染能力时也能直接看懂预测结果。

---

# 第二部分 服务层详解（续）

## 第10章 业务服务

### 本节导读

本章承接第6-9章的服务层讲解，聚焦**端到端业务场景**的五个核心服务：
- **NoticeService**：上市公司公告 AI 筛选（规则+LLM 双引擎打分，输出三池）
- **PaperTradingService**：A 股模拟盘（费用模型、持仓管理、盈亏汇总）
- **PredictFunnelService**：预测选股漏斗（板块 TopK→成分股 TopM→Kronos 预测→Top10）
- **RealtimeHub**：WebSocket 实时广播（漏斗快照+监控消息两事件）
- **FeishuNotify**：飞书通知推送（CardBuilder 卡片构建器+降噪策略）

这五个服务直接面向前端 UI 或外部通知渠道，是"用户能直接感知到"的业务能力层。

---

### 10.1 NoticeService

**文件位置**：[app/services/notice_service.py](file:///workspace/app/services/notice_service.py)

上市公司公告的 AI 筛选服务。数据来源：AkShare `stock_notice_report`，结合"规则打分（必选）+ LLM 打分（可选覆盖）"双引擎输出三池。

#### 10.1.1 数据来源

`run_notice_screen()` 中的数据抓取 [L117](file:///workspace/app/services/notice_service.py#L117)：
```python
df = await asyncio.to_thread(ak.stock_notice_report, symbol="全部", date=yyyymmdd)
```

AkShare 返回的列名：`代码 / 名称 / 公告标题 / 公告类型 / 公告日期 / 网址`，按公告发布日期过滤。

#### 10.1.2 双引擎打分

##### ① 规则打分 _rule_score()

`def _rule_score(title, notice_type, active_tags=None) -> tuple[float, str]` [L53-L70](file:///workspace/app/services/notice_service.py#L53-L70)

**BULLISH_RULES**（7 类利好关键词加权）见 [L15-L23](file:///workspace/app/services/notice_service.py#L15-L23)：

| 标签 | 权重 | 关键词 |
|------|------|--------|
| 业绩预增 | 14 | 预增、扭亏、同比增长、大幅增长、预盈 |
| 高额分红 | 12 | 分红、派息、现金红利、利润分配、送转、转增 |
| 股份回购 | 13 | 回购、增持计划、增持股份、回购股份 |
| 重大合同 | 11 | 重大合同、中标、签订、定点、订单、采购协议 |
| 资产重组 | 10 | 重组、收购、并购、资产注入、购买资产 |
| 融资获批 | 9 | 获批、审核通过、注册生效、获得批复 |
| 产品突破 | 8 | 量产、商业化、获准上市、新品发布、投产 |

基础分 = 45 分，命中多个标签累加（上限 100）。

**BEARISH_KEYWORDS**（13 个利空压分词）见 [L25-L39](file:///workspace/app/services/notice_service.py#L25-L39)：
```
减持、诉讼、处罚、立案、风险提示、停牌、终止、违约、亏损、预减、退市、ST、*ST
```
**命中任意一个直接扣到 20 分**（不再走利好加权），reason 返回"含风险词:X"。

##### ② LLM 打分：score_with_llm()（可选）

```python
llm_scores, llm_enabled = score_with_llm(shortlisted)
```
见 [L175](file:///workspace/app/services/notice_service.py#L175)，从 `app.services.notice_llm` 模块导入。

- 如果配置未开启 LLM（缺 API Key 或开关关闭）→ `llm_enabled=False`，完全使用规则分
- 开启后，LLM 返回的 `score` 会**覆盖规则分**，但保留在 0~100 的 clamp 范围
- LLM 额外返回 `reason`（一句话理由）和 `risk`（风险提示），前端单独展示

#### 10.1.3 分池阈值

`def _score_to_pool(score)` [L73-L78](file:///workspace/app/services/notice_service.py#L73-L78)：

```python
if score >= 80: return POOL_BUY      # 买入池
if score >= 65: return POOL_FOCUS    # 重点关注池
return POOL_CANDIDATE                 # 候选池
```

#### 10.1.4 股票池过滤

在候选筛选阶段做硬性过滤 [L141-L144](file:///workspace/app/services/notice_service.py#L141-L144)：
```python
if not code or "ST" in name.upper():        # 排除 ST / *ST
    continue
if not (code.startswith("6") or code.startswith("00")):  # 仅主板（6×沪市 + 00×深市主板）
    continue                                 # 排除创业板(30) + 科创板(68) + 北交所(43/83/87/92)
```

此外 score < 55 的直接丢弃不进候选，避免池子过大。

#### 10.1.5 去重 + 截断流程

多只股票可能对应同一份公告或一只股票多份公告，流程：
1. 按 rule_score 降序排序
2. 按 code dedup 去重，保留最高分那一条
3. 取 `max(10, min(limit×4, 120))` 条送 LLM 打分（控制 LLM 成本）
4. LLM 打分后按最终分截断 `max(1, min(limit, 200))` 条写入 entries

---

### 10.2 PaperTradingService

**文件位置**：[app/services/paper_trading.py](file:///workspace/app/services/paper_trading.py)

A 股模拟盘服务，含完整费用模型、持仓管理、盈亏计算、统计汇总。数据存储在 `data/funnel_state.db` 的 `paper_positions / paper_trades` 表，对应表结构详见第5章 5.3.1 节。

#### 10.2.1 费用模型

模块常量 [L18-L22](file:///workspace/app/services/paper_trading.py#L18-L22)：

| 费用项 | 默认值 | 收取方向 | 说明 |
|--------|--------|---------|------|
| `DEFAULT_COMMISSION_RATE` | 0.00025（万2.5） | 双向 | 券商佣金，买入 + 卖出都收 |
| `DEFAULT_MIN_COMMISSION` | 5.0 元 | 双向 | 单笔最低佣金（不满 5 元按 5 元收） |
| `DEFAULT_STAMP_TAX_RATE` | 0.0005（万5） | 卖出单向 | 国家印花税，仅卖出收 |
| `DEFAULT_SLIPPAGE_RATE` | 0.001（0.1%） | 双向 | 市场冲击成本模拟 |

**买入费用：_calc_buy_cost(price, qty)** [L75-L81]：
```python
slip_price = price * (1 + slippage_rate)         # 滑点：买入多付 0.1%
amount = slip_price * qty
commission = max(amount * commission_rate, 5)     # 佣金（不满 5 按 5）
# 无印花税
total_fee = commission
return actual_price, total_fee
```

**卖出费用：_calc_sell_cost(price, qty)** [L83-L90]：
```python
slip_price = price * (1 - slippage_rate)         # 滑点：卖出少得 0.1%
amount = slip_price * qty
commission = max(amount * commission_rate, 5)
stamp_tax = amount * stamp_tax_rate               # 印花税（卖出单向）
total_fee = commission + stamp_tax
return actual_price, total_fee
```

#### 10.2.2 Position Dataclass 字段

`@dataclass class Position` [L25-L44](file:///workspace/app/services/paper_trading.py#L25-L44)：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | str | 持仓 ID（uuid 前 12） |
| `symbol` | str | 股票代码 |
| `name` | str | 股票名称 |
| `direction` | str | 仅 "long"（预留做空扩展） |
| `qty` | int | 持股数（必须是 100 的整数倍，由调用方校验） |
| `cost_price` | float | 成本价（买入实际成交价，含买入费用已分摊） |
| `current_price` | float | 最新价（由 update_prices 批量刷新） |
| `pnl` | float | 浮动盈亏 = (current - cost)×qty - 预估买卖费用 |
| `pnl_pct` | float | 浮动盈亏 % |
| `opened_at` | str | ISO datetime 开仓时间 |
| `status` | str | "open" / "closed" |
| `closed_at` | str \| None | 平仓时间 |
| `close_price` | float \| None | 平仓实际成交价 |
| `realized_pnl` / `realized_pnl_pct` | float \| None | 已实现盈亏（仅 closed 有效） |
| `buy_fee` / `sell_fee` | float | 开仓 / 平仓费用（closed 实际值 / open 预估值） |
| `note` | str | 下单备注 |

#### 10.2.3 update_prices(price_map)

`def update_prices(price_map: dict[str, float])` [L230-L243](file:///workspace/app/services/paper_trading.py#L230-L243)。

被全局 `ticker_loop` 每 60 秒调用一次，传入 `{symbol: latest_price}` 映射。对所有 `status='open'` 的持仓逐条 `UPDATE current_price`，提交一次事务。刷新失败只影响价格显示，不阻塞任何 API。

#### 10.2.4 买入 / 卖出接口

**买入 open_position(symbol, name, price, qty=100, note="")** [L149-L182](file:///workspace/app/services/paper_trading.py#L149-L182)：
- 调用 `_calc_buy_cost` 获取实际成交价 + 买入费用
- 事务内插入 `paper_positions`（status=open） + 插入 `paper_trades`（action=buy）

**卖出 close_position(position_id, price, note="")** [L186-L226](file:///workspace/app/services/paper_trading.py#L186-L226)：
- 查 position，不存在返回 None
- 调用 `_calc_sell_cost` 获取卖出实际成交价 + 卖出费用
- 计算已实现盈亏：`(actual_sell - cost) * qty - (buy_fee + sell_fee)`
- 事务内 UPDATE position 为 closed + 插入 paper_trades（action=sell）

#### 10.2.5 汇总 summary()

`def get_summary() -> dict` [L262-L312](file:///workspace/app/services/paper_trading.py#L262-L312)：

| 输出字段 | 含义 |
|---------|------|
| `open_count` / `closed_count` | 未平仓 / 已平仓持仓数 |
| `total_float_pnl` / `total_float_pnl_pct` | 未平仓浮动盈亏合计 / 百分比 |
| `total_realized_pnl` | 已平仓累计盈亏 |
| `total_market_value` | 未平仓总市值 |
| `total_asset` | 总资产 = 初始资金 100 万 + total_realized + total_float - total_fee |
| `initial_capital` | 初始资金（固定 1,000,000） |
| `total_trades` | 已平仓总笔数 |
| `win_count` / `lose_count` | 盈利 / 亏损笔数 |
| `win_rate` | 胜率 = wins / closed × 100 |
| `max_drawdown` | 已平仓盈亏序列的最大回撤（元） |
| `total_fee` | 所有交易的费用总和（含 open/closed） |

#### 10.2.6 读取降级

- 持仓接口 `get_open_positions / get_closed_positions / get_summary` 全部走本地 SQLite，100% 可用
- 实时行情接口失败只影响 `update_prices` 的价格刷新，不阻塞页面
- 前端页面即使 snapshot 挂了，仍能看到持仓（价格可能是旧的）

---

### 10.3 PredictFunnelService

**文件位置**：[app/services/predict_funnel_service.py](file:///workspace/app/services/predict_funnel_service.py)

预测选股流程：**概念板块 TopK → 每板块成分股 TopM → 去重 → Kronos 三日预测 → 按 pred_max_high_pct 排序 → 三池分层**。

#### 10.3.1 工作流拆解

`async def _execute(trigger)` [L139-L304](file:///workspace/app/services/predict_funnel_service.py#L139-L304)：

**Step 1 — 热门概念板块 TopK**（默认 10）
```python
concepts, concept_board_source = await self.provider.fetch_concept_board_names_em()
```
优先东财 HTTP 接口（内部重试 3 次），失败降级同花顺概念名。按涨跌幅降序取 Top K。

**Step 2 — 每板块成分股 TopM**（默认 10）
对每个板块调用 `provider.get_concept_constituents(board_name)`，取涨幅领先 Top M。去重 + 过滤：
- 仅 `00 / 30 / 60 / 68` 开头（A 股四大板块）
- 剔除包含 `ST / * / 退` 字样的个股

**Step 3 — 逐股 Kronos 三日预测**
```python
pred = await self.kronos.predict(code, lookback=180, horizon=3)
```
计算核心指标：
```
today_close = history[-1]["close"]
pred_max_high_pct = (max(pred.high) - today_close) / today_close * 100
pred_last_close_pct = (pred[-1].close - today_close) / today_close * 100
pred_avg_close_pct = (avg(pred.close) - today_close) / today_close * 100
```
单只失败不阻塞批次，统计 `failed_count`。

**Step 4 — 三池分层**（激进阈值）
```
threshold_candidate = 2.0%   pred_max_high_pct ≥ 2%
threshold_focus     = 4.0%   ≥ 4%
threshold_buy       = 8.0%   ≥ 8%
```
三池不互斥（≥8% 也 ≥4% 也 ≥2%），但只进入最高的一个池。

**Step 5 — 持久化 + 飞书推送**
结果写入 `state_store.set_kv("predict_funnel", snapshot)`，如果 `feishu_enabled=True` 调用 `notify_predict_top()` 发飞书 Top10 卡片。

#### 10.3.2 调度

`_predict_funnel_scheduler_loop`（在 main.py lifespan 中启动）：
- 调度时间：每日 16:15 自动触发（收盘后 1 小时，K 线同步完成后）
- 防止重复：`asyncio.Lock()` 检查 `self.running`
- 进度更新：`self.progress` 六阶段（init → boards → constituents → predict → persisted → done）
- 配置向前迁移：旧版本保存的 `lookback < 60` 强制升级到新默认 180（避免预测不准）

---

### 10.4 RealtimeHub

**文件位置**：[app/services/realtime.py](file:///workspace/app/services/realtime.py)

极简 WebSocket 管理器，基于 FastAPI `WebSocket`，负责向所有在线客户端推送实时事件。

#### 10.4.1 连接管理

```python
class RealtimeHub:
    def __init__(self) -> None:
        self._clients: set[WebSocket] = set()

    async def connect(self, websocket: WebSocket) -> None:
        await websocket.accept()
        self._clients.add(websocket)

    def disconnect(self, websocket: WebSocket) -> None:
        self._clients.discard(websocket)
```
见 [L9-L18](file:///workspace/app/services/realtime.py#L9-L18)。

路由层（`app/routers/...`）在 WS 握手成功后调 `connect()`，在 WS disconnect 回调中调 `disconnect()`。

#### 10.4.2 broadcast(event, payload)

`async def broadcast(event: str, payload: dict[str, Any]) -> None` [L20-L33](file:///workspace/app/services/realtime.py#L20-L33)。

推送格式：
```json
{"event": "snapshot", "data": { ... payload ... }}
```

**广播容错**：
```python
disconnected: list[WebSocket] = []
for ws in self._clients:
    try:
        await ws.send_text(msg)
    except Exception:
        disconnected.append(ws)
for ws in disconnected:
    self._clients.discard(ws)
```
单 client 发送失败（断网 / 页面关闭）不影响其他连接，默默收集后移除。不会重试堆积。

#### 10.4.3 事件类型

| event | 触发时机 | payload 内容 |
|-------|---------|-------------|
| `"snapshot"` | FunnelService.tick() 返回 changed=True 时（每 60s + 有迁池时立即） | 漏斗三池 + 热门概念 + 热门个股 + 每只个股最新评分 |
| `"monitor_update"` | Hermes 智能监控产生新消息时（agent_monitor_messages 新增记录） | 监控消息（id + content + trigger + created_at） |

扩展方式：在业务服务的关键回调处直接 `await realtime_hub.broadcast("自定义事件", {...})`，无需改 RealtimeHub 源码。

---

### 10.5 FeishuNotify

**文件位置**：[app/services/feishu_notify.py](file:///workspace/app/services/feishu_notify.py)

飞书 Webhook 通知模块，提供纯文本 + 交互式卡片两种发送方式，以及统一的卡片构建器 `CardBuilder`。

#### 10.5.1 CardBuilder 结构

`class CardBuilder` [L80-L173](file:///workspace/app/services/feishu_notify.py#L80-L173)，风格统一为"简洁信息卡"：

```
┌─────────────────────────────────────────────┐
│ 🔮 标题（色带背景，template 颜色）           │  ← header 标题区
│ 副标题（可选）                               │
├─────────────────────────────────────────────┤
│ **板块** 10   ·   **个股** 120   ·  **耗时** 42s   ← add_kv_inline
├─────────────────────────────────────────────┤
│ ┌── KV 网格（2/3/4 列自适应）────────────┐ │
│ │ **K1**     │ **K2**     │ **K3**     │ │  ← add_kv_grid
│ │ v1         │ v2         │ v3         │ │
│ └───────────────────────────────────────┘ │
│                                             │
│ markdown 正文段落（加粗/链接/列表）          │  ← add_markdown
│ ────────────────────────────────────────    │  ← add_hr
│                                             │
┌─灰色备注──────────────────────────────────┐ │
│ Kronos · cuda:0                           │ │  ← add_note
└───────────────────────────────────────────┘ │
│ [ 跳转按钮（primary / default） ]           │  ← add_link_button
└─────────────────────────────────────────────┘
```

**颜色模板**（12 色）见 [L74-L77](file:///workspace/app/services/feishu_notify.py#L74-L77)：
`blue / wathet / turquoise / green / yellow / orange / red / carmine / violet / purple / indigo / grey`

#### 10.5.2 CardBuilder 方法一览

| 方法 | 说明 |
|------|------|
| `add_markdown(content)` | 追加一段 markdown 正文（支持 **加粗**、[链接](url)、`inline code`） |
| `add_kv_grid(items, cols=2)` | 把 [(label, value), ...] 排成 KV 网格，cols 支持 1/2/3/4 |
| `add_kv_inline(items)` | 单行 K-V 内联（"**K1** v1  ·  **K2** v2"） |
| `add_hr()` | 分隔线 |
| `add_note(text)` | 底部灰色小字备注（时间戳、设备名、环境等） |
| `add_link_button(text, url, primary=False)` | 跳转链接按钮（primary=蓝色填充，default=描边） |
| `build()` | 输出飞书卡片 JSON（`{config, header, elements}`） |

#### 10.5.3 场景卡片

##### (1) 预测选股 Top10 — notify_predict_top()

`async def notify_predict_top(trade_date, top_entries, meta)` [L197-L228](file:///workspace/app/services/feishu_notify.py#L197-L228)

- **色带**：`template="purple"`（紫色，对应"🔮 预测"主题）
- 头部 KV：板块数 / 扫描个股数 / 总耗时
- Top 10 列表格式：
  ```
  ` 1` **贵州茅台**(600519)  高 +12.3%  收 +8.5%  · 白酒、消费升级
  ```
  （序号 / 名称 / 代码 / 最高涨幅 / 收盘涨幅 / 所属概念前 2）
- 底部备注：`Kronos · cuda:0`（模型设备名）

##### (2) K 线同步完成 — notify_sync_complete()（已停用）

`async def notify_sync_complete(...)` [L180-L194](file:///workspace/app/services/feishu_notify.py#L180-L194)

当前直接返回 `{"skipped": True, reason: "..."}`，不实际发飞书。**降噪原因**：补缺任务每天至少跑一次，每次 3000+ 股票同步完成，如果都推飞书会造成通知刷屏。

#### 10.5.4 发送入口

| 函数 | 用途 |
|------|------|
| `send_feishu_text(text)` | 极简纯文本（调试用） |
| `send_feishu_card(card)` | 发送交互式卡片（推荐） |

Webhook URL 优先级：`FEISHU_WEBHOOK_URL` 环境变量 → `FEISHU_WEBHOOK` 环境变量 → 代码内默认值。

---

# 第三部分 接口与前端

## 第11章 路由层设计

### 本节导读

从第11章开始进入"接口与前端"部分。本章讲解 FastAPI 路由层的三种组织模式、15 大功能分类端点表、kline router 注入模式、GET/POST 长任务/错误响应模式。

路由层是**服务能力对外暴露的边界**：服务层（6-10章）的所有功能，都必须通过路由层（本章）才能被前端 UI、MCP Server、外部脚本调用。

---

### 11.1 路由组织方式

#### 主体路由：直接 @app 装饰器定义

主入口 [app/main.py](file:///workspace/app/main.py) 使用 FastAPI 实例 `app` 直接定义了绝大部分端点（约 40+），覆盖大盘行情、策略选股、自定义策略中心、热门股票智能、兼容接口、公告选股、Kronos 预测、Hermes Agent、智能监控、模拟盘等核心功能。

初始化阶段在模块顶部创建所有服务单例（[app/main.py#L50-L134](file:///workspace/app/main.py#L50-L134)）：

```python
_kline_store = _KlineSQLiteStore()
provider = AkshareDataProvider(kline_store=_kline_store)
market_data_client = EastmoneyMarketDataClient(store=_kline_store)
kline_cache_service = KlineCacheService(provider=provider, store=_kline_store, market_data_client=market_data_client)
service = FunnelService(provider=provider, kline_cache_service=kline_cache_service)
# ... 其余服务
```

#### 模块化路由

通过 `include_router` 注册的独立模块有两个：

1. **kline 模块** — [app/routers/kline.py](file:///workspace/app/routers/kline.py)
   - `/api/jobs/kline-cache/*`：K 线缓存同步/进度/日志/统计/检查
   - `/api/kline/{symbol}`：K 线查询
   - `/api/admin/shutdown-prepare`：管理接口（SQLite checkpoint）

2. **first_limit_alpha 模块** — [app/routers/first_limit_alpha.py](file:///workspace/app/routers/first_limit_alpha.py)
   - `/api/strategy/first-limit-alpha/*`：7 个首板 Alpha 研发链路端点

#### include_router 注册（依赖注入方式）

在 [app/main.py#L281-L282](file:///workspace/app/main.py#L281-L282) 注入依赖：

```python
app.include_router(init_kline_router(provider, kline_cache_service), prefix="/api")
app.include_router(init_first_limit_alpha_router(first_limit_alpha_service), prefix="/api")
```

---

### 11.2 路由分类（按功能分组）

#### 1. 大盘行情

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/market/hot-concepts` | GET | 热门概念板块 | [app/main.py#L302-L305](file:///workspace/app/main.py#L302-L305) |
| `/api/market/hot-stocks` | GET | 热门个股 Top30 | [app/main.py#L308-L311](file:///workspace/app/main.py#L308-L311) |
| `/api/stock/{symbol}/realtime` | GET | 个股盘中实时行情 | [app/main.py#L337-L361](file:///workspace/app/main.py#L337-L361) |
| `/api/strategy/profile` | GET | 策略参数配置档案 | [app/main.py#L377-L379](file:///workspace/app/main.py#L377-L379) |

#### 2. 策略选股

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/funnel` | GET | 漏斗三池快照 | [app/main.py#L296-L299](file:///workspace/app/main.py#L296-L299) |
| `/api/pool/move` | POST | 池间迁移股票 | [app/main.py#L323-L327](file:///workspace/app/main.py#L323-L327) |
| `/api/score/recompute` | POST | 单股重新打分 | [app/main.py#L330-L334](file:///workspace/app/main.py#L330-L334) |
| `/api/stock/{symbol}/detail` | GET | 个股详情 + K 线 | [app/main.py#L314-L320](file:///workspace/app/main.py#L314-L320) |

#### 3. 自定义策略中心

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/strategy/rules` | GET | 规则目录（动态表单） | [app/main.py#L473-L476](file:///workspace/app/main.py#L473-L476) |
| `/api/strategy/custom` | GET | 策略列表 | [app/main.py#L479-L486](file:///workspace/app/main.py#L479-L486) |
| `/api/strategy/custom/{id}` | GET | 策略详情 | [app/main.py#L489-L494](file:///workspace/app/main.py#L489-L494) |
| `/api/strategy/custom` | POST | 新增/更新策略 | [app/main.py#L497-L530](file:///workspace/app/main.py#L497-L530) |
| `/api/strategy/custom/{id}` | DELETE | 删除策略 | [app/main.py#L533-L541](file:///workspace/app/main.py#L533-L541) |
| `/api/strategy/custom/{id}/default` | POST | 设为默认 | [app/main.py#L544-L548](file:///workspace/app/main.py#L544-L548) |
| `/api/strategy/custom/{id}/scan` | GET | 上次扫描结果 | [app/main.py#L551-L556](file:///workspace/app/main.py#L551-L556) |
| `/api/strategy/custom/{id}/scan` | POST | 触发扫描 | [app/main.py#L559-L569](file:///workspace/app/main.py#L559-L569) |
| `/api/strategy/custom/{id}/backtest` | POST | 策略回测 | [app/main.py#L572-L592](file:///workspace/app/main.py#L572-L592) |

#### 4. 热门股票智能

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/strategy/hot-stock-ai` | GET | 快照 | [app/main.py#L406-L408](file:///workspace/app/main.py#L406-L408) |
| `/api/strategy/hot-stock-ai/run` | POST | 触发分析（202） | [app/main.py#L411-L416](file:///workspace/app/main.py#L411-L416) |
| `/api/strategy/hot-stock-ai/pool/move` | POST | 池间迁移 | [app/main.py#L419-L426](file:///workspace/app/main.py#L419-L426) |
| `/api/strategy/hot-stock-ai/config` | GET/POST | 配置读写 | [app/main.py#L429-L437](file:///workspace/app/main.py#L429-L437) |

#### 5. 兼容接口

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/strategy/quiet-breakout` | GET | 缩量启动快照（兼容） | [app/main.py#L440-L443](file:///workspace/app/main.py#L440-L443) |
| `/api/strategy/quiet-breakout/scan` | POST | 缩量启动扫描（兼容） | [app/main.py#L446-L468](file:///workspace/app/main.py#L446-L468) |

#### 6. 公告选股

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/notice/funnel` | GET | 公告漏斗快照 | [app/main.py#L595-L597](file:///workspace/app/main.py#L595-L597) |
| `/api/notice/keywords` | GET | 利好关键词权重 | [app/main.py#L600-L603](file:///workspace/app/main.py#L600-L603) |
| `/api/jobs/notice-screen` | POST | 触发公告筛选 | [app/main.py#L606-L613](file:///workspace/app/main.py#L606-L613) |
| `/api/notice/pool/move` | POST | 公告池迁移 | [app/main.py#L616-L621](file:///workspace/app/main.py#L616-L621) |
| `/api/notice/{symbol}/detail` | GET | 公告个股详情 | [app/main.py#L624-L629](file:///workspace/app/main.py#L624-L629) |

#### 7. Kronos 预测

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/predict/{symbol}/kronos` | GET | 单股 Kronos 预测 | [app/main.py#L364-L374](file:///workspace/app/main.py#L364-L374) |
| `/api/predict-funnel` | GET | 预测漏斗快照 | [app/main.py#L382-L384](file:///workspace/app/main.py#L382-L384) |
| `/api/predict-funnel/trigger` | POST | 触发预测扫描（202） | [app/main.py#L387-L392](file:///workspace/app/main.py#L387-L392) |
| `/api/predict-funnel/config` | GET/POST | 配置读写 | [app/main.py#L395-L403](file:///workspace/app/main.py#L395-L403) |

#### 8. K 线同步/查询（kline router）

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/jobs/kline-cache/sync` | POST | 全量同步（202） | [app/routers/kline.py#L62-L72](file:///workspace/app/routers/kline.py#L62-L72) |
| `/api/jobs/kline-cache/incremental-sync` | POST | 增量同步（202） | [app/routers/kline.py#L75-L83](file:///workspace/app/routers/kline.py#L75-L83) |
| `/api/jobs/kline-cache/batch-incremental-sync` | POST | 批量区间同步（202） | [app/routers/kline.py#L86-L99](file:///workspace/app/routers/kline.py#L86-L99) |
| `/api/jobs/kline-cache/status` | GET | 同步状态 | [app/routers/kline.py#L102-L104](file:///workspace/app/routers/kline.py#L102-L104) |
| `/api/jobs/kline-cache/progress` | GET | 进度条 | [app/routers/kline.py#L107-L109](file:///workspace/app/routers/kline.py#L107-L109) |
| `/api/jobs/kline-cache/logs` | GET | 日志分页 | [app/routers/kline.py#L112-L114](file:///workspace/app/routers/kline.py#L112-L114) |
| `/api/jobs/kline-cache/logs/{task_id}` | GET | 单任务日志详情 | [app/routers/kline.py#L117-L122](file:///workspace/app/routers/kline.py#L117-L122) |
| `/api/jobs/kline-cache/stats` | GET | 覆盖率统计 | [app/routers/kline.py#L125-L127](file:///workspace/app/routers/kline.py#L125-L127) |
| `/api/jobs/kline-cache/check` | POST | 完整性检查 | [app/routers/kline.py#L130-L133](file:///workspace/app/routers/kline.py#L130-L133) |
| `/api/jobs/kline-cache/report` | GET | 最近检查报告 | [app/routers/kline.py#L136-L141](file:///workspace/app/routers/kline.py#L136-L141) |
| `/api/kline/{symbol}` | GET | K 线查询 | [app/routers/kline.py#L147-L156](file:///workspace/app/routers/kline.py#L147-L156) |

#### 9. FirstLimit Alpha（first_limit_alpha router）

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/strategy/first-limit-alpha/status` | GET | 模块状态 | [app/routers/first_limit_alpha.py#L19-L21](file:///workspace/app/routers/first_limit_alpha.py#L19-L21) |
| `/api/strategy/first-limit-alpha/dataset/build` | POST | 数据构建 | [app/routers/first_limit_alpha.py#L24-L34](file:///workspace/app/routers/first_limit_alpha.py#L24-L34) |
| `/api/strategy/first-limit-alpha/features/build` | POST | 特征工程 | [app/routers/first_limit_alpha.py#L37-L39](file:///workspace/app/routers/first_limit_alpha.py#L37-L39) |
| `/api/strategy/first-limit-alpha/train/baseline` | POST | Baseline 训练 | [app/routers/first_limit_alpha.py#L42-L44](file:///workspace/app/routers/first_limit_alpha.py#L42-L44) |
| `/api/strategy/first-limit-alpha/train/sequence` | POST | 序列模型训练 | [app/routers/first_limit_alpha.py#L47-L49](file:///workspace/app/routers/first_limit_alpha.py#L47-L49) |
| `/api/strategy/first-limit-alpha/inference/run` | POST | 推理打分 | [app/routers/first_limit_alpha.py#L52-L54](file:///workspace/app/routers/first_limit_alpha.py#L52-L54) |
| `/api/strategy/first-limit-alpha/backtest` | POST | 回测 | [app/routers/first_limit_alpha.py#L57-L59](file:///workspace/app/routers/first_limit_alpha.py#L57-L59) |

#### 10. Hermes Agent

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/agent/status` | GET | 运行状态 | [app/main.py#L635-L637](file:///workspace/app/main.py#L635-L637) |
| `/api/agent/run` | POST | 执行任务 | [app/main.py#L640-L650](file:///workspace/app/main.py#L640-L650) |
| `/api/agent/tasks` | GET | 历史任务 | [app/main.py#L653-L656](file:///workspace/app/main.py#L653-L656) |

#### 11. 智能监控

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/agent/monitor/config` | GET/POST | 监控配置 | [app/main.py#L662-L676](file:///workspace/app/main.py#L662-L676) |
| `/api/agent/monitor/messages` | GET | 消息列表 | [app/main.py#L679-L682](file:///workspace/app/main.py#L679-L682) |
| `/api/agent/monitor/trigger` | POST | 手动触发 | [app/main.py#L685-L695](file:///workspace/app/main.py#L685-L695) |
| `/api/agent/monitor/stop` | POST | 停止监控 | [app/main.py#L698-L701](file:///workspace/app/main.py#L698-L701) |

#### 12. 模拟盘

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/paper/buy` | POST | 模拟买入 | [app/main.py#L707-L727](file:///workspace/app/main.py#L707-L727) |
| `/api/paper/sell` | POST | 模拟卖出 | [app/main.py#L730-L752](file:///workspace/app/main.py#L730-L752) |
| `/api/paper/positions` | GET | 当前持仓 | [app/main.py#L809-L825](file:///workspace/app/main.py#L809-L825) |
| `/api/paper/history` | GET | 已平仓历史 | [app/main.py#L828-L830](file:///workspace/app/main.py#L828-L830) |
| `/api/paper/summary` | GET | 账户汇总 | [app/main.py#L833-L851](file:///workspace/app/main.py#L833-L851) |
| `/api/paper/trades` | GET | 成交流水 | [app/main.py#L854-L856](file:///workspace/app/main.py#L854-L856) |
| `/api/paper/settings` | GET/POST | 账户设置 | [app/main.py#L859-L867](file:///workspace/app/main.py#L859-L867) |

#### 13. 管理接口

| 端点 | 方法 | 功能 | 文件位置 |
|------|------|------|---------|
| `/api/admin/shutdown-prepare` | POST | SQLite WAL checkpoint | [app/routers/kline.py#L53-L59](file:///workspace/app/routers/kline.py#L53-L59) |

#### 14. WebSocket

| 端点 | 事件 | 功能 | 文件位置 |
|------|------|------|---------|
| `/ws/realtime` | `snapshot` | 连接时推送漏斗+热门+个股快照 | [app/main.py#L870-L893](file:///workspace/app/main.py#L870-L893) |
| `/ws/realtime` | `monitor_update` | 智能监控新消息广播 | [app/main.py#L685-L695](file:///workspace/app/main.py#L685-L695) |

#### 15. 静态页面

| 端点 | 功能 | 文件位置 |
|------|------|---------|
| `/` | 返回 `index.html` | [app/main.py#L286-L288](file:///workspace/app/main.py#L286-L288) |
| `/notice` | 重定向 `/?tab=notice` | [app/main.py#L291-L293](file:///workspace/app/main.py#L291-L293) |

---

### 11.3 kline router 注入模式

[app/routers/kline.py](file:///workspace/app/routers/kline.py) 采用"先注册函数、后注入依赖"的模式：

```python
# 模块级全局变量（初始为 None）
_provider: AkshareDataProvider | None = None
_kline_cache_service: KlineCacheService | None = None

def init_kline_router(provider, kline_cache_service) -> APIRouter:
    """注入依赖并返回 router，供 main.py 调用。"""
    global _provider, _kline_cache_service
    _provider = provider
    _kline_cache_service = kline_cache_service
    return router
```

**设计要点**：
- router 处理函数直接读取模块级全局变量 `_provider` 和 `_kline_cache_service`
- 简化依赖注入代码，无需每个 endpoint 传参
- **测试时需要 monkeypatch** 这两个全局变量
- first_limit_alpha router 采用完全相同的模式（`_service` 全局变量）

---

### 11.4 响应模式

#### GET 查询：直接 return payload

```python
@app.get("/api/funnel")
async def get_funnel(trade_date: str | None = None):
    payload = await service.get_funnel(trade_date)
    return payload  # Pydantic model 或 dict
```

#### POST 长任务：202 Accepted + asyncio.create_task()

```python
@app.post("/api/predict-funnel/trigger")
async def trigger_predict_funnel():
    if predict_funnel_service.running:
        raise HTTPException(status_code=409, detail="预测任务已在执行中")
    asyncio.create_task(predict_funnel_service.run(trigger="manual"))
    return {"success": True, "message": "预测任务已启动，请查看进度",
            "snapshot": predict_funnel_service.get_snapshot()}
```

适用于：K 线同步（`/api/jobs/kline-cache/*`）、热门智能（`/api/strategy/hot-stock-ai/run`）、预测漏斗（`/api/predict-funnel/trigger`）等。

#### 错误处理：HTTPException + 中文 detail

```python
@app.get("/api/stock/{symbol}/detail")
async def get_stock_detail(symbol: str, ...):
    try:
        payload = await service.get_stock_detail(symbol, ...)
    except KeyError:
        raise HTTPException(status_code=404, detail="symbol not found in funnel")
    return payload
```

常见状态码：
- `400`：参数错误 / 非交易时段 / 缺少字段
- `403`：内置策略不可删除
- `404`：资源不存在（symbol / strategy / task）
- `409`：任务已在运行中（冲突）
- `500` / `503`：服务端异常（扫描失败 / 预测失败 / 执行失败）

---

## 第12章 前端架构

### 本节导读

本章讲解 Alpha 前端的零构建原生技术栈选型、8 Tab SPA 单页结构、switchTab 状态机、轮询管理器、WS+轮询双通道推送、ECharts K 线统一渲染器（含 _sliceMergedForDisplay 截断逻辑）、Kronos 预测 UI 四件套规范、Modal 系统与核心 bug 回归点。

前端通过第11章路由层的 REST API + WebSocket 与后端交互。

---

### 12.1 技术选型

- **原生 HTML/CSS/JS**：无构建工具、无框架（Vue/React 均未使用）
- **图表库**：ECharts 5（通过 CDN 引入，未打包）
- **优势**：零编译、零依赖安装、启动即运行；劣势：3000+ 行单文件 JS，模块化程度低

---

### 12.2 单页结构：index.html

入口文件 [app/static/index.html](file:///workspace/app/static/index.html) 是一个典型的单页应用（SPA）结构。

#### 8 Tab Sidebar

左侧固定导航栏（`.sidebar`）包含 8 个功能 Tab（[app/static/index.html#L14-L54](file:///workspace/app/static/index.html#L14-L54)）：

| data-tab | 名称 | 对应容器 |
|----------|------|---------|
| `market` | 大盘（总览） | `#tab-market` |
| `data` | 数据中心 | `#tab-data` |
| `funnel` | 策略选股 | `#tab-funnel` |
| `notice` | 公告选股 | `#tab-notice` |
| `predict` | 预测选股（K线预测） | `#tab-predict` |
| `hotai` | 热门智能 | `#tab-hotai` |
| `agent` | 智能监控 / 进化 | `#tab-agent` |
| `paper` | 模拟盘 | `#tab-paper` |

#### Glassmorphism 暗色主题

整体视觉风格：
- **毛玻璃背景**：`backdrop-filter: blur(12px) saturate(150%)` + `rgba(17, 24, 39, 0.65)` 半透明
- **辉光边框**：`--glass-border: rgba(255, 255, 255, 0.06)` 浅色描边
- **深色渐变**：根变量 `--bg: #0b0f19` → `--panel: #111827`
- **色彩系统**：涨红（`--up: #ef4444`）跌绿（`--down: #10b981`）符合 A 股习惯

#### Tab 容器机制

每个 Tab 对应一个 `<section id="tab-xxx" class="tab-content">` 容器（[app/static/index.html#L74](file:///workspace/app/static/index.html#L74)）：

```html
<section id="tab-market" class="tab-content active">...</section>
<section id="tab-data" class="tab-content">...</section>
<section id="tab-funnel" class="tab-content">...</section>
```

- 默认只显示加了 `.active` 类的容器
- 通过 CSS `.tab-content:not(.active) { display: none; }` 切换显隐
- 两列布局（策略/公告）额外加 `.two-col` 类启用右侧详情面板

---

### 12.3 app.js 核心机制

核心逻辑文件：[app/static/app.js](file:///workspace/app/static/app.js)（3000+ 行）。

#### 路由状态机：switchTab(name)

[app/static/app.js#L245-L263](file:///workspace/app/static/app.js#L245-L263)

```js
function switchTab(tab) {
  state.activeTab = tab;
  document.querySelectorAll('.sidebar-item[data-tab]').forEach((el) => {
    el.classList.toggle('active', el.dataset.tab === tab);
  });
  document.querySelectorAll('.tab-content').forEach((el) => {
    el.classList.toggle('active', el.id === `tab-${tab}`);
  });
  // funnel / notice 启用两列布局
  const twoCol = tab === 'funnel' || tab === 'notice';
  if (rightPanel) rightPanel.style.display = twoCol ? '' : 'none';
  if (layout) layout.classList.toggle('two-col', twoCol);
  // 触发 tab onShow 钩子
  onTabShow(tab);
}
```

初始化时读取 URL hash（`?tab=xxx`）调用 `switchTab`（[app/static/app.js#L2839](file:///workspace/app/static/app.js#L2839)）。

#### 前端状态管理：模块级 state 对象

[app/static/app.js#L1-L33](file:///workspace/app/static/app.js#L1-L33)

```js
const state = {
  activeTab: 'market',
  funnel: null,
  hotConcepts: null,
  hotStocks: null,
  syncStatus: null,
  selectedSymbol: null,
  predictFunnel: null,
  hotStockAI: null,
  paperUpdatedAt: null,
  // ... 各子模块状态
};
```

**无 Redux/Vuex**：直接读写模块级 `state` 对象，配合 DOM 渲染反映状态。数据变更→手动触发 DOM 更新（无响应式系统）。

#### 轮询管理器

每个 tab 可注册独立的 `poll()` 函数：

- **盘中（9:30-11:30, 13:00-15:00）**：10 秒轮询间隔
- **盘后**：10 分钟轮询间隔
- **可见性控制**：通过 `visibilitychange` 事件（[app/static/app.js#L2883-L2888](file:///workspace/app/static/app.js#L2883-L2888)）在页面隐藏时暂停、恢复时重启

```js
document.addEventListener('visibilitychange', () => {
  if (state.activeTab === 'paper') {
    if (document.hidden) _stopPaperPoll(); else _startPaperPoll();
  }
  if (state.activeTab === 'predict') {
    if (document.hidden) _stopPredictPoll(); else _startPredictPoll();
  }
});
```

数据中心 tab 使用独立的 `setInterval` 轮询器（[app/static/app.js#L2844-L2847](file:///workspace/app/static/app.js#L2844-L2847)）。

#### 双通道推送机制

##### 通道 1：WebSocket /ws/realtime

[app/static/app.js#L2508-L2547](file:///workspace/app/static/app.js#L2508-L2547)

```js
function connectWs() {
  const ws = new WebSocket(`${protocol}//${window.location.host}/ws/realtime`);
  ws.onmessage = (evt) => {
    const msg = JSON.parse(evt.data);
    if (msg.event === 'snapshot') {
      state.funnel = msg.data.funnel;
      state.hotConcepts = msg.data.hot_concepts;
      state.hotStocks = msg.data.hot_stocks;
      renderAll();
    } else if (msg.event === 'monitor_update') {
      appendMonitorMessage(msg.data);
    }
  };
  ws.onclose = () => {  // 指数退避重连 2s → 30s
    setTimeout(connectWs, _wsRetryDelay);
    _wsRetryDelay = Math.min(_wsRetryDelay * 1.5, 30000);
  };
}
```

##### 通道 2：setInterval 轮询兜底

当 WebSocket 断线或数据缺失时（如首次加载前），各 tab 独立的 `poll()` 函数通过 `fetch()` 拉取最新快照。双通道互为冗余：WS 负责实时推送，轮询负责首次加载 + 断线兜底。

---

### 12.4 ECharts K 线统一渲染器

#### _sliceMergedForDisplay()：数据截断

[app/static/app.js#L1024-L1031](file:///workspace/app/static/app.js#L1024-L1031)

```js
function _sliceMergedForDisplay(merged, predStartIdx, historyDays = 30) {
  const total = merged.length;
  const predIdx = Number.isFinite(predStartIdx) ? predStartIdx : total;
  const histStart = Math.max(0, predIdx - historyDays);  // 默认保留 30 根历史
  if (histStart === 0) return [merged, predIdx];
  return [merged.slice(histStart), predIdx - histStart];  // 调整预测起始索引
}
```

- 输入：历史 K 线 + 预测 K 线合并数组（可能数百根）
- 输出：最多 **30 根历史 + 3 根预测** = 33 根用于展示（避免 ECharts 拥挤）
- 返回值同时修正 `predStartIdx`（因为 slice 后索引偏移）

#### renderCandlestick()：统一渲染逻辑

所有 tab 中的 K 线图复用统一逻辑（大盘点击个股、策略详情、Kronos 预测 modal 等）：
- X 轴：日期
- 主图：蜡烛图（OHLC）
- 副图：成交量柱状图（颜色跟随涨跌）
- 预测区域：特殊虚线样式 + 背景色标记

#### Kronos 预测 UI 规范

当渲染带预测的 K 线图时（`_klinePredictOption`），严格遵循以下视觉规范：

**"预测"标签带**：
- 黄底黑字文本标签 `预 测`
- **800 字重**（`font-weight: 800`）
- **黄色辉光**（`text-shadow` + `box-shadow`）

**预测区域视觉标记**：
- **淡黄底色**：`rgba(250,204,21,.14)` 覆盖预测柱背景
- **虚线边框圈定**：0.75 alpha 透明度的黄色虚线边框 `border-style: dashed`
- **分界竖线**：1.5px 黄色虚线 `markLine` 标记历史/预测交界（`predStartIdx` 位置）

**蜡烛样式差异**：
- 历史区：实心红/绿蜡烛
- 预测区：空心虚线边框蜡烛（[app/static/app.js#L1040-L1048](file:///workspace/app/static/app.js#L1040-L1048)）

**标题栏同步**：
K 线图标题栏同步显示统计：
- 「预测3日 X%」——预测区间理论涨跌幅
- 「实际 X%」——待实际数据走出后替换
- 实际数据可用后，legend 切换显示"叠加蜡烛"对比模式

---

### 12.5 Modal 系统：renderPredictModal(symbol)

全站所有股票卡片（热门个股、漏斗命中、公告股、预测股、策略命中）点击后均复用同一个 Kronos 预测详情 Modal（`#predictModal`），禁止跳转或 `window.open`。

流程：
1. 点击卡片 → 调用 `renderPredictModal(symbol)`
2. 显示 Modal 遮罩（`display: flex`）
3. 异步 `fetch('/api/predict/{symbol}/kronos')` 获取预测数据
4. 调用 `renderCandlestick()` 渲染带预测区的 K 线图
5. 展示预测明细表格、置信度、3日/5日涨跌预测
6. 关闭按钮 / 点击遮罩空白区 → 关闭 Modal

**核心 bug 回归**：Playwright E2E 测试强制验证"点击卡片不跳转、不刷新、不 window.open，必须弹出 modal"（详见第16章 16.3.2 节）。

---

### 12.6 styles.css 体系

样式文件：[app/static/styles.css](file:///workspace/app/static/styles.css)

#### CSS 变量体系（:root）

[app/static/styles.css#L1-L42](file:///workspace/app/static/styles.css#L1-L42)

| 类别 | 变量前缀 | 示例 |
|------|---------|------|
| 背景色 | `--bg-*` | `--bg: #0b0f19` / `--panel: #111827` |
| 辉光/玻璃 | `--glass-*` / `--glow-*` | `--glass-blur: blur(12px) saturate(150%)` |
| 文字 | `--text-*` | `--text: #e6edf7` / `--muted: #94a3b8` |
| 卡片/边框 | `--card-*` / `--line-*` | `--card-bg: #111827` / `--line: #1e293b` |
| 功能色 | `--up/--down/--focus/--buy` | 涨 `--up: #ef4444` 跌 `--down: #10b981` |
| 字体 | `--font-*` | sans: Inter / mono: JetBrains Mono |
| 圆角/阴影 | `--radius-*` / `--shadow*` | `--radius: 12px` |

#### Glassmorphism 实现

```css
:root {
  --glass-bg: rgba(17, 24, 39, 0.65);
  --glass-border: rgba(255, 255, 255, 0.06);
  --glass-blur: blur(12px) saturate(150%);
}
.card {
  background: var(--glass-bg);
  backdrop-filter: var(--glass-blur);
  -webkit-backdrop-filter: var(--glass-blur);
  border: 1px solid var(--glass-border);
  border-radius: var(--radius);
}
```

#### 响应式布局

- **Grid**：大盘两列拆分（热门个股左 + K线右）、数据中心底部两列（任务历史左 + 完整性报告右）
- **Flex**：Sidebar 垂直布局、顶栏水平布局、卡片列表排列
- 无 media query 断点（桌面端优先设计，最小宽度约 1280px）

---

### 12.7 数据获取模式

#### fetch + Promise 链（无 axios）

```js
fetch('/api/funnel')
  .then(r => r.json())
  .then(data => {
    state.funnel = data;
    renderFunnel();
  })
  .catch(err => {
    renderAlertBanner('加载漏斗失败: ' + err.message, 'error');
  });
```

#### 通用错误处理：红色 alert banner

所有 `catch(err)` 路径统一调用 `renderAlertBanner(message, 'error')`，渲染页面顶部红色横条提示，3-5 秒后自动消失。Toast 样式由 `.toast-error` 类控制（[app/static/app.js#L134-L145](file:///workspace/app/static/app.js#L134-L145)）。

---

# 第四部分 研发链路、扩展与运维

## 第13章 FirstLimit Alpha研发链路

### 本节导读

从本章开始进入"研发链路、扩展与运维"部分。FirstLimit Alpha 是一个独立的**首板介入、捕捉后续连板**选股研发模块，拥有从数据构建到回测的 7 阶段完整离线研发链路。

本章与第7章（策略服务）、第8章（AI服务）、第11章（路由层 §9 FirstLimit Alpha router）内容关联：
- 策略引擎评分规则可消费 FirstLimit 输出的 TopN 候选
- 主系统的 Kronos 预测可作为 FirstLimit 的辅助特征
- 7 个 FastAPI 端点通过 router 暴露（详见第11章 §9）

---

### 13.1 模块定位

引用 [strategy/first_limit_alpha/README.md](file:///workspace/strategy/first_limit_alpha/README.md#L3-L4)：

> 面向 A 股短线场景的"首板介入、捕捉后续连板"选股模块设计文档。目标不是预测"会不会涨停"，而是识别：`前期震荡后，在第一个涨停当天介入，未来 2-5 个交易日继续强势甚至连板的候选股票。`

**核心交易逻辑**（README 总结）：
- 找出"前期横盘/震荡"
- 在"第一次涨停当天"介入
- 预测后续是否还有明显溢价或继续连板
- **收盘后决策，次日开盘介入（无未来函数）**

属于：事件驱动 + 市场情绪驱动 + 微结构特征驱动 + 强交易执行约束，非传统中低频因子选股。

---

### 13.2 目录结构

```
strategy/first_limit_alpha/
├── README.md              # 模块设计文档
├── AI_PROMPTS.md          # AI 辅助研发提示词
├── schema.py              # 配置 dataclass（SampleBuildConfig/LabelConfig/FeatureConfig/...）
├── data_builder.py        # 阶段 1：数据构建
├── labeling.py            # 阶段 2：标签
├── features.py            # 阶段 3：特征工程
├── feature_store.py       # 特征/数据持久化 helper
├── modeling.py            # 阶段 4：Baseline 建模核心
├── train_baseline.py      # 阶段 4：Baseline 训练脚本
├── sequence_dataset.py    # 阶段 5：序列数据集
├── sequence_model.py      # 阶段 5：GRU 序列模型
├── train_sequence.py      # 阶段 5：序列模型训练脚本
├── inference.py           # 阶段 6：在线推理
└── backtest.py            # 阶段 7：回测
```

---

### 13.3 7 阶段研发链路

#### 阶段 1：数据构建 — data_builder.py

**文件**：[strategy/first_limit_alpha/data_builder.py](file:///workspace/strategy/first_limit_alpha/data_builder.py)

**核心类**：`FirstLimitAlphaDataBuilder`（[strategy/first_limit_alpha/data_builder.py#L30-L33](file:///workspace/strategy/first_limit_alpha/data_builder.py#L30-L33)）

**输入**：`market_kline.db` 中 `kline_daily` 表全历史 K 线（symbol/trade_date/open/high/low/close/volume/amount），对应表结构详见第5章 5.3.2 节。

**输出**：首板样本集（每一行 = 一个"首板日"事件样本）

**样本筛选逻辑**（`_prepare_symbol_frame` 方法）：
1. 按 symbol 分组，逐条扫描
2. 计算 `board_limit_pct(symbol)`：主板 10%，创业板/科创板 20%（[strategy/first_limit_alpha/data_builder.py#L15-L18](file:///workspace/strategy/first_limit_alpha/data_builder.py#L15-L18)）
3. 条件：
   - **前 N 日无涨停**：`prior_limitup_count` 滑动窗口统计为 0
   - **当日达到涨停**：`pct_change >= limit_pct - 0.0025`（容许极小误差）
   - **前期振幅收敛**：震荡压缩条件（配合 SampleBuildConfig 参数）
4. 首板日标记为 `sample_type = "first_limit"`

---

#### 阶段 2：标签 — labeling.py

**文件**：[strategy/first_limit_alpha/labeling.py](file:///workspace/strategy/first_limit_alpha/labeling.py)

**核心函数**：`compute_sample_labels(frame, idx, label_cfg)`（[strategy/first_limit_alpha/labeling.py#L10-L68](file:///workspace/strategy/first_limit_alpha/labeling.py#L10-L68)）

**规则**：在首板日（第 idx 行）之后取未来 K 线，计算 3 套标签：

##### 标签 A：连板延续（label_continuation）
- **定义**：首板后 2 日内（`continuation_horizon=2`）**再出现任意一次涨停** → y=1
- **计算**：`next_2d_limit_up = int(cont_window["is_limit_up"].astype(bool).any())`

##### 标签 B：短期强势（label_strong_3d）
- **定义**：未来 3 日内（`strong_horizon=3`）**最高涨幅 ≥ 12%** → y=1
- **计算**：`ret_high_3d = (max_high_3d / entry_open) - 1.0`，与 `strong_threshold=0.12` 比较

##### 标签 C：断板风险（label_break_risk）
- **定义**：满足以下任一条件即为风险 → y=1
  1. 次日最低回撤 ≤ `break_drawdown_threshold`（如 -5%）
  2. 次日收盘收益 < -1%（明显走弱）
  3. 次日收盘低于首板收盘价（未接住）

标签计算同时产出辅助字段：未来每日 OHLC (`d1_open`~`d5_close`)、未来涨停计数、分阶段收益率等，供回测阶段直接使用（避免二次重算）。

---

#### 阶段 3：特征工程 — features.py

**文件**：[strategy/first_limit_alpha/features.py](file:///workspace/strategy/first_limit_alpha/features.py)

**规模**：70+ 特征，分 6 层：

##### 第 1 层：价格行为特征
- 历史收益：`ret_1d_prev` / `ret_3d` / `ret_5d` / `ret_10d` / `ret_20d`
- 振幅：`amp_3d` / `amp_5d` / `amp_10d` / `amp_20d`
- 波动率：`volatility_5d` / `volatility_10d` / `volatility_20d`
- 均线偏离：`close_vs_ma5/10/20/60`、`ma5_vs_ma10`、`ma10_vs_ma20`
- 距前高距离、高开幅度、K 线实体比例、上下影线比例

##### 第 2 层：成交与换手特征
- 成交额绝对值 + 均值比：`avg_volume_5/10/20/60` / `avg_amount_5/10/20/60`
- 换手率、量比（当日量 / 5日均量）
- 炸板次数、封单金额估算（需分时数据，若无则降级）

##### 第 3 层：板块与情绪特征
- 所属概念数量、概念热度排名
- 同题材涨停家数、连板高度（市场层面情绪）
- 炸板率（市场整体）、涨跌比 `market_adv_dec_ratio`
- 上涨比例 `market_up_ratio`、涨停比例 `market_limit_up_ratio`
- 派生：`_prepare_market_context()` 按交易日聚合市场级指标（[strategy/first_limit_alpha/features.py#L39-L58](file:///workspace/strategy/first_limit_alpha/features.py#L39-L58)）

##### 第 4 层：个股属性特征
- 总市值、流通市值
- 行业分类编码
- 次新标记（上市天数）
- 历史股性：过去 60/120 日涨停次数、连板次数

##### 第 5 层：微结构特征
- 封板时间（早板/午板/尾盘板，需分时数据）
- 首次触板时间
- 开板次数
- 回封速度（从开板到再封板的时间）
- 封板到收盘期间稳定性

##### 第 6 层：派生交互特征
- `换手 × 市值`：高换手小票 vs 高换手大盘股含义不同
- `板块热度 × 封板时间`：热门题材 + 早封板胜率高
- `量比 × 振幅压缩`：放量突破压缩区
- `连板高度 × 炸板率`：高情绪期风险/收益不对称

---

#### 阶段 4：Baseline 训练 — modeling.py + train_baseline.py

**文件**：[strategy/first_limit_alpha/modeling.py](file:///workspace/strategy/first_limit_alpha/modeling.py)、[strategy/first_limit_alpha/train_baseline.py](file:///workspace/strategy/first_limit_alpha/train_baseline.py)

##### 模型选择

- **优先 LightGBM**：`from lightgbm import LGBMClassifier`，安装失败自动回退
- **回退 RandomForest**：`from sklearn.ensemble import RandomForestClassifier`（[strategy/first_limit_alpha/modeling.py#L11-L22](file:///workspace/strategy/first_limit_alpha/modeling.py#L11-L22)）

##### 三目标并行训练

`TARGETS` 字典对应 3 套标签（[strategy/first_limit_alpha/modeling.py#L24-L28](file:///workspace/strategy/first_limit_alpha/modeling.py#L24-L28)）：
```python
TARGETS = {
    "continuation": "label_continuation",  # 连板延续
    "strong_3d":    "label_strong_3d",     # 短期强势
    "break_risk":   "label_break_risk",    # 断板风险
}
```

每个目标独立训练一个分类器，最终产出 3 个概率值 `p_continue_limit / p_strong_3d / p_fail`。

##### 切分方式：Walk-Forward Rolling（严禁随机切分）

`_split_dates()` 按**时间顺序**切分：
- **train**：前 70% 日期
- **valid**：中间 15% 日期
- **test**：后 15% 日期
- 禁止 `train_test_split(shuffle=True)` — 会引入未来信息

##### 融合评分：first_limit_score

在推理阶段加权融合（[strategy/first_limit_alpha/inference.py#L23-L30](file:///workspace/strategy/first_limit_alpha/inference.py#L23-L30)）：
```python
frame["first_limit_score"] = 100.0 * (
    0.45 * frame["proba_continuation"] +  # 连板延续：权重 45%
    0.40 * frame["proba_strong_3d"]      +  # 短期强势：权重 40%
    0.15 * (1.0 - frame["proba_break_risk"])  # 非断板风险：权重 15%
).clip(0.0, 100.0)
```

---

#### 阶段 5：序列训练 — sequence_dataset.py + sequence_model.py + train_sequence.py

**文件**：
- 数据集：[strategy/first_limit_alpha/sequence_dataset.py](file:///workspace/strategy/first_limit_alpha/sequence_dataset.py)
- 模型：[strategy/first_limit_alpha/sequence_model.py](file:///workspace/strategy/first_limit_alpha/sequence_model.py)
- 训练：[strategy/first_limit_alpha/train_sequence.py](file:///workspace/strategy/first_limit_alpha/train_sequence.py)

##### 序列输入构造

`FirstLimitSequenceDatasetBuilder`（[strategy/first_limit_alpha/sequence_dataset.py#L13-L16](file:///workspace/strategy/first_limit_alpha/sequence_dataset.py#L13-L16)）：
- 窗口长度：20-40 日 OHLCV 序列（`seq_len` 参数）
- 6 个特征通道（[strategy/first_limit_alpha/sequence_dataset.py#L45](file:///workspace/strategy/first_limit_alpha/sequence_dataset.py#L45)）：
  `ret`（收益率）/ `range`（振幅）/ `body`（实体）/ `upper`（上影）/ `lower`（下影）/ `volume_ratio`（量比）

##### GRU 序列模型

`FirstLimitSequenceModel`（[strategy/first_limit_alpha/sequence_model.py#L7-L28](file:///workspace/strategy/first_limit_alpha/sequence_model.py#L7-L28)）：
- **编码器**：单层 GRU，`input_size=6`，`hidden_size=48`
- **Dropout**：0.1 正则化
- **三输出头**（与 Baseline 对齐）：
  - `head_cont` → 连板延续
  - `head_strong` → 短期强势
  - `head_risk` → 断板风险

##### 设计目的

验证**多日时序信息相对于表格特征是否有增量价值**。如果 GRU 显著优于 LightGBM，可考虑在生产中融合两者输出。

---

#### 阶段 6：推理 — inference.py

**文件**：[strategy/first_limit_alpha/inference.py](file:///workspace/strategy/first_limit_alpha/inference.py)

**核心类**：`FirstLimitInferenceEngine`（[strategy/first_limit_alpha/inference.py#L10-L14](file:///workspace/strategy/first_limit_alpha/inference.py#L10-L14)）

**加载模型**：`joblib.load(artifact_path)`，bundle 包含：
- `models`：{continuation, strong_3d, break_risk} 三个训练好的分类器
- `feature_columns`：特征列名列表（保证特征顺序一致）
- `metadata`：训练日期、验证 AUC 等元信息

**推理流程**：
1. 输入指定交易日的**首板样本列表** DataFrame
2. 对齐特征列 → `x = frame[self.features]`
3. 三个分类器分别 `predict_proba(x)[:, 1]` → 3 列概率
4. 加权融合 → `first_limit_score`（0-100 分）
5. 按 `trade_date` + `score` 降序排序，输出 TopK 候选

---

#### 阶段 7：回测 — backtest.py

**文件**：[strategy/first_limit_alpha/backtest.py](file:///workspace/strategy/first_limit_alpha/backtest.py)

**核心类**：`FirstLimitBacktester`（[strategy/first_limit_alpha/backtest.py#L11](file:///workspace/strategy/first_limit_alpha/backtest.py#L11)）

##### TopK 选股回测参数

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `score_threshold` | 60 | 最低入选分 |
| `top_k` | 5 | 每日最多选几只 |
| `hold_days` | 3 | 最长持有天数 |
| `take_profit` | 0.08 | 止盈 8% |
| `stop_loss` | -0.05 | 止损 -5% |
| `fee_bps` | 3 | 手续费 0.03% |
| `slippage_bps` | 1 | 滑点 0.01% |

##### 出场逻辑（逐笔模拟）

```
for day in 1..hold_days:
    if 当日最高 / 入场价 - 1 >= take_profit:
        止盈出场, exit_reason = "take_profit"
    elif 当日最低 / 入场价 - 1 <= stop_loss:
        止损出场, exit_reason = "stop_loss"
    elif day == hold_days:
        收盘平仓, exit_reason = "hold_3d"
```

##### 输出指标

```python
summary = {
    "trade_count":   交易次数,
    "win_rate":      胜率,
    "avg_return":    平均单笔收益,
    "cum_return":    累计收益,
    "max_drawdown":  最大回撤,
    # 额外：盈亏比 / 夏普 / 卡玛
}
```

同时返回每笔交易明细（入场价、出场原因、日期、symbol、score）和权益曲线。

---

### 13.4 FastAPI 接口：7 个端点

对应 [app/routers/first_limit_alpha.py](file:///workspace/app/routers/first_limit_alpha.py)，端点详解见第11章 §9：

| 顺序 | 端点 | 对应阶段 |
|------|------|---------|
| 1 | `/dataset/build` | 阶段 1：数据构建 |
| 2 | `/features/build` | 阶段 3：特征构建（标签在阶段 1 已同步计算） |
| 3 | `/train/baseline` | 阶段 4：Baseline 训练 |
| 4 | `/train/sequence` | 阶段 5：序列模型训练 |
| 5 | `/inference/run` | 阶段 6：推理打分 |
| 6 | `/backtest` | 阶段 7：回测最新推理结果 |
| 7 | `/status` | 全局状态查询 |

服务封装：[app/services/first_limit_alpha_service.py](file:///workspace/app/services/first_limit_alpha_service.py) 中的 `FirstLimitAlphaService` 桥接 FastAPI 与 strategy 模块。

---

### 13.5 与 Alpha 主系统结合

#### 可复用组件
- **ConceptEngine 概念热度**：特征层第 3 层（板块与情绪）可直接消费 `FunnelService` 中的概念热度数据
- **市场情绪指标**：`predict_funnel_service` / `funnel_service` 的涨跌比、涨停数等可作为全局特征
- **Kronos 预测**：作为辅助特征（预测未来 3 日涨跌概率）提升首板质量判断

#### 接入主漏斗
- `first_limit_score` 排序后的 TopN 候选股，可作为独立候选池来源接入策略选股漏斗（`candidate` 池）
- 与其他策略（缩量启动、自定义规则）并行产生候选，再由综合分统一排序

---

## 第14章 扩展开发指南

### 本节导读

本章给出 6 种最常见扩展类型的**完整步骤 + 代码骨架 + 验证方法**：
1. 新增原子策略规则
2. 新增内置策略预设
3. 新增 API 路由（两种模式）
4. 新增 MCP 工具（铁律：严禁直连 SQL）
5. 新增后台调度循环（守护规则）
6. 新增数据表（变更前必须 checkpoint + stop，防 UEs）

扩展涉及的概念分别对应：第7章策略引擎（规则+预设）、第11章路由层、第9章 MCP Server、第2章 lifespan 后台循环、第5章数据层 schema。

---

### 14.1 新增原子策略规则

原子规则是自定义策略中心的最小单元。规则引擎定义在 [app/services/strategy_rules.py](file:///workspace/app/services/strategy_rules.py)。

#### 步骤

1. 在 `strategy_rules.py` 编写 evaluator 函数
2. 定义 `RuleSpec` 并加入 `RULE_REGISTRY`
3. 写 pytest 用例验证

#### 代码骨架：示例 rule_daily_amplitude（日振幅过滤）

假设需求：新增"指定窗口内，单日振幅最大值必须低于阈值"的规则。

##### 步骤 1：编写 evaluator

在 [app/services/strategy_rules.py](file:///workspace/app/services/strategy_rules.py) 中 `# ── 规则实现 ──` 区块（约 L137 之后）追加：

```python
def _rule_max_daily_amplitude(ctx: RuleContext, p: dict[str, Any]) -> RuleResult:
    lookback = int(p.get("lookback", 20))
    max_amp_pct = float(p.get("max_amp_pct", 8.0))
    kline = ctx.kline[-lookback:]
    if len(kline) < max(5, lookback // 2):
        return RuleResult(passed=False, label="K线不足", detail=f"至少需要 {max(5, lookback//2)} 根")
    max_pct = 0.0
    for row in kline:
        prev = float(row.get("close") or 0)
        hi = float(row.get("high") or 0)
        lo = float(row.get("low") or 0)
        if prev <= 0:
            continue
        amp = (hi - lo) / prev * 100.0
        if amp > max_pct:
            max_pct = amp
    ok = 0 < max_pct <= max_amp_pct
    return RuleResult(
        passed=ok,
        label=f"振幅 {max_pct:.1f}%",
        metric=round(max_pct, 3),
        detail=f"窗口 {lookback} 日，阈值 {max_amp_pct:.1f}%",
    )
```

函数签名固定：`rule_xxx(ctx: RuleContext, params: dict) -> RuleResult`。`ctx.kline` 按日期升序，最后一条为最新。

##### 步骤 2：定义 RuleSpec 并注册

找到 `_build_registry()` 函数（在 `RULE_REGISTRY` 之前，[app/services/strategy_rules.py#L554](file:///workspace/app/services/strategy_rules.py#L554)），在返回列表中追加：

```python
RuleSpec(
    code="max_daily_amplitude",
    title="单日振幅上限",
    category="volatility",          # price / volume / pattern / trend / filter / volatility
    description="统计指定窗口内，(最高-最低)/昨收 的最大值，要求低于阈值。用于过滤过度波动的个股。",
    params=[
        RuleParam(key="lookback", label="回看天数", type="int", default=20, min=5, max=120, step=1),
        RuleParam(key="max_amp_pct", label="最大单日振幅(%)", type="float", default=8.0, min=1.0, max=25.0, step=0.1),
    ],
    evaluator=_rule_max_daily_amplitude,
    min_kline_days=5,               # RuleSpec.evaluate() 自动拦截不足情况
),
```

#### 验证

1. **规则出现在目录中**：启动服务后 `curl http://127.0.0.1:18890/api/strategy/rules | grep max_daily_amplitude`，确认返回的 rules 列表包含新规则
2. **前端表单动态渲染**：浏览器打开策略选股 tab → 自定义策略中心 → 新建/编辑策略 → 下拉选择"单日振幅上限"，参数表应自动渲染 `lookback`（数字输入）和 `max_amp_pct`（数字输入）
3. **扫描生效**：在策略中勾选该规则保存 → 点击扫描 → 返回结果中命中个股的 rule_hits 应包含 `max_daily_amplitude` 的 passed/label/metric
4. **单测**：新建 `tests/test_custom_rules.py` 或追加到现有 `tests/test_custom_strategy.py`

---

### 14.2 新增内置策略预设

内置策略是系统预置、用户不可删除的"出厂模板"。

#### 步骤

1. 在 [app/services/custom_strategy.py](file:///workspace/app/services/custom_strategy.py) 的 `_builtin_strategies_seed()` 函数追加 `CustomStrategy` 项
2. `ensure_builtin_custom_strategies` 会在 main.py 启动时自动写入 DB（`is_builtin=1`）

#### 代码骨架

`_builtin_strategies_seed()` 定义位置：[app/services/custom_strategy.py#L125-L180](file:///workspace/app/services/custom_strategy.py#L125-L180)，已有 3 条内置策略。追加第 4 条：

```python
CustomStrategy(
    id="builtin_first_limit_helper",
    name="首板辅助过滤（内置）",
    description="价格 5-80 元 + 排除 ST/北交 + 近 20 日未涨停 + 今日接近涨停。作为 FirstLimit Alpha 的预处理过滤。",
    is_builtin=True,
    is_default=False,
    created_at=ts,
    updated_at=ts,
    rules=[
        StrategyRuleRef("exclude_boards", True, {"exclude_st": True, "exclude_bse": True}),
        StrategyRuleRef("price_range", True, {"min": 5.0, "max": 80.0}),
        StrategyRuleRef("no_limitup_recent", True, {"window": 20}),
        StrategyRuleRef("near_limit_up", True, {"tolerance_pct": 0.5}),  # 假设这是一个新规则
    ],
),
```

#### 注意：内置策略不可删除

`delete_custom_strategy()` 在 [app/services/sqlite_store.py#L283-L293](file:///workspace/app/services/sqlite_store.py#L283-L293) 中拦截：

```python
if int(row["is_builtin"]) == 1:
    raise ValueError("内置策略不可删除")  # main.py L537 转成 HTTP 403
```

对应路由层：[app/main.py#L533-L541](file:///workspace/app/main.py#L533-L541) 捕获 `ValueError` 返回 403。

内置策略的 `upsert` 也不会把 `is_builtin` 从 1 改成 0（`sqlilte_store.py#L256` 强制保留原值）。

---

### 14.3 新增 API 路由

#### 两种模式

##### 模式 A：快速原型 — 直接在 main.py 加 @app

适合独立功能、不依赖复杂注入的只读端点。追加到 [app/main.py](file:///workspace/app/main.py) 末尾（websocket 之前均可）：

```python
@app.get("/api/my-feature/summary")
async def my_feature_summary(param1: str | None = None):
    try:
        result = await service.some_method(param1)
        return result
    except ValueError as exc:
        raise HTTPException(status_code=400, detail=str(exc))
    except Exception as exc:
        raise HTTPException(status_code=500, detail=f"查询失败: {exc}")
```

##### 模式 B：模块化 — 新建 router 文件 + init 注入

适合功能集中、后续可能独立迁移的模块。参考 `kline router` 模式（第11章 §11.3）。

**步骤**：

1. 新建文件 `app/routers/my_feature.py`

```python
from __future__ import annotations
from fastapi import APIRouter, HTTPException

from app.services.funnel_service import FunnelService

router = APIRouter(tags=["MyFeature"])

_service: FunnelService | None = None

def init_my_feature_router(service: FunnelService) -> APIRouter:
    """把依赖写入模块级全局变量；供 main.py 调用。"""
    global _service
    _service = service
    return router

@router.get("/my-feature/summary")
async def get_my_feature_summary(symbol: str | None = None):
    if _service is None:
        raise HTTPException(status_code=500, detail="my_feature router 未初始化")
    try:
        return await _service.some_method(symbol)
    except KeyError:
        raise HTTPException(status_code=404, detail="symbol not found")

@router.post("/my-feature/recompute", status_code=202)
async def recompute_my_feature():
    """长任务示例：202 Accepted + create_task 后台跑"""
    if _service.running:
        raise HTTPException(status_code=409, detail="任务已在执行中")
    asyncio.create_task(_service.run_long_compute())
    return {"success": True, "message": "任务已启动", "snapshot": _service.get_snapshot()}
```

2. 在 [app/main.py#L40-L42](file:///workspace/app/main.py#L40-L42) 的 import 区块追加：
   ```python
   from app.routers.my_feature import init_my_feature_router
   ```

3. 在 [app/main.py#L281-L282](file:///workspace/app/main.py#L281-L282) 之后，`app.mount` 之前追加：
   ```python
   app.include_router(init_my_feature_router(service), prefix="/api")
   ```

#### 最佳实践

| 场景 | 做法 |
|------|------|
| 长任务（扫描/同步/预测/训练） | 返回 `202 Accepted`，用 `asyncio.create_task()` 后台执行，**禁止同步阻塞请求线程** |
| 并发保护 | 检查 service.running 标志，冲突抛 `HTTPException(409)` |
| 错误分层 | `ValueError` → 400 参数错误；`KeyError` → 404 资源不存在；`RuntimeError` → 503 不可用；兜底 Exception → 500 |
| 中文 detail | `HTTPException(status_code=400, detail="缺少必要参数 symbol")` |

---

### 14.4 新增 MCP 工具

MCP Server 是 Hermes Agent 调用 Alpha 功能的唯一入口。文件：[app/mcp_server.py](file:///workspace/app/mcp_server.py)

#### 步骤

1. 在 `mcp_server.py` 新增 `@mcp.tool()` 异步函数
2. 函数内部**只调用 Alpha 已有 REST API**，复用 `_get` / `_post` helper
3. 返回 `json.dumps(..., ensure_ascii=False, indent=2)` 字符串

#### 规则（严禁违反）

- **禁止 MCP 工具直接访问 SQL / Service 对象**，必须经过 REST API（走 HTTP 到正在运行的后端服务）
- **严禁返回编造数据**。没有数据必须明确说"无数据"，禁止生成假股票、假预测

#### 代码骨架：新增 trigger_kline_sync 工具

```python
@mcp.tool()
async def trigger_kline_sync(trade_date: str | None = None, force: bool = False) -> str:
    """触发 K 线缓存同步任务（返回提交状态，非同步结果）。

    Args:
        trade_date: 指定交易日期 YYYY-MM-DD，默认最新
        force: 是否强制重新同步已有数据
    """
    params: dict = {"force": force}
    if trade_date:
        params["trade_date"] = trade_date
    data = await _post("/api/jobs/kline-cache/sync", params)
    return json.dumps(data, ensure_ascii=False, indent=2)
```

**禁止的反模式**：

```python
# ❌ 绝对不要这样写！绕过 REST 会绕过鉴权、日志、生命周期
@mcp.tool()
async def bad_tool():
    from app.services.kline_store import KlineSQLiteStore  # 直连 DB
    store = KlineSQLiteStore()
    return json.dumps(store.query_xxx())
```

#### 验证

- 本地运行 `python -m app.mcp_server`（stdio 模式），用 MCP Inspector 连入
- 或在 Agent 对话中触发对应意图，查看 `_get` / `_post` 是否正常请求到 `ALPHA_API_BASE`（默认 http://127.0.0.1:18890）

---

### 14.5 新增后台调度循环

所有后台循环挂在 FastAPI `lifespan` 上，保证启动/关闭生命周期一致。参考现有循环（详见第2章 §2.4）：
- `_ticker_loop`（1 分钟）[app/main.py#L151-L170](file:///workspace/app/main.py#L151-L170)
- `_predict_funnel_scheduler_loop`（5 分钟）[app/main.py#L173-L197](file:///workspace/app/main.py#L173-L197)
- `kline_cache_loop`（10 分钟）[app/routers/kline.py#L162-L181](file:///workspace/app/routers/kline.py#L162-L181)

#### 步骤

1. 在 `main.py` 顶部服务单例创建区块之后，写异步循环函数
2. 在 `lifespan` startup 中 `app.state.xxx_task = asyncio.create_task(...)`
3. 在 `lifespan` shutdown 的取消 key 列表中加入对应 task 名称

#### 代码骨架

##### 1. 写循环函数

```python
async def _my_scheduler_loop() -> None:
    """我的后台任务：每 X 分钟检查一次条件，满足则执行。"""
    await asyncio.sleep(60)  # 启动延迟，等服务就绪
    while True:
        try:
            # ============= 任务逻辑 =============
            should_run = check_if_due()
            if should_run and not my_service.running:
                print(f"[my_scheduler] scheduled run at {now_cn().isoformat()}")
                asyncio.create_task(my_service.run_long_job(trigger="auto"))
            # ============= 任务逻辑结束 ==========
        except Exception as exc:
            # ⚠️ 守护规则：循环体内必须自己捕获异常
            # 否则一个异常会让整个循环永久退出，后续永不执行
            print(f"[my_scheduler] error: {exc}")
        await asyncio.sleep(300)  # 5 分钟一次（按需调整）
```

##### 2. lifespan startup 注册

在 [app/main.py#L252-L258](file:///workspace/app/main.py#L252-L258) 的 yield 之前追加：

```python
app.state.my_scheduler_task = asyncio.create_task(_my_scheduler_loop())
```

##### 3. lifespan shutdown 清理

在 [app/main.py#L262-L264](file:///workspace/app/main.py#L262-L264) 的 key 列表追加：

```python
for key in [
    "backfill_task", "ticker_task", "kline_cache_task", "hermes_task", "monitor_task",
    "predict_funnel_task", "hot_stock_ai_task",
    "my_scheduler_task",   # ← 新增
]:
```

#### 守护规则（必读）

每个 `while True` 循环必须：
1. **自己套 try/except + print 异常**，禁止异常冒泡
2. **异常后 continue**（`await asyncio.sleep` 在 try 外，确保仍会 sleep），禁止循环终止
3. **任务执行用 `asyncio.create_task`**，不要直接 await 长任务（否则阻塞调度节拍）

---

### 14.6 新增数据表（funnel_state.db）

状态存储统一走 `SQLiteStateStore`，路径 `data/funnel_state.db`。定义位置：[app/services/sqlite_store.py](file:///workspace/app/services/sqlite_store.py)，对应表结构详见第5章 5.3.1 节。

#### 步骤

1. 在 `SQLiteStateStore._init_schema()` 中加 `CREATE TABLE IF NOT EXISTS`
2. 新增 `get_xxx / set_xxx / upsert_xxx` 方法
3. **变更前先 checkpoint + stop**（严格遵守本章下文 15.1 四步流程）

#### 代码骨架：新增 my_settings 表

##### 1. 在 _init_schema() 中加表

[app/services/sqlite_store.py#L21-L81](file:///workspace/app/services/sqlite_store.py#L21-L81)，在 `custom_strategies` 的 CREATE TABLE 之后、`conn.commit()` 之前追加：

```sql
conn.execute(
    """
    CREATE TABLE IF NOT EXISTS my_settings (
        key TEXT PRIMARY KEY,
        value_json TEXT NOT NULL,
        description TEXT NOT NULL DEFAULT '',
        updated_at TEXT NOT NULL
    )
    """
)
```

##### 2. 新增 get / set / list 方法

在 `SQLiteStateStore` 类内追加：

```python
def get_my_setting(self, key: str) -> Any | None:
    with self._connect() as conn:
        self._init_schema(conn)
        row = conn.execute(
            "SELECT value_json FROM my_settings WHERE key = ?", (key,)
        ).fetchone()
        if row is None:
            return None
        return self._loads_json(row["value_json"], default=None)

def set_my_setting(self, key: str, value: Any, description: str = "") -> None:
    from datetime import datetime
    with self._connect() as conn:
        self._init_schema(conn)
        conn.execute(
            """
            INSERT INTO my_settings (key, value_json, description, updated_at)
            VALUES (?, ?, ?, ?)
            ON CONFLICT(key) DO UPDATE SET
                value_json=excluded.value_json,
                description=excluded.description,
                updated_at=excluded.updated_at
            """,
            (
                key,
                self._dumps_json(value),
                description,
                datetime.now().isoformat(timespec="seconds"),
            ),
        )
        conn.commit()

def list_my_settings(self) -> list[dict[str, Any]]:
    with self._connect() as conn:
        self._init_schema(conn)
        rows = conn.execute(
            "SELECT key, value_json, description, updated_at FROM my_settings ORDER BY key"
        ).fetchall()
        return [
            {
                "key": r["key"],
                "value": self._loads_json(r["value_json"], default=None),
                "description": r["description"],
                "updated_at": r["updated_at"],
            }
            for r in rows
        ]
```

#### 变更流程（重要！防 UEs）

参考标准四步流程（详见第15章 §15.1）：

1. **先 checkpoint**（手动执行脚本，或调 `/api/admin/shutdown-prepare`）
2. **执行 `./stop.sh` 停服**
3. **修改代码（即上面的 schema + 方法）**
4. **执行 `./start.sh` 启动服务**

禁止直接在服务运行时改 schema 代码然后 `kill -9`，极易触发 SQLite WAL fsync 打断导致 macOS 内核 UEs 卡死。UEs 问题深度解析详见第15章 §15.4。

---

## 第15章 调试与运维

### 本节导读

本章聚焦 Alpha 系统的日常运维与问题排查：
- 标准变更四步流程（防 UEs 卡死的第一道防线）
- start.sh / stop.sh / restart.sh 三个脚本详解
- UEs 进程卡死深度解析（现象/根因/唯一解法/预防措施）
- 5 类常见问题排查清单
- Kronos Benchmark 性能对比

其中 UEs 防护在第2章 §2.2（shutdown 先 checkpoint 再 cancel）和第5章 §5.5（WAL checkpoint 三层机制）中已有前置说明，本章给出完整的根因链和操作手册。

---

### 15.1 标准变更四步流程

原文引用自 [AGENTS.md#L15-L20](file:///workspace/AGENTS.md#L15-L20)：

> **每次重启服务或修改代码前，必须严格按以下顺序执行，防止 SQLite 进程卡死（UEs 内核不可中断状态）：**
>
> ```
> 1. 关闭 SQLite 连接（触发 WAL checkpoint）
> 2. 停止服务
> 3. 修改代码
> 4. 启动服务
> ```

AGENTS.md 原文原因说明（[AGENTS.md#L65-L68](file:///workspace/AGENTS.md#L65-L68)）：
> SQLite 在 WAL 模式下，若服务被强制中断（`SIGKILL` 或任务被 `CancelledError` 打断）而未完成 `fsync`，macOS 内核会让进程进入 **UEs（Uninterruptible Sleep + Exiting）** 状态，此时连 `kill -9` 也无法清除，只能重启 Mac。

lifespan shutdown 虽已内置自动 checkpoint（`_checkpoint_all_sqlite` [app/main.py#L221-L239](file:///workspace/app/main.py#L221-L239)），但手动操作时额外执行一次步骤 1 可彻底规避风险。

---

### 15.2 手动 checkpoint 脚本

在任何需要中断服务 / 代码变更 / 备份数据库之前，对两个 SQLite DB 执行 WAL checkpoint（合并 WAL 日志回主库、清空 WAL 文件）：

```bash
python3 - <<'PY'
import sqlite3
for db in ["data/funnel_state.db", "data/market_kline.db"]:
    conn = sqlite3.connect(db, timeout=5)
    conn.execute("PRAGMA wal_checkpoint(TRUNCATE)")
    conn.close()
    print(f"checkpoint OK: {db}")
PY
```

`TRUNCATE` 模式（比 `PASSIVE` / `FULL` 更彻底）：
- 将所有 WAL 页写回主数据库文件
- 将 WAL 文件截断为 0 字节
- 确保下一次打开时没有遗留 fsync 风险

---

### 15.3 服务管理脚本

#### start.sh

脚本位置：[start.sh](file:///workspace/start.sh)

执行流程：

1. **加载 .env**：类 dotenv 解析（行内 `#` 注释、支持引号、跳过空行）[start.sh#L9-L15](file:///workspace/start.sh#L9-L15)
2. **强制 arm64 Python 检查**：调用 `platform.machine()`，若不是 `arm64` 直接退出并提示 UEs 风险（Rosetta/x86_64 Python + 阻塞网络调用会 UEs）[start.sh#L35-L45](file:///workspace/start.sh#L35-L45)
3. **端口冲突检测**：`lsof -iTCP:$PORT -sTCP:LISTEN` 查占用，非目标服务占用则报错提示（UEs 状态进程跳过 is_service 判断）[start.sh#L86-L93](file:///workspace/start.sh#L86-L93)
4. **PID 文件**：写入 `.run/funnel-${PORT}.pid`，用于 stop/restart 识别
5. **uvicorn 后台启动**：通过 Python subprocess 启动，stdout/stderr 合并写入日志文件（`start_new_session=True` 免 SIGHUP 影响）[start.sh#L128-L151](file:///workspace/start.sh#L128-L151)
6. **健康检查**：启动后循环 `curl` 首页最多 10 次（每次 0.5s），确认服务响应 [start.sh#L176-L181](file:///workspace/start.sh#L176-L181)

关键变量：
- `PORT`：默认 18890
- `RELOAD`：默认 0（后台稳定），`RELOAD=1 ./start.sh` 启用开发期热重载
- `PYTHON_BIN`：优先 `$HOME/arm-python/python/bin/python3.11`，其次 homebrew，兜底 python3
- `LOG_FILE`：`logs/server-${PORT}.log`

#### stop.sh

脚本位置：[stop.sh](file:///workspace/stop.sh)

执行流程：

1. **PID 文件清理**：遍历 `.run/*.pid`，逐个验证是否是合法服务进程（ps stat 含 U 则跳过 → 是 UEs）[stop.sh#L64-L76](file:///workspace/stop.sh#L64-L76)
2. **stop_pid() 兜底**：对每个 PID 先 `kill`（SIGTERM），循环 8 秒等待退出；仍存活则 `kill -9`（SIGKILL）[stop.sh#L49-L60](file:///workspace/stop.sh#L49-L60)
3. **pgrep 兜底**：按 `uvicorn app.main:app` 进程名再查一遍，清理 PID 文件之外的残留进程 [stop.sh#L79-L92](file:///workspace/stop.sh#L79-L92)
4. **UEs 状态进程提示**：发现 ps stat=U 的进程，打印"需重启 Mac 清理"[stop.sh#L83-L86](file:///workspace/stop.sh#L83-L86) 并跳过

#### restart.sh

脚本位置：[restart.sh](file:///workspace/restart.sh)

执行流程（严格 4 步）：

1. **prepare_app_shutdown()**：`curl POST /api/admin/shutdown-prepare` → 应用内部执行 SQLite checkpoint [restart.sh#L18-L25](file:///workspace/restart.sh#L18-L25)
2. **checkpoint_sqlite()**：脚本层面再做一次独立的 PRAGMA wal_checkpoint(TRUNCATE) 双保险 [restart.sh#L27-L46](file:///workspace/restart.sh#L27-L46)
3. **调用 stop.sh**
4. **调用 start.sh**

推荐：**任何代码变更后的重启都走 `./restart.sh`，不要手动 kill + 手动 start**。

---

### 15.4 UEs 进程卡死定位

#### 现象

- `ps aux | grep uvicorn` 显示 STAT 列 = **U**（Uninterruptible Sleep）或 **UEs** / **U+**
- `kill -9 <PID>` **无效**：进程仍在
- `lsof -iTCP:18890` 显示端口被该 PID 占着，但 curl 无法访问
- `cat /proc/<pid>/status`（macOS：`ps -p <pid> -o stat=,wchan=`）显示 wchan 停在 `fsync` / `sqlitevfs` 相关内核函数

#### 原因

SQLite 在 WAL 模式下执行写入 checkpoint 时会做 `fsync(fd)`。如果：
- 此时刚好收到 `SIGKILL`（无法捕获）
- 或 asyncio 任务在 `asyncio.to_thread(checkpoint)` 期间被 `CancelledError` 打断
- 或 Python 解释器异常退出

macOS 内核的 HFS+ / APFS 文件系统 `fsync` 调用一旦进入不可中断睡眠且被打断时处理不当，会导致进程进入僵尸级 U 态，调度器永不调度、信号永不处理。

#### 唯一解法

**重启 Mac**。没有任何用户态命令能清掉 UEs 进程（包括 `kill -9`、`sudo kill -9`、`gdb` detach、`signal` 注入）。

#### 预防

1. **严格遵守 4 步流程**：变更前先 checkpoint → stop → 改代码 → start
2. **禁止直接 kill -9**：哪怕服务看起来"卡死"，先 `curl /api/admin/shutdown-prepare`，再 `./stop.sh` 走 SIGTERM → SIGKILL 二级流程
3. **lifespan shutdown 顺序**：先 `_checkpoint_all_sqlite`（单独线程同步完成），再 `task.cancel()` 取消所有后台任务 — [app/main.py#L260-L270](file:///workspace/app/main.py#L260-L270)，顺序不能颠倒（详见第2章 §2.2）
4. **强制 arm64 Python**：start.sh L35-L45 检查，Rosetta 模式下网络调用 + SQLite 写入叠加时 UEs 概率高得多

---

### 15.5 日志

- **路径**：`logs/server-${PORT}.log`（如 `logs/server-18890.log`）
- **级别**：默认 INFO（uvicorn 默认）。若需 DEBUG：
  ```bash
  RELOAD=1 UVICORN_EXTRA_ARGS="--log-level debug" ./start.sh
  # 或手动修改 start.sh 中的 UVICORN_ARGS += ("--log-level", "debug")
  ```
- **轮转**：本项目未内置 logrotate，定期 `gzip` / 清理旧日志即可
- **快速 tail**：`tail -n 500 -f logs/server-18890.log | grep -E "error|ERROR|\[ticker\]|\[kline-cache\]"`

---

### 15.6 常见问题排查

#### 漏斗不刷新（策略选股页数据不更新）

1. 搜日志：`grep "\[ticker\] error" logs/server-18890.log`
2. 如果发现 ticker_loop 异常退出：检查 `_ticker_loop` 的 try/except 是否被某异常类型穿透（理论上 Exception 全捕获，但 BaseException 如 KeyboardInterrupt 除外）
3. 临时恢复：`./restart.sh`
4. 根本修复：在 `_ticker_loop` [app/main.py#L151-L170](file:///workspace/app/main.py#L151-L170) 的 except 分支追加 `import traceback; traceback.print_exc()` 找到具体出错行

#### K线同步卡住（数据中心进度条不动）

1. **看进度接口**：`curl http://127.0.0.1:18890/api/jobs/kline-cache/progress` — 确认 `total` / `done` / `current_symbol` 字段
2. **查积压队列**：调用 `kline_cache_service._queue`（调试时加临时路由或 log）看是否有阻塞
3. **看日志**：`grep "\[kline-cache\]" logs/server-18890.log | tail -30`
4. **常见根因**：AkShare 接口限流 / 网络超时 — `provider.get_realtime_snapshot` 在某只股票上卡住 60s 超时。等超时后会自动继续（`run_if_due` 内部有超时控制）

#### Kronos 预测失败

1. 典型报错 HTTP 400：`ValueError: K线数据不足，需要至少 180 根，实际只有 XX 根`
2. **解决**：先去"数据中心"tab → 全量补缺，或针对该股增量补历史 K 线（Kronos 默认 lookback=180）
3. 预测失败 503：模型未下载（`NeoQuasar/Kronos-mini` 等），首次调用时会触发 huggingface 下载，失败通常是网络问题
4. 预测失败 500：pandas DataFrame 空值等具体异常，detail 会带中文信息 `预测失败: xxx`

#### Agent 失败（daily_review / notice_review / full_diagnosis）

1. **检查 OPENAI_API_KEY**：`.env` 文件中是否正确配置，启动日志第一行有提示"已加载 .env"
2. **检查 HERMES_AGENT_URL**：如需独立 Hermes 服务，确认 URL 可 curl
3. **查熔断计数**：`curl http://127.0.0.1:18890/api/agent/status` 看 `consecutive_failures` 字段，连续失败会触发降级逻辑（详见第9章 §9.1.3 熔断机制）
4. 具体执行错误：`curl http://127.0.0.1:18890/api/agent/tasks?limit=5` 看最近任务的 `message` / `error` 字段

#### 模拟盘价格不更新（持仓浮盈/浮亏不变）

模拟盘价格源分 4 级（`_get_realtime_price` 返回的 `price_source` 字段，[app/main.py#L764-L776](file:///workspace/app/main.py#L764-L776)）：

| source | 含义 | 原因 |
|--------|------|------|
| `em_live` | 东财实时（最佳） | 正常 |
| `db_fallback` / `db_fallback_after_timeout` | 东财超时后，从 K 线 DB 取昨收 | 东财 spot 接口超时/限流 |
| `stale_cache` | 连 DB fallback 都失败，用内存旧缓存 | 长期断网或 DB 锁 |
| `none` | 完全无法获取价格 | 该代码停牌/退市，或数据严重缺失 |

判断：
```bash
curl http://127.0.0.1:18890/api/paper/summary | grep price_source
```

如果长期是 `stale_cache` 而非 `em_live`：
- 检查网络连通性（东财 spot 是否被防火墙拦截）
- 检查 Python 是否是 arm64（Rosetta 模式下网络栈不稳）
- `tail logs/server-18890.log | grep "price update"` 看具体失败堆栈

---

### 15.7 Benchmark Kronos

位置：[tests/benchmark_kronos.py](file:///workspace/tests/benchmark_kronos.py)

用法：

```bash
python -m tests.benchmark_kronos
```

功能：
- 比较 3 个 Kronos 模型（Kronos-mini 4.1M / Kronos-small 24.7M / Kronos-base 102.3M）
- 不同 `sample_count`（1/10/20/50/100）下的加载时间、推理耗时
- 输出 JSON：`tests/benchmark_kronos_results.json`
- 用作选型参考：通常 `Kronos-mini` 性价比最优

---

## 第16章 测试体系

### 本节导读

本章讲解 Alpha 项目的双轨测试体系：后端 API 回归（pytest）+ 前端 E2E（Playwright），以及单测清单、一键测试脚本和 GitHub Actions CI 配置。

测试覆盖的端点和场景与第11章路由层（端点定义）、第12章前端架构（UI交互）直接对应。

---

### 16.1 双轨测试总览

Alpha 项目采用**后端 API 回归（pytest）+ 前端 E2E（Playwright）**双轨并行：

| 维度 | API 回归（pytest） | 前端 E2E（Playwright） |
|------|-------------------|---------------------|
| 目标 | 所有 read-only + 非破坏性端点返回码/字段正确 | UI 渲染、Tab 切换、核心交互无 JS 错误 |
| 运行方式 | `python3 -m pytest tests/test_api_regression.py -v` | `cd tests/e2e && npx playwright test` |
| 前提 | 后端已启动在 127.0.0.1:18890 | 后端已启动 + 已安装 Playwright Chromium |
| 典型耗时 | 1-3 分钟 | 2-5 分钟 |

配合**单元测试**（strategy / store / service 层）做细粒度验证。

---

### 16.2 API 回归测试

位置：[tests/test_api_regression.py](file:///workspace/tests/test_api_regression.py)

#### 结构

- 使用 `httpx.Client`（同步）或 `httpx.AsyncClient`（异步）连接**运行中的后端服务**（不是 TestClient）
- 每个 test class 对应一个功能模块
- 模块级 fixture `client` 共享 HTTP 连接（scope=module）
- BASE_URL 默认 `http://127.0.0.1:18890`，可通过环境变量 `ALPHA_TEST_BASE_URL` 覆盖

#### 覆盖分类（12 大类 / 40+ 端点）

引用 [tests/test_api_regression.py#L33-L100](file:///workspace/tests/test_api_regression.py#L33-L100) 及后续章节：

| 大类 | Test class | 用例数 | 典型端点 |
|------|-----------|--------|---------|
| 1. 静态页 | `TestStaticPages` | 5 | `/`、`/notice`、`/static/app.js`、`/static/styles.css`、`/static/index.html` |
| 2. 大盘行情 | `TestMarket` | 5 | `hot-concepts`、`hot-stocks`、`stock/realtime`、`stock/detail` × 2 |
| 3. 策略漏斗 | `TestFunnel` | 3+ | `funnel` 快照、`pool/move`（如开启）、`score/recompute` |
| 4. Kronos 预测 | `TestPredict` | 4 | `/api/predict/{symbol}/kronos`、`/api/predict-funnel`、`trigger`、`config` |
| 5. K线同步 | `TestKline` | 6 | `status`、`progress`、`stats`、`logs`、`report`、`/api/kline/{symbol}` |
| 6. 公告选股 | `TestNotice` | 2 | `funnel`、`keywords` |
| 7. Hermes Agent | `TestAgent` | 4 | `status`、`tasks`、`monitor/config`、`monitor/messages` |
| 8. 模拟盘 | `TestPaper` | 5 | `positions`、`history`、`summary`、`trades`、`settings` |
| 9. 已移除接口 | `TestRemovedEndpoints` | 21 | 21 个已删除端点必须返回 404（防回归） |
| 10. 公告详情 | `TestNoticeDetail` | 1 | `/api/notice/{symbol}/detail` |
| 11. 自定义策略 | `TestCustomStrategy` | 3+ | `rules`、`custom` CRUD 部分 |
| 12. 性能巡检 | `TestPerformance` | 5 | 关键只读端点 < 3s |

#### 性能巡检

对高频只读端点设置响应时间上限（典型 < 3s，盘后高峰期 < 5s 可接受）：

```python
@pytest.mark.performance
class TestPerformance:
    def test_funnel_under_3s(self, client):
        import time
        t0 = time.perf_counter()
        r = client.get("/api/funnel")
        assert r.status_code == 200
        elapsed = time.perf_counter() - t0
        assert elapsed < 3.0, f"funnel 响应过慢: {elapsed:.2f}s"
```

可以用 `-m performance` 只跑性能用例：
```bash
python3 -m pytest tests/test_api_regression.py -v -m performance
```

#### 兼容性测试

- **legacy 接口**：`/api/strategy/quiet-breakout`（缩量启动兼容接口）必须仍能正常返回 200 + 数据结构
- 防止重构旧接口时"顺手删掉"导致上游集成方故障

#### 移除接口回归（防误加回）

`TestRemovedEndpoints` 类覆盖 21 个历史上已删除的端点：
- `/api/proposal/*`（提案管理体系，已整体移除）
- `/api/hermes-ai/*`（Hermes 旧版 AI 能力，已合并到 agent/*）
- 其他历史遗留端点

每个端点必须返回 **404**。如果哪天有人手滑重新加回来，此测试立即报错拦截。

#### 运行命令

```bash
python3 -m pytest tests/test_api_regression.py -v --tb=short
# 只跑某类
python3 -m pytest tests/test_api_regression.py::TestMarket -v
# 带持续时间报告
python3 -m pytest tests/test_api_regression.py -v --durations=0
```

---

### 16.3 前端 E2E 测试（Playwright）

位置：[tests/e2e/ui.spec.js](file:///workspace/tests/e2e/ui.spec.js)

#### 环境准备

```bash
cd tests/e2e
npm install                               # 首次
npx playwright install chromium           # 首次，下载 Chromium 浏览器
npx playwright test                       # 运行全部
```

配置文件：`tests/e2e/playwright.config.js`（baseURL 默认指向 `http://127.0.0.1:18890`）

#### 覆盖 14 个用例

对应 [tests/e2e/ui.spec.js](file:///workspace/tests/e2e/ui.spec.js)：

##### 1. 页面整体加载与切换（8 用例）
- `首页正常加载`：sidebar 可见、market tab 默认 active、0 关键 JS 错误
- `切换到 [大盘] tab 不崩溃`
- `切换到 [数据中心] tab 不崩溃`
- `切换到 [策略选股] tab 不崩溃`
- `切换到 [公告选股] tab 不崩溃`
- `切换到 [预测选股] tab 不崩溃`
- `切换到 [智能监控] tab 不崩溃`
- `切换到 [模拟盘] tab 不崩溃`

每个 tab 切换用例都会：
1. 监听 pageerror 事件
2. 过滤非关键错误（ResizeObserver / favicon / WebSocket 404 / 断网 — IGNORED_ERRORS 正则白名单）
3. 断言 `#tab-xxx` 有 `.active` 类且可见
4. 断言 0 个关键 JS 错误

##### 2. 策略选股 tab — 自定义策略中心（2 用例）
- **视觉尺寸**：`#strategyCenterArea` 区块渲染后 width ≥ 600px 且 height ≥ 200px（内容足够多，防止 regress 成空壳）
- **规则卡片 ≥ 3 张**：`.sc-rule-card` count ≥ 3（内置规则集至少 3+）
- **核心 bug 回归 — 点击策略命中卡片**：
  - 点击 `.sc-hit-card` 后
  - 断言 `framenavigated` 事件**未触发**（不跳转）
  - 断言 `page.url()` 不变（不刷新）
  - 断言 `window.__opened.length === 0`（不 window.open）
  - 断言 `#predictModal` 可见（必须弹出 Kronos modal）
  - 对应历史 bug：旧版点击卡片后 `window.open` 新开页或跳转，体验差且丢失上下文

##### 3. 数据中心 tab（1 用例）
- **任务历史 grid cell 无水平 overlap**：取前 3 行任务，逐行检查 `.dc-td` 单元格 boundingBox，相邻单元格 x 坐标差绝对值 ≤ 2px 容差视为正常（禁止文字重叠）

##### 4. 智能进化（agent）tab（1 用例）
- **不出现提案管理子 tab**：确认 `.proposal-subtab` / `提案管理` 文本不存在（L23 决策移除了提案管理体系后，防止前端残留）

##### 5. 模拟盘 tab（1 用例）
- **主容器渲染**：`#tab-paper.active`、账户余额卡片、持仓列表容器可见

##### 6. 全站巡检（1 用例，汇总）
- **0 关键 JS 错误**：遍历所有 7 个 tab 后，汇总 pageerror 列表，必须为空

#### 运行参数

```bash
# 只跑策略中心
npx playwright test -g "策略中心"
# 调试模式（可视化逐步骤）
npx playwright test --ui
# 录视频
npx playwright test --video on
# 报告
npx playwright show-report
```

---

### 16.4 单元测试

`tests/` 目录下多个细粒度单测文件：

| 文件 | 测试内容 | 核心类/函数 |
|------|---------|------------|
| `tests/test_kline_store.py` | K 线 SQLite 读写正确性 | `KlineSQLiteStore` 的 save/get/update |
| `tests/test_sqlite_store.py` | 状态存储 CRUD | `SQLiteStateStore` 的 funnel_state / custom_strategies / kv_store |
| `tests/test_concept_engine.py` | 概念热度引擎 | `ConceptEngine` 概念匹配、热度计算 |
| `tests/test_custom_strategy.py` | 自定义策略模型+扫描 | `CustomStrategy`、扫描器规则逻辑 |
| `tests/test_data_provider_parse.py` | 数据解析 | `AkshareDataProvider` 的字段解析、归一化 |
| `tests/test_first_limit_alpha.py` | 首板模型全链路 | data_builder → labeling → features → modeling → inference → backtest |
| `tests/test_hot_stocks.py` / `test_hot_stock_ai_service.py` | 热门智能 | `HotStockAIService` 三池迁移 + 调度 |
| `tests/test_funnel_concepts.py` | 策略漏斗 + 概念 | `FunnelService` 综合 |
| `tests/test_notice_service.py` | 公告服务 | `NoticeService` 关键词打分 + 池管理 |
| `tests/test_paper_routes.py` | 模拟盘路由 | paper_trading 买入/卖出/平仓逻辑 |
| `tests/test_market_data_client.py` | 行情客户端 | `EastmoneyMarketDataClient` |
| `tests/test_kline_cache_service.py` | K线缓存服务 | 同步队列、去重、进度 |
| `tests/test_transition_rules.py` | 状态转移规则 | 池间转移条件校验 |
| `tests/test_tradingagents_adapter.py` | TradingAgents 适配器 | HTTP 请求、异常降级 |

单测不依赖后端运行，直接 import service 测（除 `test_api_regression.py` / E2E 外）。

---

### 16.5 一键测试脚本

位置：[tests/run_all_tests.sh](file:///workspace/tests/run_all_tests.sh)

#### 执行流程

```bash
./tests/run_all_tests.sh
```

1. **检查后端在线**：curl `/api/agent/status` 失败则直接 exit 1（提示先 `./start.sh`）
2. **API 回归**：`python3 -m pytest tests/test_api_regression.py -v --tb=short`，结果 tee 到 `tests/test_report.txt`
3. **E2E**：`cd tests/e2e && npm install`（首次）+ `npx playwright test`，结果追加到报告
4. **汇总**：全部通过输出 ✓，有失败输出 ✗ 并指向 `tests/test_report.txt`

返回码：
- `0`：双轨全通过
- 非 0：API_STATUS 或 E2E_STATUS 任一失败（见脚本末尾判断）

---

### 16.6 CI：GitHub Actions

位置：[.github/workflows/tests.yml](file:///workspace/.github/workflows/tests.yml)

典型触发条件：
- push 到 `main` 分支
- PR 打开/更新到 `main`

CI 步骤：
1. Checkout 代码
2. Setup Python 3.11 + Node 20
3. pip install -r requirements.txt
4. npm install（e2e 目录）+ playwright install chromium
5. 启动后端服务（`./start.sh` + 健康检查等待就绪）
6. 运行 `tests/run_all_tests.sh`
7. 上传 test_report.txt / Playwright trace / 截图作为 artifacts
8. 失败时通知（邮件 / Slack / webhook，按仓库配置）

---

# 附录：交叉引用速查表

| 我想知道… | 先查哪一章哪一节 |
|-----------|-----------------|
| 系统启动时干了什么？ | 第2章 2.1 FastAPI lifespan 启动流程 + 2.3 全局单例初始化依赖链（12层） |
| 改了策略参数为什么不生效？ | 第3章 3.1 四层配置来源优先级（默认值< YAML < 环境变量 < 运行时KV） |
| 某条评分是怎么算出来的？ | 第7章 7.1 StrategyEngine 盘中评分四维度公式（量价/概念/形态/动量） |
| 漏斗三池的状态存在哪里？ | 第5章 5.3.1 funnel_state.db 10张表完整字段 → `funnel_state` 表 |
| 为什么重启后昨日漏斗为空？ | 第7章 7.2 FunnelService.trade_date 跨日重置逻辑 |
| 如何新增一条选股规则？ | 第14章 14.1 新增原子策略规则（完整步骤+代码骨架） |
| Kronos 预测失败报"历史K线不足"？ | 第8章 8.1 KronosPredictService.lookback=180 + 第15章 15.6.3 常见问题 Kronos 预测失败 |
| Agent 任务连续失败后不执行了？ | 第9章 9.1.3 HermesRuntime 熔断机制（3次失败→冷却1h） |
| 点击策略卡片浏览器跳新页面？ | 第12章 12.5 Modal 系统；第16章 16.3.2 E2E 核心 bug 回归（点击卡片不跳转/不刷新/不window.open） |
| 进程卡死后 kill -9 也清不掉？ | 第2章 2.2 shutdown 顺序；第5章 5.5 WAL checkpoint；第15章 15.4 UEs 卡死深度解析（现象/根因/唯一解法/预防） |
| 如何在本地跑全量测试？ | 第16章 16.5 一键脚本 run_all_tests.sh（API回归+E2E双轨） |
| Kronos 模型选 mini/small/base 哪个？ | 第8章 8.1.3 Kronos 三件套模型规格 + 第15章 15.7 Benchmark Kronos 对比脚本 |
| MCP 工具为什么不能直连 SQL？ | 第9章 9.3 MCP Server 同源 REST 调用；第14章 14.4 新增 MCP 工具铁律 |
| 新增数据表后服务无法启动？ | 第14章 14.6 新增数据表变更流程（必须先checkpoint+stop，禁止运行时改schema） |
| 飞书通知不推送？ | 第10章 10.5 FeishuNotify（检查 FEISHU_WEBHOOK_URL 环境变量 + notify_sync_complete 已停用降噪） |

---

分章版详见同目录 01-架构总览.md ~ 16-测试体系.md