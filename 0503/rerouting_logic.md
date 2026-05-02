# RSU 智能路徑清理與車道保護邏輯

本文檔說明 RSU 如何在不造成死結的前提下，透過主動干預確保緊急車輛 (EV) 的最優通行效率。

## 核心策略：智能三位一體控制 (Intelligent Trinity Control)

在最終版本中，我們從單純的「強制停止」轉向了更為精細的交通流管理：

### 1. 車道保留 (Lane Reservation) —— 最重要防線
- **機制**：當 EV 進入 RSU 範圍 (120m) 時，RSU 對該路段所有**非 EV 車道**的車輛下達 `LaneChangeMode(512)` 指令。
- **目的**：鎖死側邊車道的變道權限，嚴禁任何車輛切入 EV 所在的快車道。這在 EV 到達前就建立了一條「虛擬專用道」，徹底解決了隨機車輛亂切入導致的頻繁剎車。

### 2. 對向借道與清空 (Opposite-lane Driving & Clearing) —— 終極解方
- **EV 權限**：給予 EV `LaneChangeMode(8448)`，允許其在主車道塞車時主動穿越中心線借道超車。
- **對向助推**：RSU 監測對向路段 (`sameOrOppositeRoad`)，一旦 EV 準備借道，RSU 會自動「助推」對向來車加速離開，避免正面死結，確保 EV 有足夠的物理空間進行超車。

### 3. 智能助推與降級 (Strategic Clearance Boost)
- **機制**：不再使用強制停止 (`setSpeed(0.1)`)，而是使用 `setSpeed(max)` 配合 `SpeedMode(0)`。
- **階層化觸發**：
    - **遠距離 (35m 以外)**：不干預，讓車流自然流動，防止下游擁塞壓縮。
    - **近距離 (35m 以內)**：如果車輛擋住 EV 且無法換道，RSU 才啟動助推，將障礙物「彈開」。

## 實作細節 (Implementation Details)

### 安全性加固 (Crash Prevention)
- **路口保護**：判斷路段關係時自動跳過 SUMO 內部路口 ID (`:junction`)，防止 TraCI 因處理虛擬路段而導致伺服器崩潰。
- **身份鎖定**：修復了 EV 在 Mix Zone 頻繁輪轉偽名的 Bug。EV 保持穩定的 `EV_leader1` 標識，讓 RSU 的優先權追蹤邏輯 100% 準確。

### 性能數據驗證
透過上述邏輯，全系統在超高密度場景下的表現如下：
- **旅行時間**：從基準的 152s 降至 **110s** (縮短 27.6%)。
- **停止次數**：達成全線 **0 次停止**。
- **平均車速**：從 15.5 m/s 提升至 **20.8 m/s**。

## 參數配置

| 參數 | 設定值 | 說明 |
| :--- | :--- | :--- |
| `kEvPrioritySpeedMode` | 7 | 忽略紅燈、忽略安全距離 |
| `kEvPriorityLaneChangeMode` | 8448 | 強制變道 + 允許對向借道 |
| `Lane Reservation Radius` | 120m | 車道鎖定生效半徑 |

---

## 2026-05-03 更新：Tier1（Constrained Cooperative Yielding）

本專案程式碼已將 Tier1 更新為「不隨機失控、但理想讓路」的規則式偏好（deterministic bias），並且與 Full System（Corridor）做到**模式隔離**：

### 模式隔離（不互相污染）

- `enableCorridor = true`：走 Full System corridor（清道/車道保留/較高介入）
- `enableCorridor = false && enableTier1 = true`：走 Tier1 only 路徑，**不會執行 corridor / 強制清道邏輯**

### Tier1 距離分層（以一般車到 EV 的距離 d）

| 距離區間 | 規則層級 | 行為（bias） |
| --- | --- | --- |
| `d < 20m` | Hard constraint | `mustNotBlock` + 偏好變道 |
| `20m ≤ d < 50m` | Strong yield | 偏好變道 + 偏好減速 |
| `50m ≤ d < 120m` | Soft influence | 偏好減速 |

### 核心差異（避免失控）

- **禁止阻擋（mustNotBlock）≠ 強制移開**：不使用 `setSpeed(0)` 清空、也不做全域停車
- **偏好（bias）≠ 隨機（random/probability）**：不引入隨機機率，避免行為爆炸
- **防補位（cooldown）**：車輛剛讓完路後，短時間內禁止立刻切回（避免補位卡住 EV）
