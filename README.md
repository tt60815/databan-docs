# Databan 交易系統完整架構報告

> 最後更新：2026-03-12

---

## 目錄

- [一、系統總覽](#一系統總覽)
- [二、Flask App — 歷史資料每日自動更新](#二flask-app--歷史資料每日自動更新)
  - [2.1 排程工廠架構](#21-排程工廠架構)
  - [2.2 資料庫排程表](#22-資料庫排程表)
  - [2.3 Cron 觸發機制](#23-cron-觸發機制)
  - [2.4 股票資料更新流程](#24-股票資料更新流程)
  - [2.5 排程 API 端點](#25-排程-api-端點)
  - [2.6 背景任務監控](#26-背景任務監控)
  - [2.7 部署配置](#27-部署配置)
- [三、IB Dashboard — 即時技術指標與策略引擎](#三ib-dashboard--即時技術指標與策略引擎)
  - [3.1 系統架構圖](#31-系統架構圖)
  - [3.2 即時 K 線聚合器 (tf_aggregator.py)](#32-即時-k-線聚合器-tf_aggregatorpy)
  - [3.3 即時技術指標計算 (indicators.py)](#33-即時技術指標計算-indicatorspy)
  - [3.4 策略狀態機引擎 (strategy_engine.py)](#34-策略狀態機引擎-strategy_enginepy)
  - [3.5 自動下單管理器 (order_manager.py)](#35-自動下單管理器-order_managerpy)
  - [3.6 主應用程式 (app.py)](#36-主應用程式-apppy)
  - [3.7 Dashboard 前端 (dashboard.html)](#37-dashboard-前端-dashboardhtml)
  - [3.8 部署配置](#38-部署配置)
- [四、兩系統的關聯與資料流](#四兩系統的關聯與資料流)
- [五、完整檔案清單](#五完整檔案清單)

---

## 一、系統總覽

Databan 交易系統由兩個獨立服務組成，部署在不同的 GCP VM 上：

| 系統 | 部署位置 | 功能 | 技術棧 |
|------|----------|------|--------|
| **flask_app** | databan-vm (104.155.207.176) | 歷史資料管理、回測、條件掃描、每日自動更新 | Flask + Gunicorn/gevent + Databento Historical API |
| **ib_dashboard** | ib-gateway-vm (35.236.189.25) | IB Gateway 監控、即時指標、策略引擎、自動交易 | Flask + ib_insync + Databento Live API |

兩者共用同一個 Cloud SQL PostgreSQL 資料庫。

```
                    ┌─────────────────────────────┐
                    │     Cloud SQL (PostgreSQL)   │
                    │     trading database         │
                    │                              │
                    │  - *_1m / *_5m / *_1d 表     │
                    │  - selected_strategies       │
                    │  - scan_tasks                │
                    │  - schedule_configs          │
                    │  - schedule_runs             │
                    │  - ib_trades                 │
                    └──────────┬───────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                                 │
   ┌──────────▼──────────┐          ┌───────────▼──────────┐
   │  databan-vm         │          │  ib-gateway-vm       │
   │  flask_app :8080    │          │  ib_dashboard :8080  │
   │                     │          │                      │
   │  - 歷史資料管理     │          │  - IB Gateway 連線   │
   │  - 每日自動更新     │          │  - 即時 K 線聚合     │
   │  - 條件掃描回測     │          │  - 即時指標計算      │
   │  - 排程工廠         │          │  - 策略狀態機        │
   └─────────────────────┘          │  - 自動下單          │
                                    └──────────────────────┘
```

---

## 二、Flask App — 歷史資料每日自動更新

### 2.1 排程工廠架構

排程系統由三個檔案協作：

```
flask_app/
├── schedule_tasks.py    ← 任務定義 + 執行引擎 + cron 解析
├── data_manager.py      ← DB schema + 資料更新邏輯 (refresh_stock_generator)
└── app.py               ← Flask API 端點 + 背景執行緒
```

**執行流程：**

```
系統 crontab (UTC 22:00, Mon-Fri)
  │
  ▼
curl POST /api/schedules/cron-trigger
  │  Header: X-Cron-Secret: databan-cron-2024
  │
  ▼
Flask API 端點 (app.py:2828-2854)
  │
  ├─ 驗證 X-Cron-Secret
  ├─ 讀取所有 active 排程 (schedule_configs)
  ├─ 逐一檢查 should_run_now(cron_expression, now)
  │
  ▼ (若符合)
execute_task_sync(config)  (schedule_tasks.py)
  │
  ├─ task_type = "stock_update_all"
  │   └─ run_stock_update_all_task({days: 3})
  │       └─ 遍歷所有 symbols → refresh_stock_generator(symbol, days=3)
  │
  ├─ task_type = "stock_update"
  │   └─ run_stock_update_task({symbol: "TSLA", days: 3})
  │       └─ refresh_stock_generator("TSLA", days=3)
  │
  └─ task_type = "combo_backtest"
      └─ run_combo_backtest_task({config_id: 1})

  │
  ▼
更新 schedule_configs (last_run_at, last_status)
建立 schedule_runs 紀錄 (started_at, completed_at, status, error)
```

### 2.2 資料庫排程表

#### `schedule_configs` 表

```sql
CREATE TABLE schedule_configs (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,          -- 排程名稱
    task_type       VARCHAR(50) NOT NULL,           -- 'stock_update' / 'stock_update_all' / 'combo_backtest'
    description     TEXT,
    cron_expression VARCHAR(100),                   -- '0 17 * * 1-5' (ET)
    is_active       BOOLEAN DEFAULT true,
    task_params     JSONB DEFAULT '{}',             -- {"days": 3} / {"symbol": "TSLA"} / {"config_id": 1}
    last_run_at     TIMESTAMP,
    last_status     VARCHAR(20),                    -- 'success' / 'failed'
    last_error      TEXT,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);
```

#### `schedule_runs` 表

```sql
CREATE TABLE schedule_runs (
    id              SERIAL PRIMARY KEY,
    config_id       INTEGER REFERENCES schedule_configs(id) ON DELETE CASCADE,
    started_at      TIMESTAMP DEFAULT NOW(),
    completed_at    TIMESTAMP,
    status          VARCHAR(20) DEFAULT 'running',  -- 'running' / 'success' / 'failed'
    result_summary  TEXT,
    error_message   TEXT
);
```

### 2.3 Cron 觸發機制

#### 設定腳本 `setup_cron.sh`

```bash
#!/bin/bash
# 部署在 databan-vm 上

# 1. 透過 API 建立排程配置
curl -X POST http://localhost:8080/api/schedules \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "每日股票資料更新",
    "task_type": "stock_update_all",
    "cron_expression": "0 17 * * 1-5",
    "is_active": true,
    "task_params": {"days": 3}
  }'

# 2. 加入系統 crontab
# UTC 22:00 = ET 17:00-18:00（收盤後 1-2 小時）
echo "0 22 * * 1-5 curl -s -X POST http://localhost:8080/api/schedules/cron-trigger \
  -H 'X-Cron-Secret: databan-cron-2024' \
  >> /home/user/cron-update.log 2>&1" >> /etc/crontab
```

#### Cron 觸發端點安全驗證

```python
# app.py:2828-2854
@app.route('/api/schedules/cron-trigger', methods=['POST'])
def api_schedule_cron_trigger():
    secret = request.headers.get('X-Cron-Secret', '')
    expected = os.environ.get('CRON_SECRET', 'databan-cron-2024')
    if secret != expected:
        return jsonify({'error': 'unauthorized'}), 403

    configs = get_schedule_configs(active_only=True)
    now = datetime.now()
    executed = []

    for config in configs:
        if should_run_now(config['cron_expression'], now):
            execute_task_sync(config)
            executed.append(config['name'])

    return jsonify({'executed': executed, 'count': len(executed)})
```

#### Cron 解析器

```python
# schedule_tasks.py — 使用 croniter 庫
from croniter import croniter

def should_run_now(cron_expr, now):
    """檢查 cron 表達式是否匹配當前時間（分鐘精度）"""
    cron = croniter(cron_expr, now - timedelta(minutes=1))
    next_time = cron.get_next(datetime)
    return abs((next_time - now).total_seconds()) < 60

def get_next_run_time(cron_expr, now):
    """計算下次執行時間"""
    cron = croniter(cron_expr, now)
    return cron.get_next(datetime)
```

### 2.4 股票資料更新流程

#### `refresh_stock_generator(symbol, days=3)`

7 步驟 generator，每步 yield 進度訊息：

```
Step 1/7: 檢查/建立資料表
  └─ 確認 {symbol}_1m, {symbol}_5m, ..., {symbol}_1d 表存在

Step 2/7: 偵測資料缺口
  └─ 查詢 DB 中最新的 1m 資料時間
  └─ 若缺口 > days → 自動擴大 days 範圍

Step 3/7: 下載 1 分線資料
  └─ Databento Historical API: EQUS.MINI ohlcv-1m
  └─ 日期範圍：過去 N 天 → 現在

Step 4/7: 保留舊資料
  └─ 取出相同時間範圍的舊 OHLCV 數據用於比對

Step 5/7: 寫入新資料 (Upsert)
  └─ INSERT ... ON CONFLICT (ts) DO UPDATE
  └─ 不刪除歷史資料，只更新/新增

Step 6/7: 驗證一致性
  └─ 比對舊 vs 新 OHLCV 值，記錄差異

Step 7/7: 重新計算多週期 + 指標
  └─ 聚合：1m → 3m → 5m → 15m → 30m → 1h → 2h → 3h → 4h → 1d
  └─ 指標：MACD, KD, 趨勢, 交叉信號 (ThreadPoolExecutor, 4 workers)
```

**支援的任務類型：**

| task_type | 功能 | 預設參數 |
|-----------|------|----------|
| `stock_update` | 單一 symbol 增量更新 | `{symbol: "TSLA", days: 3}` |
| `stock_update_all` | 所有 symbol 批次更新 | `{days: 3}` |
| `combo_backtest` | 多時框組合回測 | `{config_id: 1}` |

### 2.5 排程 API 端點

| 方法 | 端點 | 功能 |
|------|------|------|
| GET | `/schedule-factory` | 排程工廠 Web UI |
| GET | `/api/schedules` | 列出所有排程 |
| POST | `/api/schedules` | 建立新排程 |
| GET | `/api/schedules/<id>` | 取得單一排程 |
| PUT | `/api/schedules/<id>` | 更新排程（cron、啟用、參數） |
| DELETE | `/api/schedules/<id>` | 刪除排程 |
| POST | `/api/schedules/<id>/run` | 手動執行（SSE 即時進度） |
| GET | `/api/schedules/<id>/runs` | 執行歷史紀錄 |
| GET | `/api/schedule-task-types` | 可用任務類型列表 |
| POST | `/api/schedules/cron-trigger` | **Cron 觸發入口** |

### 2.6 背景任務監控

```python
# app.py — 心跳檢查器（daemon thread）
def _heartbeat_checker():
    while True:
        time.sleep(300)  # 每 5 分鐘檢查
        # 偵測卡住的任務（超過 5 分鐘無心跳）
        # 標記為 'interrupted'（不自動重啟）

# app.py — 啟動時恢復
def _startup_recovery():
    # 檢查未完成的 scan tasks
    # 標記為 'interrupted'
```

### 2.7 部署配置

#### Dockerfile (flask_app)

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "--bind", "0.0.0.0:8080",
     "--workers", "2",
     "--worker-class", "gevent",    # 支援高並發 I/O
     "--timeout", "300"]            # 5 分鐘超時（長任務）
```

#### docker-compose.yml

```yaml
services:
  web:
    build: ./flask_app
    ports: ["8080:8080"]
    environment:
      - DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
      - DATABENTO_API_KEY
    resources:
      limits:
        cpus: "3.5"
        memory: 14G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 30s

  caddy:              # HTTPS 反向代理
    image: caddy:2
    ports: ["80:80", "443:443"]
```

#### 關鍵依賴

```
flask, gunicorn, gevent          # Web 框架
croniter==2.0.1                  # Cron 表達式解析
databento                        # 股票資料 API
psycopg2-binary                  # PostgreSQL
pandas, numpy, numba             # 資料處理
exchange_calendars               # 交易日曆
```

---

## 三、IB Dashboard — 即時技術指標與策略引擎

### 3.1 系統架構圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ib_dashboard Flask App (:8080)                    │
│                                                                     │
│  ┌─ Thread 1: _ib_thread ─────────────────────────────────────────┐│
│  │  asyncio event loop + nest_asyncio                             ││
│  │  IB Gateway (localhost:4002, paper trading)                    ││
│  │                                                                ││
│  │  迴圈（每 1 秒）：                                             ││
│  │  ├─ ib.connect() / 重連                                       ││
│  │  ├─ order_manager.process_queue()  ← 執行下單                 ││
│  │  └─ ib.sleep(1)                   ← 處理 IB 事件             ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌─ Thread 2: _databento_thread ──────────────────────────────────┐│
│  │  Databento Live: EQUS.MINI / trades schema                    ││
│  │  → _live_prices = {TSLA: {price, size, timestamp}, ...}       ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌─ Thread 3: _databento_bars_thread ─────────────────────────────┐│
│  │  Databento Live: EQUS.MINI / ohlcv-1m schema                 ││
│  │  → _live_bars = {TSLA: deque([bar, ...], maxlen=60)}          ││
│  │  → strategy_engine.on_bar(sym, bar)  ← 餵給策略引擎          ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌─ Thread 4: _strategy_engine_init ──────────────────────────────┐│
│  │  載入 active 策略 (selected_strategies 表)                    ││
│  │  Databento Historical API 暖機（當天 9:30 起的 1m bars）      ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌─ Core Objects ─────────────────────────────────────────────────┐│
│  │                                                                ││
│  │  strategy_engine = StrategyEngine(DB_CONFIG)                  ││
│  │    ├─ aggregator = TimeframeAggregator()                      ││
│  │    ├─ _indicator_cache = {}                                   ││
│  │    └─ strategies = [StrategyInstance, ...]                     ││
│  │                                                                ││
│  │  order_manager = OrderManager(DB_CONFIG, shares=10, max=5)    ││
│  │    ├─ _queue = Queue()                                        ││
│  │    ├─ _positions = {strategy_id: ManagedPosition}             ││
│  │    └─ _inflight = set()                                       ││
│  │                                                                ││
│  └────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 即時 K 線聚合器 (tf_aggregator.py)

**檔案：** `ib_dashboard/tf_aggregator.py`

**功能：** 將 Databento 即時 1m K 棒聚合為多時框 K 棒

#### 支援的時框

```python
TF_MINUTES = {
    '1m': 1,   '3m': 3,   '5m': 5,   '15m': 15,  '30m': 30,
    '1h': 60,  '2h': 120, '3h': 180, '4h': 240,
}
```

#### 聚合邏輯

```python
class TimeframeAggregator:
    def on_bar(self, symbol: str, bar_1m: dict) -> dict:
        """
        收到 1m bar 時呼叫
        回傳：{tf: completed_bar} 已完成的高 TF K 棒
        """
```

**核心演算法：**

```
收到 1m bar (例：10:03 TSLA)
  │
  ├─ 解析時間，轉換到 NY 時區
  ├─ 計算 mins_since_open = (bar_time - 9:30) in minutes
  ├─ 過濾盤前/盤後（只處理 9:30-16:00）
  │
  └─ 對每個配置的 TF（3m, 5m, 15m ...）：
      │
      ├─ bar_index = mins_since_open // tf_minutes
      │   例：10:03 → mins=33 → 5m bar_index=6 (第 7 根 5m K 棒)
      │
      ├─ 若 bar_index 改變（新 TF K 棒開始）：
      │   └─ 將上一根 TF K 棒標記為「完成」→ 存入 bars deque
      │
      └─ 更新當前 TF K 棒：
          ├─ open = 第一根 1m 的 open
          ├─ high = max(所有 1m 的 high)
          ├─ low  = min(所有 1m 的 low)
          ├─ close = 最後一根 1m 的 close
          └─ volume = sum(所有 1m 的 volume)
```

**記憶體管理：**

```python
_data = {
    'TSLA': {
        '5m': {
            'current': {ts, open, high, low, close, volume},  # 正在構建中
            'current_index': 6,
            'current_date': date(2026, 3, 12),
            'bars': deque([...], maxlen=200)  # 已完成的 K 棒（最多 200 根）
        },
        '15m': { ... },
        '1h': { ... },
    }
}
```

**市場時段對齊：**
- 所有 TF 從 9:30 ET 開盤時間對齊（不是從午夜算）
- 盤前/盤後的 bar 會被過濾掉
- 跨日時自動重置 current_date

### 3.3 即時技術指標計算 (indicators.py)

**檔案：** `ib_dashboard/indicators.py`

**功能：** 對聚合後的 K 棒計算技術指標（純 pandas/numpy，輕量版本）

#### 核心函式

```python
BUFFER_BARS = 30  # 最少需要 30 根 K 棒（EMA 暖機）

def calculate_indicators(bars: list) -> list:
    """
    輸入：[{ts, open, high, low, close, volume}, ...]
    輸出：同樣的 list，每個 bar 新增指標欄位
    """
```

#### 計算的指標完整列表

| 分類 | 指標欄位 | 計算方式 |
|------|----------|----------|
| **MACD** | `dif` | EMA(close, 12) - EMA(close, 26) |
| | `dea` | EMA(dif, 9) |
| | `macd_hist` | (dif - dea) * 2 |
| **KDJ** | `k` | RSV 的 3 期 SMA |
| | `d` | K 的 3 期 SMA |
| | `j` | 3K - 2D |
| | *RSV* | (close - lowest_low_9) / (highest_high_9 - lowest_low_9) * 100 |
| **Heikin-Ashi** | `ha_open` | (prev_ha_open + prev_ha_close) / 2 |
| | `ha_close` | (open + high + low + close) / 4 |
| | `ha_high` | max(high, ha_open, ha_close) |
| | `ha_low` | min(low, ha_open, ha_close) |
| **趨勢** | `dif_trend` | DIF 連續上升/下降計數 (+N/-N) |
| | `k_trend` | K 值連續上升/下降計數 |
| | `d_trend` | D 值連續上升/下降計數 |
| | `j_trend` | J 值連續上升/下降計數 |
| | `dea_trend` | DEA 連續上升/下降計數 |
| | `macd_hist_trend` | MACD 柱連續上升/下降計數 |
| **交叉信號** | `dif_cross` | +1=金叉（DIF 上穿 DEA）/ -1=死叉 / 0=無 |
| | `kd_cross` | +1=金叉（K 上穿 D）/ -1=死叉 / 0=無 |
| **位置判斷** | `k_above_50` | K > 50 → 1, else 0 |
| | `d_above_50` | D > 50 → 1, else 0 |
| | `k_above_d` | K > D → 1, else 0 |
| | `dif_above_dea` | DIF > DEA → 1, else 0 |
| | `dif_above_zero` | DIF > 0 → 1, else 0 |
| | `dea_above_zero` | DEA > 0 → 1, else 0 |
| **交叉後計數** | `dif_up_since_cross` | DIF 金叉後持續上升幾根 |
| | `dif_down_since_cross` | DIF 死叉後持續下降幾根 |
| | `k_up_since_cross` | K 金叉後持續上升幾根 |
| | `k_down_since_cross` | K 死叉後持續下降幾根 |
| | `d_up/down_since_cross` | D 同上 |
| | `j_up/down_since_cross` | J 同上 |
| | `dea_up/down_since_cross` | DEA 同上 |
| **均高/均低** | `avg_high_buy` | high > prev_ha_high → 1（突破上一根 HA 高點） |
| | `avg_low_sell` | low < prev_ha_low → 1（跌破上一根 HA 低點） |

#### 指標快取機制

```python
# strategy_engine.py 中的快取邏輯
_indicator_cache = {}        # {(symbol, tf): list[dict]}
_indicator_bar_count = {}    # {(symbol, tf): int}

# 只在 bars 數量變化時重新計算（避免重複計算）
cache_key = (symbol_upper, tf)
current_count = len(bars)
if self._indicator_bar_count.get(cache_key) != current_count:
    indicators = calculate_indicators(bars)
    self._indicator_cache[cache_key] = indicators
    self._indicator_bar_count[cache_key] = current_count
```

### 3.4 策略狀態機引擎 (strategy_engine.py)

**檔案：** `ib_dashboard/strategy_engine.py`

#### 狀態機定義

```
┌────────┐   Entry 條件     ┌──────────┐   Target 確認    ┌──────────┐
│  IDLE  │ ──────────────→  │ WATCHING │ ─────────────→   │ HOLDING  │
│ 等待中 │                  │  觀察中  │                   │  持倉中  │
└────────┘                  └──────────┘                   └──────────┘
    ▲                           │                              │
    │     Close 條件觸發        │                              │
    └───────────────────────────┘                              │
    │                                                          │
    │          3 種出場觸發                                     │
    └──────────────────────────────────────────────────────────┘
              1. 止損/止盈
              2. 均高買/均低賣
              3. 條件出場
```

#### StrategyInstance 資料結構

```python
@dataclass
class StrategyInstance:
    # 基本資訊
    strategy_id: int          # selected_strategies.id
    symbol: str               # 'TSLA'
    entry_tf: str             # '5m' — 進場判斷用的時框
    observe_tf: str           # '15m' — 觀察確認用的時框
    direction: str            # 'long' / 'short'
    exit_mode: str            # 'exit_condition' / 'prev_high' / ...

    # 條件配置
    entry_conditions: list    # 進場條件 [{field, operator, value}, ...]
    watch_conditions: list    # 觀察條件 [{timeframe, conditions: [...]}, ...]
    target_conditions: list   # 目標確認 [{field, operator, value}, ...]
    close_conditions: list    # 取消條件 [{timeframe, conditions: [...]}, ...]
    exit_conditions: list     # 出場條件 [{timeframe, conditions: [...]}, ...]

    # 止損/止盈（預設 5%/5%）
    price_exit: dict = {
        'stop_loss':    {'enabled': True, 'percent': 5},
        'take_profit':  {'enabled': True, 'percent': 5},
        'priority':     'stop_loss_first'
    }

    # 均高買/均低賣
    avg_hl_mode: dict = {'enabled': False}

    # 執行狀態
    state: StrategyState      # IDLE / WATCHING / HOLDING
    signal_time: datetime     # 最近信號時間
    signal_price: float       # 最近信號價格
    entry_time: datetime      # 進場時間
    entry_price: float        # 進場價格
    signals: list             # 信號歷史（最多 50 筆）
```

#### 完整評估流程

```python
def on_bar(self, symbol: str, bar_1m: dict):
    """每收到一根 1m bar 觸發一次"""

    # 1. 每 5 分鐘自動重載策略
    if time.time() - self._last_reload > 300:
        self.reload_strategies()

    # 2. 聚合到高 TF
    self.aggregator.on_bar(symbol, bar_1m)

    # 3. 取該 symbol 的策略
    symbol_strategies = [s for s in self.strategies if s.symbol == symbol]

    # 4. 計算需要的 TF 指標
    for tf in needed_tfs:
        bars = self.aggregator.get_all_bars(symbol, tf)
        if len(bars) >= 30:  # BUFFER_BARS
            indicators = calculate_indicators(bars)

    # 5. 對每個策略評估狀態機
    for strat in symbol_strategies:
        self._evaluate(strat, entry_row, observe_row)
```

#### HOLDING 狀態三優先級出場

```python
def _evaluate(self, strat, entry_row, observe_row):
    # ...
    elif strat.state == StrategyState.HOLDING:

        # ═══ 優先級 1：止損/止盈 ═══
        # Long 止損：current_low <= entry_price * (1 - 5/100)
        # Long 止盈：current_high >= entry_price * (1 + 5/100)
        # Short 止損：current_high >= entry_price * (1 + 5/100)
        # Short 止盈：current_low <= entry_price * (1 - 5/100)
        #
        # priority = "stop_loss_first" → 先檢查止損
        # priority = "take_profit_first" → 先檢查止盈

        # ═══ 優先級 2：均高/均低出場 ═══
        # Long:  avg_low_sell == 1 → 出場
        # Short: avg_high_buy == 1 → 出場

        # ═══ 優先級 3：條件出場 ═══
        # 遍歷 exit_conditions → check_conditions() → 滿足則出場
```

#### 條件檢查函式

```python
def check_condition(row_data: dict, cond: dict) -> bool:
    """
    支援的 operator: >, >=, <, <=
    支援 field-to-field 比較（compare_field）
    過濾 NaN 和 None 值
    """

def check_conditions(row_data: dict, conditions: list) -> bool:
    """多條件 AND：全部滿足才回傳 True"""
```

#### 策略載入與狀態保留

```python
def reload_strategies(self):
    """從 DB 載入 active 策略，保留舊策略的執行狀態"""

    # SELECT ... FROM selected_strategies WHERE status = 'active'

    # 保留舊策略的狀態（跨 reload 不中斷）
    for strat in new_strategies:
        old = old_state.get(strat.strategy_id)
        if old:
            strat.state = old.state
            strat.entry_price = old.entry_price
            strat.signals = old.signals
            # ...
```

#### 信號紀錄

| 信號類型 | 意義 | 狀態轉換 |
|----------|------|----------|
| `ENTRY_SIGNAL` | 進場條件觸發 | IDLE → WATCHING |
| `CLOSE_CANCEL` | 觀察期間取消 | WATCHING → IDLE |
| `TARGET_ENTRY` | 目標確認，實際進場 | WATCHING → HOLDING |
| `EXIT` | 出場（含原因和 PnL） | HOLDING → IDLE |

### 3.5 自動下單管理器 (order_manager.py)

**檔案：** `ib_dashboard/order_manager.py`

#### 跨線程通訊模型

```
Databento Bars Thread              IB Thread
(strategy_engine)                  (_ib_thread)
       │                                │
       │ submit_order(OrderRequest)      │
       │        │                        │
       │   ┌────▼────┐                  │
       │   │  Lock   │ ← 防重複         │
       │   └────┬────┘                  │
       │        │                        │
       │   ┌────▼────────────────┐      │
       │   │   queue.Queue       │──────▶ process_queue()
       │   │   (thread-safe)     │      │      │
       │   └─────────────────────┘      │  ┌───▼───────┐
       │                                │  │ _execute_  │
       │                                │  │  order()   │
       │                                │  └───┬───────┘
       │                                │      │
       │                                │  ┌───▼───────┐
       │                                │  │ ib.place   │
       │                                │  │  Order()   │
       │                                │  └───┬───────┘
       │                                │      │
       │                                │  ┌───▼───────┐
       │                                │  │ _on_fill() │
       │                                │  │ callback   │
       │                                │  └───┬───────┘
       │                                │      │
       │                                │      ▼
       │                                │  DB: ib_trades
```

#### 安全機制一覽

| 機制 | 實作 | 說明 |
|------|------|------|
| 模擬倉限定 | IB Gateway | `TRADING_MODE=paper`, port 4002 |
| 小量下單 | `default_shares=10` | ~$2,500/筆 (TSLA) |
| 最大持倉 | `max_total_positions=5` | 最多 5 個策略同時持倉 |
| 重複防護 | `_inflight` set | 同一策略不重複下單 |
| 已持倉防護 | `_positions` dict | 同策略不重複進場 |
| 止損保護 | `price_exit` | 預設 5% stop-loss |
| 全域開關 | `config['enabled']` | API 可即時停用交易 |
| IB 斷線 | Queue 保留 | 未連線時放回 Queue 等重連 |
| 訂單取消 | `cancelledEvent` | 清除 inflight 狀態 |

#### DB 表：ib_trades

```sql
CREATE TABLE ib_trades (
    id              SERIAL PRIMARY KEY,
    strategy_id     INTEGER,            -- selected_strategies.id
    symbol          VARCHAR(20) NOT NULL,
    direction       VARCHAR(10) NOT NULL,  -- 'long' / 'short'
    quantity        INTEGER NOT NULL,
    entry_price     FLOAT NOT NULL,     -- IB fill 成交價
    entry_time      TIMESTAMP NOT NULL,
    exit_price      FLOAT,              -- NULL = 持倉中
    exit_time       TIMESTAMP,
    exit_reason     VARCHAR(50),        -- 'stop_loss' / 'take_profit' / 'exit_condition' / 'avg_low_sell' / ...
    pnl_pct         FLOAT,             -- (exit-entry)/entry * 100
    pnl_dollar      FLOAT,             -- (exit-entry) * quantity
    entry_order_id  INTEGER,           -- IB orderId
    exit_order_id   INTEGER,
    created_at      TIMESTAMP DEFAULT NOW()
);
```

### 3.6 主應用程式 (app.py)

**檔案：** `ib_dashboard/app.py` (877 行)

#### 啟動順序

```
模組載入
  │
  ├─ 1. _init_trade_tables()
  │     → CREATE TABLE IF NOT EXISTS ib_trades
  │
  ├─ 2. order_manager = OrderManager(DB_CONFIG, shares=10, max=5)
  │
  ├─ 3. strategy_engine = StrategyEngine(DB_CONFIG)
  │     strategy_engine.set_order_manager(order_manager)
  │
  ├─ 4. Thread: _ib_thread()
  │     → IB 連線 → order_manager.set_ib(ib)
  │     → 每 1 秒：process_queue() + ib.sleep(1)
  │
  ├─ 5. Thread: _databento_thread()
  │     → Databento Live trades 串流 → _live_prices
  │
  ├─ 6. Thread: _databento_bars_thread()
  │     → Databento Live ohlcv-1m → _live_bars
  │     → strategy_engine.on_bar(sym, bar)
  │
  └─ 7. Thread: _strategy_engine_init()
        → reload_strategies()
        → _warmup_with_historical() (當天已有的 1m bars)
```

#### API 端點完整列表

| 分類 | 方法 | 端點 | 功能 |
|------|------|------|------|
| **健康** | GET | `/health` | 基本健康檢查 |
| **IB** | GET | `/api/ib/status` | IB 連線狀態 |
| | GET | `/api/ib/account` | 帳戶摘要（淨值、購買力等） |
| | GET | `/api/ib/positions` | IB 實際持倉 |
| | GET | `/api/ib/orders` | 活躍訂單 |
| | GET | `/api/ib/trades` | 今日成交 |
| **即時數據** | GET | `/api/live/prices` | 即時 tick 價格 |
| | GET | `/api/live/bars` | 即時 1m K 棒 (`?symbol=TSLA&limit=60`) |
| | GET | `/api/live/indicators` | 即時指標值 (`?symbol=TSLA&tf=5m`) |
| **資料庫** | GET | `/api/db/symbols` | DB symbol 列表 |
| | GET | `/api/db/selected-strategies` | 已選策略 |
| | GET | `/api/db/scan-tasks` | 掃描任務 |
| **策略引擎** | GET | `/api/strategy/status` | 策略狀態（IDLE/WATCHING/HOLDING） |
| | GET | `/api/strategy/signals` | 最近信號紀錄 |
| | POST | `/api/strategy/reload` | 手動重載策略 |
| **自動交易** | GET | `/api/trading/positions` | 策略持倉（含即時 PnL） |
| | GET | `/api/trading/trades` | 已完成交易紀錄 |
| | GET | `/api/trading/orders` | inflight 訂單 + Queue |
| | GET/POST | `/api/trading/config` | 交易設定（啟用、股數、上限） |

### 3.7 Dashboard 前端 (dashboard.html)

**檔案：** `ib_dashboard/templates/dashboard.html` (722 行)

#### 面板排列順序

```
┌─────────────────────────────────────────┐
│  IB Gateway 監控 Dashboard   ●          │  ← 連線燈號
├─────────────────────────────────────────┤
│  即時策略狀態  ● 運行中 (3 策略) [重載]  │  ← 策略狀態表格
│  ID | Symbol | 方向 | TF | 狀態 | 價格  │
├─────────────────────────────────────────┤
│  最近信號紀錄                            │  ← 信號時間線
│  時間 | Symbol | 類型 | 方向 | PnL       │
├─────────────────────────────────────────┤
│  即時策略持倉                    ★ 新增  │  ← 持倉 + 即時 PnL
│  策略ID | Symbol | 進場價 | 現價 | PnL%  │
├─────────────────────────────────────────┤
│  策略交易紀錄                    ★ 新增  │  ← 歷史交易 + 出場原因
│  策略ID | 進/出場價 | PnL% | PnL$ | 原因 │
├─────────────────────────────────────────┤
│  連線狀態     │  帳戶摘要                │
│  ● 已連線     │  淨值 / 現金 / 購買力    │
├──────────────┼──────────────────────────┤
│  當前持倉     │  活躍訂單                │
│  (IB 實際)    │  (IB 掛單)               │
├─────────────────────────────────────────┤
│  今日成交記錄                            │
├─────────────────────────────────────────┤
│  即時價格  ● 串流中                      │
│  [TSLA $250.00] [AAPL $180.00] ...      │
├─────────────────────────────────────────┤
│  資料庫 Symbols │  最近掃描任務           │
└─────────────────────────────────────────┘
  最後更新：15:30:05         ← 每 5 秒自動刷新
```

#### 自動刷新

```javascript
// 12 個 API 同時並行請求
async function refreshAll() {
  await Promise.all([
    refreshStrategyStatus(),      // /api/strategy/status
    refreshSignalLog(),           // /api/strategy/signals
    refreshTradingPositions(),    // /api/trading/positions    ★ 新增
    refreshTradingTrades(),       // /api/trading/trades       ★ 新增
    refreshStatus(),              // /api/ib/status
    refreshAccount(),             // /api/ib/account
    refreshPositions(),           // /api/ib/positions
    refreshOrders(),              // /api/ib/orders
    refreshTrades(),              // /api/ib/trades
    refreshLivePrices(),          // /api/live/prices
    refreshSymbols(),             // /api/db/symbols
    refreshScanTasks(),           // /api/db/scan-tasks
  ]);
}

setInterval(refreshAll, 5000);  // 每 5 秒刷新
```

### 3.8 部署配置

#### Dockerfile (ib_dashboard)

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y libpq-dev
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "--bind", "0.0.0.0:8080",
     "--workers", "1",          # ib_insync 要求單 worker
     "--timeout", "120",
     "app:app"]
```

#### requirements.txt

```
flask
ib_insync               # IB Gateway API
psycopg2-binary          # PostgreSQL
gunicorn
databento                # 即時市場數據
numpy
pandas
pytz                     # 時區處理
nest_asyncio             # asyncio 巢狀 event loop
```

#### 環境變數

```bash
# 資料庫
DB_HOST=<cloud-sql-private-ip>
DB_PORT=5432
DB_NAME=trading
DB_USER=postgres
DB_PASSWORD=<password>

# IB Gateway
IB_HOST=127.0.0.1
IB_PORT=4002             # paper trading port
IB_CLIENT_ID=10

# 市場數據
DATABENTO_API_KEY=<key>
```

---

## 四、兩系統的關聯與資料流

```
                    ┌───────────────────────────────────┐
                    │        Cloud SQL PostgreSQL        │
                    │                                   │
                    │  ┌─────────────────────────────┐  │
                    │  │    *_1m / *_5m / *_1d 表     │  │
                    │  │    (歷史 K 線 + 指標)        │  │
                    │  │                              │  │
                    │  │  flask_app 寫入（每日更新）   │  │
                    │  │  ib_dashboard 不直接讀取      │  │
                    │  └─────────────────────────────┘  │
                    │                                   │
                    │  ┌─────────────────────────────┐  │
                    │  │    selected_strategies 表    │  │
                    │  │    (策略配置)                │  │
                    │  │                              │  │
                    │  │  flask_app 寫入（使用者選擇）│  │
                    │  │  ib_dashboard 讀取（載入策略）│  │
                    │  └─────────────────────────────┘  │
                    │                                   │
                    │  ┌─────────────────────────────┐  │
                    │  │    ib_trades 表              │  │
                    │  │    (自動交易紀錄)            │  │
                    │  │                              │  │
                    │  │  ib_dashboard 寫入            │  │
                    │  └─────────────────────────────┘  │
                    │                                   │
                    │  ┌─────────────────────────────┐  │
                    │  │  schedule_configs/runs 表    │  │
                    │  │  (排程配置與執行紀錄)        │  │
                    │  │                              │  │
                    │  │  flask_app 讀寫               │  │
                    │  └─────────────────────────────┘  │
                    └───────────────────────────────────┘

    flask_app (databan-vm)              ib_dashboard (ib-gateway-vm)
    ┌───────────────────┐               ┌───────────────────┐
    │                   │               │                   │
    │  每日收盤後       │               │  盤中即時          │
    │  UTC 22:00        │               │  9:30-16:00 ET    │
    │                   │               │                   │
    │  ┌─────────────┐  │               │  ┌─────────────┐  │
    │  │ Databento   │  │               │  │ Databento   │  │
    │  │ Historical  │  │               │  │ Live        │  │
    │  │ API         │  │               │  │ API         │  │
    │  │             │  │               │  │             │  │
    │  │ 下載過去    │  │               │  │ 即時 1m bar │  │
    │  │ N 天 1m     │  │               │  │ 串流        │  │
    │  │ bars        │  │               │  └──────┬──────┘  │
    │  └──────┬──────┘  │               │         │         │
    │         │         │               │   ┌─────▼──────┐  │
    │   ┌─────▼──────┐  │               │   │ TF 聚合器  │  │
    │   │ Upsert DB  │  │               │   │ 1m→5m→15m  │  │
    │   │ 1m bars    │  │               │   └─────┬──────┘  │
    │   └─────┬──────┘  │               │         │         │
    │         │         │               │   ┌─────▼──────┐  │
    │   ┌─────▼──────┐  │               │   │ 指標計算   │  │
    │   │ 聚合多 TF  │  │               │   │ MACD/KDJ   │  │
    │   │ 3m~1d      │  │               │   └─────┬──────┘  │
    │   └─────┬──────┘  │               │         │         │
    │         │         │               │   ┌─────▼──────┐  │
    │   ┌─────▼──────┐  │               │   │ 策略評估   │  │
    │   │ 重算指標   │  │               │   │ 狀態機     │  │
    │   │ MACD/KDJ   │  │               │   └─────┬──────┘  │
    │   │ (4 workers)│  │               │         │         │
    │   └────────────┘  │               │   ┌─────▼──────┐  │
    │                   │               │   │ IB 下單    │  │
    │                   │               │   │ 模擬倉     │  │
    │                   │               │   └────────────┘  │
    └───────────────────┘               └───────────────────┘
```

### 資料流時間線

```
   收盤後                                    下一個交易日
   16:00 ET                                  9:30 ET
     │                                         │
     ▼                                         ▼
17:00 ET (UTC 22:00)                      ┌─ Databento Live 訂閱
     │                                    │   ohlcv-1m 串流
     ├─ cron 觸發                         │
     │   /api/schedules/cron-trigger      ├─ _warmup_with_historical()
     │                                    │   灌入 9:30 到現在的 1m bars
     ├─ stock_update_all                  │
     │   下載過去 3 天 1m bars            ├─ strategy_engine 開始評估
     │   upsert 到 DB                     │
     │   重算 3m~1d 聚合                  ├─ 每根 1m bar：
     │   重算指標                         │   ├─ 聚合到 5m/15m/1h
     │                                    │   ├─ 計算指標
     ├─ 完成，寫入 schedule_runs          │   ├─ 評估策略狀態機
     │                                    │   └─ 觸發 → 下單
     ▼                                    │
  DB 資料已更新                           └─ 16:00 ET 收盤
  等待下一個交易日
```

---

## 五、完整檔案清單

### flask_app/

| 檔案 | 行數 | 功能 |
|------|------|------|
| `app.py` | ~2900 | Flask 主應用 + API 端點 + 背景執行緒 |
| `data_manager.py` | ~3500 | 資料管理 + DB 操作 + 排程表 CRUD |
| `schedule_tasks.py` | ~380 | 任務定義 + 執行引擎 + cron 解析 |
| `multi_tf_engine.py` | ~500 | 多時框回測引擎 |
| `backtest.py` | ~500 | 單策略回測（止損/止盈/均高低邏輯） |
| `Dockerfile` | ~15 | Docker 容器配置 |
| `requirements.txt` | ~20 | Python 依賴（含 croniter） |
| `templates/` | 多檔 | Jinja2 前端模板 |

### ib_dashboard/

| 檔案 | 行數 | 功能 |
|------|------|------|
| `app.py` | 877 | Flask 主應用 + 4 個背景執行緒 + API 端點 |
| `strategy_engine.py` | 628 | 策略狀態機 + 條件評估 + 下單整合 |
| `order_manager.py` | 327 | 跨線程下單管理 + IB 執行 + DB 紀錄 |
| `tf_aggregator.py` | ~200 | 即時 1m→多 TF K 棒聚合 |
| `indicators.py` | ~300 | 即時技術指標計算（MACD/KDJ/HA/交叉） |
| `setup_cron.sh` | ~50 | Cron 排程設定腳本 |
| `Dockerfile` | ~15 | Docker 容器配置 |
| `requirements.txt` | ~10 | Python 依賴 |
| `templates/dashboard.html` | 722 | 單頁 Dashboard 前端 |

### 共用

| 檔案 | 功能 |
|------|------|
| `docker-compose.yml` | flask_app + Caddy 部署 |
| `setup_stock.py` | 初始股票資料匯入腳本 |
