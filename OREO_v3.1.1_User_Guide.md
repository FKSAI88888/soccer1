# OREO v3.1.1 — 每輪操作指南
- 需要 Python 3.8+

---

## 一、賽前準備（開賽前2-3天）

### Step 1：下載數據

前往 [football-data.co.uk/germanym.php](https://www.football-data.co.uk/germanym.php) 下載最新 **D2.csv**。(Season 25/26 Bundesliga 2)

下載後**改名為當前輪次**，保存到：

```
OREO_v3.1/data/user_downloads/R29.csv   ← 以當前輪次命名
```

（football-data.co.uk 下載的原始文件叫 D2.csv，改名後放入即可）

### Step 2：在 CSV 末尾手動加入當輪賽程行

打開 D2.csv，在最後一行之後加入每場比賽的空白行。

**等待Pinnacle開盤後再填**，把賠率一併加進去（信號覆蓋最完整）：
(https://www.pinnacle.com/en/soccer/germany-bundesliga-2/matchups/#all)

```
Div,Date,Time,HomeTeam,AwayTeam,PSH,P>2.5
D2,12/04/2026,13:00,Hannover,Hamburg,2.10,1.85
D2,12/04/2026,13:00,Paderborn,Elversberg,1.95,1.90
...（共9場）
```

- Date格式：`DD/MM/YYYY`
- Time格式：`HH:MM`（不填秒數）
- `PSH`：Pinnacle主賠率
- `P>2.5`：Pinnacle大球賠率
- csv_processor 自動用這些計算 FairH / FairOver，不需另外告知 Carazinho

**賠率還沒出？** 留空也沒問題，Paqueta算FairH=None，Toney自動跳過需要賠率的信號（AH-01、AV-10），其他信號不受影響。

**可選（AH-01信號用）**：可額外加兩列 `AHh`（亞盤開盤）和 `AHCh`（亞盤收盤）。沒有跳過。

### Step 3：更新歷史數據庫

```bash
cd "D:/Documents/Financial/Football Prediction/OREO_v3.1"
python src/data/append_2526.py --csv data/user_downloads/R29.csv
```

（將 R29 改為當前輪次）

輸出：`data/master_rolling.csv` 更新完成（確認無報錯）

### Step 4：生成當輪賽前 JSON

```bash
python src/data/csv_processor.py --csv data/user_downloads/R29.csv --output data/raw/R29_fixtures.json
```

（兩處 R29 都改為當前輪次）

確認輸出顯示：`✓ R29: 9 fixtures` 且賠率不全為 None

---

## 二、AI 預測（開新 Claude 對話串）

### Step 5：喚醒 Carazinho

開新對話串，第一句說：

> **「Carazinho，現在是2526 R29賽前，請執行預測流程。fixtures已在 data/raw/R29_fixtures.json。」**

Carazinho 會自動依序呼叫：
1. **Paqueta**（計算特徵：GapBucket/FairH/form3等）
2. **Toney**（掃描信號，核查迴避）
3. **小剛**（挑戰報告，反駁論點）
4. **Tonali**（整合注碼，生成推薦）

### Step 6：審閱推薦

輸出：`reports/recommendation_R29_draft.md`

審閱內容：
- 觸發信號及方向
- 注碼建議（單場上限1.5u）
- 小剛的風險提醒
- 迴避場次清單

---

## 三、賽後更新（週一）

### Step 7：更新賽果

重新下載最新 D2.csv（已含本輪賽果），改名為 `R29.csv`，覆蓋放到 `data/user_downloads/R29.csv`。

```bash
python src/data/append_2526.py --csv data/user_downloads/R29.csv
```

master_rolling.csv 自動填入本輪 FTHG/FTAG/FTR/TTC 等賽果欄位。

---

## 快速參考

| 信號優先級 | 注碼 |
|-----------|------|
| P1+P1 重疊 | 1.5u |
| P1+P2 重疊 | 1.5u |
| P2+P2 重疊 | 1.0u |
| 任何 + P4 | 只加 0.25u |
| 單場硬上限 | 1.5u |
| avoid=true | 0u（封鎖） |

| Time_Slot | 時間 |
|-----------|------|
| TS=1 | 週五 17:30 |
| TS=2 | 週六 13:00 |
| TS=3 | 週六 20:30 |
| TS=4 | 週日 13:30 |
| TS=-1 | 其他（含R1/R34全輪） |

---

**整個流程約需：** Step 1-4 手動操作 10分鐘，Step 5-6 AI預測 5-10分鐘。