# OREO v3.1.1 工作繼承文件
**最後更新：2026-04-10**
**用途：新對話串開始時，Claude必須先完整讀取此文件再執行任何工作**

---

## 一、項目背景

Bundesliga 2（德國乙組）投注研究系統。
基於歷史數據信號掃描，半自動化生成每輪下注推薦。

- 訓練集：2122 + 2223 + 2526 R1-R24（828場）
- 排除賽季：2324（賠率環境異常低）、2425（賠率環境異常高）
- 實盤目標：2526 R25-R34（每5輪生成信號表現報告）

---

## 二、數據定義（所有代碼的唯一標準）

### 2.1 數據來源
- 訓練數據：`research/processed_master.csv`（1440場，5賽季）
- 欄位命名以此文件為準，不以任何腳本為準

### 2.2 賠率環境基準
```
賽季    AvgO均值   環境
2122    1.719      正常（訓練集）
2223    1.737      正常（訓練集）
2324    1.591      異常低（排除）
2425    1.921      異常高（排除）
2526    1.646      正常（訓練+目標）
```
⚠️ 若AvgO均值偏離1.70超過0.15，所有賠率閾值需重新校準。

### 2.3 基礎欄位衍生
```python
RoundNum = Round.str.replace("R","").astype(int)
IsOver25 = (FTHG + FTAG) >= 3
t_sum    = (1/AvgH) + (1/AvgD) + (1/AvgA)
FairH    = (1/AvgH) / t_sum
FairA    = (1/AvgA) / t_sum
tou      = (1/Avg_over25) + (1/Avg_under25)
FairOver = (1/Avg_over25) / tou
AvgO     = Avg_over25
AvgU     = Avg_under25
TS       = Time_Slot（-1視為NaN排除）
```

### 2.4 ROI定義
```python
大球ROI = Σ[(AvgO-1) if IsOver25 else -1] / 場數
小球ROI = Σ[(AvgU-1) if NOT IsOver25 else -1] / 場數
主勝ROI = Σ[(AvgH-1) if FTR=="H" else -1] / 場數
客勝ROI = Σ[(AvgA-1) if FTR=="A" else -1] / 場數
```

### 2.5 GapBucket（唯一正確定義）
```python
# g = HomeWindowScore - AwayWindowScore
# 正數=主場較強，負數=客場較強
def gap_bucket(g):
    if g <= -8: return "E"   # 客場明顯強
    if g <= -3: return "D"   # 客場略強
    if g <=  2: return "C"   # 均等
    if g <=  7: return "B"   # 主場略強
    return       "A"          # 主場明顯強（≥+8）
```
⚠️ 門檻值：A=≥+8（不是≥+9），E=≤-8（不是≤-5）。之前有錯誤定義，以此為準。

### 2.6 窗口結構
```python
WINDOW_BINS = {
    "W1":(1,6), "W2":(7,12), "W3":(13,17),
    "W4":(18,23), "W5":(24,28), "W6":(29,34)
}
# W3→W4之間有冬歇期（R17後約1個月）
# W4起始評分用H1_cumul（R17後的累積積分）
```

### 2.7 W0基準分
```python
W0_returning = prev_season_pts_W5 + prev_season_pts_W6  # 留隊球隊
W0_from1B    = 6.22  # 從1B降組（跨5季固定值）
W0_from3B    = 6.00  # 從3B升組（跨5季固定值）
```

### 2.8 升降班名單
```python
NEWCOMERS = {
    2122: {"from_1B":["Bremen","Paderborn"],
           "from_3B":["Hansa Rostock","Dresden","Ingolstadt"]},
    2223: {"from_1B":["Greuther Furth","Bielefeld"],
           "from_3B":["Magdeburg","Braunschweig","Kaiserslautern"]},
    2324: {"from_1B":["Schalke 04","Hertha"],
           "from_3B":["Elversberg","Osnabruck","Wehen"]},
    2425: {"from_1B":["FC Koln","Darmstadt"],
           "from_3B":["Ulm","Preußen Münster","Regensburg"]},
    2526: {"from_1B":["Bochum","Holstein Kiel"],
           "from_3B":["Bielefeld","Dresden"]},
}
```

### 2.9 賽況標籤 H_label / A_label（唯一正確定義）
```python
# 每輪賽前計算，基於當輪積分榜
dist_top3 = pts_3rd  - team_pts   # 距第3名差距
dist_bot3 = team_pts - pts_16th   # 距第16名差距

if dist_top3 <= 6:                       label = "PROM"
elif dist_bot3 <= 6:                     label = "RELEG"
elif dist_top3 > 6 and dist_bot3 > 6:   label = "MID_SAFE"
else:                                    label = "MARGIN"

# 特殊情況：R1全隊積分=0，全部標記為 "MARGIN"
```
⚠️ 有四個標籤（PROM/RELEG/MID_SAFE/MARGIN），不是三個。
⚠️ 不是按排名位置（第1-6名等），是按積分差距。

### 2.10 近況標籤
```python
H_form3 = 主隊最近3場積分（勝3平1負0），不足3場→NaN，每季獨立計算
A_form3 = 客隊最近3場積分，同上
H_form5 = 主隊最近5場積分（0-15），不足5場→NaN
A_form5 = 客隊最近5場積分，同上
```

---

## 三、已確認信號（OREO v3最終結果）

### 3.1 正向信號（P1-P4）

| ID | 名字 | 方向 | n | 訓練ROI | R18-24 ROI | 優先級 | 注碼 |
|----|------|------|---|---------|-----------|--------|------|
| AH-01 | 亞盤移動利主 | ah_home | 118 | +17.2% | −3.5%(n=2) | P1 | 1.0u |
| NEW-01 | 主PROM×H_form3中(4-6) | over25 | 175 | +17.4% | +42.3% | P1 | 1.0u |
| NEW-02 | 客PROM×A_form3低(≤3) | over25 | 80 | +23.3% | +40.5% | P1 | 1.0u |
| GBA-H | GapBucket=A×主勝 | home | 148 | +16.8% | +21.7% | P1 | 0.75u |
| NEW-03 | 主PROM×H_form3低(≤3) | over25 | 83 | +10.6% | +38.2% | P2 | 0.75u |
| NEW-04 | H_form3低×GapBucket=A | over25 | 44 | +11.2% | +41.7% | P2 | 0.75u |
| GBA-TS2 | GapBucket=A×TS=2×主勝 | home | 51 | +26.0% | +40.0% | P2 | 0.75u |
| GBB-H | GapBucket=B×主勝 | home | 74 | +8.6% | +8.6% | P2 | 0.5u |
| F-26 | Karlsruhe×大球 | over25 | 92 | +9.2% | −11.1% | P4 | 0.25u |
| GBE-TS2-U | GapBucket=E×TS=2×小球 | under25 | 49 | +5.4% | +25.5% | P4 | 0.25u |
| W6-OV | R29-34×大球 | over25 | 108 | +6.3% | N/A | P4 | 0.25u |
| MID-LATE | 主MID_SAFE×R18+×大球 | over25 | 88 | +8.8% | N/A(n=3) | P4 | 0.25u |

### 3.2 注碼規則
```
單場上限：1.5u
P1+P1重疊：1.5u
P1+P2重疊：1.5u
P2+P2重疊：1.0u
任何+P4：只加0.25u，不翻倍
迴避信號觸發：該場所有正向信號失效
```

### 3.3 迴避信號（AV-01至AV-12）
完整列表在 `config/model_params.json` 的 `avoid_signals` 欄位。
核心原則：
- 任何GapBucket=A的場次不買客勝（AV-03）
- PROM主場不買小球（AV-01、AV-05、AV-07）
- Karlsruhe不買小球（AV-08）

### 3.4 已移除信號（不得重新啟用，除非新賽季重新驗證）
- Karlsruhe_over25：已降P4觀察
- AvgO 1.7-1.8 × 大球：冬歇後−20.5%，移除
- H_form3_hi × 小球：冬歇後−45.2%，移除
- GBE_TS4 × 主勝：冬歇後翻負，移除
- form_diff_pos × 大球：2526全段均負，移除
- TS3 × 平局：2526全段均負，移除

---

## 四、系統架構（v3.1.1）

### 4.1 Subagent團隊（5人）

| 代號 | 名字 | 職責 |
|------|------|------|
| coordinator | Carazinho | 主控，唯一對外接口 |
| feature-engineer | Paqueta | 計算所有特徵，預留ELO/Poisson注入窗口 |
| signal-scanner | Toney | 對照model_params.json觸發信號，核查迴避 |
| independent-validator | 小剛 | 反駁者，信息隔離，主動尋找失敗論點 |
| bet-sizing | Tonali | 整合推薦，生成recommendation_R{N}.md |

（大奎、阿洛已退場：web scraping不穩定，改由用戶上傳CSV）

### 4.2 每輪流程（v3.1.1）
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

每5輪：Carazinho生成信號表現報告
```

### 4.3 信息隔離（小剛的設計）
```
小剛只能看：觸發條件、歷史失敗率、當前賽季非量化資訊
小剛不能看：推薦方向和注碼
小剛必須回答：
  1. 這個信號的反面論點是什麼？
  2. 歷史數據中有多少次相同信號最終失敗？
  3. 當前賽季有哪些非量化因素可能使模型失效？
  4. 這個信號的歷史失敗場次有什麼共同特徵？本輪觸發場次是否相似？
```

### 4.4 數據管道（v3.1.1）
```
data/user_downloads/D2.csv          ← 用戶每輪下載，手動加賽程行
    ↓ append_2526.py
data/master_rolling.csv             ← 歷史數據庫（R1-最新，含所有衍生列）
    ↓ csv_processor.py
data/raw/R{N}_fixtures.json         ← 當輪賽前數據（供Paqueta讀取）

衍生列計算規則：
- Time_Slot：R1/R34全部=-1；其他：Fri 17:30=1, Sat 12:00=2, Sat 19:30=3, Sun 12:30=4
- TTC = HC + AC（總角球）
- HomeScore/AwayScore = W0 + 已完成窗口積分（窗口內鎖定）
- W4+用H1_cumul錨點（W0+W1+W2+W3）
- AH-01：用戶可選在CSV賽程行加AHh/AHCh欄位，無數據時自動跳過
```

### 4.5 安全原則
- 本地Git，永不push到GitHub
- `.private/`目錄加密，不進Git，財務金額只存於此
- 所有雲端同步服務排除
- model_params.json每次變更必須git commit並記錄原因

### 4.6 ELO/Poisson預留窗口
```json
"elo_module": {"enabled": false, "weight": 0.0}
"poisson_module": {"enabled": false, "weight": 0.0}
```
啟用時：在feature-engineer層注入為額外特徵列，不改動signal_engine邏輯。

---

## 五、目錄結構（v3.1.1）

```
OREO_v3.1/
├── .claude/agents/         ← 5個Subagent定義（Carazinho/Paqueta/Toney/小剛/Tonali）
│   │                         大奎/阿洛已退場，各自md保留退場說明
├── src/
│   ├── data/
│   │   ├── csv_processor.py            ← 生成當輪R{N}_fixtures.json（v3.1.1新增）
│   │   ├── append_2526.py              ← 更新master_rolling.csv（v3.1.1新增）
│   │   └── scraper_v311_archived.py    ← 大奎舊腳本（已歸檔）
│   ├── features/feature_engineering.py ← Paqueta用
│   ├── models/
│   │   ├── signal_engine.py            ← Toney用
│   │   ├── validator_v311_archived.py  ← 阿洛舊腳本（已歸檔）
│   │   ├── elo_model.py                ← 預留空檔
│   │   └── poisson_model.py            ← 預留空檔
│   └── output/report_generator.py      ← Tonali用（已移除HKJC比對）
├── config/model_params.json ← 信號大腦
├── data/
│   ├── user_downloads/    ← 用戶下載的D2.csv放這裡（不進Git）
│   ├── master_rolling.csv ← 歷史數據庫（2122-2526，含所有衍生列）
│   ├── raw/               ← R{N}_fixtures.json（不進Git）
│   ├── features/
│   ├── signals/
│   └── results/           ← 實盤記錄（不含金額）
├── reports/               ← 每輪推薦清單
├── docs/superpowers/plans/ ← 實施計劃文檔
├── .private/              ← 加密，不進Git
├── .gitignore
├── README.md
└── HANDOVER.md            ← 本文件
```

---

## 六、當前進度（v3.1.1，更新至2026-04-10）

### 已完成
- [x] 項目骨架和Git初始化
- [x] config/model_params.json（所有信號、迴避條件、ELO預留窗口）
- [x] src/features/feature_engineering.py（Paqueta）
- [x] src/models/signal_engine.py（Toney）
- [x] src/output/report_generator.py（Tonali）— 已移除HKJC比對（v3.1.1）
- [x] src/data/csv_processor.py（v3.1.1新增，取代大奎賽前功能）
- [x] src/data/append_2526.py（v3.1.1新增，更新master_rolling.csv）
- [x] data/master_rolling.csv（2122-2526 R1-R29，含所有衍生列）
- [x] .claude/agents/ 全部更新至v3.1.1流程
- [x] 2526 R25-R28 實盤完成（recommendation reports已生成）

### 當前狀態
- 2526 賽季進行中，現為 **R29**（2026-04-10起踢）
- master_rolling.csv：R1-R28有賽果，R29為空白賽程行
- 下一步：R29踢完後重新下載D2.csv，執行append_2526.py更新賽果

### v3.1.1 關鍵設計決策
- 大奎/阿洛退場：改由用戶手動下載D2.csv + csv_processor.py
- HKJC比對移除：實際操作中未採納，從Tonali輸出中删除
- AH-01保留但標記「手動輸入」：無數據時自動跳過，不報錯
- 賠率優先級：Pinnacle開盤 → Pinnacle收盤 → BbAv均值 → None
- Time_Slot：R1和R34全部=-1；其他按時間映射

---

## 七、重要決策記錄（不得推翻，除非明確說明）

1. **不用ELO/Poisson作為主模型**：現階段直接用信號掃描結果，ELO/Poisson在後續輪次空檔補入。
2. **2324/2425不作為驗證集**：賠率環境異常，finding在這兩季的表現不作為通過/失敗依據。
3. **HKJC比對已移除**：實際操作中未採納，Tonali不再輸出HKJC比對行（v3.1.1起）。
4. **大奎/阿洛已退場**：改由用戶手動下載D2.csv + csv_processor.py取代。阿洛的格式核查由csv_processor.py報錯取代。
5. **independent-validator（小剛）與數據驗證分開**：信息隔離需要，不合併。
6. **每5輪生成表現報告**（不是每10輪）。
7. **P4信號統一0.25u**：觀察組不忘記但謹慎下注。

---

## 八、新對話串開始時的指示

**給新對話串的Claude：**

1. 完整讀取本文件
2. 讀取 `.claude/agents/carazinho.md`（Carazinho是orchestrator，定義了完整v3.1.1執行流程）
3. 確認當前進度（第六章）
4. 繼續「進行中」的任務，不重新設計已決定的架構
5. 數據定義以第二章為唯一標準，任何腳本或其他文件有衝突時以本文件為準
6. 如需查閱完整信號統計數據，讀取 `config/model_params.json`
7. 每輪賽前執行順序：
   a. 用戶下載D2.csv並手動加入當輪賽程空白行
   b. `python src/data/append_2526.py`（更新master_rolling.csv）
   c. `python src/data/csv_processor.py --csv <path> --output data/raw/R{N}_fixtures.json`
   d. 喚醒 Paqueta → Toney → 小剛 → Tonali
