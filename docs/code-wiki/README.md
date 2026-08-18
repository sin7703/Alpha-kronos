# Alpha Code Wiki（开发者文档）

> 本 Wiki 面向**开发者与维护者**，深度讲解 Alpha 量化选股系统的代码实现原理、模块关系与扩展方式。
> 若你想了解产品功能与使用方法，请先阅读项目根目录的 [README.md](file:///workspace/README.md)。

---

## 如何阅读

推荐阅读顺序（从宏观到微观）：

```
01-架构总览 → 02-入口与生命周期 → 05-数据层设计
                        ↓
              03-配置系统 · 04-数据模型
                        ↓
         06/07/08/09/10 服务层详解（按需选读）
                        ↓
              11-路由层 · 12-前端架构
                        ↓
         14-扩展开发指南 · 15-调试与运维 · 16-测试体系
```

---

## 目录索引

### 第一部分：架构与基础

| # | 章节 | 核心内容 |
|---|------|---------|
| 01 | [架构总览.md](01-架构总览.md) | 项目定位、五层分层架构图、23个核心服务一览、5个后台调度循环、端到端数据流 |
| 02 | [入口与生命周期.md](02-入口与生命周期.md) | FastAPI lifespan、7个 startup 任务、shutdown checkpoint 顺序、全局单例初始化依赖链、路由注册 |
| 03 | [配置系统.md](03-配置系统.md) | 四层配置来源优先级、StrategyConfig 19项参数详解、11个环境变量清单、KV Store 配置存储格式 |
| 04 | [数据模型.md](04-数据模型.md) | 18 个 Pydantic 模型分类、StockCard 15 字段详解、PoolName Literal、KlinePoint 与 OHLCV+Amount 的取舍 |
| 05 | [数据层设计.md](05-数据层设计.md) | 双 SQLite 隔离设计、WAL + synchronous=NORMAL、10+ 张表逐字段用途、WAL checkpoint 三层机制、UEs 根因链、KlineSQLiteStore vs SQLiteStateStore 对比 |

### 第二部分：服务层详解（5 篇）

| # | 章节 | 覆盖模块 |
|---|------|---------|
| 06 | [服务层详解-数据服务.md](06-服务层详解-数据服务.md) | AkshareDataProvider（多源适配+缓存+降级）、EastmoneyMarketDataClient（async硬超时+重试）、KlineCacheService（队列+并发+三模式）、KlineSQLiteStore（批量upsert） |
| 07 | [服务层详解-策略服务.md](07-服务层详解-策略服务.md) | StrategyEngine（盘中评分公式+迁池规则）、FunnelService（tick 6步流程）、自定义策略中心（12原子规则+AND组合+3内置预设）、BacktestLab（180日TopK回测） |
| 08 | [服务层详解-AI服务.md](08-服务层详解-AI服务.md) | KronosPredictService（惰性加载/自适应设备/串行锁/GIL隔离四件套+推理流程）、HotStockAIService（六因子评分+三池阈值+轻量模式）、TradingAgentsAdapter（命令行调用+决策映射）、FirstLimitAlphaService（7步离线研发） |
| 09 | [服务层详解-Agent服务.md](09-服务层详解-Agent服务.md) | HermesRuntime（双模式/并发2/熔断3次冷却1h/180s超时/JSON容错）、HermesMemory（任务/监控配置/监控消息三张表）、MCP Server（16个工具分类/通过httpx同源REST调用） |
| 10 | [服务层详解-业务服务.md](10-服务层详解-业务服务.md) | NoticeService（双引擎打分+7类利好13利空）、PaperTradingService（费用模型+持仓/交易表+summary指标）、PredictFunnelService（板块→成分股→Kronos→Top10）、RealtimeHub（WS两事件广播）、FeishuNotify（CardBuilder+降噪） |

### 第三部分：接口与前端

| # | 章节 | 核心内容 |
|---|------|---------|
| 11 | [路由层设计.md](11-路由层设计.md) | 路由组织三模式、15 大功能分类端点表、kline router 注入模式、GET/POST/长任务/错误响应模式 |
| 12 | [前端架构.md](12-前端架构.md) | 原生零构建技术栈、8 Tab SPA 结构、Glassmorphism 主题、switchTab 状态机、轮询管理器、WS+轮询双通道、ECharts K 线统一渲染器、Kronos 预测 UI 四件套规范、Modal 系统 |

### 第四部分：研发链路与扩展

| # | 章节 | 核心内容 |
|---|------|---------|
| 13 | [FirstLimit Alpha 研发链路.md](13-FirstLimit%20Alpha%20研发链路.md) | 首板介入连板预测模块定位、7 阶段端到端（数据构建→3套标签→70+特征6层→Baseline LightGBM→GRU序列→在线推理→TopK回测）、7 个 FastAPI 端点、与主系统结合建议 |
| 14 | [扩展开发指南.md](14-扩展开发指南.md) | 6 种扩展类型的完整步骤+代码骨架+验证方法：新增原子规则/新增内置策略/新增API路由/新增MCP工具/新增后台循环/新增数据表 |
| 15 | [调试与运维.md](15-调试与运维.md) | 标准变更四步流程、手动 checkpoint 脚本、start/stop/restart 三脚本详解、UEs 卡死深度解析（现象/原因/解法/预防）、5 类常见问题排查清单、Kronos Benchmark |
| 16 | [测试体系.md](16-测试体系.md) | 双轨测试（pytest API 回归 + Playwright E2E）、12 大类 40+ 端点回归覆盖、14 个 E2E 用例详情（含核心 bug 回归）、单测清单、一键脚本、GitHub Actions CI |

---

## 交叉引用速查

| 我想知道… | 先看哪篇 |
|-----------|---------|
| 系统启动时干了什么？ | 02-入口与生命周期.md |
| 改了策略参数为什么不生效？ | 03-配置系统.md（四层优先级） |
| 某条评分是怎么算出来的？ | 07-策略服务.md → StrategyEngine 评分算法 |
| 漏斗三池的状态存在哪里？ | 05-数据层设计.md → funnel_state 表 |
| 为什么重启后昨日漏斗为空？ | 07-策略服务.md → FunnelService.trade_date 跨日重置逻辑 |
| 如何新增一条选股规则？ | 14-扩展开发指南.md → 14.1 节 |
| Kronos 预测失败报"历史K线不足"？ | 08-AI服务.md + 15-调试与运维.md → 数据中心补数 |
| Agent 任务连续失败后不执行了？ | 09-Agent服务.md → 熔断机制（3次失败冷却1h） |
| 点击策略卡片浏览器跳新页面？ | 12-前端架构.md → Modal 系统；16-测试体系.md → E2E 核心 bug 回归 |
| 进程卡死后 kill -9 也清不掉？ | 05-数据层设计.md + 15-调试与运维.md → UEs 说明 |
| 如何在本地跑全量测试？ | 16-测试体系.md → 一键脚本 run_all_tests.sh |

---

## 维护约定

- 本 Wiki **只增不改**的精神：当系统架构/API/模块发生**非兼容**变化时，新增章节而不是删除旧章节，旧章节在开头标记"已废弃，见 xxx.md"。
- 所有代码引用统一使用 `file:///workspace/绝对路径#L起始-L结束` 格式，确保 IDE 中可一键跳转。
- 若发现 Wiki 内容与代码不一致，**以代码为准**，并在对应章节顶部加"⚠️ 内容待更新"标记后更新。
- Wiki 内容分工：README.md 讲"产品是什么 / 怎么用"，本 Code Wiki 讲"代码怎么实现 / 怎么扩展 / 怎么调试"。
