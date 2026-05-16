# Backtest Spec

週二開工。先驗 Phase 1 live (週一 9:15) 再開始。

## 目標

回答兩個問題：

1. **單一策略**：「開盤量比爆量」這個訊號真的能在歷史上抓到當天會漲的股票嗎？
2. **多策略比較**：4 個策略中哪一個勝率 / 平均報酬 / 風報比最高？

## 資料

| 資料源 | 深度 | 用途 |
|--------|------|------|
| yfinance `period=60d, interval=5m` | 60 個交易日 × 50 檔 × 5分K | 主回測資料 |
| yfinance `period=60d, interval=1d` | 60 日 × 50 檔 × 日K | 算前收 / 漲跌停 |

> 想要更深 → 付費 FinMind 或自建 tick 倉。**先用 60 天驗證概念**。

## 出場規則（三套都跑，並列在報告裡）

| 規則 | 程式變數 | 邏輯 |
|------|---------|------|
| **EOD** | `exit_eod` | 13:25 出場（avoid 13:25-13:30 競價段），純看當沖一日勝率 |
| **R-multiple** | `exit_r` | 進場後 +2% take profit OR -1% stop loss，先到先出 |
| **VWAP** | `exit_vwap` | 進場後 close 連續 3 根跌破 VWAP 出場，動態出場 |

## 進場規則（每個策略不同，先寫策略 1）

### Strategy 1: 開盤量比爆量

- **訊號時機**：每天 9:15 sharp（收盤前 4 小時，給足走勢時間）
- **進場條件**：09:00~09:15 累積量 ÷ 過去 5 交易日 09:00~09:15 平均量 ≥ 3.0
- **進場價**：9:15 該根 5分K 的 close
- **方向**：long only（量比爆量假設突破向上；short 之後另寫）

### Strategy 2-4: 之後寫的時候各自寫規格

## Backtest 引擎輸出

每個 (strategy, exit_rule) 組合算這幾個 metric：

| Metric | 公式 |
|--------|------|
| `total_signals` | 60 天總命中訊號數 |
| `win_rate` | 賺錢訊號 ÷ 總訊號 |
| `avg_return` | 平均報酬 % |
| `median_return` | 中位數報酬 % |
| `max_drawdown` | 最深單筆虧損 % |
| `r_value` | avg_return ÷ avg_loss_when_lose（標準 R 值） |
| `expectancy` | (win_rate × avg_win) - ((1-win_rate) × avg_loss) |
| `signals_per_day` | 訊號數 ÷ 交易日 |

## 報告 UI

新增前端 tab `/backtest`。表格：

```
Strategy 1: 開盤量比爆量 (60 天)
┌─────────┬───────┬──────────┬──────────┬───────┐
│ 出場規則  │ 訊號  │ 勝率      │ 平均報酬  │ R 值  │
├─────────┼───────┼──────────┼──────────┼───────┤
│ EOD     │ 87    │ 54.0%    │ +0.42%   │ 0.85  │
│ R 2/1   │ 87    │ 47.1%    │ +0.18%   │ 1.05  │
│ VWAP    │ 87    │ 62.1%    │ +0.55%   │ 1.23  │
└─────────┴───────┴──────────┴──────────┴───────┘
```

訊號清單可下鑽到單一交易日：
```
2026-04-15: 命中 3 檔
  2330 台積電  量比 3.2x  9:15 → 13:25 +1.8%  ✅
  2454 聯發科  量比 4.1x  9:15 → 13:25 -0.3%  ❌
  2317 鴻海    量比 3.5x  9:15 → 13:25 +0.9%  ✅
```

## Backend 新 endpoint

```
GET /api/daytrade/backtest/run?strategy=opening-volume-burst&exit=eod,r,vwap&days=60
  → {
    strategy: "opening-volume-burst",
    period: {start: "2026-02-15", end: "2026-05-16", days: 60},
    exits: {
      eod:  {total: 87, wins: 47, win_rate: 0.540, avg_return: 0.42, ...},
      r:    {total: 87, wins: 41, win_rate: 0.471, ...},
      vwap: {total: 87, wins: 54, win_rate: 0.621, ...},
    },
    signals: [
      {date: "2026-04-15", id: "2330", volume_ratio: 3.2,
       entry_price: 1085, exits: {eod: {price: 1104, return: 1.8}, ...}},
      ...
    ]
  }
```

實作位置：和現有 scan 同檔 `/api/daytrade/...`。Cache 結果 24 小時（資料 60 天內不變）。

## 為什麼這樣設計

- **3 套出場規則並列**：避免 cherry-pick。同樣訊號不同出場結果差很大，全部列出來才知道策略真正的特性
- **EOD 是當沖標準**：確保訊號是真當沖適用，不是「波段順便」
- **R 值是風報比經典**：所有 quant 文獻都用，方便和其他系統比較
- **VWAP 出場是實戰最近**：實際 day trader 通常會盯 VWAP

## 程式時數估算

| 任務 | 時間 |
|------|------|
| Backtest 框架 (data loader, exit rule engine, metrics) | 2-3 hr |
| Strategy 1 backtest specialization | 0.5 hr |
| 前端 backtest tab UI + 表格 + 下鑽 | 2 hr |
| Strategy 2 (VWAP 反轉) + backtest | 3 hr |
| Strategy 3 (跳空+接力) + backtest | 2 hr |
| Strategy 4 (5分K 突破前高) + backtest | 2 hr |
| 對比 + 砍弱策略 | 0.5 hr |

合計 ~12-13 hr。分 3-4 個 session 做完。
