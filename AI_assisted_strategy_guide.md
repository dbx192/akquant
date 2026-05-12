# 用 AI 写 AKQuant 策略：指南与模板

版本：2026-05-12  
作者：为 dbx192 定制的 AI 协助策略写作指南

---

## 一、目的与适用场景
本指南用于：
- 指导如何使用 AKQuant 编写策略；
- 提供给 AI（例如 ChatGPT）用于生成、校验与改进策略代码的规范与 prompt 模板；
- 包含策略模板、测试流程与常见注意点，便于快速落地回测与检验。

适用场景：
- 快速原型（想法 -> 策略代码 -> 回测）；
- 大量参数化策略生成与网格搜索；
- 用 AI 协助生成完整策略文件与单元/回测测试脚本。

---

## 二、AKQuant 编写策略的核心要点（速查）
- 策略类需继承 `akquant.Strategy`，主要回调：
  - `on_start()`：回测启动时（可选）
  - `on_bar(bar)`：收到 K 线（Bar）时
  - `on_order(order)`, `on_trade(trade)`, `on_timer()` 等事件回调
- 获取历史数据：
  - `self.get_history(count=N, symbol=s, field="close")` — 返回包含当前 Bar 的最近 N 条数据
  - 为避免未来函数，若基于“昨日”计算指标，应请求 N+1 条后用 `[:-1]`
- 预热期：
  - `self.warmup_period = <n>` 需至少为用于指标计算的最大窗口
- 下单与仓位控制：
  - `self.buy(symbol, quantity)` / `self.sell(symbol, quantity)`
  - `self.order_target_percent(symbol, target_percent)` / `self.order_target_weights(...)`
  - `self.close_position(symbol)`
  - `self.get_position(symbol)` 获取当前仓位（数量）
- 复杂订单：
  - `place_bracket_order(...)`, `create_oco_order_group(...)`
- 多标的：
  - `run_backtest(data=dict(symbol: DataFrame), symbols=[...])`
  - 在策略中使用 `self.get_history(..., symbol=s)` 指定标的
- 交易限制（A 股）：`lot_size`、回测参数需传入 `run_backtest`
- 流式回测/实时：使用 `on_event` 回调或 `run_backtest(..., on_event=...)`

---

## 三、最佳实践（写给 AI 的约定）
当让 AI 生成策略代码时，建议在 prompt 中明确以下信息（这些信息也应写入策略的 docstring）：
1. 目标：例如“做横截面动量轮动”，“日内 5 分钟突破策略”等。
2. 数据与频率：日线/分钟，是否复权（A 股需 qfq/前复权）。
3. 风险约束：
   - 最大回撤限额（绝对值或 %）
   - 单笔最大仓位 / 仓位比率
   - 多头/空头允许与否（`allow_short`）
4. 交易成本：佣金、印花税、滑点、最低手续费、最小交易手数（lot_size）
5. 执行逻辑细节（进场/出场/止损/止盈）
6. 预热期与指标窗口（例如 short=5,long=20 -> warmup_period=20）
7. 是否需要参数化（用于网格搜索），若需要，请指定参数范围
8. 额外输出：日志、回测报告、orders_df 导出、图表等

---

## 四、给 AI 的 Prompt 模板（可复制）
下面给出多个不同粒度的 prompt 模板，直接填入参数即可。

基本 prompt（生成策略骨架）：
```
你是一个懂 AKQuant 的 Python 开发器。请生成一个继承 akquant.Strategy 的策略代码文件，满足：
- 目标：{简短描述目标}
- 频率：{日线/分钟}
- 数据复权：{qfq/不复权}
- 交易逻辑：{具体的进出场逻辑}
- 风险限制：最大仓位 {max_percent}，单笔最大仓位 {single_percent}，是否允许做空 {allow_short}
- 期望输出：包含 docstring、参数化（PARAM_MODEL 可选）、注释、以及一个 main() 示例，展示如何调用 run_backtest 运行回测并输出 result.report(...)
请确保：
- 使用 self.get_history 时正确设置 warmup_period，并避免未来函数（必要时取 N+1 并用 [:-1]）
- 所有下单都使用框架方法（order_target_percent / buy / sell / close_position）
- 注释说明如何在本地运行（依赖 akshare、如何准备 data）
```

高级 prompt（包含单元测试与优化）：
```
在上面的要求基础上，请同时生成：
- 一个用于单个策略的最小化 pytest 测试文件，使用合成数据验证基本进出场行为（例如买入后仓位 > 0）
- 一个参数化的 PARAM_MODEL（或类似结构）用于网格搜索（指定参数名与范围）
- 一个简单的网格搜索脚本示例（多进程）说明如何调用 run_backtest 多次并收集指标
- 并在代码末尾生成 README 段落，说明策略假设和风险参数
```

提示 AI 排查“未来函数”：
```
请检查代码是否使用了当日 bar 的 close 直接作为指标输入导致未来函数。如有请改为取历史数据并切片 [:-1]。
```

---

## 五、常用策略模板（直接可用的代码片段）

1) 最小策略骨架（给 AI 用的模板）
```python
from typing import Any
import akquant as aq
from akquant import Strategy, Bar

class TemplateStrategy(Strategy):
    """
    目标：{填入目标}
    频率：{日线/分钟}
    参数：
        param1: ...
    """
    def __init__(self, param1=..., *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.param1 = param1
        self.warmup_period = 20  # 根据需要修改

    def on_bar(self, bar: Bar) -> None:
        symbol = bar.symbol
        # 获取历史数据（包含当前 bar）
        closes = self.get_history(count=self.warmup_period, symbol=symbol, field="close")
        if len(closes) < self.warmup_period:
            return
        # 计算指标，防止未来函数处理示例：
        # historical_closes = self.get_history(count=self.warmup_period+1, symbol=symbol, field="close")[:-1]
        # use historical_closes for signals that should not peek current bar
        # 交易逻辑...
```

2) 双均线（示例，可交付给 AI 做变体）
```python
import numpy as np
from akquant import Strategy, Bar

class DualMA(Strategy):
    def __init__(self, short_window=5, long_window=20, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.short_window = short_window
        self.long_window = long_window
        self.warmup_period = long_window

    def on_bar(self, bar: Bar):
        s = bar.symbol
        closes = self.get_history(count=self.long_window, symbol=s, field="close")
        if len(closes) < self.long_window:
            return
        short_ma = np.mean(closes[-self.short_window:])
        long_ma = np.mean(closes[-self.long_window:])
        pos = self.get_position(s)
        if short_ma > long_ma and pos == 0:
            self.order_target_percent(symbol=s, target_percent=0.95)
        elif short_ma < long_ma and pos > 0:
            self.close_position(symbol=s)
```

---

## 六、如何让 AI 帮你“写策略并迭代”——实用工作流
1. 定义实验目标（在 prompt 里写清楚）：策略目标、数据、约束、评估指标（年化、最大回撤、夏普等）。
2. 用上面的 prompt 让 AI 生成策略代码 + docstring + small test。
3. 本地运行回测（或远程 CI），记录 result.metrics 与 orders_df。
4. 把回测结果（含 equity_curve、orders_df、metrics JSON）贴回给 AI：
   - 提示 AI 分析哪些交易导致最大回撤、低胜率或高滑点；
   - 请求改进建议（例如：加入止损、过滤条件、持仓限制）。
5. 让 AI 输出改进后的代码（commit/patch 风格），并再次回测。
6. 当满足目标后，导出报告并做参数优化（网格/贝叶斯），或进行 WFO（Walk-Forward）验证。

---

## 七、AI 生成策略时的常见检查清单（让 AI 自动生成并自测）
- [ ] 是否设置了合适的 warmup_period？
- [ ] get_history 是否被误用（是否含当前 bar 导致未来函数）？
- [ ] 下单是否考虑 lot_size、最小手续费、取整逻辑？
- [ ] 是否包含日志（关键下单/止损/止盈）？
- [ ] 是否包含单元测试（合成数据）或最小回测示例？
- [ ] 是否明确风险参数（最大仓位、止损、最大回撤触发动作）？
- [ ] 是否把策略参数暴露为可调对象（以便做优化）？
- [ ] 如果是跨标策略：有没有确保对齐时间片（timestamp bucket）？

---

## 八、进阶：让 AI 做“策略改进分析”（回测结果 -> 改进建议）
当你把回测的关键输出（例如 metrics、equity_curve csv、orders_df）交给 AI，建议提供：
- 目标衡量标准（例如提升年化 2%，或把最大回撤降到 < 10%）
- 哪些变动允许（增加止损、降低杠杆、改变信号窗）
- 是否允许加复杂度（加入 ML 模型、因子表达式引擎）

示例 prompt：
```
我把回测结果（metrics: {...}, equity_curve.csv, orders_df.csv）给你。目标是把最大回撤从 18% 降到 < 10%，同时保持年化 >= 8%。请：
1) 分析主要亏损原因（列出 top-5 交易或时间段）
2) 给出 3 个改进方向（每个方向给出具体代码改动）
3) 给出对应的回测超参数与预期影响
```

---

## 九、测试与部署建议
- 单元测试：用合成 DataFrame 测试入场/出场在可控条件下触发。
- 回测多频次：先日线回测，再分样本做分钟线或 Tick 仿真（如果策略需要）。
- Walk-Forward：使用 AKQuant 的 WFO 框架（docs/advanced/ml.md）验证泛化能力。
- 流式回测：用 `on_event` 即时监控回测过程并实时中断或采样关键事件。
- 实盘前：用 Paper Trading（broker 模拟）做一段时间的在线验证，注意滑点与延迟。

---

## 十、示例：完整的 AI 请求示例（把所有东西放进一个 prompt）
```
请生成一个 AKQuant 策略，要求如下：
- 策略名：CrossMomentum
- 目标：横截面动量轮动，月度调仓，选出前10只动量最强股票等权持有
- 数据：日线，使用 akshare 获取并复权 (adjust="qfq")
- 风险：单标最大仓位 10%，总仓位最大 80%，禁止做空，最大回撤触发自动清仓（回撤阈值 15%）
- 参数化：lookback=[60,120], top_n=[5,10,20]
- 输出：策略代码（含 docstring）、main 示例（如何用 run_backtest 对字典数据回测）、PARAM_MODEL、一个最小 pytest 文件（模拟 3 天数据验证轮动触发）、以及 README 摘要
请同时说明如何避免未来函数、以及如何在回测结果不佳时迭代（给 3 条具体建议）
```

---

## 十一、常见坑与 FAQ
- 误用 get_history：记得包含/剔除当前 bar 的差别；
- A 股整手问题：使用 `lot_size` 并用 order_target_percent，框架会自动向下取整；
- 滑点/费率遗漏：回测结果对滑点很敏感，务必配置 realistic 的 commission/slippage；
- 数据对齐：多标策略要确保 time index 一致（或用框架的多标对齐方法）；
- Warm-start：恢复运行时需注意热启动（warm start）与指标预热数据；

---

## 十二、附录：回测运行最小示例
```python
import akquant as aq
import akshare as ak
from strategies.cross_momentum import CrossMomentum  # 假设 AI 生成的策略

# 准备数据（示例）
symbols = ["sh600000", "sh600004"]
data = {}
for s in symbols:
    df = ak.stock_zh_a_daily(symbol=s, start_date="20220101", end_date="20231231", adjust="qfq")
    df["symbol"] = s.replace("sh", "").replace("sz", "")
    data[df["symbol"].iloc[0]] = df

# 运行回测
result = aq.run_backtest(
    strategy=CrossMomentum,
    data=data,
    symbols=list(data.keys()),
    initial_cash=1_000_000,
    commission_rate=0.0003,
    min_commission=5.0,
    stamp_tax_rate=0.001,
    lot_size=100,
)

result.report(filename="cross_momentum_report.html", show=False)
```

---

## 十三、总结（给你的行动项）
1. 把你想实现的策略目标、数据样本和风险参数发给 AI，使用本指南中的 Prompt 模版。
2. 运行 AI 生成的代码并做一次回测，把 metrics 与 orders_df 返回给 AI 请求改进建议。
3. 采用网格搜索或 WFO 验证策略稳健性，再做 paper/live 前的模拟验证。

---

如果你愿意，我可以：
- 根据你的具体策略想法（写一段一句话的目标）直接生成一个完整的 AKQuant 策略文件 + 最小回测脚本；
- 或者把该文档转成 README.md / template 文件并提交到你的仓库（我可以帮你创建文件并提交 — 需你确认要写入的 repo 与分支）。
