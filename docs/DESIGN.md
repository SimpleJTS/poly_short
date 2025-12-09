# Polymarket 短期套利交易警报系统 - 设计文档

## 1. 系统概述

### 1.1 项目背景

Polymarket 是一个去中心化预测市场平台，允许用户对各类事件（包括加密货币价格走势）进行押注。本系统专注于监控**快速结算的二元期权市场**，特别是"BTC是否会在某时间点达到特定价格"类型的短期合约。

### 1.2 核心目标

- **实时监控**：追踪 Polymarket 上的短期 BTC 价格预测市场
- **定价偏差检测**：通过模型计算公允价格，发现市场定价错误
- **套利警报**：当市场价格与模型价格存在显著偏差时发出警报
- **交易建议**：基于期望值(EV)计算，给出最优下注金额建议

### 1.3 系统输出示例

```
┌─────────────────────────────────────────────────────────────┐
│  目标价: $90,077.77    现价: $90,049.54                     │
│  状态: 落后 $-28.23    CB权重: 70%                          │
├─────────────────────────────────────────────────────────────┤
│  YES: 买入 42¢ / 卖出 41¢                                   │
│  NO:  买入 60¢ / 卖出 59¢                                   │
├─────────────────────────────────────────────────────────────┤
│  模型公允: YES 42¢ vs NO 57¢                                │
│  剩余: 9m 18s    Vol: 5                                     │
├─────────────────────────────────────────────────────────────┤
│  建议买YES: $5 (EV:+$0.0)                                   │
├─────────────────────────────────────────────────────────────┤
│  ⚠️  ALARM: YES 被低估 7.0¢                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 系统架构

### 2.1 高层架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Polymarket 套利警报系统                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐ │
│  │  数据采集层  │───▶│  计算引擎层  │───▶│      警报与展示层        │ │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘ │
│         │                  │                       │               │
│         ▼                  ▼                       ▼               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐ │
│  │ Polymarket  │    │  定价模型   │    │   终端 TUI 显示         │ │
│  │   API       │    │  EV 计算    │    │   Telegram/Discord Bot  │ │
│  │ Coinbase    │    │  套利检测   │    │   日志记录              │ │
│  │ Binance     │    │             │    │                         │ │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块划分

```
poly_short/
├── src/
│   ├── __init__.py
│   ├── main.py                 # 程序入口
│   ├── config.py               # 配置管理
│   │
│   ├── data/                   # 数据采集层
│   │   ├── __init__.py
│   │   ├── polymarket.py       # Polymarket API 客户端
│   │   ├── price_feeds.py      # 价格数据源 (Coinbase, Binance)
│   │   └── aggregator.py       # 多源价格聚合器
│   │
│   ├── engine/                 # 计算引擎层
│   │   ├── __init__.py
│   │   ├── pricing_model.py    # 公允价格定价模型
│   │   ├── ev_calculator.py    # 期望值计算器
│   │   ├── arbitrage.py        # 套利检测逻辑
│   │   └── kelly.py            # Kelly准则仓位管理
│   │
│   ├── alerts/                 # 警报层
│   │   ├── __init__.py
│   │   ├── detector.py         # 警报条件检测
│   │   ├── notifier.py         # 通知发送器
│   │   └── channels/           # 通知渠道
│   │       ├── telegram.py
│   │       ├── discord.py
│   │       └── terminal.py
│   │
│   ├── ui/                     # 展示层
│   │   ├── __init__.py
│   │   ├── tui.py              # 终端 UI (Rich/Textual)
│   │   └── formatters.py       # 数据格式化
│   │
│   └── utils/                  # 工具类
│       ├── __init__.py
│       ├── logger.py
│       └── helpers.py
│
├── tests/                      # 测试
├── docs/                       # 文档
├── config/                     # 配置文件
│   ├── settings.yaml
│   └── markets.yaml
└── requirements.txt
```

---

## 3. 核心数据模型

### 3.1 市场数据结构

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Optional

class MarketType(Enum):
    BTC_PRICE_TARGET = "btc_price_target"
    ETH_PRICE_TARGET = "eth_price_target"
    CUSTOM = "custom"

@dataclass
class MarketQuote:
    """市场报价"""
    bid: float          # 买入价 (买方愿意支付的最高价)
    ask: float          # 卖出价 (卖方愿意接受的最低价)
    spread: float       # 买卖价差

    @property
    def mid(self) -> float:
        """中间价"""
        return (self.bid + self.ask) / 2

@dataclass
class BinaryMarket:
    """二元期权市场"""
    market_id: str
    condition: str              # 例如: "BTC >= $90,077.77"
    target_price: float         # 目标价格
    settlement_time: datetime   # 结算时间

    yes_quote: MarketQuote      # YES 合约报价
    no_quote: MarketQuote       # NO 合约报价

    volume: int                 # 交易量
    liquidity: float            # 流动性深度

    @property
    def time_remaining(self) -> float:
        """剩余时间(秒)"""
        return (self.settlement_time - datetime.utcnow()).total_seconds()

@dataclass
class PriceData:
    """现货价格数据"""
    symbol: str                 # 例如: "BTC-USD"
    price: float                # 当前价格
    source: str                 # 数据源: coinbase, binance
    timestamp: datetime
    weight: float = 1.0         # 在聚合计算中的权重

@dataclass
class AggregatedPrice:
    """聚合后的价格"""
    weighted_price: float       # 加权平均价
    prices: list[PriceData]     # 各数据源价格
    confidence: float           # 置信度 (0-1)

@dataclass
class ModelPricing:
    """模型定价结果"""
    fair_yes: float             # YES 公允价格 (0-1)
    fair_no: float              # NO 公允价格 (0-1)
    probability: float          # 事件发生概率
    confidence: float           # 模型置信度

@dataclass
class TradeSignal:
    """交易信号"""
    side: str                   # "YES" 或 "NO"
    action: str                 # "BUY" 或 "SELL"
    market_price: float         # 当前市场价
    fair_price: float           # 模型公允价
    mispricing: float           # 定价偏差
    ev: float                   # 期望值
    suggested_size: float       # 建议下注金额
    kelly_fraction: float       # Kelly比例
    urgency: str                # "LOW", "MEDIUM", "HIGH"

@dataclass
class AlertEvent:
    """警报事件"""
    timestamp: datetime
    market: BinaryMarket
    signal: TradeSignal
    alert_type: str             # "UNDERVALUED", "OVERVALUED", "ARBITRAGE"
    message: str
    severity: int               # 1-5
```

---

## 4. 核心算法设计

### 4.1 公允价格定价模型

基于**Black-Scholes变体**和**实时价格距离**计算YES/NO的公允价格。

```python
import math
from scipy.stats import norm

class PricingModel:
    """
    二元期权定价模型

    核心思想：根据当前价格与目标价格的距离，
    结合剩余时间和波动率，计算目标达成的概率
    """

    def __init__(self, volatility: float = 0.02):
        """
        Args:
            volatility: 预期波动率 (日内短期市场建议 0.01-0.03)
        """
        self.volatility = volatility

    def calculate_fair_price(
        self,
        current_price: float,
        target_price: float,
        time_remaining_seconds: float,
    ) -> ModelPricing:
        """
        计算公允价格

        使用简化的二元期权定价公式:
        P(达标) = N(d2)

        其中:
        d2 = [ln(S/K) - 0.5*σ²*T] / (σ*√T)

        S = 当前价格
        K = 目标价格
        σ = 波动率
        T = 剩余时间 (年化)
        """
        # 时间转换为年化
        T = time_remaining_seconds / (365 * 24 * 3600)

        if T <= 0:
            # 已到期，直接判断
            prob = 1.0 if current_price >= target_price else 0.0
        else:
            # 计算d2
            sigma_sqrt_t = self.volatility * math.sqrt(T)

            if sigma_sqrt_t < 1e-10:
                # 波动率极小，近似确定性结果
                prob = 1.0 if current_price >= target_price else 0.0
            else:
                d2 = (math.log(current_price / target_price) -
                      0.5 * self.volatility**2 * T) / sigma_sqrt_t
                prob = norm.cdf(d2)

        # 概率边界处理
        prob = max(0.01, min(0.99, prob))

        return ModelPricing(
            fair_yes=round(prob, 4),
            fair_no=round(1 - prob, 4),
            probability=prob,
            confidence=self._calculate_confidence(T, current_price, target_price)
        )

    def _calculate_confidence(
        self,
        time_years: float,
        current: float,
        target: float
    ) -> float:
        """
        计算模型置信度

        - 距离结算越近，置信度越高
        - 价格与目标差距越大，置信度越高
        """
        time_factor = max(0, 1 - time_years * 365)  # 1天内逐渐增加
        distance_pct = abs(current - target) / target
        distance_factor = min(1, distance_pct * 20)  # 5%差距=满置信

        return (time_factor * 0.6 + distance_factor * 0.4)
```

### 4.2 多源价格聚合

```python
class PriceAggregator:
    """
    多数据源价格聚合器

    支持配置不同交易所的权重
    """

    def __init__(self, weights: dict[str, float] = None):
        self.weights = weights or {
            "coinbase": 0.70,   # Polymarket 结算通常参考 Coinbase
            "binance": 0.20,
            "kraken": 0.10,
        }

    def aggregate(self, prices: list[PriceData]) -> AggregatedPrice:
        """加权平均聚合"""
        if not prices:
            raise ValueError("No price data available")

        total_weight = 0
        weighted_sum = 0

        for p in prices:
            w = self.weights.get(p.source, 0.1)
            p.weight = w
            weighted_sum += p.price * w
            total_weight += w

        weighted_price = weighted_sum / total_weight

        # 计算置信度 (基于数据源数量和一致性)
        price_variance = sum((p.price - weighted_price)**2 for p in prices) / len(prices)
        consistency = 1 / (1 + price_variance / 100)  # 归一化
        source_coverage = min(1, len(prices) / 3)
        confidence = consistency * 0.7 + source_coverage * 0.3

        return AggregatedPrice(
            weighted_price=weighted_price,
            prices=prices,
            confidence=confidence
        )
```

### 4.3 期望值(EV)与Kelly仓位计算

```python
class EVCalculator:
    """
    期望值计算器

    EV = P(win) * profit - P(lose) * loss
    """

    def calculate_ev(
        self,
        fair_prob: float,      # 模型计算的真实概率
        market_price: float,   # 市场价格 (0-1)
        bet_amount: float = 1.0
    ) -> float:
        """
        计算期望值

        买YES的EV:
        - 胜利: 赚取 (1 - market_price) / market_price * bet_amount
        - 失败: 亏损 bet_amount

        EV = fair_prob * win_amount - (1 - fair_prob) * bet_amount
        """
        if market_price <= 0 or market_price >= 1:
            return 0

        win_multiplier = (1 - market_price) / market_price
        win_amount = win_multiplier * bet_amount

        ev = fair_prob * win_amount - (1 - fair_prob) * bet_amount
        return round(ev, 4)


class KellyCalculator:
    """
    Kelly准则仓位计算

    Kelly公式: f* = (bp - q) / b

    b = 赔率
    p = 胜率
    q = 败率 = 1 - p
    """

    def __init__(self, max_fraction: float = 0.25, confidence_factor: float = 0.5):
        """
        Args:
            max_fraction: 最大仓位比例 (防止过度下注)
            confidence_factor: 信心因子 (通常取半Kelly)
        """
        self.max_fraction = max_fraction
        self.confidence_factor = confidence_factor

    def calculate_position(
        self,
        fair_prob: float,
        market_price: float,
        bankroll: float
    ) -> float:
        """计算建议下注金额"""
        if market_price <= 0 or market_price >= 1:
            return 0

        # 赔率 b = (1 - price) / price
        b = (1 - market_price) / market_price
        p = fair_prob
        q = 1 - p

        # Kelly 比例
        kelly = (b * p - q) / b

        if kelly <= 0:
            return 0  # 负EV，不下注

        # 应用信心因子和最大限制
        adjusted_kelly = min(kelly * self.confidence_factor, self.max_fraction)

        return round(bankroll * adjusted_kelly, 2)
```

### 4.4 套利检测

```python
class ArbitrageDetector:
    """
    套利机会检测器

    检测类型:
    1. YES/NO 内部套利 (买YES + 买NO < 1)
    2. 模型定价偏差套利 (市场价 vs 公允价)
    3. 跨市场套利 (暂不实现)
    """

    def __init__(
        self,
        min_edge: float = 0.03,      # 最小优势阈值 (3%)
        min_ev: float = 0.01,         # 最小EV阈值
        urgency_threshold: float = 300  # 紧急度阈值 (秒)
    ):
        self.min_edge = min_edge
        self.min_ev = min_ev
        self.urgency_threshold = urgency_threshold

    def detect(
        self,
        market: BinaryMarket,
        model_pricing: ModelPricing,
        bankroll: float = 100
    ) -> Optional[TradeSignal]:
        """检测套利机会"""

        # 1. 检查 YES/NO 内部套利
        internal_arb = self._check_internal_arbitrage(market)
        if internal_arb:
            return internal_arb

        # 2. 检查模型定价偏差
        return self._check_mispricing(market, model_pricing, bankroll)

    def _check_internal_arbitrage(self, market: BinaryMarket) -> Optional[TradeSignal]:
        """
        内部套利: 如果 YES.ask + NO.ask < 1，可以同时买入两边锁定利润
        """
        total_cost = market.yes_quote.ask + market.no_quote.ask

        if total_cost < 0.98:  # 考虑交易成本，需要2%以上空间
            profit_pct = 1 - total_cost
            return TradeSignal(
                side="BOTH",
                action="BUY",
                market_price=total_cost,
                fair_price=1.0,
                mispricing=profit_pct,
                ev=profit_pct,
                suggested_size=50,  # 固定金额套利
                kelly_fraction=1.0,
                urgency="HIGH"
            )

        return None

    def _check_mispricing(
        self,
        market: BinaryMarket,
        model: ModelPricing,
        bankroll: float
    ) -> Optional[TradeSignal]:
        """检查定价偏差"""

        ev_calc = EVCalculator()
        kelly_calc = KellyCalculator()

        # 检查 YES 是否被低估
        yes_market = market.yes_quote.ask  # 买入价
        yes_edge = model.fair_yes - yes_market

        if yes_edge >= self.min_edge:
            ev = ev_calc.calculate_ev(model.fair_yes, yes_market)
            if ev >= self.min_ev:
                size = kelly_calc.calculate_position(
                    model.fair_yes, yes_market, bankroll
                )
                return TradeSignal(
                    side="YES",
                    action="BUY",
                    market_price=yes_market,
                    fair_price=model.fair_yes,
                    mispricing=yes_edge,
                    ev=ev,
                    suggested_size=size,
                    kelly_fraction=size / bankroll if bankroll > 0 else 0,
                    urgency=self._calc_urgency(market.time_remaining)
                )

        # 检查 NO 是否被低估
        no_market = market.no_quote.ask
        no_edge = model.fair_no - no_market

        if no_edge >= self.min_edge:
            ev = ev_calc.calculate_ev(model.fair_no, no_market)
            if ev >= self.min_ev:
                size = kelly_calc.calculate_position(
                    model.fair_no, no_market, bankroll
                )
                return TradeSignal(
                    side="NO",
                    action="BUY",
                    market_price=no_market,
                    fair_price=model.fair_no,
                    mispricing=no_edge,
                    ev=ev,
                    suggested_size=size,
                    kelly_fraction=size / bankroll if bankroll > 0 else 0,
                    urgency=self._calc_urgency(market.time_remaining)
                )

        return None

    def _calc_urgency(self, time_remaining: float) -> str:
        if time_remaining < 60:
            return "HIGH"
        elif time_remaining < self.urgency_threshold:
            return "MEDIUM"
        return "LOW"
```

---

## 5. 数据采集层设计

### 5.1 Polymarket API 客户端

```python
import aiohttp
import asyncio
from typing import Optional

class PolymarketClient:
    """
    Polymarket API 客户端

    官方API: https://docs.polymarket.com/
    CLOB API: https://clob.polymarket.com/
    """

    BASE_URL = "https://clob.polymarket.com"
    GAMMA_URL = "https://gamma-api.polymarket.com"

    def __init__(self, api_key: Optional[str] = None):
        self.api_key = api_key
        self._session: Optional[aiohttp.ClientSession] = None

    async def _get_session(self) -> aiohttp.ClientSession:
        if self._session is None or self._session.closed:
            headers = {}
            if self.api_key:
                headers["Authorization"] = f"Bearer {self.api_key}"
            self._session = aiohttp.ClientSession(headers=headers)
        return self._session

    async def get_market(self, condition_id: str) -> dict:
        """获取市场详情"""
        session = await self._get_session()
        url = f"{self.GAMMA_URL}/markets/{condition_id}"
        async with session.get(url) as resp:
            return await resp.json()

    async def get_orderbook(self, token_id: str) -> dict:
        """获取订单簿"""
        session = await self._get_session()
        url = f"{self.BASE_URL}/book"
        params = {"token_id": token_id}
        async with session.get(url, params=params) as resp:
            return await resp.json()

    async def get_price(self, token_id: str) -> dict:
        """获取最新价格"""
        session = await self._get_session()
        url = f"{self.BASE_URL}/price"
        params = {"token_id": token_id, "side": "buy"}
        async with session.get(url, params=params) as resp:
            return await resp.json()

    async def search_markets(
        self,
        query: str = "BTC",
        closed: bool = False,
        limit: int = 20
    ) -> list[dict]:
        """搜索市场"""
        session = await self._get_session()
        url = f"{self.GAMMA_URL}/markets"
        params = {
            "closed": str(closed).lower(),
            "limit": limit,
            # 可添加更多过滤条件
        }
        async with session.get(url, params=params) as resp:
            data = await resp.json()
            # 过滤包含关键词的市场
            return [m for m in data if query.lower() in m.get("question", "").lower()]

    async def close(self):
        if self._session:
            await self._session.close()
```

### 5.2 价格数据源

```python
class CoinbasePriceFeed:
    """Coinbase 价格数据源"""

    BASE_URL = "https://api.coinbase.com/v2"
    WS_URL = "wss://ws-feed.exchange.coinbase.com"

    async def get_spot_price(self, symbol: str = "BTC-USD") -> PriceData:
        """获取现货价格"""
        async with aiohttp.ClientSession() as session:
            url = f"{self.BASE_URL}/prices/{symbol}/spot"
            async with session.get(url) as resp:
                data = await resp.json()
                return PriceData(
                    symbol=symbol,
                    price=float(data["data"]["amount"]),
                    source="coinbase",
                    timestamp=datetime.utcnow()
                )

    async def subscribe_ticker(self, symbol: str, callback):
        """订阅实时价格 (WebSocket)"""
        import websockets

        async with websockets.connect(self.WS_URL) as ws:
            subscribe_msg = {
                "type": "subscribe",
                "product_ids": [symbol],
                "channels": ["ticker"]
            }
            await ws.send(json.dumps(subscribe_msg))

            async for message in ws:
                data = json.loads(message)
                if data.get("type") == "ticker":
                    price_data = PriceData(
                        symbol=symbol,
                        price=float(data["price"]),
                        source="coinbase",
                        timestamp=datetime.utcnow()
                    )
                    await callback(price_data)


class BinancePriceFeed:
    """Binance 价格数据源"""

    BASE_URL = "https://api.binance.com/api/v3"

    async def get_spot_price(self, symbol: str = "BTCUSDT") -> PriceData:
        async with aiohttp.ClientSession() as session:
            url = f"{self.BASE_URL}/ticker/price"
            params = {"symbol": symbol}
            async with session.get(url, params=params) as resp:
                data = await resp.json()
                return PriceData(
                    symbol=symbol,
                    price=float(data["price"]),
                    source="binance",
                    timestamp=datetime.utcnow()
                )
```

---

## 6. 警报与通知设计

### 6.1 警报条件

```python
@dataclass
class AlertCondition:
    """警报条件配置"""
    min_edge: float = 0.05          # 最小定价偏差 5%
    min_ev: float = 0.02            # 最小EV 2%
    max_time_remaining: int = 600   # 仅关注10分钟内到期的
    min_time_remaining: int = 30    # 至少30秒才能执行
    min_volume: int = 3             # 最小交易量
    min_liquidity: float = 100      # 最小流动性 $100

class AlertDetector:
    """警报检测器"""

    def __init__(self, conditions: AlertCondition = None):
        self.conditions = conditions or AlertCondition()

    def should_alert(
        self,
        market: BinaryMarket,
        signal: TradeSignal
    ) -> bool:
        """判断是否应该发送警报"""
        c = self.conditions

        # 时间窗口检查
        if not (c.min_time_remaining <= market.time_remaining <= c.max_time_remaining):
            return False

        # 信号强度检查
        if signal.mispricing < c.min_edge:
            return False

        if signal.ev < c.min_ev:
            return False

        # 流动性检查
        if market.volume < c.min_volume:
            return False

        if market.liquidity < c.min_liquidity:
            return False

        return True
```

### 6.2 通知渠道

```python
from abc import ABC, abstractmethod

class NotificationChannel(ABC):
    """通知渠道抽象基类"""

    @abstractmethod
    async def send(self, alert: AlertEvent) -> bool:
        pass


class TelegramNotifier(NotificationChannel):
    """Telegram 通知"""

    def __init__(self, bot_token: str, chat_id: str):
        self.bot_token = bot_token
        self.chat_id = chat_id

    async def send(self, alert: AlertEvent) -> bool:
        url = f"https://api.telegram.org/bot{self.bot_token}/sendMessage"

        message = self._format_message(alert)

        async with aiohttp.ClientSession() as session:
            payload = {
                "chat_id": self.chat_id,
                "text": message,
                "parse_mode": "Markdown"
            }
            async with session.post(url, json=payload) as resp:
                return resp.status == 200

    def _format_message(self, alert: AlertEvent) -> str:
        m = alert.market
        s = alert.signal

        return f"""
🚨 *{alert.alert_type} ALERT*

*Market:* {m.condition}
*Remaining:* {m.time_remaining // 60}m {m.time_remaining % 60}s

*Signal:* BUY {s.side}
*Market Price:* {s.market_price:.2%}
*Fair Price:* {s.fair_price:.2%}
*Edge:* {s.mispricing:.1%}
*EV:* +${s.ev:.2f}

*Suggested:* ${s.suggested_size:.0f}
*Urgency:* {s.urgency}
        """.strip()


class TerminalNotifier(NotificationChannel):
    """终端通知 (声音+闪烁)"""

    async def send(self, alert: AlertEvent) -> bool:
        # 终端铃声
        print("\a", end="")

        # 彩色输出
        from rich.console import Console
        from rich.panel import Panel

        console = Console()

        panel = Panel(
            self._format_alert(alert),
            title=f"⚠️ {alert.alert_type}",
            border_style="red" if alert.severity >= 4 else "yellow"
        )
        console.print(panel)
        return True
```

---

## 7. 终端UI设计

### 7.1 使用 Rich 库的实时显示

```python
from rich.live import Live
from rich.table import Table
from rich.layout import Layout
from rich.panel import Panel
from rich.console import Console

class TradingDashboard:
    """实时交易仪表盘"""

    def __init__(self):
        self.console = Console()
        self.current_market: Optional[BinaryMarket] = None
        self.current_price: Optional[AggregatedPrice] = None
        self.current_signal: Optional[TradeSignal] = None

    def create_layout(self) -> Layout:
        """创建布局"""
        layout = Layout()

        layout.split_column(
            Layout(name="header", size=3),
            Layout(name="main"),
            Layout(name="footer", size=3)
        )

        layout["main"].split_row(
            Layout(name="market", ratio=2),
            Layout(name="signal", ratio=1)
        )

        return layout

    def render_price_panel(self) -> Panel:
        """渲染价格面板"""
        if not self.current_market or not self.current_price:
            return Panel("Waiting for data...", title="Price Status")

        m = self.current_market
        p = self.current_price

        diff = p.weighted_price - m.target_price
        status = "领先" if diff >= 0 else "落后"
        status_color = "green" if diff >= 0 else "red"

        content = f"""
目标价: ${m.target_price:,.2f}    现价: ${p.weighted_price:,.2f}
状态: [{status_color}]{status} ${diff:+,.2f}[/]    CB权重: {p.prices[0].weight:.0%}
        """.strip()

        return Panel(content, title="价格状态")

    def render_quotes_panel(self) -> Panel:
        """渲染报价面板"""
        if not self.current_market:
            return Panel("No market data", title="Market Quotes")

        m = self.current_market

        table = Table(show_header=True)
        table.add_column("Side")
        table.add_column("Bid", justify="right")
        table.add_column("Ask", justify="right")
        table.add_column("Spread", justify="right")

        table.add_row(
            "YES",
            f"{m.yes_quote.bid:.0%}",
            f"{m.yes_quote.ask:.0%}",
            f"{m.yes_quote.spread:.1%}"
        )
        table.add_row(
            "NO",
            f"{m.no_quote.bid:.0%}",
            f"{m.no_quote.ask:.0%}",
            f"{m.no_quote.spread:.1%}"
        )

        return Panel(table, title="市场报价")

    def render_signal_panel(self) -> Panel:
        """渲染信号面板"""
        if not self.current_signal:
            return Panel("No signal", title="Trading Signal")

        s = self.current_signal

        if s.mispricing >= 0.05:
            border_style = "bold red"
            title = "⚠️ ALARM"
        elif s.mispricing >= 0.03:
            border_style = "yellow"
            title = "Signal"
        else:
            border_style = "dim"
            title = "Signal"

        content = f"""
建议买{s.side}: ${s.suggested_size:.0f} (EV:+${s.ev:.2f})

{s.side} 被低估 {s.mispricing:.1%}
市场价 {s.market_price:.0%} vs 公允价 {s.fair_price:.0%}
        """.strip()

        return Panel(content, title=title, border_style=border_style)

    async def run(self, update_interval: float = 0.5):
        """运行仪表盘"""
        layout = self.create_layout()

        with Live(layout, console=self.console, refresh_per_second=4) as live:
            while True:
                # 更新各面板
                layout["header"].update(
                    Panel(f"剩余: {self._format_time()}", title="Polymarket Arbitrage Monitor")
                )
                layout["market"].update(self.render_quotes_panel())
                layout["signal"].update(self.render_signal_panel())

                await asyncio.sleep(update_interval)

    def _format_time(self) -> str:
        if not self.current_market:
            return "--:--"
        remaining = self.current_market.time_remaining
        mins = int(remaining // 60)
        secs = int(remaining % 60)
        return f"{mins}m {secs}s"
```

---

## 8. 配置管理

### 8.1 配置文件结构

```yaml
# config/settings.yaml

# 数据源配置
data_sources:
  price_feeds:
    coinbase:
      enabled: true
      weight: 0.70
      api_key: ${COINBASE_API_KEY}  # 环境变量引用
    binance:
      enabled: true
      weight: 0.20
    kraken:
      enabled: false
      weight: 0.10

  polymarket:
    api_key: ${POLYMARKET_API_KEY}
    refresh_interval: 500  # 毫秒

# 模型参数
model:
  volatility: 0.02  # 日内波动率估计
  confidence_factor: 0.5  # Kelly 信心因子

# 警报配置
alerts:
  conditions:
    min_edge: 0.05        # 最小优势 5%
    min_ev: 0.02          # 最小EV
    min_time: 30          # 最小剩余时间(秒)
    max_time: 600         # 最大剩余时间(秒)
    min_volume: 3
    min_liquidity: 100

  channels:
    terminal:
      enabled: true
      sound: true
    telegram:
      enabled: true
      bot_token: ${TELEGRAM_BOT_TOKEN}
      chat_id: ${TELEGRAM_CHAT_ID}
    discord:
      enabled: false
      webhook_url: ${DISCORD_WEBHOOK}

# 风险管理
risk:
  max_position: 50        # 单笔最大下注
  max_exposure: 200       # 总敞口限制
  bankroll: 1000          # 资金池

# 监控的市场
markets:
  # 可以配置多个市场类型
  - type: btc_price_target
    enabled: true
  - type: eth_price_target
    enabled: false
```

### 8.2 配置加载器

```python
import os
import yaml
from pathlib import Path

class Config:
    """配置管理器"""

    def __init__(self, config_path: str = "config/settings.yaml"):
        self.config_path = Path(config_path)
        self._config = self._load_config()

    def _load_config(self) -> dict:
        """加载并解析配置"""
        with open(self.config_path) as f:
            content = f.read()

        # 替换环境变量
        content = self._substitute_env_vars(content)

        return yaml.safe_load(content)

    def _substitute_env_vars(self, content: str) -> str:
        """替换 ${VAR_NAME} 格式的环境变量"""
        import re

        pattern = r'\$\{([^}]+)\}'

        def replacer(match):
            var_name = match.group(1)
            return os.environ.get(var_name, "")

        return re.sub(pattern, replacer, content)

    def get(self, key: str, default=None):
        """获取配置值，支持点号分隔的路径"""
        keys = key.split(".")
        value = self._config

        for k in keys:
            if isinstance(value, dict):
                value = value.get(k)
            else:
                return default

            if value is None:
                return default

        return value
```

---

## 9. 主程序流程

```python
# src/main.py

import asyncio
import signal
from typing import Optional

class TradingMonitor:
    """主监控程序"""

    def __init__(self, config: Config):
        self.config = config
        self.running = False

        # 初始化组件
        self.polymarket = PolymarketClient(
            api_key=config.get("data_sources.polymarket.api_key")
        )
        self.price_aggregator = PriceAggregator(
            weights={
                "coinbase": config.get("data_sources.price_feeds.coinbase.weight", 0.7),
                "binance": config.get("data_sources.price_feeds.binance.weight", 0.2),
            }
        )
        self.pricing_model = PricingModel(
            volatility=config.get("model.volatility", 0.02)
        )
        self.arbitrage_detector = ArbitrageDetector(
            min_edge=config.get("alerts.conditions.min_edge", 0.05),
            min_ev=config.get("alerts.conditions.min_ev", 0.02)
        )
        self.alert_detector = AlertDetector(AlertCondition(
            min_edge=config.get("alerts.conditions.min_edge", 0.05),
            min_ev=config.get("alerts.conditions.min_ev", 0.02),
            min_time_remaining=config.get("alerts.conditions.min_time", 30),
            max_time_remaining=config.get("alerts.conditions.max_time", 600),
        ))

        # 通知渠道
        self.notifiers: list[NotificationChannel] = []
        self._setup_notifiers()

        # UI
        self.dashboard = TradingDashboard()

    def _setup_notifiers(self):
        """设置通知渠道"""
        if self.config.get("alerts.channels.terminal.enabled"):
            self.notifiers.append(TerminalNotifier())

        if self.config.get("alerts.channels.telegram.enabled"):
            self.notifiers.append(TelegramNotifier(
                bot_token=self.config.get("alerts.channels.telegram.bot_token"),
                chat_id=self.config.get("alerts.channels.telegram.chat_id")
            ))

    async def run(self):
        """主运行循环"""
        self.running = True

        # 启动UI
        ui_task = asyncio.create_task(self.dashboard.run())

        # 监控循环
        monitor_task = asyncio.create_task(self._monitor_loop())

        try:
            await asyncio.gather(ui_task, monitor_task)
        except asyncio.CancelledError:
            pass
        finally:
            await self.shutdown()

    async def _monitor_loop(self):
        """监控主循环"""
        interval = self.config.get("data_sources.polymarket.refresh_interval", 500) / 1000

        while self.running:
            try:
                await self._update_cycle()
            except Exception as e:
                print(f"Error in update cycle: {e}")

            await asyncio.sleep(interval)

    async def _update_cycle(self):
        """单次更新周期"""
        # 1. 获取目标市场
        markets = await self.polymarket.search_markets("BTC")

        for market_data in markets:
            # 2. 构建市场对象
            market = self._parse_market(market_data)

            if market.time_remaining <= 0 or market.time_remaining > 600:
                continue  # 跳过不在窗口内的市场

            # 3. 获取现货价格
            prices = await self._fetch_prices()
            aggregated = self.price_aggregator.aggregate(prices)

            # 4. 计算公允价格
            model_pricing = self.pricing_model.calculate_fair_price(
                current_price=aggregated.weighted_price,
                target_price=market.target_price,
                time_remaining_seconds=market.time_remaining
            )

            # 5. 检测套利
            signal = self.arbitrage_detector.detect(
                market=market,
                model_pricing=model_pricing,
                bankroll=self.config.get("risk.bankroll", 1000)
            )

            # 6. 更新UI
            self.dashboard.current_market = market
            self.dashboard.current_price = aggregated
            self.dashboard.current_signal = signal

            # 7. 发送警报
            if signal and self.alert_detector.should_alert(market, signal):
                alert = AlertEvent(
                    timestamp=datetime.utcnow(),
                    market=market,
                    signal=signal,
                    alert_type="UNDERVALUED" if signal.side == "YES" else "OVERVALUED",
                    message=f"{signal.side} undervalued by {signal.mispricing:.1%}",
                    severity=4 if signal.mispricing >= 0.07 else 3
                )

                for notifier in self.notifiers:
                    await notifier.send(alert)

    async def shutdown(self):
        """清理关闭"""
        self.running = False
        await self.polymarket.close()


async def main():
    config = Config()
    monitor = TradingMonitor(config)

    # 信号处理
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, lambda: asyncio.create_task(monitor.shutdown()))

    await monitor.run()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 10. 部署与运维

### 10.1 依赖管理

```txt
# requirements.txt

# 核心
aiohttp>=3.8.0
websockets>=10.0
pydantic>=2.0

# 数学计算
scipy>=1.10.0
numpy>=1.24.0

# 终端UI
rich>=13.0.0
textual>=0.40.0

# 配置
pyyaml>=6.0
python-dotenv>=1.0.0

# 日志
structlog>=23.0.0

# 测试
pytest>=7.0.0
pytest-asyncio>=0.21.0
```

### 10.2 Docker 部署

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/
COPY config/ ./config/

# 环境变量
ENV PYTHONUNBUFFERED=1

CMD ["python", "-m", "src.main"]
```

### 10.3 监控与日志

```python
# src/utils/logger.py

import structlog
from datetime import datetime

def setup_logging():
    """配置结构化日志"""
    structlog.configure(
        processors=[
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.add_log_level,
            structlog.processors.JSONRenderer()
        ],
        wrapper_class=structlog.BoundLogger,
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
    )

logger = structlog.get_logger()

# 使用示例
logger.info("signal_detected",
    side="YES",
    edge=0.07,
    ev=0.15,
    market_id="abc123"
)
```

---

## 11. 自动交易模块

### 11.1 交易执行架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          自动交易执行层                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌──────────┐ │
│  │  信号接收   │───▶│  风控检查   │───▶│  订单生成   │───▶│ 执行引擎  │ │
│  └─────────────┘    └─────────────┘    └─────────────┘    └──────────┘ │
│         │                  │                  │                 │       │
│         ▼                  ▼                  ▼                 ▼       │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌──────────┐ │
│  │ TradeSignal │    │ 仓位限制    │    │ 市价/限价   │    │ CLOB API │ │
│  │ from Arb    │    │ 敞口控制    │    │ 滑点保护    │    │ 订单提交 │ │
│  │ Detector    │    │ 频率限制    │    │ 紧急度优先  │    │ 状态追踪 │ │
│  └─────────────┘    └─────────────┘    └─────────────┘    └──────────┘ │
│                                                                         │
│                              ┌─────────────┐                            │
│                              │  订单管理   │                            │
│                              │  ─────────  │                            │
│                              │ • 挂单追踪  │                            │
│                              │ • 成交确认  │                            │
│                              │ • 自动撤单  │                            │
│                              │ • 仓位同步  │                            │
│                              └─────────────┘                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.2 订单数据模型

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional
import uuid

class OrderSide(Enum):
    BUY = "BUY"
    SELL = "SELL"

class OrderType(Enum):
    MARKET = "MARKET"       # 市价单 - 立即成交
    LIMIT = "LIMIT"         # 限价单 - 指定价格
    FOK = "FOK"             # Fill or Kill - 全部成交或取消
    IOC = "IOC"             # Immediate or Cancel - 立即成交剩余取消

class OrderStatus(Enum):
    PENDING = "PENDING"         # 待提交
    SUBMITTED = "SUBMITTED"     # 已提交
    OPEN = "OPEN"               # 挂单中
    PARTIALLY_FILLED = "PARTIALLY_FILLED"  # 部分成交
    FILLED = "FILLED"           # 完全成交
    CANCELLED = "CANCELLED"     # 已取消
    REJECTED = "REJECTED"       # 被拒绝
    EXPIRED = "EXPIRED"         # 已过期

@dataclass
class Order:
    """订单"""
    order_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    market_id: str = ""
    token_id: str = ""              # YES 或 NO 的 token ID
    side: OrderSide = OrderSide.BUY
    order_type: OrderType = OrderType.LIMIT

    # 价格与数量
    price: float = 0.0              # 限价单价格 (0-1)
    size: float = 0.0               # 下注金额 (USDC)
    filled_size: float = 0.0        # 已成交金额

    # 状态
    status: OrderStatus = OrderStatus.PENDING
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)

    # 交易所返回
    exchange_order_id: Optional[str] = None
    fill_price: Optional[float] = None      # 平均成交价
    error_message: Optional[str] = None

    # 来源信号
    signal: Optional[TradeSignal] = None

    @property
    def is_active(self) -> bool:
        return self.status in (OrderStatus.SUBMITTED, OrderStatus.OPEN, OrderStatus.PARTIALLY_FILLED)

    @property
    def is_complete(self) -> bool:
        return self.status in (OrderStatus.FILLED, OrderStatus.CANCELLED, OrderStatus.REJECTED, OrderStatus.EXPIRED)

    @property
    def remaining_size(self) -> float:
        return self.size - self.filled_size


@dataclass
class Position:
    """持仓"""
    market_id: str
    token_id: str
    side: str                       # "YES" or "NO"

    size: float                     # 持有数量 (shares)
    avg_cost: float                 # 平均成本
    current_price: float            # 当前市价

    unrealized_pnl: float = 0.0     # 未实现盈亏
    realized_pnl: float = 0.0       # 已实现盈亏

    @property
    def market_value(self) -> float:
        return self.size * self.current_price

    @property
    def total_cost(self) -> float:
        return self.size * self.avg_cost


@dataclass
class TradeRecord:
    """成交记录"""
    trade_id: str
    order_id: str
    market_id: str
    side: str

    price: float
    size: float
    fee: float

    timestamp: datetime
    pnl: Optional[float] = None     # 平仓时的盈亏
```

### 11.3 风控模块

```python
from dataclasses import dataclass
from typing import Optional
import time

@dataclass
class RiskLimits:
    """风控限制配置"""
    # 仓位限制
    max_position_size: float = 50           # 单笔最大下注 (USDC)
    max_total_exposure: float = 200         # 总敞口上限
    max_position_per_market: float = 100    # 单市场最大敞口

    # 亏损限制
    max_daily_loss: float = 50              # 日最大亏损
    max_drawdown_pct: float = 0.10          # 最大回撤比例 (10%)

    # 频率限制
    min_order_interval: float = 1.0         # 最小下单间隔 (秒)
    max_orders_per_minute: int = 10         # 每分钟最大订单数

    # 信号质量限制
    min_edge: float = 0.03                  # 最小优势
    min_ev: float = 0.01                    # 最小期望值
    min_time_remaining: int = 30            # 最小剩余时间 (秒)
    max_slippage: float = 0.02              # 最大滑点容忍


class RiskManager:
    """风控管理器"""

    def __init__(self, limits: RiskLimits, initial_balance: float):
        self.limits = limits
        self.initial_balance = initial_balance
        self.current_balance = initial_balance

        # 状态追踪
        self.daily_pnl: float = 0.0
        self.peak_balance: float = initial_balance
        self.positions: dict[str, Position] = {}
        self.order_timestamps: list[float] = []

        # 熔断状态
        self.is_halted: bool = False
        self.halt_reason: Optional[str] = None

    def check_order(self, order: Order, signal: TradeSignal) -> tuple[bool, Optional[str]]:
        """
        检查订单是否通过风控

        Returns:
            (通过, 拒绝原因)
        """
        # 1. 熔断检查
        if self.is_halted:
            return False, f"Trading halted: {self.halt_reason}"

        # 2. 信号质量检查
        if signal.mispricing < self.limits.min_edge:
            return False, f"Edge too low: {signal.mispricing:.2%} < {self.limits.min_edge:.2%}"

        if signal.ev < self.limits.min_ev:
            return False, f"EV too low: {signal.ev:.4f} < {self.limits.min_ev}"

        # 3. 仓位检查
        if order.size > self.limits.max_position_size:
            return False, f"Order size {order.size} exceeds max {self.limits.max_position_size}"

        total_exposure = self._calculate_total_exposure()
        if total_exposure + order.size > self.limits.max_total_exposure:
            return False, f"Would exceed total exposure limit: {total_exposure + order.size} > {self.limits.max_total_exposure}"

        market_exposure = self._get_market_exposure(order.market_id)
        if market_exposure + order.size > self.limits.max_position_per_market:
            return False, f"Would exceed market exposure limit"

        # 4. 亏损检查
        if self.daily_pnl < -self.limits.max_daily_loss:
            self._halt("Daily loss limit reached")
            return False, "Daily loss limit reached"

        drawdown = (self.peak_balance - self.current_balance) / self.peak_balance
        if drawdown > self.limits.max_drawdown_pct:
            self._halt(f"Max drawdown reached: {drawdown:.2%}")
            return False, f"Max drawdown reached: {drawdown:.2%}"

        # 5. 频率检查
        now = time.time()
        self.order_timestamps = [t for t in self.order_timestamps if now - t < 60]

        if len(self.order_timestamps) >= self.limits.max_orders_per_minute:
            return False, "Order rate limit exceeded"

        if self.order_timestamps and (now - self.order_timestamps[-1]) < self.limits.min_order_interval:
            return False, "Min order interval not met"

        return True, None

    def on_order_submitted(self, order: Order):
        """订单提交后更新状态"""
        self.order_timestamps.append(time.time())

    def on_trade(self, trade: TradeRecord):
        """成交后更新状态"""
        self.daily_pnl += (trade.pnl or 0) - trade.fee
        self.current_balance += (trade.pnl or 0) - trade.fee
        self.peak_balance = max(self.peak_balance, self.current_balance)

    def _calculate_total_exposure(self) -> float:
        return sum(p.market_value for p in self.positions.values())

    def _get_market_exposure(self, market_id: str) -> float:
        return sum(
            p.market_value for p in self.positions.values()
            if p.market_id == market_id
        )

    def _halt(self, reason: str):
        self.is_halted = True
        self.halt_reason = reason

    def reset_daily(self):
        """每日重置"""
        self.daily_pnl = 0
        self.is_halted = False
        self.halt_reason = None
```

### 11.4 Polymarket 交易执行器

```python
import hmac
import hashlib
import time
from typing import Optional
import aiohttp

class PolymarketTrader:
    """
    Polymarket CLOB 交易执行器

    API 文档: https://docs.polymarket.com/
    需要 API Key 进行签名认证
    """

    CLOB_URL = "https://clob.polymarket.com"

    def __init__(
        self,
        api_key: str,
        api_secret: str,
        passphrase: str,
        risk_manager: RiskManager
    ):
        self.api_key = api_key
        self.api_secret = api_secret
        self.passphrase = passphrase
        self.risk_manager = risk_manager

        self._session: Optional[aiohttp.ClientSession] = None
        self.active_orders: dict[str, Order] = {}

    async def _get_session(self) -> aiohttp.ClientSession:
        if self._session is None or self._session.closed:
            self._session = aiohttp.ClientSession()
        return self._session

    def _sign_request(self, method: str, path: str, body: str = "") -> dict:
        """生成签名 headers"""
        timestamp = str(int(time.time() * 1000))
        message = timestamp + method.upper() + path + body

        signature = hmac.new(
            self.api_secret.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()

        return {
            "POLY_API_KEY": self.api_key,
            "POLY_SIGNATURE": signature,
            "POLY_TIMESTAMP": timestamp,
            "POLY_PASSPHRASE": self.passphrase,
            "Content-Type": "application/json"
        }

    async def execute_signal(self, signal: TradeSignal, market: BinaryMarket) -> Optional[Order]:
        """
        执行交易信号

        流程:
        1. 构建订单
        2. 风控检查
        3. 提交订单
        4. 追踪状态
        """
        # 1. 构建订单
        order = self._build_order(signal, market)

        # 2. 风控检查
        passed, reason = self.risk_manager.check_order(order, signal)
        if not passed:
            order.status = OrderStatus.REJECTED
            order.error_message = reason
            return order

        # 3. 提交订单
        try:
            result = await self._submit_order(order)
            order.exchange_order_id = result.get("orderID")
            order.status = OrderStatus.SUBMITTED
            self.active_orders[order.order_id] = order
            self.risk_manager.on_order_submitted(order)

        except Exception as e:
            order.status = OrderStatus.REJECTED
            order.error_message = str(e)

        return order

    def _build_order(self, signal: TradeSignal, market: BinaryMarket) -> Order:
        """根据信号构建订单"""
        # 确定 token_id
        if signal.side == "YES":
            token_id = market.yes_token_id
            quote = market.yes_quote
        else:
            token_id = market.no_token_id
            quote = market.no_quote

        # 根据紧急度选择订单类型
        if signal.urgency == "HIGH":
            order_type = OrderType.MARKET
            price = quote.ask * 1.01  # 略高于卖一价确保成交
        else:
            order_type = OrderType.LIMIT
            price = min(quote.ask, signal.fair_price * 0.98)  # 尝试更好价格

        return Order(
            market_id=market.market_id,
            token_id=token_id,
            side=OrderSide.BUY,
            order_type=order_type,
            price=price,
            size=signal.suggested_size,
            signal=signal
        )

    async def _submit_order(self, order: Order) -> dict:
        """提交订单到 CLOB"""
        session = await self._get_session()

        path = "/order"
        body = {
            "tokenID": order.token_id,
            "side": order.side.value,
            "type": order.order_type.value,
            "price": str(order.price),
            "size": str(order.size),
        }

        if order.order_type == OrderType.FOK:
            body["timeInForce"] = "FOK"
        elif order.order_type == OrderType.IOC:
            body["timeInForce"] = "IOC"

        import json
        body_str = json.dumps(body)
        headers = self._sign_request("POST", path, body_str)

        async with session.post(
            f"{self.CLOB_URL}{path}",
            headers=headers,
            data=body_str
        ) as resp:
            if resp.status != 200:
                error = await resp.text()
                raise Exception(f"Order failed: {error}")
            return await resp.json()

    async def cancel_order(self, order_id: str) -> bool:
        """撤销订单"""
        session = await self._get_session()

        path = f"/order/{order_id}"
        headers = self._sign_request("DELETE", path)

        async with session.delete(
            f"{self.CLOB_URL}{path}",
            headers=headers
        ) as resp:
            if resp.status == 200:
                if order_id in self.active_orders:
                    self.active_orders[order_id].status = OrderStatus.CANCELLED
                return True
            return False

    async def get_order_status(self, order_id: str) -> dict:
        """查询订单状态"""
        session = await self._get_session()

        path = f"/order/{order_id}"
        headers = self._sign_request("GET", path)

        async with session.get(
            f"{self.CLOB_URL}{path}",
            headers=headers
        ) as resp:
            return await resp.json()

    async def sync_positions(self) -> list[Position]:
        """同步持仓"""
        session = await self._get_session()

        path = "/positions"
        headers = self._sign_request("GET", path)

        async with session.get(
            f"{self.CLOB_URL}{path}",
            headers=headers
        ) as resp:
            data = await resp.json()

            positions = []
            for p in data:
                positions.append(Position(
                    market_id=p["conditionId"],
                    token_id=p["tokenId"],
                    side=p["outcome"],
                    size=float(p["size"]),
                    avg_cost=float(p["avgCost"]),
                    current_price=float(p.get("currentPrice", p["avgCost"]))
                ))

            # 更新风控管理器
            self.risk_manager.positions = {p.token_id: p for p in positions}

            return positions

    async def close_position(self, position: Position) -> Optional[Order]:
        """平仓"""
        # 卖出持有的份额
        order = Order(
            market_id=position.market_id,
            token_id=position.token_id,
            side=OrderSide.SELL,
            order_type=OrderType.MARKET,
            price=position.current_price * 0.99,  # 略低于市价确保成交
            size=position.size
        )

        try:
            result = await self._submit_order(order)
            order.exchange_order_id = result.get("orderID")
            order.status = OrderStatus.SUBMITTED
            return order
        except Exception as e:
            order.status = OrderStatus.REJECTED
            order.error_message = str(e)
            return order
```

### 11.5 订单管理器

```python
import asyncio
from typing import Callable, Optional

class OrderManager:
    """
    订单生命周期管理

    - 追踪活跃订单
    - 自动过期撤单
    - 部分成交处理
    - 事件回调
    """

    def __init__(
        self,
        trader: PolymarketTrader,
        on_fill: Optional[Callable] = None,
        on_cancel: Optional[Callable] = None
    ):
        self.trader = trader
        self.on_fill = on_fill
        self.on_cancel = on_cancel

        self._running = False

    async def start(self, poll_interval: float = 1.0):
        """启动订单追踪"""
        self._running = True

        while self._running:
            await self._poll_orders()
            await asyncio.sleep(poll_interval)

    async def stop(self):
        self._running = False

    async def _poll_orders(self):
        """轮询订单状态"""
        for order_id, order in list(self.trader.active_orders.items()):
            if not order.is_active:
                continue

            try:
                status = await self.trader.get_order_status(order.exchange_order_id)
                await self._update_order(order, status)
            except Exception as e:
                print(f"Error polling order {order_id}: {e}")

    async def _update_order(self, order: Order, status: dict):
        """更新订单状态"""
        old_status = order.status
        new_status_str = status.get("status", "").upper()

        # 映射状态
        status_map = {
            "OPEN": OrderStatus.OPEN,
            "FILLED": OrderStatus.FILLED,
            "PARTIALLY_FILLED": OrderStatus.PARTIALLY_FILLED,
            "CANCELLED": OrderStatus.CANCELLED,
            "EXPIRED": OrderStatus.EXPIRED,
        }

        if new_status_str in status_map:
            order.status = status_map[new_status_str]

        # 更新成交信息
        filled = float(status.get("filledSize", 0))
        if filled > order.filled_size:
            order.filled_size = filled
            order.fill_price = float(status.get("avgFillPrice", order.price))

            if self.on_fill:
                await self.on_fill(order)

        # 完成的订单移出活跃列表
        if order.is_complete:
            del self.trader.active_orders[order.order_id]

            if order.status == OrderStatus.CANCELLED and self.on_cancel:
                await self.on_cancel(order)

    async def cancel_stale_orders(self, max_age_seconds: float = 30):
        """撤销过期订单"""
        now = datetime.utcnow()

        for order in list(self.trader.active_orders.values()):
            age = (now - order.created_at).total_seconds()

            if age > max_age_seconds and order.status == OrderStatus.OPEN:
                await self.trader.cancel_order(order.exchange_order_id)
```

### 11.6 自动交易主循环

```python
class AutoTrader:
    """
    自动交易主控

    整合信号检测、风控、执行
    """

    def __init__(self, config: Config):
        self.config = config

        # 风控
        self.risk_manager = RiskManager(
            limits=RiskLimits(
                max_position_size=config.get("risk.max_position", 50),
                max_total_exposure=config.get("risk.max_exposure", 200),
                max_daily_loss=config.get("risk.max_daily_loss", 50),
            ),
            initial_balance=config.get("risk.bankroll", 1000)
        )

        # 交易执行
        self.trader = PolymarketTrader(
            api_key=config.get("trading.api_key"),
            api_secret=config.get("trading.api_secret"),
            passphrase=config.get("trading.passphrase"),
            risk_manager=self.risk_manager
        )

        # 订单管理
        self.order_manager = OrderManager(
            trader=self.trader,
            on_fill=self._on_fill,
            on_cancel=self._on_cancel
        )

        # 模式
        self.paper_trading = config.get("trading.paper_mode", True)
        self.auto_execute = config.get("trading.auto_execute", False)

        # 统计
        self.signals_received = 0
        self.orders_executed = 0
        self.orders_filled = 0

    async def on_signal(self, signal: TradeSignal, market: BinaryMarket):
        """
        接收交易信号

        根据配置决定自动执行或仅记录
        """
        self.signals_received += 1

        # Paper trading 模式
        if self.paper_trading:
            await self._paper_trade(signal, market)
            return

        # 自动执行模式
        if self.auto_execute:
            order = await self.trader.execute_signal(signal, market)

            if order and order.status == OrderStatus.SUBMITTED:
                self.orders_executed += 1
                print(f"Order submitted: {order.order_id} - {signal.side} @ {signal.market_price:.2%}")
            elif order:
                print(f"Order rejected: {order.error_message}")

    async def _paper_trade(self, signal: TradeSignal, market: BinaryMarket):
        """模拟交易"""
        print(f"[PAPER] Would {signal.action} {signal.side}")
        print(f"        Size: ${signal.suggested_size:.2f}")
        print(f"        Price: {signal.market_price:.2%}")
        print(f"        Edge: {signal.mispricing:.2%}")
        print(f"        EV: +${signal.ev:.2f}")

    async def _on_fill(self, order: Order):
        """成交回调"""
        self.orders_filled += 1

        pnl = 0  # 买入时 pnl 为 0，卖出时计算
        fee = order.filled_size * 0.001  # 假设 0.1% 手续费

        trade = TradeRecord(
            trade_id=str(uuid.uuid4()),
            order_id=order.order_id,
            market_id=order.market_id,
            side=order.side.value,
            price=order.fill_price,
            size=order.filled_size,
            fee=fee,
            timestamp=datetime.utcnow(),
            pnl=pnl
        )

        self.risk_manager.on_trade(trade)

        print(f"Order filled: {order.order_id}")
        print(f"  Price: {order.fill_price:.2%}")
        print(f"  Size: ${order.filled_size:.2f}")

    async def _on_cancel(self, order: Order):
        """撤单回调"""
        print(f"Order cancelled: {order.order_id}")

    async def run(self):
        """启动自动交易"""
        # 同步持仓
        await self.trader.sync_positions()

        # 启动订单管理
        order_task = asyncio.create_task(self.order_manager.start())

        try:
            while True:
                await asyncio.sleep(1)
        finally:
            await self.order_manager.stop()
            order_task.cancel()

    def get_stats(self) -> dict:
        """获取统计信息"""
        return {
            "signals_received": self.signals_received,
            "orders_executed": self.orders_executed,
            "orders_filled": self.orders_filled,
            "fill_rate": self.orders_filled / self.orders_executed if self.orders_executed > 0 else 0,
            "daily_pnl": self.risk_manager.daily_pnl,
            "current_balance": self.risk_manager.current_balance,
            "is_halted": self.risk_manager.is_halted,
        }
```

### 11.7 交易配置

```yaml
# config/settings.yaml 追加

# 交易配置
trading:
  # 模式
  paper_mode: true          # 模拟交易模式 (强烈建议先用此模式)
  auto_execute: false       # 自动执行 (需关闭 paper_mode)

  # API 认证
  api_key: ${POLYMARKET_API_KEY}
  api_secret: ${POLYMARKET_API_SECRET}
  passphrase: ${POLYMARKET_PASSPHRASE}

  # 执行参数
  default_order_type: "LIMIT"   # MARKET / LIMIT / FOK / IOC
  max_slippage: 0.02            # 最大滑点 2%
  order_timeout: 30             # 订单超时撤单 (秒)

# 风控配置
risk:
  bankroll: 1000                # 总资金
  max_position: 50              # 单笔最大
  max_exposure: 200             # 总敞口
  max_daily_loss: 50            # 日亏损上限
  max_drawdown_pct: 0.10        # 最大回撤 10%

  # 频率限制
  min_order_interval: 1.0       # 最小下单间隔
  max_orders_per_minute: 10     # 每分钟最大订单
```

---

## 12. 风险提示

### 12.1 系统性风险

| 风险类型 | 描述 | 缓解措施 |
|---------|------|---------|
| API延迟 | 数据延迟导致信号失效 | 监控延迟指标，设置最大延迟阈值 |
| 价格源故障 | 单一价格源错误 | 多源聚合 + 异常检测 |
| 流动性不足 | 无法以预期价格成交 | 检查深度，限制单笔金额 |
| 模型失效 | 极端行情下定价模型失准 | 设置置信度门槛，异常波动暂停 |

### 12.2 操作建议

1. **从模拟开始**：先运行 paper trading 模式验证策略
2. **小额试水**：真实交易从最小仓位开始
3. **持续监控**：保持对系统运行状态的实时关注
4. **定期复盘**：分析交易记录，优化模型参数

---

## 13. 后续迭代规划

### Phase 1: MVP (当前文档)
- [x] 核心定价模型
- [x] 单市场监控
- [x] 终端警报
- [x] 自动交易执行 (买入/卖出)
- [x] 风控模块

### Phase 2: 增强
- [ ] 多市场并行监控
- [ ] 历史数据回测框架
- [ ] WebSocket 实时订单更新

### Phase 3: 进阶
- [ ] 机器学习定价模型
- [ ] 跨市场套利
- [ ] 完整的风控系统

---

*文档版本: v1.0*
*最后更新: 2024-01*
