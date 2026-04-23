# 繞路與廊道策略代碼邏輯 (Rerouting & Corridor Logic)

本文檔說明模擬中用於提升緊急車輛 (EV) 通行效率的繞路與交通管理邏輯。

## 核心邏輯步驟

1.  **EV 偵測 (EV Detection)**:
    RSU 定期掃描範圍內的車輛，識別帶有特定前綴 (如 `EV_`) 的緊急車輛。
    - **核心函數**: `rsu::handleSelfMsg` 中的 EV 掃描循環。

2.  **路徑清理觸發 (Corridor Triggering)**:
    當 EV 進入 RSU 的候選半徑 (`candidateRadius`) 時，啟動廊道清理模式。
    - **核心函數**: `rsu::hold(bool corridorEnable, bool evAssistEnable)`

3.  **前方車輛清理 (Vehicle Clearing)**:
    針對在 EV 前方且位於相同路徑上的車輛，執行避讓指令。
    - **核心函數**: `rsu::applyCorridorClearing`
    - **邏輯**:
        - 若車輛與 EV 在同車道：執行強制換道 (`tryForcedLaneChange`)。
        - 若無法換道或接近路口：執行減速 (`boundedSlowDown`) 或停在停止線 (`applyGapOnlyHold`)。

4.  **EV 優先權輔助 (EV Assistance)**:
    調整 EV 自身的行駛參數，使其能以更小的跟車距離與更積極的方式通過。
    - **核心函數**: `rsu::applyEvAssist`
    - **邏輯**: 調整 `tau` (反應時間) 與 `minGap` (最小間距)，並在紅燈時嘗試安全越線。

## 核心函數說明

| 函數名稱 | 功能描述 | 關鍵參數 |
| :--- | :--- | :--- |
| `applyCorridorClearing` | 判斷前方車輛位置並下達避讓指令 | `kCorridorLookaheadMeters` (120m) |
| `tryForcedLaneChange` | 強制車輛切換至相鄰車道 | `kLaneChangeDurationSec` (3s) |
| `applyEvAssist` | 優化 EV 行駛行為，減少前車阻礙 | `kEvAggressiveTauSec` (0.3s) |
| `boundedSlowDown` | 平滑減速指令，避免引發後車碰撞 | `kMaxComfortDecelMps2` (3.5 m/s²) |

## 策略效果表 (數據源自最近一次模擬)

| 指標項目 | 數據值 | 單位 | 說明 |
| :--- | :--- | :--- | :--- |
| **EV 平均時速 (Full System)** | 17.87 | m/s | 啟用繞路策略後的表現 |
| **EV 平均時速 (Baseline)** | 15.49 | m/s | 未啟用策略的基準表現 |
| **強制換道次數 (Lane Changes)** | 9 | 次 | RSU 成功引導車輛避讓的次數 |
| **減速介入次數 (Slowdowns)** | 198 | 次 | RSU 執行速度限制以騰出空間的次數 |
| **受影響車輛總數** | 753 | 輛次 | 模擬期間受到交通管理的普通車輛總量 |
