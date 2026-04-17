# OREO v3.1.1 — Bundesliga 2 投注研究系統

## 系統概覽

基於歷史數據信號掃描的半自動化投注推薦系統。
訓練集：2122 + 2223 + 2526 R1-R24（828場）
實盤目標：2526 R25-R34

## Subagent團隊（v3.1.1，5人）

| 代號 | 名字 | 職責 |
|------|------|------|
| coordinator | Carazinho | 主控，唯一對外接口 |
| feature-engineer | Paqueta | 計算所有特徵 |
| signal-scanner | Toney | 信號觸發與迴避核查 |
| independent-validator | 小剛 | 反駁者，尋找失敗論點 |
| bet-sizing | Tonali | 生成最終推薦清單 |

（大奎/阿洛已退場，詳見 `.claude/agents/daquei.md` / `alo.md`）

## 每輪流程（v3.1.1）

```
賽前：
  用戶下載D2.csv + 手動加入當輪賽程空白行
  → python src/data/append_2526.py        （更新master_rolling.csv）
  → python src/data/csv_processor.py      （生成R{N}_fixtures.json）
  → Carazinho → Paqueta → Toney → 小剛 → Tonali
  → reports/recommendation_R{N}_draft.md

賽後（週一）：
  用戶重新下載最新D2.csv（含本輪賽果）
  → python src/data/append_2526.py        （更新master_rolling.csv）

每5輪：Carazinho 生成信號表現報告
```

## 安全原則

- 本地Git，永不push到GitHub
- `.private/` 目錄加密，不進Git
- 財務金額只存於 `.private/`
- 所有雲端同步服務已在 `.gitignore` 排除

## 目錄結構

```
OREO_v3.1/
├── .claude/agents/         ← 5個Subagent定義（大奎/阿洛退場說明保留）
├── src/
│   ├── data/
│   │   ├── csv_processor.py            ← 生成當輪R{N}_fixtures.json
│   │   ├── append_2526.py              ← 更新master_rolling.csv
│   │   └── scraper_v311_archived.py    ← 大奎舊腳本（已歸檔）
│   ├── features/feature_engineering.py ← Paqueta用
│   ├── models/
│   │   ├── signal_engine.py            ← Toney用
│   │   └── validator_v311_archived.py  ← 阿洛舊腳本（已歸檔）
│   └── output/report_generator.py      ← Tonali用
├── config/model_params.json ← 信號大腦（Git追蹤）
├── data/
│   ├── master_rolling.csv  ← 歷史數據庫（2122-2526，含所有衍生列）
│   ├── raw/                ← R{N}_fixtures.json（不進Git）
│   ├── features/
│   ├── signals/
│   └── results/            ← 實盤記錄（不含金額）
├── reports/                ← 每輪推薦清單
└── HANDOVER.md             ← 主要參考文件
```
