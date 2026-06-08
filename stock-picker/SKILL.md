---
name: stock-picker
description: A股选股策略与市场分析工具。基于多套选股标准（优质/平时/财报）、技术指标（MACD背离、均线系统、K线形态）、时间节点（节假日、政策事件）、板块轮动规律进行综合选股决策。当用户问及"选股"、"买什么股票"、"分析某股"、"市场怎么走"、"热点板块"、"做T"、"技术分析"时使用。覆盖选股、排雷、择时、板块分析、做T操作等场景。
---

# 选股策略

## 工作流程

当用户请求选股或市场分析时，按以下顺序执行：

### 1. 理解用户需求
- 确定用户是想选股、分析某只股票、分析大盘还是了解某板块
- 确定用户关注的周期（短线/中线/长线）
- 确定用户的风格（稳健/激进）
- **必须询问用户的板块/行业偏好**（如电力、新能源、科技、消费、医药等），用 `question` 工具提供选项让用户选择。用户可选具体板块或"没有偏好，你来定"。

### 2. 选股逻辑（按场景匹配）

**用户想要筛选股票** → 先判断选哪套标准：
- 追求**优质长线** → 使用优质选股标准（详见 [selection-criteria.md](references/selection-criteria.md#优质选股)）
- **日常筛选** → 使用平时选股标准（详见 [selection-criteria.md#平时选股](references/selection-criteria.md#平时选股)）
- **财报季** → 使用财报选股标准（详见 [selection-criteria.md#财报季选股](references/selection-criteria.md#财报季选股)）
- **短线博弈** → 使用短线选股标准（详见 [selection-criteria.md#短线博弈策略](references/selection-criteria.md#短线博弈策略)）

### 3. 排雷检查
对候选股票逐一排查（详见 [risk-warnings.md](references/risk-warnings.md)）：
- 是否近期有解禁
- 是否亏损股/下降趋势
- 是否质押股
- 商誉是否过高
- 是否有减持计划

### 4. 技术面确认
对通过排雷的股票做技术面确认（详见 [tech-indicators.md](references/tech-indicators.md)）：
- 日K线20日均线是否抬头向上
- 股价是否在5日均线之上
- MACD是否有背离
- 成交量是否健康（量价齐缩是洗盘，放量滞涨要小心）
- 是否处于收敛三角形末端即将变盘

### 5. 板块与大盘环境
分析当前市场环境（详见 [sector-analysis.md](references/sector-analysis.md)）：
- 当前热点板块是什么
- 是否有龙头旗帜
- 市场情绪（涨跌比、赚钱效应）
- 板块轮动到哪个阶段

### 6. 择时建议
结合时间节点给出操作建议（详见 [timing-events.md](references/timing-events.md)）：
- 当前是否临近特殊日期
- 是否有重大政策事件
- 节假日效应
- 美联储会议周期

## 核心原则

- **谋时不如乘势**：趋势不轻易形成，形成不轻易改变
- **成交量是一切的基础**：没有成交量的配合，技术形态不可信
- **小周期服从大周期**：日线 > 30分钟线，矛盾时以大周期为准
- **收盘价最重要**：突破/跌破压力支撑线超过3%才能确认趋势
- **5日线下不买股**：股价在5日线以下时不要买入

## 资源文件

- **`references/selection-criteria.md`** — 四套选股标准（优质/平时/财报/短线）
- **`references/tech-indicators.md`** — 技术指标用法（MACD、K线、均线、成交量）
- **`references/timing-events.md`** — 时间节点与事件驱动
- **`references/sector-analysis.md`** — 板块分析与热点判断
- **`references/risk-warnings.md`** — 风险排查清单
