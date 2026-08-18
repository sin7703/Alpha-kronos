# 13. FirstLimit Alpha 研发链路

## 模块定位

引用 [strategy/first_limit_alpha/README.md](file:///workspace/strategy/first_limit_alpha/README.md#L3-L4)：

> 面向 A 股短线场景的"首板介入、捕捉后续连板"选股模块设计文档。目标不是预测"会不会涨停"，而是识别：`前期震荡后，在第一个涨停当天介入，未来 2-5 个交易日继续强势甚至连板的候选股票。`

**核心交易逻辑**（README 总结）：
- 找出"前期横盘/震荡"
- 在"第一次涨停当天"介入
- 预测后续是否还有明显溢价或继续连板
- **收盘后决策，次日开盘介入（无未来函数）**

属于：事件驱动 + 市场情绪驱动 + 微结构特征驱动 + 强交易执行约束，非传统中低频因子选股。

## 目录结构

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

## 7 阶段研发链路

---

### 阶段 1：数据构建 — data_builder.py

**文件**：[strategy/first_limit_alpha/data_builder.py](file:///workspace/strategy/first_limit_alpha/data_builder.py)

**核心类**：`FirstLimitAlphaDataBuilder`（[strategy/first_limit_alpha/data_builder.py#L30-L33](file:///workspace/strategy/first_limit_alpha/data_builder.py#L30-L33)）

**输入**：`market_kline.db` 中 `kline_daily` 表全历史 K 线（symbol/trade_date/open/high/low/close/volume/amount）

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

### 阶段 2：标签 — labeling.py

**文件**：[strategy/first_limit_alpha/labeling.py](file:///workspace/strategy/first_limit_alpha/labeling.py)

**核心函数**：`compute_sample_labels(frame, idx, label_cfg)`（[strategy/first_limit_alpha/labeling.py#L10-L68](file:///workspace/strategy/first_limit_alpha/labeling.py#L10-L68)）

**规则**：在首板日（第 idx 行）之后取未来 K 线，计算 3 套标签：

#### 标签 A：连板延续（label_continuation）
- **定义**：首板后 2 日内（`continuation_horizon=2`）**再出现任意一次涨停** → y=1
- **计算**：`next_2d_limit_up = int(cont_window["is_limit_up"].astype(bool).any())`

#### 标签 B：短期强势（label_strong_3d）
- **定义**：未来 3 日内（`strong_horizon=3`）**最高涨幅 ≥ 12%** → y=1
- **计算**：`ret_high_3d = (max_high_3d / entry_open) - 1.0`，与 `strong_threshold=0.12` 比较

#### 标签 C：断板风险（label_break_risk）
- **定义**：满足以下任一条件即为风险 → y=1
  1. 次日最低回撤 ≤ `break_drawdown_threshold`（如 -5%）
  2. 次日收盘收益 < -1%（明显走弱）
  3. 次日收盘低于首板收盘价（未接住）

标签计算同时产出辅助字段：未来每日 OHLC (`d1_open`~`d5_close`)、未来涨停计数、分阶段收益率等，供回测阶段直接使用（避免二次重算）。

---

### 阶段 3：特征工程 — features.py

**文件**：[strategy/first_limit_alpha/features.py](file:///workspace/strategy/first_limit_alpha/features.py)

**规模**：70+ 特征，分 6 层：

#### 第 1 层：价格行为特征
- 历史收益：`ret_1d_prev` / `ret_3d` / `ret_5d` / `ret_10d` / `ret_20d`
- 振幅：`amp_3d` / `amp_5d` / `amp_10d` / `amp_20d`
- 波动率：`volatility_5d` / `volatility_10d` / `volatility_20d`
- 均线偏离：`close_vs_ma5/10/20/60`、`ma5_vs_ma10`、`ma10_vs_ma20`
- 距前高距离、高开幅度、K 线实体比例、上下影线比例

#### 第 2 层：成交与换手特征
- 成交额绝对值 + 均值比：`avg_volume_5/10/20/60` / `avg_amount_5/10/20/60`
- 换手率、量比（当日量 / 5日均量）
- 炸板次数、封单金额估算（需分时数据，若无则降级）

#### 第 3 层：板块与情绪特征
- 所属概念数量、概念热度排名
- 同题材涨停家数、连板高度（市场层面情绪）
- 炸板率（市场整体）、涨跌比 `market_adv_dec_ratio`
- 上涨比例 `market_up_ratio`、涨停比例 `market_limit_up_ratio`
- 派生：`_prepare_market_context()` 按交易日聚合市场级指标（[strategy/first_limit_alpha/features.py#L39-L58](file:///workspace/strategy/first_limit_alpha/features.py#L39-L58)）

#### 第 4 层：个股属性特征
- 总市值、流通市值
- 行业分类编码
- 次新标记（上市天数）
- 历史股性：过去 60/120 日涨停次数、连板次数

#### 第 5 层：微结构特征
- 封板时间（早板/午板/尾盘板，需分时数据）
- 首次触板时间
- 开板次数
- 回封速度（从开板到再封板的时间）
- 封板到收盘期间稳定性

#### 第 6 层：派生交互特征
- `换手 × 市值`：高换手小票 vs 高换手大盘股含义不同
- `板块热度 × 封板时间`：热门题材 + 早封板胜率高
- `量比 × 振幅压缩`：放量突破压缩区
- `连板高度 × 炸板率`：高情绪期风险/收益不对称

---

### 阶段 4：Baseline 训练 — modeling.py + train_baseline.py

**文件**：[strategy/first_limit_alpha/modeling.py](file:///workspace/strategy/first_limit_alpha/modeling.py)、[strategy/first_limit_alpha/train_baseline.py](file:///workspace/strategy/first_limit_alpha/train_baseline.py)

#### 模型选择

- **优先 LightGBM**：`from lightgbm import LGBMClassifier`，安装失败自动回退
- **回退 RandomForest**：`from sklearn.ensemble import RandomForestClassifier`（[strategy/first_limit_alpha/modeling.py#L11-L22](file:///workspace/strategy/first_limit_alpha/modeling.py#L11-L22)）

#### 三目标并行训练

`TARGETS` 字典对应 3 套标签（[strategy/first_limit_alpha/modeling.py#L24-L28](file:///workspace/strategy/first_limit_alpha/modeling.py#L24-L28)）：
```python
TARGETS = {
    "continuation": "label_continuation",  # 连板延续
    "strong_3d":    "label_strong_3d",     # 短期强势
    "break_risk":   "label_break_risk",    # 断板风险
}
```

每个目标独立训练一个分类器，最终产出 3 个概率值 `p_continue_limit / p_strong_3d / p_fail`。

#### 切分方式：Walk-Forward Rolling（严禁随机切分）

`_split_dates()` 按**时间顺序**切分：
- **train**：前 70% 日期
- **valid**：中间 15% 日期
- **test**：后 15% 日期
- 禁止 `train_test_split(shuffle=True)` — 会引入未来信息

#### 融合评分：first_limit_score

在推理阶段加权融合（[strategy/first_limit_alpha/inference.py#L23-L30](file:///workspace/strategy/first_limit_alpha/inference.py#L23-L30)）：
```python
frame["first_limit_score"] = 100.0 * (
    0.45 * frame["proba_continuation"] +  # 连板延续：权重 45%
    0.40 * frame["proba_strong_3d"]      +  # 短期强势：权重 40%
    0.15 * (1.0 - frame["proba_break_risk"])  # 非断板风险：权重 15%
).clip(0.0, 100.0)
```

---

### 阶段 5：序列训练 — sequence_dataset.py + sequence_model.py + train_sequence.py

**文件**：
- 数据集：[strategy/first_limit_alpha/sequence_dataset.py](file:///workspace/strategy/first_limit_alpha/sequence_dataset.py)
- 模型：[strategy/first_limit_alpha/sequence_model.py](file:///workspace/strategy/first_limit_alpha/sequence_model.py)
- 训练：[strategy/first_limit_alpha/train_sequence.py](file:///workspace/strategy/first_limit_alpha/train_sequence.py)

#### 序列输入构造

`FirstLimitSequenceDatasetBuilder`（[strategy/first_limit_alpha/sequence_dataset.py#L13-L16](file:///workspace/strategy/first_limit_alpha/sequence_dataset.py#L13-L16)）：
- 窗口长度：20-40 日 OHLCV 序列（`seq_len` 参数）
- 6 个特征通道（[strategy/first_limit_alpha/sequence_dataset.py#L45](file:///workspace/strategy/first_limit_alpha/sequence_dataset.py#L45)）：
  `ret`（收益率）/ `range`（振幅）/ `body`（实体）/ `upper`（上影）/ `lower`（下影）/ `volume_ratio`（量比）

#### GRU 序列模型

`FirstLimitSequenceModel`（[strategy/first_limit_alpha/sequence_model.py#L7-L28](file:///workspace/strategy/first_limit_alpha/sequence_model.py#L7-L28)）：
- **编码器**：单层 GRU，`input_size=6`，`hidden_size=48`
- **Dropout**：0.1 正则化
- **三输出头**（与 Baseline 对齐）：
  - `head_cont` → 连板延续
  - `head_strong` → 短期强势
  - `head_risk` → 断板风险

#### 设计目的
验证**多日时序信息相对于表格特征是否有增量价值**。如果 GRU 显著优于 LightGBM，可考虑在生产中融合两者输出。

---

### 阶段 6：推理 — inference.py

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

### 阶段 7：回测 — backtest.py

**文件**：[strategy/first_limit_alpha/backtest.py](file:///workspace/strategy/first_limit_alpha/backtest.py)

**核心类**：`FirstLimitBacktester`（[strategy/first_limit_alpha/backtest.py#L11](file:///workspace/strategy/first_limit_alpha/backtest.py#L11)）

#### TopK 选股回测参数

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `score_threshold` | 60 | 最低入选分 |
| `top_k` | 5 | 每日最多选几只 |
| `hold_days` | 3 | 最长持有天数 |
| `take_profit` | 0.08 | 止盈 8% |
| `stop_loss` | -0.05 | 止损 -5% |
| `fee_bps` | 3 | 手续费 0.03% |
| `slippage_bps` | 1 | 滑点 0.01% |

#### 出场逻辑（逐笔模拟）

```
for day in 1..hold_days:
    if 当日最高 / 入场价 - 1 >= take_profit:
        止盈出场, exit_reason = "take_profit"
    elif 当日最低 / 入场价 - 1 <= stop_loss:
        止损出场, exit_reason = "stop_loss"
    elif day == hold_days:
        收盘平仓, exit_reason = "hold_3d"
```

#### 输出指标

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

## FastAPI 接口：7 个端点

详见 11-路由层设计.md §9，对应 [app/routers/first_limit_alpha.py](file:///workspace/app/routers/first_limit_alpha.py)：

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

## 与 Alpha 主系统结合

### 可复用组件
- **ConceptEngine 概念热度**：特征层第 3 层（板块与情绪）可直接消费 `FunnelService` 中的概念热度数据
- **市场情绪指标**：`predict_funnel_service` / `funnel_service` 的涨跌比、涨停数等可作为全局特征
- **Kronos 预测**：作为辅助特征（预测未来 3 日涨跌概率）提升首板质量判断

### 接入主漏斗
- `first_limit_score` 排序后的 TopN 候选股，可作为独立候选池来源接入策略选股漏斗（`candidate` 池）
- 与其他策略（缩量启动、自定义规则）并行产生候选，再由综合分统一排序
