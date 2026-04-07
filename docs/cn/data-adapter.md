# ValueCell 数据适配器深度解析

## 学习目标

阅读本文后，你将能够：

- 解释 ValueCell 为什么需要数据适配器系统，以及它解决了什么问题
- 描述 Adapter Manager 的路由机制、自动故障转移（Failover）和缓存策略
- 区分三个适配器（Yahoo Finance、AKShare、BaoStock）各自的市场覆盖、数据类型和限制
- 理解统一 Ticker 格式（`EXCHANGE:SYMBOL`）和 Ticker 转换机制
- 跟踪从 FastAPI lifespan 到 Agent 使用市场数据的完整链路
- 知道如何添加一个新的数据适配器

---

## 1. 为什么需要数据适配器

ValueCell 是一个面向全球金融市场的多智能体平台。Agent 在执行研究、分析和策略任务时，需要获取：

- **实时行情**：股票当前价格、涨跌幅、成交量
- **历史 K 线**：日线、周线、分钟线等历史数据
- **资产信息**：公司名称、交易所、币种、行业分类
- **资产搜索**：根据关键词查找股票、ETF、指数

问题在于，不同市场的数据散落在不同的数据源中，每个数据源的 API 接口、Ticker 格式、字段命名都不一样：

| 市场 | 数据源 | Ticker 格式示例 |
|------|--------|-----------------|
| 美股 | Yahoo Finance | `AAPL`（无后缀） |
| 港股 | Yahoo Finance | `0700.HK` |
| A 股（上海） | Yahoo Finance | `600519.SS` |
| A 股（深圳） | AKShare / BaoStock | `000001`（纯数字） |
| 加密货币 | Yahoo Finance | `BTC-USD` |

如果让每个 Agent 直接对接这些数据源，会导致：

1. **重复代码**：每个 Agent 都要处理不同数据源的 API 差异
2. **维护困难**：数据源 API 变更时，需要修改多处
3. **路由复杂**：Agent 需要知道"美股用 Yahoo Finance，A 股用 AKShare"这样的路由逻辑

数据适配器系统通过引入**统一接口 + Manager 路由**的方式解决了这些问题。Agent 只需要调用 `AdapterManager` 的统一方法，不需要关心底层使用了哪个数据源。

---

## 2. 架构总览

### 2.1 核心文件

数据适配器系统的代码位于 `python/valuecell/adapters/assets/`：

```
python/valuecell/adapters/assets/
├── __init__.py              # 模块入口，导出公共 API
├── types.py                 # 核心数据类型定义（Asset、AssetPrice、Exchange 等）
├── base.py                  # 抽象基类 BaseDataAdapter
├── manager.py               # AdapterManager + WatchlistManager
├── yfinance_adapter.py      # Yahoo Finance 适配器
├── akshare_adapter.py       # AKShare 适配器
├── baostock_adapter.py      # BaoStock 适配器
└── i18n_integration.py      # 国际化支持（资产名称本地化）
```

上层服务位于 `python/valuecell/server/services/assets/`：

```
python/valuecell/server/services/assets/
├── __init__.py
└── asset_service.py         # 高层服务层，封装 AdapterManager + i18n
```

### 2.2 架构分层

```
  Agent（通过工具调用）
       ↓
  AssetService（服务层，含 i18n 格式化）
       ↓
  AdapterManager（统一路由 + 自动 Failover）
       ↓
  ┌────────────┬────────────┬────────────┐
  │ YFinance   │ AKShare    │ BaoStock   │
  │ Adapter    │ Adapter    │ Adapter    │
  └─────┬──────┴─────┬──────┴─────┬──────┘
        ↓            ↓            ↓
  Yahoo Finance  东方财富等     证券宝
  API            国内数据源     API
```

### 2.3 统一 Ticker 格式

ValueCell 内部使用统一的 Ticker 格式：`EXCHANGE:SYMBOL`

| 交易所 | 枚举值 | Ticker 示例 |
|--------|--------|-------------|
| 纳斯达克 | `NASDAQ` | `NASDAQ:AAPL` |
| 纽交所 | `NYSE` | `NYSE:JPM` |
| 美国证券交易所 | `AMEX` | `AMEX:GLD` |
| 上海证券交易所 | `SSE` | `SSE:600519` |
| 深圳证券交易所 | `SZSE` | `SZSE:000001` |
| 北京证券交易所 | `BSE` | `BSE:835368` |
| 香港联交所 | `HKEX` | `HKEX:00700` |
| 加密货币 | `CRYPTO` | `CRYPTO:BTC` |

每个适配器都实现了 `convert_to_source_ticker()` 和 `convert_to_internal_ticker()` 方法，负责内部格式和外部格式之间的转换。

---

## 3. BaseDataAdapter —— 抽象基类

所有数据适配器都继承自 `BaseDataAdapter`（定义在 `base.py`），它定义了适配器必须实现的接口：

```python
class BaseDataAdapter(ABC):
    @abstractmethod
    def _initialize(self) -> None: ...

    @abstractmethod
    def search_assets(self, query: AssetSearchQuery) -> List[AssetSearchResult]: ...

    @abstractmethod
    def get_asset_info(self, ticker: str) -> Optional[Asset]: ...

    @abstractmethod
    def get_real_time_price(self, ticker: str) -> Optional[AssetPrice]: ...

    @abstractmethod
    def get_historical_prices(self, ticker, start_date, end_date, interval) -> List[AssetPrice]: ...

    @abstractmethod
    def get_capabilities(self) -> List[AdapterCapability]: ...

    @abstractmethod
    def convert_to_source_ticker(self, internal_ticker: str) -> str: ...

    @abstractmethod
    def convert_to_internal_ticker(self, source_ticker, default_exchange) -> str: ...
```

`AdapterCapability` 数据类描述了适配器支持的资产类型和交易所：

```python
@dataclass
class AdapterCapability:
    asset_type: AssetType          # 如 STOCK、ETF、INDEX、CRYPTO
    exchanges: Set[Exchange]       # 支持的交易所集合
```

基类还提供了默认实现的方法：

- `get_multiple_prices()`：批量获取价格（默认逐个调用 `get_real_time_price()`）
- `validate_ticker()`：验证 Ticker 是否属于本适配器支持的交易所
- `get_supported_asset_types()`：从 capabilities 中提取资产类型列表
- `get_supported_exchanges()`：从 capabilities 中提取交易所集合

---

## 4. AdapterManager —— 核心调度器

`AdapterManager`（定义在 `manager.py`）是整个适配器系统的核心，承担三个关键职责：

### 4.1 路由表机制

AdapterManager 维护一个 **Exchange → Adapters** 的路由表：

```python
self.exchange_routing: Dict[str, List[BaseDataAdapter]] = {}
```

当注册新适配器时，`_rebuild_routing_table()` 方法会遍历所有适配器的 capabilities，建立交易所到适配器的映射。例如，`SSE`（上海证券交易所）会同时映射到 AKShare 和 BaoStock 两个适配器。

### 4.2 Ticker 缓存

为了加速重复查询，AdapterManager 维护了一个 Ticker → Adapter 的缓存：

```python
self._ticker_cache: Dict[str, BaseDataAdapter] = {}
```

当首次查询某个 Ticker 时，Manager 会遍历该交易所的所有适配器，找到第一个 `validate_ticker()` 返回 `True` 的适配器并缓存结果。

### 4.3 自动故障转移（Failover）

这是 AdapterManager 最重要的设计之一。所有核心方法（`get_asset_info()`、`get_real_time_price()`、`get_historical_prices()`）都实现了自动故障转移：

```
1. 获取主适配器（从缓存或路由表）
2. 尝试主适配器
   ├── 成功 → 返回结果
   └── 失败 → 进入 Failover
3. Failover：遍历同一交易所的其他适配器
   ├── 尝试备选适配器
   │   ├── 成功 → 更新缓存，返回结果
   │   └── 失败 → 继续尝试下一个
   └── 全部失败 → 返回 None 或空列表
```

这意味着，对于 A 股市场，如果 AKShare 的 API 暂时不可用，系统会自动切换到 BaoStock 尝试获取数据。

### 4.4 并行搜索

`search_assets()` 方法使用 `ThreadPoolExecutor` 并行查询所有适配器，然后对结果进行智能去重：

- 按 `(symbol, country)` 去重
- 按交易所优先级（NASDAQ > NYSE > AMEX）选择最佳结果
- 按相关性分数排序

如果所有适配器都没有返回结果，还会触发 **LLM 回退搜索**（`_fallback_search_assets()`）：使用 LLM 根据用户查询生成可能的 Ticker 候选列表，然后逐个验证。

### 4.5 全局单例

AdapterManager 通过全局单例模式管理：

```python
_adapter_manager: Optional[AdapterManager] = None

def get_adapter_manager() -> AdapterManager:
    global _adapter_manager
    if _adapter_manager is None:
        _adapter_manager = AdapterManager()
    return _adapter_manager
```

### 4.6 线程安全设计

AdapterManager 使用 `threading.RLock`（可重入锁）保护内部状态：

```python
self._lock = threading.RLock()          # 保护 adapters 和 routing table
self._cache_lock = threading.Lock()     # 保护 ticker cache
```

- 所有修改操作（`register_adapter`、`_rebuild_routing_table`）都在 `_lock` 下执行
- Ticker 缓存的读写使用独立的 `_cache_lock`
- `search_assets()` 使用 `ThreadPoolExecutor` 并行查询，每个适配器的调用在独立线程中执行

### 4.7 模块导出

`__init__.py` 中导出的公共 API：

```python
__all__ = [
    "Asset", "AssetPrice", "AssetSearchResult", "AssetSearchQuery",
    "AssetType", "MarketStatus", "DataSource", "Exchange",
    "MarketInfo", "LocalizedName", "Watchlist", "WatchlistItem",
    "BaseDataAdapter", "AdapterCapability",
    "YFinanceAdapter", "AKShareAdapter",  # 注意：BaoStockAdapter 未导出
    "AdapterManager", "WatchlistManager",
    "get_adapter_manager", "get_watchlist_manager",
    "reset_managers",
    "AssetI18nService", "get_asset_i18n_service", "reset_asset_i18n_service",
]
```

> **注意**：`BaoStockAdapter` 未包含在 `__all__` 导出列表中，但 `configure_baostock()` 方法仍可正常使用（内部通过延迟导入 `from .baostock_adapter import BaoStockAdapter`）。

---

## 5. Yahoo Finance 适配器

**文件**：`python/valuecell/adapters/assets/yfinance_adapter.py`
**依赖库**：`yfinance`
**数据源**：Yahoo Finance API

### 5.1 能力范围

| 能力 | 说明 |
|------|------|
| 资产搜索 | 通过 `yf.Search()` API 搜索股票、ETF、指数等 |
| 实时行情 | 通过 `Ticker.history(period="1d", interval="1m")` 获取 |
| 历史数据 | 支持从 1 分钟到月线的多种时间粒度 |
| 批量价格 | 通过 `yf.download()` 批量获取多个资产价格 |
| 资产信息 | 包含行业、市值、PE 比率、股息率、贝塔系数等 |

### 5.2 支持的市场

| 资产类型 | 支持的交易所 |
|----------|-------------|
| 股票（STOCK） | NASDAQ、NYSE、AMEX、SSE、SZSE、HKEX |
| ETF | NASDAQ、NYSE、AMEX、SSE、SZSE、HKEX |
| 指数（INDEX） | NASDAQ、NYSE、SSE、SZSE、HKEX |
| 加密货币（CRYPTO） | CRYPTO |

### 5.3 Ticker 转换规则

从内部格式转换为 Yahoo Finance 格式：

| 内部格式 | Yahoo Finance 格式 | 规则 |
|----------|-------------------|------|
| `NASDAQ:AAPL` | `AAPL` | 美股不加后缀 |
| `NYSE:JPM` | `JPM` | 美股不加后缀 |
| `SSE:600519` | `600519.SS` | 添加 `.SS` 后缀 |
| `SZSE:000001` | `000001.SZ` | 添加 `.SZ` 后缀 |
| `HKEX:00700` | `0700.HK` | 去前导零后补 4 位 + `.HK` |
| `NASDAQ:IXIC`（指数） | `^IXIC` | 添加 `^` 前缀 |
| `CRYPTO:BTC` | `BTC-USD` | 添加 `-USD` 后缀 |

### 5.4 历史数据时间粒度

支持的 interval 值：

| 内部格式 | Yahoo Finance 格式 | 说明 |
|----------|-------------------|------|
| `1m` | `1m` | 1 分钟 |
| `5m` | `5m` | 5 分钟 |
| `15m` | `15m` | 15 分钟 |
| `60m` | `60m` | 60 分钟 |
| `1h` | `1h` | 1 小时 |
| `1d` | `1d` | 日线（默认） |
| `1w` | `1wk` | 周线 |
| `1mo` | `1mo` | 月线 |

### 5.5 特点与限制

- **无需 API Key**：完全免费，不需要注册
- **覆盖面广**：美股、港股、A 股、加密货币均有覆盖
- **重试机制**：内置指数退避重试（默认 3 次，退避基数 0.5 秒）
- **多级降级**：获取资产信息时，依次尝试 `info` → `Search API` → `fast_info`
- **限制**：Yahoo Finance 是非官方 API，可能存在频率限制；A 股数据不如专业数据源全面

---

## 6. AKShare 适配器

**文件**：`python/valuecell/adapters/assets/akshare_adapter.py`
**依赖库**：`akshare`
**数据源**：东方财富（East Money）等国内金融数据接口

### 6.1 能力范围

| 能力 | 说明 |
|------|------|
| 资产搜索 | 不支持搜索（`search_assets()` 返回空列表） |
| 实时行情 | 通过东方财富 1 分钟 API 获取最新数据 |
| 历史数据 | 支持从 1 分钟到月线的多种时间粒度，前复权（qfq） |
| 资产信息 | 通过雪球（Xueqiu）API 获取详细公司信息，支持中英文名称 |

### 6.2 支持的市场

| 资产类型 | 支持的交易所 |
|----------|-------------|
| 股票（STOCK） | SSE、SZSE、BSE、HKEX、NASDAQ、NYSE、AMEX |
| ETF | SSE、SZSE、BSE、HKEX、NASDAQ、NYSE、AMEX |
| 指数（INDEX） | SSE、SZSE、BSE、HKEX、NASDAQ、NYSE、AMEX |

### 6.3 Ticker 转换规则

从内部格式转换为 AKShare 格式：

| 内部格式 | AKShare 格式 | 说明 |
|----------|-------------|------|
| `SSE:600519` | `600519` | A 股直接使用数字代码 |
| `SZSE:000001` | `000001` | A 股直接使用数字代码 |
| `BSE:430047` | `430047` | 北交所直接使用数字代码 |
| `HKEX:00700` | `00700` | 港股直接使用数字代码 |
| `NASDAQ:AAPL` | `105.AAPL` | 美股需加交易所代码前缀 |
| `NYSE:JPM` | `106.JPM` | 美股需加交易所代码前缀 |
| `AMEX:GLD` | `107.GLD` | 美股需加交易所代码前缀 |

美股交易所代码映射：

| 交易所 | AKShare 代码 |
|--------|-------------|
| NASDAQ | `105` |
| NYSE | `106` |
| AMEX | `107` |
| 美股指数 | `100` |

### 6.4 市场区分逻辑

AKShare 对不同市场使用不同的 API 调用：

**实时行情**：

| 市场 | API 函数 | 说明 |
|------|----------|------|
| A 股 | `stock_zh_a_hist_min_em()` | 1 分钟数据 |
| 港股个股 | `stock_hk_hist_min_em()` | 1 分钟数据 |
| 港股指数 | `stock_hk_index_spot_em()` | 指数现货数据 |
| 美股个股 | `stock_us_hist_min_em()` | 分钟数据 |
| 美股指数 | `index_us_stock_sina()` | 新浪指数数据 |

**历史数据**：

| 市场 | API 函数 | 复权方式 |
|------|----------|----------|
| A 股 | `stock_zh_a_hist()` | 前复权（qfq） |
| 港股个股 | `stock_hk_hist()` | 前复权（qfq） |
| 港股指数 | `stock_hk_index_daily_em()` | 无复权 |
| 美股个股 | `stock_us_hist()` | 前复权（qfq） |
| 美股指数 | `index_us_stock_sina()` | 无复权 |

### 6.5 字段映射

AKShare 的 API 返回字段名不稳定（中英文混用、版本间可能变化），适配器内部维护了字段映射表来处理这个问题：

```python
# 以 A 股为例
"a_shares": {
    "code": ["代码", "symbol", "ts_code"],
    "name": ["名称", "name", "short_name"],
    "price": ["最新价", "close", "price", "latest"],
    "open": ["今开", "开盘", "open"],
    ...
}
```

`_get_field_name()` 方法按优先顺序依次检查 DataFrame 中是否存在这些字段名，自动适配不同版本的 API 返回。

### 6.6 特点与限制

- **无需 API Key**：完全免费，不需要注册
- **A 股数据丰富**：覆盖沪深北三交易所，数据来源为东方财富等主流国内数据源
- **跨市场覆盖**：同时支持 A 股、港股和美股
- **不支持搜索**：需要配合 Yahoo Finance 的搜索能力，或使用 Manager 的 LLM 回退搜索
- **雪球 Token**：获取详细资产信息时需要 `XUEQIU_TOKEN` 环境变量
- **API 不稳定**：AKShare 的 API 字段名可能随版本变化，适配器通过字段映射机制应对

---

## 7. BaoStock 适配器

**文件**：`python/valuecell/adapters/assets/baostock_adapter.py`
**依赖库**：`baostock`
**数据源**：证券宝（http://baostock.com）

### 7.1 能力范围

| 能力 | 说明 |
|------|------|
| 资产搜索 | 按代码或名称搜索（`query_stock_basic()`） |
| 实时行情 | 获取最近交易日的日线数据作为"实时"价格 |
| 历史数据 | 支持分钟线、日线、周线、月线 |
| 资产信息 | 通过 `query_stock_basic()` 获取基本信息 |

### 7.2 支持的市场

| 资产类型 | 支持的交易所 |
|----------|-------------|
| 股票（STOCK） | SSE、SZSE |
| 指数（INDEX） | SSE、SZSE |
| ETF | SSE、SZSE |

> **注意**：BaoStock 仅支持上海证券交易所（SSE）和深圳证券交易所（SZSE），不支持北交所（BSE）和港股。

### 7.3 Ticker 转换规则

| 内部格式 | BaoStock 格式 | 说明 |
|----------|--------------|------|
| `SSE:600000` | `sh.600000` | 交易所前缀 + `.` + 代码 |
| `SZSE:000001` | `sz.000001` | 交易所前缀 + `.` + 代码 |

指数识别规则：

| 交易所 | 指数代码特征 | 示例 |
|--------|-------------|------|
| 上海 | `sh.000xxx` | `sh.000001`（上证指数） |
| 深圳 | `sz.399xxx` | `sz.399001`（深证成指） |

### 7.4 线程安全与连接管理

BaoStock 的特殊之处在于它使用**全局会话状态**（非线程安全）。适配器通过以下机制处理这个问题：

1. **全局锁**：使用 `threading.Lock()` 确保同一时间只有一个 API 调用
2. **登录管理**：每次 API 调用前检查登录状态，支持会话 TTL（默认 300 秒）
3. **超时保护**：使用 `func_timeout` 库对 API 调用设置超时（默认 10 秒）
4. **自动重试**：超时或错误时重新登录并重试一次

```
API 调用流程：
1. 获取全局锁
2. 检查登录状态（是否已登录 + 是否超过 TTL）
3. 如需要，重新登录
4. 执行 API 调用（带超时）
5. 检查结果 error_code
   ├── error_code == "0" → 返回结果
   └── error_code != "0" → 重新登录，重试一次
```

### 7.5 历史数据时间粒度

| 内部格式 | BaoStock 格式 | 说明 |
|----------|--------------|------|
| `5m` | `5` | 5 分钟 |
| `15m` | `15` | 15 分钟 |
| `30m` | `30` | 30 分钟 |
| `60m` | `60` | 60 分钟 |
| `1d` | `d` | 日线（默认） |
| `1w` | `w` | 周线 |
| `1mo` | `m` | 月线 |

> **注意**：BaoStock 不支持指数的分钟线数据。对于指数，只支持日线、周线和月线。

### 7.6 Pandas 2.0 兼容性

BaoStock 的 `get_data()` 方法内部使用了 `DataFrame.append()`，该方法在 pandas 2.0 中已被移除。适配器通过 `_get_data_safe()` 方法处理了这个兼容性问题：

```python
def _get_data_safe(self, rs):
    try:
        return rs.get_data()
    except AttributeError as e:
        if "append" in str(e):
            # pandas 2.0 兼容方案
            return pd.DataFrame(rs.data, columns=rs.fields)
```

### 7.7 特点与限制

- **无需 API Key**：完全免费，不需要注册
- **A 股历史数据全面**：提供完整的沪深两市历史 K 线数据
- **支持搜索**：可按股票代码或名称搜索
- **非线程安全**：需要通过全局锁机制处理并发
- **非真正实时**：通过获取最近交易日数据模拟"实时"价格
- **市场覆盖有限**：仅支持沪深两市，不支持北交所和港股

---

## 8. 初始化机制

### 8.1 FastAPI Lifespan Hook

三个适配器在 FastAPI 应用的 lifespan 阶段自动初始化。代码位于 `python/valuecell/server/api/app.py`：

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    # ... 数据库初始化 ...

    # 初始化并配置数据适配器
    try:
        logger.info("Configuring data adapters...")
        manager = get_adapter_manager()

        # Configure Yahoo Finance（免费，无需 API Key）
        try:
            manager.configure_yfinance()
            logger.info("✓ Yahoo Finance adapter configured")
        except Exception as e:
            logger.info(f"✗ Yahoo Finance adapter failed: {e}")

        # Configure AKShare（免费，无需 API Key，已优化）
        try:
            manager.configure_akshare()
            logger.info("✓ AKShare adapter configured (optimized)")
        except Exception as e:
            logger.info(f"✗ AKShare adapter failed: {e}")

        # Configure BaoStock（免费，无需 API Key）
        try:
            manager.configure_baostock()
            print("✓ BaoStock adapter configured")
        except Exception as e:
            print(f"✗ BaoStock adapter failed: {e}")

        print("Data adapters configuration completed")
    except Exception as e:
        logger.info(f"Error configuring adapters: {e}")

    yield
    # Shutdown
    logger.info("ValueCell Server shutting down...")
```

### 8.2 配置方法

AdapterManager 提供了三个配置方法，每个方法创建对应的适配器实例并注册到路由表：

```python
def configure_yfinance(self, **kwargs):
    adapter = YFinanceAdapter(**kwargs)
    self.register_adapter(adapter)

def configure_akshare(self, **kwargs):
    adapter = AKShareAdapter(**kwargs)
    self.register_adapter(adapter)

def configure_baostock(self, **kwargs):
    adapter = BaoStockAdapter(**kwargs)
    self.register_adapter(adapter)
```

`register_adapter()` 在注册适配器后会自动调用 `_rebuild_routing_table()` 重建路由表。

### 8.3 容错设计

每个适配器的初始化都包裹在独立的 `try-except` 中，某个适配器初始化失败不会影响其他适配器的正常工作。例如，如果 BaoStock 的 `baostock` 库没有安装，系统仍然可以正常使用 Yahoo Finance 和 AKShare。

---

## 9. Agent 如何使用市场数据

### 9.1 通过 AssetService

Agent 通常不直接调用 AdapterManager，而是通过更高层的 `AssetService`（位于 `python/valuecell/server/services/assets/asset_service.py`）访问数据。AssetService 封装了 AdapterManager 和国际化服务：

```python
class AssetService:
    def __init__(self):
        self.adapter_manager = get_adapter_manager()
        self.watchlist_manager = get_watchlist_manager()
        self.i18n_service = get_asset_i18n_service()
```

AssetService 提供的主要方法：

| 方法 | 说明 |
|------|------|
| `search_assets()` | 搜索资产，返回带本地化名称的结果 |
| `get_asset_info()` | 获取资产详细信息 |
| `get_asset_price()` | 获取实时价格（带格式化） |
| `get_multiple_prices()` | 批量获取价格 |
| `get_historical_prices()` | 获取历史 K 线数据 |
| `add_to_watchlist()` | 添加到自选股 |
| `get_watchlist()` | 获取自选股列表 |

### 9.2 调用链路

以获取实时价格为例，完整的调用链路：

```
Agent 工具调用
  → AssetService.get_asset_price("SSE:600519")
    → AdapterManager.get_real_time_price("SSE:600519")
      → 从缓存或路由表获取适配器
      → 路由到 AKShareAdapter（SSE 交易所）
        → AKShareAdapter.get_real_time_price("SSE:600519")
          → 转换为 AKShare 格式：symbol = "600519"
          → 调用 ak.stock_zh_a_hist_min_em(symbol="600519", period="1")
          → 将 DataFrame 转换为 AssetPrice 对象
      → 如果 AKShare 失败，自动尝试 BaoStockAdapter
    → i18n_service 格式化价格和涨跌幅
  → 返回带格式化的价格数据
```

### 9.3 智能路由

Agent 不需要指定使用哪个数据源。路由完全自动：

1. **解析 Ticker**：从 `SSE:600519` 提取交易所 `SSE`
2. **查找路由表**：找到支持 SSE 的适配器列表（AKShare、BaoStock、YFinance）
3. **验证 Ticker**：找到第一个 `validate_ticker()` 返回 `True` 的适配器
4. **缓存结果**：将 Ticker → Adapter 的映射缓存起来
5. **自动 Failover**：主适配器失败时，自动尝试同交易所的其他适配器

### 9.4 i18n 支持

AssetService 通过 `AssetI18nService` 提供国际化支持：

- **本地化资产名称**：同一资产在不同语言下显示不同名称（如 Apple Inc. / 苹果公司）
- **货币格式化**：根据币种和语言格式化价格（如 USD 的 `$1,234.56`，CNY 的 `¥12,345.67`）
- **涨跌幅格式化**：根据语言习惯显示涨跌幅
- **市值格式化**：大数字的缩写格式化（如 $1.23T）

---

## 10. 三个适配器对比

| 维度 | Yahoo Finance | AKShare | BaoStock |
|------|--------------|---------|----------|
| **数据源** | Yahoo Finance API | 东方财富等国内数据源 | 证券宝 |
| **依赖库** | `yfinance` | `akshare` | `baostock` |
| **API Key** | 不需要 | 不需要 | 不需要 |
| **资产搜索** | 支持（`yf.Search`） | 不支持 | 支持（按代码/名称） |
| **实时行情** | 真正实时（1 分钟级） | 近似实时（1 分钟级） | 最近交易日数据 |
| **历史数据** | 全面（分钟到月线） | 全面（分钟到月线） | 全面（5 分钟到月线） |
| **复权方式** | Yahoo 内部处理 | 前复权（qfq） | 前复权（非指数）/不复权（指数） |
| **资产信息** | 丰富（行业、PE、市值等） | 详细（中英文名称、行业） | 基础（代码、名称、类型） |
| **线程安全** | 安全 | 安全 | 需要全局锁 |
| **批量获取** | 支持（`yf.download`） | 逐个获取 | 逐个获取 |
| **A 股（沪深）** | 支持（`.SS` / `.SZ`） | 支持（原生） | 支持（原生） |
| **A 股（北交所）** | 不支持 | 支持 | 不支持 |
| **港股** | 支持（`.HK`） | 支持 | 不支持 |
| **美股** | 支持（原生） | 支持（交易所代码前缀） | 不支持 |
| **加密货币** | 支持（`-USD`） | 不支持 | 不支持 |
| **指数** | 支持 | 支持 | 支持（仅日线以上） |
| **ETF** | 支持 | 支持 | 支持 |
| **LLM 回退搜索** | Manager 层统一提供 | Manager 层统一提供 | Manager 层统一提供 |
| **主要优势** | 覆盖面最广，信息最丰富 | A 股数据最专业 | A 股历史数据稳定 |
| **主要限制** | 非官方 API，A 股数据不够专业 | 不支持搜索，API 字段不稳定 | 仅沪深两市，非线程安全 |

---

## 11. 适配器选择决策指南

在实际开发中，选择合适的适配器直接影响数据获取的成功率和效率。下表按照典型需求场景给出推荐方案：

| 需求场景 | 首选适配器 | 备选方案 | 说明 |
|----------|-----------|----------|------|
| A 股实时行情 | AKShare | YFinance / BaoStock | AKShare 通过东方财富接口获取 1 分钟级数据，延迟最低 |
| A 股历史 K 线（稳定性优先） | BaoStock | AKShare | BaoStock 数据源稳定，适合回测场景 |
| A 股历史 K 线（细节优先） | AKShare | BaoStock | AKShare 支持前复权且字段更丰富 |
| A 股（北交所） | AKShare | 无 | 仅 AKShare 支持 BSE |
| 美股实时行情 | YFinance | AKShare | YFinance 覆盖最全、信息最丰富 |
| 美股资产信息 | YFinance | AKShare | YFinance 提供行业、PE、市值等详细信息 |
| 港股实时行情 | YFinance | AKShare | YFinance 对港股支持成熟 |
| 港股历史数据 | AKShare | YFinance | AKShare 使用东方财富港股接口，数据粒度更细 |
| 加密货币 | YFinance | 无 | 仅 YFinance 支持加密货币 |
| 资产搜索 | YFinance | BaoStock / LLM 回退 | YFinance 通过 `yf.Search()` 提供最全面的搜索能力 |
| 批量价格获取 | YFinance | 无 | 仅 YFinance 支持 `yf.download()` 批量下载 |
| 回测与量化研究 | BaoStock + AKShare | YFinance | BaoStock 数据稳定，AKShare 数据丰富，二者互为补充 |

> **路由优先级提示**：AdapterManager 按注册顺序和 `validate_ticker()` 结果决定主适配器。对于同一交易所存在多个适配器的情况，Failover 机制会自动按路由表顺序依次尝试。开发者无需手动指定，但了解优先级有助于排查问题。

---

## 12. Ticker 转换速查

三个适配器对同一内部 Ticker 的格式转换各不相同。下表列出常见市场的转换对照，供快速查阅：

| 内部格式 | YFinance | AKShare | BaoStock | 说明 |
|----------|----------|---------|----------|------|
| `NASDAQ:AAPL` | `AAPL` | `105.AAPL` | -- | 美股纳斯达克 |
| `NYSE:JPM` | `JPM` | `106.JPM` | -- | 美股纽交所 |
| `AMEX:GLD` | `GLD` | `107.GLD` | -- | 美股证券交易所 |
| `SSE:600519` | `600519.SS` | `600519` | `sh.600519` | 上海证券交易所 |
| `SZSE:000001` | `000001.SZ` | `000001` | `sz.000001` | 深圳证券交易所 |
| `BSE:430047` | -- | `430047` | -- | 北京证券交易所 |
| `HKEX:00700` | `0700.HK` | `00700` | -- | 香港联交所 |
| `NASDAQ:IXIC` | `^IXIC` | `100.IXIC` | -- | 纳斯达克指数 |
| `CRYPTO:BTC` | `BTC-USD` | -- | -- | 加密货币 |

表中 `--` 表示该适配器不支持对应市场。转换规则由各适配器的 `convert_to_source_ticker()` 方法实现，逆向转换由 `convert_to_internal_ticker()` 方法实现。

**特殊转换规则备忘**：

- YFinance 港股：去前导零后补 4 位再加 `.HK`（如 `HKEX:00700` → `0700.HK`）
- YFinance 指数：添加 `^` 前缀（如 `NASDAQ:IXIC` → `^IXIC`）
- AKShare 美股：添加交易所代码前缀（NASDAQ=`105`、NYSE=`106`、AMEX=`107`、指数=`100`）
- BaoStock 指数：上海 `sh.000xxx`、深圳 `sz.399xxx`

---

## 13. 常见问题

### Q1：为什么获取 A 股数据有时会失败？

A 股数据的获取受多种因素影响：

- **API 频率限制**：东方财富等国内数据源对高频请求有速率限制，短时间内大量调用可能被限流
- **交易时段限制**：部分 API（尤其是实时行情接口）仅在交易时段（09:30-15:00）返回有效数据，非交易时段可能返回空结果或前一个交易日的数据
- **数据源不稳定**：AKShare 依赖的底层接口可能因数据源服务器维护、接口变更等原因暂时不可用
- **网络环境**：从海外服务器访问国内数据源可能存在较高的网络延迟或连接超时

建议：利用 AdapterManager 的自动 Failover 机制，确保至少配置两个 A 股适配器（AKShare + BaoStock），同时设置合理的重试策略。

### Q2：Ticker 格式转换错误怎么排查？

排查步骤：

1. **确认内部格式正确**：检查 Ticker 是否符合 `EXCHANGE:SYMBOL` 格式，交易所部分必须是 `Exchange` 枚举中定义的值（如 `SSE`、`SZSE`、`NASDAQ` 等）
2. **检查交易所匹配**：确认该交易所确实在目标适配器的 `get_capabilities()` 返回的 `exchanges` 集合中
3. **查看转换日志**：适配器在转换失败时会记录日志，关注 `convert_to_source_ticker()` 和 `convert_to_internal_ticker()` 的输出
4. **手动验证**：调用适配器的转换方法，检查输出是否符合数据源的格式要求（如 YFinance 的 `.SS` / `.SZ` 后缀、AKShare 的交易所代码前缀等）

### Q3：如何添加新的数据源？

完整的添加流程包含以下步骤：

1. 在 `python/valuecell/adapters/assets/` 下创建新的适配器文件，继承 `BaseDataAdapter` 并实现所有抽象方法
2. 在 `types.py` 的 `DataSource` 枚举中添加新的数据源标识
3. 在 `AdapterManager` 中添加 `configure_xxx()` 配置方法，内部通过延迟导入创建适配器实例并调用 `register_adapter()` 注册
4. 在 `app.py` 的 lifespan 函数中添加初始化调用，并用 `try-except` 包裹以保证单个适配器失败不影响其他适配器
5. 在 `__init__.py` 中更新导入和 `__all__` 导出列表

关键注意事项：准确声明 `AdapterCapability`，因为路由表的构建完全依赖该声明；`get_*` 方法出错时应返回 `None` 或空列表，不要将异常抛出到 Manager 层。

### Q4：AdapterManager 的缓存何时失效？

AdapterManager 维护两种缓存：

- **Ticker 缓存**（`_ticker_cache`）：记录 Ticker 到适配器的映射。缓存会在以下情况下更新：
  - 首次查询某个 Ticker 时，通过 `validate_ticker()` 确定适配器并写入缓存
  - Failover 成功时，主适配器失败后切换到备选适配器，缓存会更新为成功的适配器
  - 调用 `register_adapter()` 注册新适配器时，`_rebuild_routing_table()` 会重建路由表（但不会主动清除 Ticker 缓存，因为已缓存的映射仍然有效）
- **路由表**（`exchange_routing`）：在每次调用 `register_adapter()` 后通过 `_rebuild_routing_table()` 完全重建

缓存的读写分别使用独立的锁（`_cache_lock` 和 `_lock`）保护，确保线程安全。

### Q5：为什么 BaoStock 获取数据很慢？

BaoStock 性能受限的原因：

- **全局锁机制**：BaoStock 库内部使用全局会话状态（非线程安全），适配器通过 `threading.Lock()` 确保同一时间只有一个 API 调用，导致所有请求串行执行
- **单线程 API**：每次 API 调用都需要先检查登录状态、可能需要重新登录，增加了额外的网络往返
- **网络延迟**：BaoStock 的数据源服务器（baostock.com）部署在国内，从海外服务器访问时延迟较高
- **超时保护**：默认 10 秒超时，超时后会重新登录并重试一次，最坏情况下单次调用可能耗时 20 秒以上

建议：如果对性能有较高要求，优先使用 AKShare 获取 A 股数据，将 BaoStock 作为 Failover 备选。对于历史数据回测等批量场景，可以考虑在非高峰时段预取数据。

---

## 14. 自测检查清单

阅读完本文后，可以使用以下清单检验自己的掌握程度：

- [ ] **能够解释为什么需要数据适配器**：理解统一接口 + Manager 路由的设计动机，以及它解决了重复代码、维护困难和路由复杂三个核心问题
- [ ] **能够列出全部三个适配器及其市场覆盖**：YFinance（全球多市场，加密货币）、AKShare（A 股为主，兼有港股美股）、BaoStock（仅沪深两市）
- [ ] **能够描述 Failover 机制的工作流程**：主适配器失败后，AdapterManager 自动遍历同一交易所的其他适配器，成功后更新缓存
- [ ] **能够解释统一 Ticker 格式及转换逻辑**：内部使用 `EXCHANGE:SYMBOL` 格式，各适配器通过 `convert_to_source_ticker()` 和 `convert_to_internal_ticker()` 进行双向转换
- [ ] **能够描述 app.py lifespan 中的初始化流程**：三个适配器在 FastAPI lifespan 阶段独立初始化，各自用 `try-except` 包裹，互不影响
- [ ] **能够列出 BaseDataAdapter 的抽象方法**：`_initialize()`、`search_assets()`、`get_asset_info()`、`get_real_time_price()`、`get_historical_prices()`、`get_capabilities()`、`convert_to_source_ticker()`、`convert_to_internal_ticker()`
- [ ] **能够解释 BaoStock 的线程安全机制**：全局锁保护并发、登录状态 TTL 管理、`func_timeout` 超时保护、失败自动重登录重试
- [ ] **能够描述并行搜索与去重逻辑**：`ThreadPoolExecutor` 并行查询所有适配器，按 `(symbol, country)` 去重，按交易所优先级选择最佳结果，全部无结果时触发 LLM 回退搜索

---

## 15. 如何扩展新的数据适配器

### 15.1 创建适配器类

在 `python/valuecell/adapters/assets/` 下创建新文件，继承 `BaseDataAdapter`：

```python
# my_adapter.py
from .base import BaseDataAdapter, AdapterCapability
from .types import AssetType, Exchange, DataSource, ...

class MyAdapter(BaseDataAdapter):
    def __init__(self, **kwargs):
        super().__init__(DataSource.MY_SOURCE, **kwargs)

    def _initialize(self) -> None:
        # 初始化适配器配置
        pass

    def get_capabilities(self) -> List[AdapterCapability]:
        return [
            AdapterCapability(
                asset_type=AssetType.STOCK,
                exchanges={Exchange.NASDAQ, Exchange.NYSE},
            ),
        ]

    def search_assets(self, query) -> List[AssetSearchResult]:
        # 实现搜索逻辑
        pass

    def get_asset_info(self, ticker) -> Optional[Asset]:
        # 实现资产信息获取
        pass

    def get_real_time_price(self, ticker) -> Optional[AssetPrice]:
        # 实现实时价格获取
        pass

    def get_historical_prices(self, ticker, start_date, end_date, interval) -> List[AssetPrice]:
        # 实现历史数据获取
        pass

    def convert_to_source_ticker(self, internal_ticker) -> str:
        # 实现内部格式 → 数据源格式的转换
        pass

    def convert_to_internal_ticker(self, source_ticker, default_exchange) -> str:
        # 实现数据源格式 → 内部格式的转换
        pass
```

### 15.2 注册数据源枚举

在 `types.py` 的 `DataSource` 枚举中添加新的数据源：

```python
class DataSource(str, Enum):
    YFINANCE = "yfinance"
    AKSHARE = "akshare"
    BAOSTOCK = "baostock"
    MY_SOURCE = "my_source"  # 新增
```

### 15.3 添加配置方法

在 `AdapterManager` 中添加配置方法：

```python
def configure_my_source(self, **kwargs):
    from .my_adapter import MyAdapter
    adapter = MyAdapter(**kwargs)
    self.register_adapter(adapter)
```

### 15.4 在启动时初始化

在 `python/valuecell/server/api/app.py` 的 lifespan 函数中添加初始化代码：

```python
try:
    manager.configure_my_source()
    logger.info("My source adapter configured")
except Exception as e:
    logger.info(f"My source adapter failed: {e}")
```

### 15.5 更新模块导出

在 `__init__.py` 中添加导入和导出。

### 15.6 关键注意事项

1. **统一 Ticker 格式**：所有适配器必须支持 `EXCHANGE:SYMBOL` 内部格式
2. **capability 声明**：准确声明支持的资产类型和交易所，路由表依赖这个声明
3. **容错设计**：`get_*` 方法在出错时应返回 `None` 或空列表，不要抛出异常到 Manager 层
4. **数据库持久化**：在获取资产信息时，建议调用 `asset_repo.upsert_asset()` 将数据保存到数据库，后续查询可以直接使用缓存
5. **线程安全**：如果底层库不是线程安全的，需要像 BaoStock 那样使用锁机制

---

## 16. 继续阅读

| 你想了解什么 | 推荐阅读 |
|-------------|----------|
| ValueCell 整体架构 | [architecture.md](./architecture.md) |
| 从源码开始深入 | [source-guide.md](./source-guide.md) |
| 开发自己的 Agent | [agent-extension.md](./agent-extension.md) |
| 配置系统详解 | [configuration.md](./configuration.md) |
| 项目概览 | [overview.md](./overview.md) |
