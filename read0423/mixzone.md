# Mix Zone 啟動與結束生命週期邏輯

本文檔說明 RSU 如何管理 Mix Zone 階段的轉換與車輛同步，以及最新的假名池隔離機制。

## 生命週期階段 (FSM States)

模擬中的 Mix Zone 遵循五階段有限狀態機 (FSM)：

1.  **INACTIVE**: 初始狀態，監控 EV 距離。
2.  **WAIT_READY**: 已偵測到 EV，廣播啟動指令，等待車輛確認 (ACK)。
3.  **BSP_ACTIVE (Tb)**: 廣播階段，車輛在此期間發送最後一批舊假名 BSM，並準備進入靜默。
4.  **SILENCE (Ts)**: 靜默階段，所有參與車輛停止發送 BSM。
5.  **DRAINING**: 排空階段 (Grace Period)。EV 離開後，RSU 不會立刻終止 Session，而是每隔一定時間 (預設 4 秒) 為尚未離開的車輛無靜默發送新假名，直到車輛駛離或超時。

## 三階層假名池隔離設計 (Tiered Pseudonym Pools)

為防止觀察者透過假名來源進行關聯分析，系統實作了三個獨立的假名池：

1.  **EV_Pseud.txt**: **領頭車 (EV) 專用池**。EV 始終使用此池，即使進入 Mix Zone 也不改變來源，確保最高級別的憑證安全。
2.  **Veh_pseud.txt**: **普通車日常池**。普通車在 Mix Zone 覆蓋範圍外進行例行性假名更換時使用。
3.  **pseudonyms.txt**: **Mix Zone 專用池**。RSU 啟動 BSP 或在 DRAINING 階段時，專門從此池為普通車提取並分配假名。

> [!NOTE]
> `PseudonymPool` 實作了切片機制 (Slicing)，確保車輛自主更換與 RSU 分配時，不會從同一個假名文件的相同區塊提取，徹底避免假名碰撞 (Collision)。

## 啟動邏輯 (Start Logic)

-   **觸發函數**: `rsu::evaluateTrigger()`
-   **條件**:
    1.  EV 與 RSU 的距離 $\le$ `mzTrigger` (150m)。
    2.  當前範圍內的候選車輛數 $\ge \theta$ (最小匿名門檻)。
-   **啟動動作**: 
    - 呼叫 `startMixZone()`。
    - 鎖定當前成員 (Frozen Set)。
    - 生成該 Session 專用的新假名。
    - 發送 `CMD_START_BSP` 指令給車輛。

## 同步與靜默邏輯 (Sync & Silence)

-   **同步點**: `rsu::triggerBspPhase()`
    - 當 90% 的車輛回覆 `COMM_READY` 或計時器結束時觸發。
-   **靜默開始**: `ENTER_TS`
    - RSU 廣播 `CMD_START_SILENCE`。
    - 車輛進入 `inSilence = 1` 狀態。

## 結束邏輯與無靜默換假名 (Grace Period Rotation)

-   **觸發條件**: EV 離開 RSU 覆蓋範圍 (`!evExists`)。
-   **排空邏輯 (DRAINING)**:
    - RSU 啟動 `nextDrainingRotation_` 計時器。
    - 每隔 `drainingRotationInterval` (預設 4 秒)，RSU 發送 `UPDATE_PSEUDONYM`。
    - 車輛收到後立即切換假名並持續廣播（**無靜默期**），確保在駛離 Mix Zone 邊緣時不會被輕易追蹤。
-   **徹底結束**: `rsu::deactivateMixZone()`
    - 所有車輛離開半徑或經過 `mzDuration` (60s) 超時。
    - 廣播 `RESUME_BROADCAST`，車輛恢復使用 `Veh_pseud.txt` 日常池。

## 生命周期關鍵參數表

| 參數名稱 | 數值 | 單位 | 說明 |
| :--- | :--- | :--- | :--- |
| `mzTrigger` | 150 | m | 觸發 Mix Zone 的 EV 距離 |
| `theta` | 2 (依設定) | 輛 | 最低匿名集合大小門檻 |
| `bspBroadcast` (Tb) | 4 | s | 進入靜默前的廣播同步時間 |
| `bspSilence` (Ts) | 1 | s | 車輛停止通訊的靜默時長 |
| `commReadyTimeout` | 4 | s | 等待車輛 ACK 的最長時間 |
| `drainingRotationInterval` | 4 | s | DRAINING 期間無靜默換假名間隔 |

## 實測結果表 (Simulation Results)

以下為最近一次 Full System 模擬中，EV 經過各個 RSU 時的 Mix Zone 啟動與周邊車輛統計數據（由 `calc_mz_table.py` 從 `trip_full.csv` 計算）：

- **Unique Other Vehicles**: 整段 EV 通過期間，曾出現在 120m 內的**不重複**普通車輛數
- **Max Simultaneous Others**: 任一時刻，**同時**在場的普通車最大數量（恆 ≤ Unique）

| RSU | Approach Time (s) | Leave Time (s) | Unique Other Vehicles (In 120m) | Max Simultaneous Others |
| --- | --- | --- | --- | --- |
| RSU[3] | 265.0 | 286.0 | 0 | 0 |
| RSU[4] | 295.0 | 311.0 | 0 | 0 |
| RSU[7] | 160.0 | 172.0 | 7 | 6 |
| RSU[8] | 177.0 | 256.0 | 7 | 5 |
| RSU[11] | 102.0 | 114.0 | 10 | 7 |
| RSU[12] | 118.0 | 156.0 | 13 | 8 |
| RSU[15] | 69.0 | 78.0 | 9 | 9 |
| RSU[16] | 82.0 | 98.0 | 9 | 9 |
| RSU[20] | 61.0 | 65.0 | 0 | 0 |
