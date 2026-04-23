# Mix Zone 啟動與結束生命週期邏輯

本文檔說明 RSU 如何管理 Mix Zone 階段的轉換與車輛同步。

## 生命週期階段 (FSM States)

模擬中的 Mix Zone 遵循五階段有限狀態機 (FSM)：

1.  **INACTIVE**: 初始狀態，監控 EV 距離。
2.  **WAIT_READY**: 已偵測到 EV，廣播啟動指令，等待車輛確認 (ACK)。
3.  **BSP_ACTIVE (Tb)**: 廣播階段，車輛在此期間發送最後一批舊假名 BSM，並準備進入靜默。
4.  **SILENCE (Ts)**: 靜默階段，所有參與車輛停止發送 BSM。
5.  **DRAINING**: 結束階段，等待車輛恢復發送新假名或超時。

## 啟動邏輯 (Start Logic)

-   **觸發函數**: `rsu::evaluateTrigger()`
-   **條件**:
    1.  EV 與 RSU 的距離 $\le$ `mzTrigger` (120m)。
    2.  當前範圍內的候選車輛數 $\ge \theta$ (最小匿名門檻)。
-   **啟動動作**: 
    - 呼叫 `startMixZone()`。
    - 鎖定當前成員 (Frozen Set)。
    - 生成該 Session 專用的新假名。
    - 發送 `CMD_START_BSP` 指令給車輛。

## 同步與靜默邏輯 (Sync & Silence)

-   **同步點**: `rsu::triggerBspPhase()`
    - 當 90% 的車輛回覆 `COMM_READY` 或 4 秒計時器結束時觸發。
-   **靜默開始**: `ENTER_TS`
    - RSU 廣播 `CMD_START_SILENCE`。
    - 車輛進入 `inSilence = 1` 狀態。

## 結束邏輯 (End Logic)

-   **結束函數**: `rsu::deactivateMixZone()`
-   **觸發條件**:
    1.  EV 離開 RSU 覆蓋範圍。
    2.  或是 Session 持續時間超過 `mzDuration`。
-   **結束動作**:
    - 廣播 `MZ_RESUME` 指令。
    - 車輛恢復 BSM 發送並切換至預先計算好的新假名。
    - 清除 RSU 狀態，紀錄隱私指標。

## 生命周期關鍵參數表

| 參數名稱 | 數值 | 單位 | 說明 |
| :--- | :--- | :--- | :--- |
| `mzTrigger` | 120 | m | 觸發 Mix Zone 的 EV 距離 |
| `theta` | 3 (依設定) | 輛 | 最低匿名集合大小門檻 |
| `bspBroadcast` (Tb) | 4 | s | 進入靜默前的廣播同步時間 |
| `bspSilence` (Ts) | 1 | s | 車輛停止通訊的靜默時長 |
| `commReadyTimeout` | 4 | s | 等待車輛 ACK 的最長時間 |
