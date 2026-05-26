# engine_next

## 1. 项目定位

`engine_next` 是一个面向 A 股短线看盘、竞价分析、开盘确认、盘中跟踪和盘后复盘的运行时引擎。

它不是自动交易系统，当前更准确的定位是：

- 用统一的数据生命周期管理盘前、竞价、开盘、盘中、盘后、夜间复盘。
- 把 Redis、TDengine、离线因子、竞价锚点、热板、昨日涨停池等数据整理成一套可稳定消费的运行时上下文。
- 在不同阶段输出“主叙事、题材判断、强弱验证、个股观察、风险提示、复盘结论”。

当前系统的核心价值不是“直接下单”，而是：

- 减少盘中临时取数和重复计算。
- 明确数据来自哪里、何时可用、何时失效。
- 让策略判断尽量建立在统一口径的数据切片和验证链上。

---

## 2. 当前能力总览

当前代码已经具备以下能力：

- 启动时自检离线链路是否完整，识别 `daily_kline`、`daily_factors`、`chip_peaks`、`daily_dde`、`hot_plates`、`yest_limit_pool`、`stock_plate_mapping`、`auction_anchor` 的缺口。
- 在允许的阶段触发轻量修复或补数，避免盘中做过重的全量同步。
- 从 Redis 读取实时或准实时行情快照，统一兼容老 `stock:quote:*` 和新 `q2:*` 行情结构。
- 读取并恢复竞价锚点，包括 `09:20`、`09:24`、`09:25` 的竞价快照，以及 `market:auction:anchor:{date}` 锚点归档。
- 读取 Kaipan 热板、昨日涨停池、个股涨停原因，并写回 Redis 作为题材和板块运行时缓存。
- 把离线日线、因子、筹码、DDE 与盘中分时、竞价数据拼成 `StockStateSnapshot` 和 `StockSelectionContext`。
- 生成阶段化输出：
  - 盘前预案
  - 竞价分析
  - 开盘确认
  - 盘中跟踪
  - 盘后总结
  - 夜间复盘
- 输出主叙事、题材碰撞、EAX 预期差、梯队映射、高标生死簿、核心观察池、风险提示等看盘文本。

---

## 3. 模块分层

### 3.1 启动与生命周期

- `app_main.py`
  - 主程序入口。
  - 负责按时间阶段驱动启动、自检、渲染、盘中循环、盘后循环。
- `runtime/controllers/startup_bootstrap_controller.py`
  - 启动审计入口。
  - 负责缓存启动审计结果，避免反复全量做同一轮启动检查。
- `runtime/startup_runtime_coordinator.py`
  - 启动阶段的总协调器。
  - 连接“自检报告”和“实际补数动作”。
- `runtime/startup_self_check.py`
  - 生成就绪状态、缺口报告、修复建议、风险说明。
- `runtime/offline_sync_executor.py`
  - 生成离线同步决策和同步范围。
  - 只允许服务器环境执行正式离线同步。

### 3.2 数据获取与标准化

- `runtime/intraday_data_hub.py`
  - 盘中取数主入口。
  - 统一读取 Redis 行情、Kaipan 热板、昨日涨停池、THS 热榜、竞价锚点。
- `runtime/intraday_context_builder.py`
  - 把分散的数据源组装成运行时上下文。
  - 生成股票快照、板块映射、昨日涨停映射、热板映射、行情新鲜度统计。
- `runtime/tick_window_tracker.py`
  - 维护近端滚动分时窗口。
  - 生成 `speed_1m`、`amount_2m` 等瞬时指标。
- `runtime/rust_snapshot_bridge.py`
  - 把 Rust/C++ 侧产出的快照字段统一到 Python 运行时字段。

### 3.3 策略与展示

- `runtime/controllers/auction_runtime_controller.py`
  - 当前最核心的策略输出控制器。
  - 负责盘前、竞价、开盘、盘中、盘后视图渲染。
- `strategy_skill_layer/context_pipeline.py`
  - 生成相对值 profile、题材内排名、候选池过滤。
- `strategy_skill_layer/shape_engine.py`
  - 形态识别、分时结构识别、形态标签和风险标签。
- `strategy_skill_layer/theme_selection_context_factory.py`
  - 生成题材交易上下文，判断题材是否可交易、是否假强、是否需要等待确认。
- `strategy_skill_layer/trap_guards.py`
  - 骗炮、高开转虚、弱承接等防守约束。
- `runtime/theme_fact_aggregator.py`
  - 汇总题材层面的 `2m` 成交、转强、延续、回流、分歧等信号。
- `strategy_skill_layer/slice_comparison.py`
  - 做 TopN 切片比较和开盘两分钟切片比较。

### 3.4 盘后与复盘

- `runtime/controllers/postmarket_runtime_controller.py`
  - 盘后轻刷新。
- `runtime/controllers/night_recap_controller.py`
  - `17:40+` 夜间复盘执行入口。
- `audit/recap_pipeline.py`
  - 收盘后回溯当日策略、板块、涨停、热板、验证结果。

---

## 4. 数据源、格式、字段、存储与生命周期

本系统的核心原则是：

- 盘中优先读 Redis 轻量缓存。
- 正式日线、因子、DDE、筹码长期存 TDengine。
- 重数据尽量在盘后或夜间准备好，盘中只消费“整理后的视图”。

### 4.1 Redis 行情

来源：

- 旧行情键：`stock:quote:{symbol}`
- 新行情键：`q2:{symbol}`，前缀可由 `REDIS_Q2_PREFIX` 配置，默认 `q2:`

标准化位置：

- `runtime/intraday_data_hub.py`
  - `_standardize_legacy_quote`
  - `_standardize_q2_quote`

标准化后常用字段：

| 字段 | 含义 | 单位/口径 |
| --- | --- | --- |
| `symbol` | 6 位股票代码 | 字符串 |
| `name` | 股票名称 | 字符串 |
| `price` | 最新价 | 元 |
| `pre_close` | 昨收 | 元 |
| `amount` | 累计成交额 | 元 |
| `volume` | 累计成交量 | 股/源口径 |
| `time` | 行情时间 | `HH:MM:SS` |
| `timestamp` | 时间戳 | 毫秒或标准化整数 |
| `bid_amount` | 买一承接金额 | 元 |
| `auction_amount_yuan` | 竞价金额 | 元 |
| `bid_amount_yuan` | 竞价买单金额 | 元 |
| `ask_amount_yuan` | 竞价卖单金额 | 元 |
| `phase` | 行情阶段 | Q2 原始阶段值 |
| `limit_state` | 涨跌停状态 | Q2 原始状态值 |
| `speed_1m` | 1 分钟涨速 | 比例值，`0.01=1%` |
| `amount_2m` | 开盘前 2 分钟或滚动 2 分钟成交额 | 元 |
| `amount_5m` | 5 分钟成交额 | 元 |
| `vector_3m` | 3 分钟方向向量 | 比例值 |
| `vector_5m` | 5 分钟方向向量 | 比例值 |
| `large_net_yuan` | 大单净额 | 元 |
| `source` | 来源标记 | `redis_quote` / `redis_q2` |

读取时机：

- 盘前用于构建历史快照上下文。
- 竞价用于竞价视图和锚点兜底。
- 开盘/盘中用于实时或滞后快照判断。

更新时机：

- 主循环 `_v2_tick_pump` 在 `09:15-11:50 / 12:55-15:10` 近 3 秒轮询。

生命周期：

- 这是运行时快照，不是正式归档真值。
- 同一交易日内不断覆盖更新。

### 4.2 竞价锚点

来源：

- Redis 归档：
  - `market:auction:{date}:0920`
  - `market:auction:{date}:0924`
  - `market:auction:{date}:0925`
  - `market:auction:anchor:{yyyymmdd}`

读取入口：

- `runtime/intraday_data_hub.py`
  - `fetch_auction_anchor`

标准化字段：

| 字段 | 含义 |
| --- | --- |
| `symbol` | 股票代码 |
| `name` | 名称 |
| `tag` | `0920` / `0924` / `0925` |
| `price` | 竞价价格 |
| `change_pct` | 竞价涨跌幅，统一成比例值 |
| `amount` | 竞价金额 |
| `bid_amount` | 竞价承接金额 |
| `snapshot_total_stocks` | 快照覆盖股票数 |
| `snapshot_high_open_count` | 高开数量 |
| `snapshot_low_open_count` | 低开数量 |
| `snapshot_flat_open_count` | 平开数量 |
| `snapshot_limit_up_count` | 涨停数量 |
| `snapshot_limit_down_count` | 跌停数量 |
| `snapshot_total_auction_amount_yuan` | 全市场竞价总额 |
| `snapshot_total_limit_up_bid_amount_yuan` | 涨停承接总额 |
| `source` | `redis_0920` / `redis_0924` / `redis_0925` / `redis_anchor` |

读取时机：

- `09:20` 以后可以读取预览。
- `09:25:10` 以后正式读取 `09:25` 锚点。
- `09:25+` 如果缺失，允许恢复。

更新时机：

- 原始 C++ 竞价采集器在竞价阶段落 Redis。
- Python 侧在缺失时允许兜底恢复并回写。

生命周期：

- 当日竞价锚点是竞价阶段真值。
- 可被开盘确认和盘后复盘继续消费。

### 4.3 热板 `hot_plates`

来源：

- Kaipan

运行时缓存：

- `cache:hot_plates:{trade_date}`
- `cache:hot_plates_meta:{trade_date}`

字段：

| 字段 | 含义 |
| --- | --- |
| `plate_name` | 题材/板块名 |
| `rank` | 热板排名 |
| `strength` | 强度值 |
| `hot` | 热度值 |
| `change_pct` | 板块涨幅 |
| `net_inflow_yi` | 板块净流入，亿元 |
| `trade_date` | 所属交易日 |
| `source` | 来源 |

读取时机：

- 盘前读取上一交易日热板，做延续、迁移、主线背景。
- 竞价/盘中/盘后可读取当日热板。

更新时机：

- 启动修复可刷新。
- `_v2_tick_pump` 盘中允许同步 Kaipan 热板。
- `09:25 finalize` 会刷新“今日热板”。
- `17:40` 盘后轻刷新再次固化。

生命周期：

- 昨日热板用于“延续/兑现/迁移”判断。
- 今日热板用于竞价和盘中主线确认。
- `meta` 用于判断新鲜度、缓存日期、签名。

### 4.4 昨日涨停池 `yest_limit_pool`

来源：

- Kaipan

运行时缓存：

- `cache:yest_limit_pool:{trade_date}`
- `cache:yest_limit_pool_meta:{trade_date}`

字段：

| 字段 | 含义 |
| --- | --- |
| `symbol` | 股票代码 |
| `name` | 名称 |
| `lb_days` | 连板天数 |
| `plate` | 所属板块 |
| `seal_time` | 封板时间 |
| `turnover` | 换手 |
| `close_pct` | 收盘涨跌幅 |
| `source` | 来源 |

读取时机：

- 盘前是核心输入，用于：
  - 晋级率
  - 红开率
  - 核按钮率
  - 梯队映射
  - 高标生死簿
  - 竞价验证

更新时机：

- `08:30`、`09:00` 启动检查会尝试保障。
- `09:25 finalize` 和 `09:26 follow-up` 也会刷新。
- 盘后可再次轻刷新。

生命周期：

- 这是“上一交易日正式上下文”。
- 不是实时数据，但对竞价和开盘判断非常关键。

### 4.5 涨停原因与动态板块映射

来源：

- Kaipan 涨停原因、板块原因

运行时缓存：

- `market:stock_plate`
- `market:stock_reason`
- `config:plate_mapping:s2p`

用途：

- 修正个股所属板块。
- 把过泛化题材收敛成更适合短线看盘的主板块。
- 给题材聚合、热板匹配、主叙事输出提供更可信的板块归属。

生命周期：

- `market:stock_plate` 和 `market:stock_reason` 属于运行时富化缓存。
- `config:plate_mapping:s2p` 属于静态或半静态映射。

### 4.6 日线 `daily_kline`

正式来源：

- Baostock

正式存储：

- TDengine

运行时就绪键：

- `cache:kline_ready:{trade_date}`

正式字段：

| 字段 | 含义 |
| --- | --- |
| `trade_date` | 交易日 |
| `symbol` | 股票代码 |
| `open/high/low/close` | OHLC |
| `preclose` | 前收 |
| `pct_chg` | 当日涨跌幅 |
| `amount` | 成交额 |
| `source` | 来源 |

更新时机：

- 正式口径在盘后和夜间同步。
- 启动时如缺失，仅在允许的窗口做补数决策。

生命周期：

- 是后续因子、筹码、复盘的基础正式数据。

### 4.7 因子 `daily_factors`

来源：

- 基于日线和离线管线计算得到

运行时缓存：

- `cache:stock_extra:{trade_date}`

正式字段核心包含：

- `change_pct_5d`
- `avg_turnover_5d`
- `limit_up_days_5`
- `real_market_cap`
- `avg_cost`
- `bias_20`
- `profit_ratio`
- `vol_ratio`
- `rsi_6`
- `concentration`
- `ma5/ma10/ma20`
- `macd_dif/macd_dea/macd_hist`
- `kdj_k/kdj_d/kdj_j`
- `boll_up/boll_mid/boll_low`
- `t2_lb_days`
- `t2_pct`
- `structure_score_base`
- `shape_platform_ready`
- `shape_breakout_ready`
- `shape_repair_ready`
- `shape_overheat_risk`
- `shape_chip_cleanliness`
- `shape_trend_health`
- `shape_t2_repair_bias`
- `theme_core_base`

用途：

- 个股形态。
- 风险约束。
- 题材内角色。
- 日 K 高低位判断。
- 多因子基础评分。

生命周期：

- 盘中不重算，直接读取 `cache:stock_extra:{date}`。
- 如果缓存缺失，只在启动或允许窗口做修复。

### 4.8 筹码 `chip_peaks`

运行时缓存：

- `cache:chip_peaks:{trade_date}`

字段：

- `peak_price`
- `avg_cost`
- `profit_ratio`
- `loss_ratio`
- `concentration`
- `dense_area_count`

用途：

- 判断筹码密集、获利盘、套牢盘、筹码干净度。
- 给修复、平台突破、反包等形态提供辅助判断。

### 4.9 DDE `daily_dde`

运行时缓存：

- `cache:dde_ready:{trade_date}`

字段：

- `ddje`
- `ddx`
- `ddy`
- `ddz`

用途：

- 作为个股资金性质和补充参考，不单独决定交易。

### 4.10 THS 热榜 `hot_rank`

来源：

- THS

运行时缓存：

- `cache:hot_rank:{trade_date}`
- `cache:hot_rank_meta:{trade_date}`

字段：

- `symbol`
- `rank`
- `heat`
- `name`

用途：

- 反映注意力，不是主线真值。
- 只能作为辅助热度代理，不能代替热板和板块联动。

---

## 5. 核心运行时数据模型

### 5.1 `StockStateSnapshot`

位置：

- `domain/models.py`

这是单票运行时基础快照，主要包含：

- 基本标识：
  - `symbol`
  - `name`
  - `plate`
- 板块/梯队：
  - `lb_days`
  - `leader_rank_in_theme`
  - `board_time_rank`
- 竞价与分时：
  - `open_pct`
  - `current_pct`
  - `auction_amount`
  - `volume_intensity`
  - `vol_ratio`
  - `speed_1m`
  - `amount_2m`
  - `amount_5m`
  - `vector_3m`
  - `vector_5m`
- 筹码和技术：
  - `concentration`
  - `profit_ratio`
  - `bias_20`
  - `rsi_6`
- DDE：
  - `ddje`
  - `ddx`
  - `ddy`
  - `ddz`
- 形态基础：
  - `structure_score_base`
  - `shape_platform_ready`
  - `shape_breakout_ready`
  - `shape_repair_ready`
  - `shape_overheat_risk`
  - `shape_chip_cleanliness`
  - `shape_trend_health`
  - `shape_t2_repair_bias`
- 题材和市场属性：
  - `theme_core_base`
  - `market_cap_yi`
  - `amount_day_yi`
  - `plate_persistence_score`
  - `hot_plate_days`
  - `ths_hot_rank`
  - `ths_hot_heat`

### 5.2 `StockSelectionContext`

这是“单票策略消费上下文”，主要用于打分、过滤和输出。

核心字段包括：

- 题材角色：
  - `is_true_leader`
  - `is_front_row`
  - `leader_bucket`
- 热度与活跃度：
  - `hot_rank`
  - `hot_heat`
  - `heat_flow_score`
  - `turnover_quality_score`
  - `activity_score`
- 形态与执行：
  - `kline_pattern`
  - `kline_score`
  - `structure_score`
  - `chip_score`
  - `auction_score`
  - `timing_score`
  - `open_undertake_score`
  - `shape_quality_score`
  - `execution_quality_score`
- 题材可交易性：
  - `theme_tradable`
  - `theme_fakeout_level`
  - `theme_x_score`
  - `open_confirm_state`
- 高低位和相对排名：
  - `daily_height_bucket`
  - `stock_amount_2m_rank_in_theme_pct`
  - `stock_amount_ratio_2m_rank_in_theme_pct`
  - `stock_execution_rank_in_theme_pct`
  - `stock_shape_rank_in_theme_pct`
- 汇总分：
  - `total_score`

### 5.3 题材与会话事实

- `SessionFacts`
  - 全市场情绪、机会、风险、对局、量能、主叙事等。
- `ThemeFact`
  - 单题材资金、强度、转强、延续、分歧、热度、切换状态。
- `ThemeTradeFact`
  - 单题材交易结论，例如延续可做、切换试错、回流修复、兑现回避、龙头独活。
- `LadderFact`
  - 梯队分布、红开率、晋级率、极值特征。

---

## 6. 程序启动顺序与阶段时间线

时间线定义位于：

- `runtime/original_timeline.py`

### 6.1 盘前准备与离线同步

| 时间 | 阶段 | 触发组件 | 主要动作 |
| --- | --- | --- | --- |
| `00:00-09:25` | `PREMARKET` | `DataLifecycle.on_startup` | 同步元数据、同步上一交易日离线数据、触发因子流水线 |
| `01:00-08:30` | `NIGHT` | `DataLifecycle.on_startup` | 生成前一交易日复盘审计 |
| `08:30` | `PREMARKET` | `Orchestrator.run_guardian` | 重置竞价状态、执行启动自检、加载昨日涨停池 |
| `09:00` | `PREMARKET` | `Orchestrator.run_guardian` | 第二次启动检查，逻辑同 `08:30` |

重要规则：

- `09:00` 前允许较重修复。
- `09:00` 后盘前只允许小范围修复，避免与竞价和开盘冲突。
- 正式离线同步仅允许服务器环境。

### 6.2 竞价阶段

| 时间 | 阶段 | 触发组件 | 主要动作 |
| --- | --- | --- | --- |
| `09:15-09:20` | `AUCTION` | `t1.cpp` | 试撮合积累窗口 |
| `09:20:03-09:23:59` | `AUCTION` | `t1.cpp` | 输出 `09:20` 竞价快照 |
| `09:24:10-09:24:59` | `AUCTION` | `t1.cpp` | 输出 `09:24` 竞价快照 |
| `09:25:10-09:29:59` | `AUCTION` | `t1.cpp` | 输出 `09:25` 锚点快照 |
| `09:25+` | `AUCTION` | 启动修复链 | 若锚点缺失，允许恢复竞价分析 |
| `09:26` | `AUCTION` | `Orchestrator.run_guardian` | 刷新昨日涨停池并跑竞价分析 |

关键实现点：

- `auction_runtime_controller.py` 在 `09:25:10` 前会把锚点标记为 `waiting_finalization`，不提前做正式竞价分析。
- `09:25 finalize` 会刷新：
  - 今日热板
  - 昨日涨停池
  - 竞价锚点
  - `market runtime summary`

### 6.3 盘中阶段

| 时间 | 阶段 | 触发组件 | 主要动作 |
| --- | --- | --- | --- |
| `09:15-11:50 / 12:55-15:10` | `INTRADAY` | `_v2_tick_pump` | 轮询 Redis 行情，刷新 Kaipan 热板，向 Rust bridge 推 tick |
| `09:30-15:00` | `INTRADAY` | `Orchestrator.run_guardian` | 大约每 3 分钟做一轮盘中分析 |
| `09:30-09:35` | `INTRADAY` | `RiskSentinel` | 开盘止损路径 |
| `09:35-15:00` | `INTRADAY` | `RiskSentinel` | 盘中追踪、冲高回落、跳水、炸板等监控 |
| `11:25-11:40` | `INTRADAY` | `execute_analysis` | 午间特殊分支 |
| `14:57-15:00` | `INTRADAY` | C++ 工具 | 收盘竞价窗口 |

### 6.4 盘后与夜间

| 时间 | 阶段 | 触发组件 | 主要动作 |
| --- | --- | --- | --- |
| `15:05` | `POSTMARKET` | `Orchestrator.run_guardian` | 标记收盘，降低循环频率 |
| `17:40` | `POSTMARKET` | `Orchestrator + DataLifecycle` | 预加载水位、执行 EOD 生命周期、触发最终复盘脚本 |
| `17:40+` | `POSTMARKET` | `RecapEngine` | 对照热板、昨日涨停、Wencai 涨停真值，生成复盘报告 |
| `night mode` | `NIGHT` | `run_guardian` | 夜间心跳和轮动消息 |

---

## 7. 启动自检、就绪等级与修复原则

相关代码：

- `runtime/startup_self_check.py`
- `runtime/startup_runtime_coordinator.py`
- `runtime/controllers/startup_bootstrap_controller.py`

系统会对以下数据集做 readiness 审计：

- `daily_kline`
- `daily_factors`
- `chip_peaks`
- `daily_dde`
- `yest_limit_pool`
- `hot_plates`
- `stock_plate_mapping`
- `auction_anchor`

输出重点包括：

- 当前阶段 `phase`
- 正式离线日 `formal_offline_date`
- 缺口矩阵
- 是否允许正式同步
- 建议动作
- 是否仍处于大修复窗口

常见就绪等级：

- `trade_ready`
- `trade_ready_degraded`
- `observe_only`

原则：

- 如果正式离线数据不完整，但竞价/盘中必要上下文存在，可以降级观察，不强行报交易机会。
- 如果竞价锚点缺失，优先恢复锚点，不等待重离线。
- 如果热板、昨日涨停池、个股板块映射缺失，可以做轻量补数。

---

## 8. 数据从底层到展示的策略链路

当前系统的策略链不是“直接在输出里临时拼字符串”，而是以下链路：

### 8.1 数据采集层

输入包括：

- Redis 行情
- 竞价快照
- Kaipan 热板
- 昨日涨停池
- 运行时个股板块
- 离线因子
- 筹码
- DDE
- Rust/C++ 分时扩展字段

### 8.2 运行时上下文层

`IntradayContextBuilder` 做的事情：

- 合并 quote、auction、factor、chip、dde、hot plate、yest limit。
- 给每只股票补齐名称、板块、题材列表、涨停原因。
- 统计行情新鲜度、快照覆盖率、缓存日期、新鲜度签名。
- 生成 `StockStateSnapshot`。

### 8.3 会话与题材事实层

`build_session_facts`、`theme_fact_aggregator`、题材碰撞模块会生成：

- 市场情绪分
- 对局环境
- 热门题材
- 延续 / 新发酵 / 兑现
- 板块涨幅、净流入、成交额
- 题材内转强数、涨停数、前排股
- 梯队晋级率、红开率、核按钮率

### 8.4 个股策略上下文层

`context_pipeline.py`、`shape_engine.py`、`theme_selection_context_factory.py` 会计算：

- 个股在题材内的相对排名
- 开盘两分钟成交额是否处于题材 TopN
- 竞价金额与开盘承接的关系
- 个股日 K 高低位
- 题材是否可交易
- 是否存在高开转虚、假强、骗炮风险
- 是否属于修复、突破、平台、反包、龙头独活、后排杂毛

### 8.5 候选池过滤与降权层

系统不是单一“通过/不过”，而是多层过滤：

- 形态范围压缩 `shape_eval_scope`
- 候选池过滤 `filter_trade_candidates`
- 观察池过滤 `filter_watch_candidates`
- 题材可交易性判断
- 骗炮/过热/高位风险守卫
- 开盘确认状态守卫

当前思路已经从“硬过滤”逐步向“软降权 + 备选保留”演进。

### 8.6 展示层

`auction_runtime_controller.py` 最终输出：

- 主叙事
- 情绪总览
- 竞价总览 / 竞价结构
- 数据对撞
- EAX 预期差
- 昨日涨停反馈
- 竞价预案 / 开盘验证 / 盘中观察
- 高标生死簿
- 梯队映射
- 核心观察池 / 执行图
- 风险提示
- 盘后故事线和复盘校验

---

## 9. 各阶段的“策略链路”如何落地

### 9.1 盘前

底层输入：

- 上一交易日热板
- 上一交易日昨日涨停池
- 上一交易日离线因子 / 筹码 / DDE

主要函数链：

- `StartupBootstrapController.execute`
- `RuntimeStartupCoordinator.build_plan`
- `RuntimeStartupCoordinator.execute_allowed_repairs`
- `LiveRuntimeController` 构建上下文
- `AuctionRuntimeController.render_premarket_view`

输出重点：

- 上一交易日收盘定性
- 主线/副线
- 涨停主线/次主线
- 昨日机会与明日预案
- 数据缺口和就绪等级

### 9.2 竞价

底层输入：

- `09:25` 竞价锚点
- 今日热板
- 昨日涨停池
- 板块映射
- 个股竞价金额、竞价涨跌幅

主要函数链：

- `IntradayDataHub.fetch_auction_anchor`
- `IntradayContextBuilder.build_context`
- `build_auction_plate_bucket_stats`
- `build_auction_snapshot_delta_stats`
- `build_context_strategy_bundle_for_symbols`
- `AuctionRuntimeController.render_auction_view`

输出重点：

- 竞价总览
- 题材碰撞
- EAX 预期差
- 高标生死簿
- 竞价龙头
- 竞价执行图

### 9.3 开盘确认

底层输入：

- 开盘后的 `speed_1m`
- `amount_2m`
- `vector_3m`
- 今日热板继续演化
- 昨日涨停反馈

主要函数链：

- `TickWindowTracker`
- `shape_engine`
- `slice_comparison.build_opening_2m_slice_comparison`
- `AuctionRuntimeController` 中 opening confirm 逻辑

输出重点：

- 开盘验证点
- 承接是否成立
- 是否低开转强
- 是否高开转虚
- 是否仅龙头独活
- 是否有 1 到 3 个确认后候选

### 9.4 盘中

底层输入：

- 滚动行情
- 热板切换
- 分时成交额切片
- 昨日涨停、中位股、高位股反馈

主要函数链：

- `_v2_tick_pump`
- `LiveRuntimeController`
- `IntradayContextBuilder`
- `AuctionRuntimeController.render_intraday_view`

输出重点：

- 主线是否迁移
- 哪些题材兑现、哪些题材回流、哪些题材转强
- 龙头独活还是板块联动
- 当前只观察还是允许试错

### 9.5 盘后 / 夜间复盘

底层输入：

- 当日热板
- 前一交易日热板
- 前一交易日昨日涨停池
- Wencai 当日涨停真值
- 当日运行时快照和策略输出

主要函数链：

- `PostmarketRuntimeController`
- `NightRecapController`
- `audit/recap_pipeline.py`

输出重点：

- 主叙事是否正确
- 竞价预判是否被开盘验证
- 开盘候选是否产生实际赚钱效应
- 哪些题材是兑现、哪些是切换、哪些是假强

---

## 10. 关键缓存键与用途

### 10.1 行情与竞价

| 键 | 用途 |
| --- | --- |
| `stock:quote:{symbol}` | 老行情键 |
| `q2:{symbol}` | 新行情键 |
| `market:auction:{date}:0920` | `09:20` 竞价快照 |
| `market:auction:{date}:0924` | `09:24` 竞价快照 |
| `market:auction:{date}:0925` | `09:25` 竞价快照 |
| `market:auction:anchor:{yyyymmdd}` | 竞价锚点归档 |

### 10.2 离线与运行时缓存

| 键 | 用途 |
| --- | --- |
| `cache:kline_ready:{date}` | 日线就绪视图 |
| `cache:stock_extra:{date}` | 因子缓存 |
| `cache:chip_peaks:{date}` | 筹码缓存 |
| `cache:dde_ready:{date}` | DDE 缓存 |
| `cache:hot_plates:{date}` | 热板缓存 |
| `cache:hot_plates_meta:{date}` | 热板元数据 |
| `cache:yest_limit_pool:{date}` | 昨日涨停池 |
| `cache:yest_limit_pool_meta:{date}` | 昨日涨停池元数据 |
| `cache:hot_rank:{date}` | THS 热榜缓存 |
| `cache:hot_rank_meta:{date}` | THS 热榜元数据 |

### 10.3 板块与原因富化

| 键 | 用途 |
| --- | --- |
| `market:stock_plate` | 个股主板块映射 |
| `market:stock_reason` | 个股涨停原因原文 |
| `config:plate_mapping:s2p` | 静态板块映射配置 |

### 10.4 运行时摘要

| 键 | 用途 |
| --- | --- |
| `market:runtime:summary:{date}` | 市场运行时摘要 |
| `market:recap:{date}:report` | 盘后复盘报告 |

---

## 11. 数据更新时机与生命周期总结

### 11.1 哪些是正式数据

- `daily_kline`
- `daily_factors`
- `chip_peaks`
- `daily_dde`
- `hot_plates`
- `yest_limit_pool`

这类数据通常：

- 在盘后或夜间形成正式口径。
- 盘中尽量不重算。
- 在 Redis 只保留运行时消费所需的轻量字段。

### 11.2 哪些是运行时快照

- Redis 行情
- 竞价锚点
- `market:stock_plate`
- `market:stock_reason`
- `market:runtime:summary`

这类数据通常：

- 当日覆盖更新。
- 面向盘中判断，不作为永久正式真值。

### 11.3 哪些是辅助代理

- `hot_rank`
- Wencai `limit_truth`
- `broken_boards`
- `first_failed`

这类数据通常：

- 作为辅助验证。
- 不能替代主线判断的核心真值。

---

## 12. 当前策略体系的边界

当前系统已经能做：

- 统一阶段化输出。
- 基于热板、昨日涨停、竞价、2 分钟成交额、1 分钟涨速、形态、多因子做综合判断。
- 输出“可做/禁做/只观察/龙头观察/确认后看”。

当前系统仍然不应该被误解为：

- 自动得出稳定盈利答案。
- 单靠一个评分就能识别全部主线迁移。
- 不需要复盘校验就可以长期稳定实盘。

它的正确使用方式是：

- 先看主叙事和题材碰撞。
- 再看开盘验证和 2 分钟承接。
- 最后再看个股观察池和风险提示。

不要把辅助热度、单票瞬时拉升、孤立涨停直接当成主线。

---

## 13. 推荐的排查顺序

当你看到系统输出“不准”“看不懂盘面”“不知道买啥”时，优先按下面顺序排查：

1. `readiness` 是否已经降级成 `observe_only`。
2. `auction_anchor`、`hot_plates`、`yest_limit_pool` 是否齐全。
3. 当前读到的是实时行情、滞后快照，还是仅历史快照。
4. 主叙事是否引用了“今日热板”而不是只引用“昨日热板”。
5. 开盘两分钟相关字段是否真实可用：
   - `amount_2m`
   - `speed_1m`
   - `vector_3m`
6. 个股是否被题材不可交易、骗炮保护、高位风险、假强保护降权掉。
7. 输出层是否只是“观察名单”，而不是“确认后候选”。

---

## 14. 代码阅读顺序建议

如果要继续维护这个系统，建议按下面顺序看代码：

1. `app_main.py`
2. `runtime/original_timeline.py`
3. `runtime/controllers/startup_bootstrap_controller.py`
4. `runtime/startup_runtime_coordinator.py`
5. `runtime/intraday_data_hub.py`
6. `runtime/intraday_context_builder.py`
7. `runtime/controllers/auction_runtime_controller.py`
8. `strategy_skill_layer/context_pipeline.py`
9. `strategy_skill_layer/shape_engine.py`
10. `strategy_skill_layer/theme_selection_context_factory.py`
11. `audit/recap_pipeline.py`

---

## 15. 本 README 对应的代码事实来源

本文档主要依据以下文件整理，尽量只描述当前代码已实现的能力：

- `app_main.py`
- `domain/models.py`
- `runtime/original_timeline.py`
- `runtime/controllers/startup_bootstrap_controller.py`
- `runtime/startup_runtime_coordinator.py`
- `runtime/startup_self_check.py`
- `runtime/offline_sync_executor.py`
- `runtime/intraday_data_hub.py`
- `runtime/intraday_context_builder.py`
- `runtime/controllers/auction_runtime_controller.py`
- `runtime/controllers/postmarket_runtime_controller.py`
- `runtime/controllers/night_recap_controller.py`
- `contracts/schema_contracts.py`
- `contracts/source_semantics.py`
- `source_policies/intraday_network_policy.py`

如果后续代码修改了阶段触发、Redis 键、字段含义或策略链路，应该优先同步更新本文件。
