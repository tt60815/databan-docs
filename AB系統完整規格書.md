# Databan AB 系統完整規格書

> 版本：v3.0（2026-03-21）
> 架構：DB 為唯一真相來源

---

## 一、系統總覽

```
系統 A（回測分析）                    系統 B（即時交易）
databan-vm                           ib-gateway-vm
104.155.207.176:8080                 35.236.189.25:8080
                    ↕
            Cloud SQL PostgreSQL
            {symbol}_{tf} 表 ← 唯一真相來源
```

| 系統 | 職責 | 技術 |
|------|------|------|
| A | 歷史資料下載、多週期聚合、指標計算、條件掃描、回測、策略篩選、格子止盈分析 | Flask + Gunicorn + PostgreSQL |
| B | 即時行情接收、指標即時計算寫回 DB、策略評估、自動下單、日內平倉 | Flask + Gunicorn + IB Gateway + Databento Live |

---

## 二、資料庫（唯一真相來源）

### 2.1 核心表：`{symbol}_{tf}`

每檔股票每個週期一張表（如 `tsla_2m`、`aapl_4m`）。系統 A 寫歷史，系統 B 寫即時，格式完全一致。

**63 個欄位：**

```
主鍵與價格：
  ts (TIMESTAMPTZ PK), ts_ny (TIMESTAMP)
  open, high, low, close (DECIMAL 12,4), volume (BIGINT)

MACD：
  macd, macd_signal, macd_hist (DECIMAL 12,4)

KDJ：
  k, d, j (DECIMAL 8,4)

Heikin-Ashi：
  ha_open, ha_high, ha_low, ha_close (DECIMAL 12,4)
  prev_ha_high, prev_ha_low (DOUBLE PRECISION)

趨勢計數：
  dif_trend, dea_trend, k_trend, d_trend, j_trend, macd_hist_trend (INTEGER)

變化量：
  dif_change, dea_change, k_change, d_change, j_change, macd_hist_change (DECIMAL)

交叉信號：
  dif_cross, kd_cross (SMALLINT: 1=金叉, -1=死叉, 0=無)

交叉後計數：
  dif/k/d/j/dea 各有 up_since_cross, down_since_cross (INTEGER)

位置判斷：
  k_above_50, d_above_50, k_above_d (SMALLINT: 1/-1)
  dif_above_dea, dif_above_zero, dea_above_zero (SMALLINT: 1/-1)

均線訊號：
  avg_high_buy, avg_low_sell (SMALLINT: 1=觸發, 0=未觸發)

純升降：
  dif_no_down/up_since_cross, dea_no_down/up_since_cross (SMALLINT)

J 交叉位置：
  j_down_1st, j_down_2nd, j_up_1st, j_up_2nd (SMALLINT)

MACD 柱拐點：
  macd_hist_turning, macd_hist_turning_dir (SMALLINT)
  macd_hist_turning_count, macd_hist_consecutive (INTEGER)

Pattern：
  k/d/j/dif/dea/macd_hist_pattern_since_cross (VARCHAR 500)
```

### 2.2 策略表：`selected_strategies`

| 欄位 | 說明 |
|------|------|
| id | 主鍵 |
| best_result_id | 對應 scan_best_results |
| task_id | 掃描任務 ID |
| symbol, entry_tf, observe_tf | 股票 + 分線配對 |
| direction | long / short |
| intraday_mode | intraday / non_intraday |
| exit_mode | normal / range / prev_high / prev_hl |
| conditions (JSONB) | `[{"key":"entry_xxx","value":1}]` |
| condition_label | 顯示用文字 |
| optimal_grid_level | 最佳止盈格數 (0=不用) |
| trades_count, win_rate, ev, weighted_ev, total_pnl_pct, profit_factor | 回測統計 |
| status | active / inactive |

### 2.3 交易記錄表：`ib_trades`

| 欄位 | 說明 |
|------|------|
| strategy_id | 對應 selected_strategies.id |
| symbol, direction, quantity | 交易基本資訊 |
| entry_price, entry_time, entry_order_id | 進場 |
| exit_price, exit_time, exit_order_id, exit_reason | 出場 |
| pnl_pct, pnl_dollar | 盈虧 |

### 2.4 其他表

| 表 | 用途 | 誰寫 |
|---|------|------|
| `scan_tasks` | 掃描任務進度 | 系統 A |
| `scan_results` | 回測交易列表 (JSONB) | 系統 A |
| `scan_best_results` | 最佳條件組合排名 | 系統 A |
| `strategy_packs` | 策略包 | 系統 A |
| `strategy_pack_items` | 策略包內容 | 系統 A |
| `live_bars_1m` | 即時 1m bar 備份 | 系統 B |

### 2.5 欄位名映射

| DB 欄位 | 系統 B indicators.py | 說明 |
|---------|---------------------|------|
| `macd` | `dif` | DIF 線 |
| `macd_signal` | `dea` | DEA 線 |
| 其餘 65 個 | 完全相同 | — |

---

## 三、系統 A 流程

### 3.1 資料下載

```
Databento Historical API (EQUS.MINI, ohlcv-1m)
    ↓ 過濾 9:30~16:00 ET
    ↓
{symbol}_1m 表 (INSERT ON CONFLICT UPDATE)
    ↓
聚合 → {symbol}_3m, 5m, ..., 1d (SQL aggregation)
    ↓
計算 67 指標 (Python EMA/KDJ/HA) → UPDATE 表
```

### 3.2 條件掃描

```
輸入：股票、進場條件(KD金叉)、出場條件(KD死叉)、分線比例(1:2)
    ↓
背景線程 × 6 workers：
  對每個分線配對 → 產生交易列表 → scan_results
  對每組觀察條件 → 統計勝率/EV → scan_best_results
    ↓
排名 → 使用者勾選 → selected_strategies
```

### 3.3 格子止盈分析

```
API: /api/trade-analyzer/grid-analysis
    ↓
對 1~20 格模擬出場：
  range = prev_ha_high - prev_ha_low (prev_hl)
       或 prev_ha_high × 2 - entry_price (prev_high)
  每格 = range / 20
  如果 max_profit_pct >= N × 每格 → 算賺 N 格
    ↓
找最佳 N (最高 EV) → 寫入 optimal_grid_level
```

### 3.4 頁面

| URL | 功能 |
|-----|------|
| `/` | → 重定向 `/scan-results` |
| `/trade-analyzer` | 掃描設定 + 啟動 |
| `/scan-results` | 結果排名 + 勾選 |
| `/strategy-packs` | 策略包管理 + 部署 |
| `/data` | 股票資料管理 |
| `/schedule-factory` | 排程系統 |

---

## 四、系統 B 流程

### 4.1 啟動順序

```
1. 建表 ib_trades, live_bars_1m
2. 初始化 OrderManager (Queue, Lock, 10股/筆, 上限5筆)
3. 初始化 StrategyEngine
4. 啟動 4 個 Daemon 線程
```

### 4.2 四個線程

```
Thread 1: IB Gateway
├── 連線 localhost:4002 (Paper Trading)
├── 每秒處理下單 Queue
├── 15:55 ET 強制平倉 intraday HOLDING
└── 9:30 ET 前重置 EOD flag

Thread 2: Databento 即時價格
└── trades schema → _live_prices[symbol]

Thread 3: Databento 1m K 棒（核心交易線程）
├── ohlcv-1m schema → 每分鐘一根完成 bar
├── _save_bar_to_db() → live_bars_1m 備份
└── strategy_engine.on_bar() → 交易決策

Thread 4: 策略初始化（一次性）
├── 載入 selected_strategies
├── 聚合器暖機（從 live_bars_1m 讀 1m bar）
└── 重置所有策略 → IDLE
```

### 4.3 即時交易決策流程

```
1m bar 到達（Databento Live）
    ↓
聚合器：2m bar 完成了嗎？
    ├── 沒有 → return
    └── 完成 →
        ↓
取 completed bars → calculate_indicators() [~41ms]
        ↓
寫入 DB {symbol}_{tf} 表 [~10ms]
  ├── INSERT OHLCV (ON CONFLICT UPDATE)
  └── UPDATE 63 個指標欄位
        ↓
從 DB 讀最新指標 [~5ms]  ← DB 是唯一真相
        ↓
策略評估 [~0.01ms]
        ↓
下單（如果觸發）[~1ms → Queue]

全程 ~57ms
```

### 4.4 策略狀態機

```
┌─ IDLE ───────────────────────────────────┐
│  KD 金叉 (kd_cross=1) → WATCHING          │
└──────────────────────────────────────────┘
    ↓
┌─ WATCHING ───────────────────────────────┐
│  1. 觀察條件反轉 → 回 IDLE（取消）         │
│  2. 觀察條件全滿足 → 繼續                  │
│  3. Target 滿足 → HOLDING                 │
│     ├── entry_price = close               │
│     ├── 算 grid_target_price              │
│     └── 下進場單 + MOC 安全網              │
└──────────────────────────────────────────┘
    ↓
┌─ HOLDING ────────────────────────────────┐
│  優先：格子止盈（close 到 target）          │
│  保底：KD 死叉 (kd_cross=-1)              │
│  強制：15:55 ET 日內平倉                   │
└──────────────────────────────────────────┘
```

### 4.5 格子止盈計算

```
進場時：
  prev_hl: range = prev_ha_high - prev_ha_low
  prev_high: range = prev_ha_high × 2 - entry_price (long)
                   = entry_price - prev_ha_low × 2 (short)

  grid_size = range / 20
  grid_move = optimal_grid_level × grid_size

  long:  target = entry_price + grid_move
  short: target = entry_price - grid_move

持倉中：
  每根 bar 檢查 close vs target
  到價 → 出場 (exit_reason='grid_take_profit')
```

### 4.6 下單執行

```
OrderRequest → Queue
    ↓
IB 線程 process_queue()：
    ├── 檢查 IB 連線（斷線 → 丟棄訂單，策略下根 bar 重新評估）
    ├── 市價單 ib.placeOrder()
    ├── intraday → 同時掛 MOC 反向單
    ↓
Fill Callback：
    ├── 進場 → ManagedPosition + INSERT ib_trades
    └── 出場 → 算 PnL + UPDATE ib_trades + 取消 MOC
```

### 4.7 日內收盤（三重保險）

```
A. IB 線程：15:55 ET 強制平倉（每秒檢查）
B. 策略引擎：on_bar 時 15:55 ET 檢查
C. MOC 安全網：IB ~15:50 ET 自動執行
   └── 提前出場時自動取消 MOC
```

---

## 五、效能

| 指標 | 數值 |
|------|------|
| 指標計算 | ~41 ms |
| 寫 DB | ~10 ms |
| 讀 DB | ~5 ms |
| 策略評估 | ~0.01 ms |
| **全程決策** | **~57 ms** |
| IB 下單 | ~1 秒（Queue 間隔） |
| 聚合器暖機 | ~14 秒 |

---

## 六、目前部署

| 項目 | 值 |
|------|---|
| 策略數 | 181 個 |
| 股票 | TSLA(91), AAPL(49), MSFT(26), NVDA(16), AMD(11), META(7) |
| 每筆股數 | 10 股 |
| 最大同時持倉 | 5 筆 |
| IB 帳戶 | DUH720646 (Paper Trading) |
| Databento | EQUS.MINI, 57 symbols |

---

## 七、安全防護

| 防護 | 實作 |
|------|------|
| SQL 注入 | `_safe_table_name()` 白名單 + 欄位名白名單 |
| 重複下單 | `_inflight` set + `_lock` |
| 持倉上限 | `max_total_positions` 檢查 |
| 日內平倉 | 三重保險（IB 線程 + 策略引擎 + MOC） |
| IB 斷線 | 丟棄訂單，下根 bar 重新評估 |
| 暖機殘留 | `_warmup_mode` + 完成後重置 IDLE |
| 孤兒持倉 | 手動清理（待自動化） |
| 容器掛掉 | `--restart unless-stopped` |

---

## 八、剩餘待辦（全低優先）

| # | 問題 | 影響 |
|---|------|------|
| 1 | 19 處 `except Exception: pass` 不 log | 難排查 |
| 2 | ALTER TABLE 失敗靜默 | 表結構不一致 |
| 3 | diff API trade_since 寫死 | 長期使用要改 |
| 4 | table name regex 允許 __ | 理論不安全 |
| 5 | 過夜持倉自動偵測 | 目前手動清 |
| 6 | 異常通知（Slack/Email） | 斷線沒通知 |
| 7 | 下單改固定金額 | 目前固定股數 |
