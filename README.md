# 當沖掃描 — Daytrade TW

iPhone PWA 當沖策略掃描 app，第一階段策略：**開盤量比爆量**。

## 是什麼

每天 9:00 開盤後，掃描台灣 50 成分（50 檔）中
「09:00 ~ 現在累積量 > 過去 5 交易日同時段均量 × N 倍」的股票。
盤中 60 秒自動刷新。

## Stack

| 層 | 技術 | 部署 |
|----|------|------|
| Frontend | React 18 + Babel standalone inline（單檔 `www/index.html`） | Vercel |
| Backend | 共用 `taiwan-stock-app` Flask | Mac + ngrok（必要）/ Render fallback（資料源不通） |
| Data | yfinance 5d × 1m（基準）+ `/api/intraday/{id}`（即時） | Mac |

## 為什麼要 Mac+ngrok

Render free tier 上 yfinance 被 rate-limit、MIS 也被擋，所以盤中即時資料只能走 Mac。
打開 Mac、跑 `start.command`，ngrok 把 `127.0.0.1:8080` 開到外網。

## API

| Endpoint | 回傳 |
|----------|------|
| `GET /api/daytrade/universe` | `{stocks: [{id, name}], total, source}` |
| `GET /api/daytrade/scan/opening-volume-burst?threshold=3.0` | `{status, hits: [...], universe_size, threshold, until_time, market_open}` |

詳見 [taiwan-stock-app#56](https://github.com/Kevin666-glitch/taiwan-stock-app/pull/56)。

## 本機開發

```bash
# 1. 確認 Mac Flask + ngrok 在跑
bash /Users/oliver/Documents/taiwan-stock-app/start.command

# 2. 起 static server
python3 -m http.server 8766 --directory www

# 3. 開瀏覽器
open http://127.0.0.1:8766
```

`window.API_BASES` 在 `index.html` `<head>` 自動切換：
- `localhost` / `127.0.0.1` → `http://127.0.0.1:8080`
- 其他 → `https://renegade-silliness-stony.ngrok-free.dev`

## 部署

Vercel 設定：
- Output directory: `www`
- Framework preset: Other
- 無 build command

## 加到 iPhone 主畫面

1. iPhone Safari 開 Vercel URL
2. 分享 → 加入主畫面
3. 名稱 "當沖掃描"
4. 開啟 app → standalone PWA 全螢幕

## 週一 9:15 驗證 checklist

- [ ] Mac 開機，`start.command` 已跑（Flask + ngrok 都 up）
- [ ] iPhone 開 PWA，看到「盤中」pill
- [ ] threshold 設 3.0，至少看到 1-5 檔命中（如果沒有，降到 2.0 看看）
- [ ] 60 秒後自動更新（看「N 秒前」變化）
- [ ] 點 status pill 手動 refresh 可用

## Roadmap

| Phase | 內容 |
|-------|------|
| 1.0 | ✅ 開盤量比爆量 + 50 檔 universe + Mac+ngrok backend |
| 1.1 | mini intraday spark / 漲跌停價標示 |
| 1.2 | Pull-to-refresh |
| 2.0 | VWAP 反轉 / 開盤跳空 / 5分K 突破前高 |
| 3.0 | 進出場 journal（localStorage） |
| 4.0 | 策略 backtest |
| 5.0 | iOS Web Push notification |
