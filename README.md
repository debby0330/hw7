# ARIA v4.0 — The Accessible Auditor
### Week 7 Assignment | 花蓮市災害可及性評估系統

> *"A risk map tells you what is broken. A network analysis tells you if you can still save someone."*

---

## 專案概述 Project Overview

本專案針對颱風鳳凰（Typhoon Fung-wong）情境，建立花蓮市整合式道路網路災害可及性自動評估系統（ARIA v4.0）。系統結合 OpenStreetMap 道路網路、即時降雨壅塞動態權重，以及地形風險疊加，模擬 130.5 mm/hr 極端降雨下蘇花公路封閉、花蓮山區道路中斷的交通可及性崩潰狀況。

---

## 專案結構 Project Structure

```
ex7/
├── Week7-Complete.ipynb         # 主要分析筆記本
├── .env                         # 環境變數設定
├── data/
│   └── terrain_risk_audit.json  # Week 4 地形風險避難所資料（311 筆）
└── output/
    ├── hualien_network.graphml  # 花蓮市道路網路圖資
    ├── bottleneck_map.png       # Top-5 瓶頸節點視覺化地圖
    ├── isochrone_comparison.png # 等時線災前/災後比較圖
    ├── accessibility_table.csv  # 可及性衝擊表
    ├── ai_strategy_report.txt   # Gemini AI 災防建議報告
    └── README.md                # 本文件
```

---

## 資料來源 Data Sources

| 資料類型 | 來源 | 說明 |
|---------|------|------|
| 道路網路 | OpenStreetMap via OSMnx | 花蓮市，5km 半徑，`network_type='drive'` |
| 降雨資料 | 內建隨機模擬（seed=42） | 可替換為 W5 fungwong JSON 或 W6 Kriging TIF |
| 地形風險 | Week 4 `terrain_risk_audit.json` | 311 筆避難所屬性（風險等級、高程、坡度、河距） |

---

## 分析流程 Analysis Pipeline

```
[Part A] 道路網路擷取（OSMnx）
         → 投影 EPSG:3826
         → 計算基礎行車時間（依 highway 類型速度預設）
              ↓
[Part B] Betweenness Centrality → Top-5 瓶頸節點
         → 疊加 Week 4 地形風險（Nominatim 地理編碼 + sjoin_nearest）
         → 視覺化：路網底圖 + 彩色星形標記（顏色 = 風險等級）
              ↓
[Part C] 降雨 → 壅塞係數映射（rain_to_congestion）
         → 套用動態權重（travel_time_adj）
         → 5 個關鍵設施 × 固定 5min / 10min 等時線
         → 計算面積縮減比例，標記孤立設施（縮減 ≥ 80%）
              ↓
[Part D] 等時線災前 / 災後比較視覺化
              ↓
[Part F] Gemini AI 災防策略報告（Bonus）
```

---

## 核心方法說明 Core Methods

### 速度預設值

| highway 類型 | 速度 (km/h) |
|-------------|------------|
| motorway | 110 |
| primary | 60 |
| secondary | 40 |
| residential | 30 |
| unclassified | 30 |

OSM 的 `maxspeed` 屬性經常缺失，以上預設值確保行車時間計算的一致性。

### 降雨 → 壅塞係數（Slide 12 閾值法）

| 每小時雨量 | 壅塞係數 cf | 狀態描述 |
|-----------|------------|---------|
| < 10 mm/hr | 0.0 | 正常行駛 |
| 10–40 mm/hr | 0.3 | 輕微減速 |
| 40–80 mm/hr | 0.6 | 嚴重延誤 |
| > 80 mm/hr | 0.9 | 近乎無法通行 |

調整後行車時間：`travel_time_adj = length / (speed_m/s × (1 − cf))`

cf ≥ 0.95 時路段視為完全中斷（`travel_time_adj = inf`）。

### 等時線計算

- 固定閾值：**5 分鐘（300 秒）** 和 **10 分鐘（600 秒）**
- Dijkstra 單源最短路徑（`nx.single_source_dijkstra_path_length`）
- 可達節點轉換為 `shapely.MultiPoint.convex_hull` 凸包多邊形
- 面積縮減比：`shrinkage = 1 − (A_after / A_before)`
- **孤立判定**：5min 縮減比 ≥ 80% → 標記為孤立設施

---

## 環境設定 Environment Setup

### 安裝套件

```bash
pip install osmnx networkx geopandas pandas numpy matplotlib shapely
pip install python-dotenv geopy google-generativeai
```

### .env 設定

```env
GOOGLE_API_KEY=your_google_api_key_here
OUTPUT_DIR=C:\Users\user\CascadeProjects\ex7\output
NETWORK_PLACE=Hualien City, Taiwan
NETWORK_DIST=5000
NETWORK_CRS=EPSG:3826
CONGESTION_METHOD=threshold
CONGESTION_BREAK_1=10
CONGESTION_BREAK_2=40
CONGESTION_BREAK_3=80
TERRAIN_RISK_PATH=C:\Users\user\CascadeProjects\ex7\data\terrain_risk_audit.json
```

---

## AI 診斷日誌 AI Diagnostic Log

### 問題 1：OSMnx 擷取逾時

**問題描述**：`ox.graph_from_address()` 呼叫 Overpass API 時偶發逾時，特別是在尖峰時段或網路不穩時。

**解決方案**：
- 等待數分鐘後重試，或加入 `ox.settings.timeout = 300`
- 成功後立即以 `ox.save_graphml()` 儲存，後續直接讀取避免重複下載

---

### 問題 2：地形風險 JSON 無座標欄位

**問題描述**：`terrain_risk_audit.json` 僅有 `address` 文字欄位，無 `lat`/`lon`，無法直接做空間疊加。

**解決方案**：
- 使用 `geopy.Nominatim` 對 `address` 進行地理編碼取得座標
- 套用 `RateLimiter`（間隔 1.1 秒）避免請求過於頻繁
- 以 `gpd.sjoin_nearest(max_distance=2000)` 在 2km 範圍內找最近避難所疊合

---

### 問題 3：等時線多邊形形狀異常

**問題描述**：`alphashape` 的 alpha 參數難以調整，在稀疏節點區域容易產生不規則孔洞或面積過大的異常多邊形。

**解決方案**：改用 `shapely.MultiPoint.convex_hull` 取凸包，形狀穩定且面積計算可信。

---

### 問題 4：Gemini API 模型名稱與 Quota 限制

**問題描述**：
1. `gemini-1.5-flash` 已不支援，出現 `404 NotFound`
2. `gemini-2.5-pro` 免費 RPM 極低，頻繁呼叫出現 quota exceeded

**解決方案**：
1. 以 `genai.list_models()` 確認可用模型名稱（使用 `gemini-2.5-flash`）
2. 在呼叫前加入 `time.sleep(5)` 避免連續觸發限制

---

## 主要發現 Key Findings

> 執行完 notebook 後填入實際數值。

- **最脆弱瓶頸節點**：Node `[填入 Top-1 ID]`，Centrality = `[值]`，鄰近地形風險：`[填入]`
- **最大可及性損失**：Node `[填入]`，5min 縮減 `[x]%`，10min 縮減 `[x]%`
- **孤立設施**：`[填入節點清單，或 None]`
- **救援優先順序**：依 Shrinkage % 由高至低排序

---

## 提交清單 Submission Checklist

- [x] `Week7-Complete.ipynb` — 完整執行含輸出結果
- [x] `output/hualien_network.graphml`
- [x] `output/bottleneck_map.png`
- [x] `output/isochrone_comparison.png`
- [x] `output/accessibility_table.csv`
- [ ] `output/ai_strategy_report.txt` — 需填入 Gemini API Key
- [x] `README.md`

---

## 評分對照 Grading Reference

| 項目 | 配分 | 對應 Cell |
|------|------|-----------|
| 道路網路擷取 + 基礎行車時間 + GraphML 儲存 | 15% | S2, S3, S4, S15 |
| Betweenness Centrality + Top-5 + W4 疊加 | 20% | S5, S6, S7 |
| 動態可及性分析（壅塞 + 等時線 + 縮減比） | 30% | S8, S9, S10, S11, S12 |
| 專業標準（.env + GraphML + README + AI 日誌） | 15% | S14, S15, S18 |
| 視覺化品質（路網圖 + 等時線比較） | 10% | S7, S13 |
| **Bonus**：AI 策略報告 | 10% | S16, S17 |

---

*ARIA v4.0 | Week 7 Assignment | April 2026*
