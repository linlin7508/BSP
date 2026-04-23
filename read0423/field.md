# 模擬輸出檔案欄位說明

本文檔詳細說明模擬生成的 CSV 檔案中各欄位的含義與取值邏輯。

## 1. 核心指標日誌 (`paper_metrics_full.csv`)
記錄模擬全局與各 RSU 的即時控制狀態。

| 欄位名稱 | 說明 | 來源/計算方式 |
| :--- | :--- | :--- |
| `time` | 模擬時間 | `simTime()` |
| `rsu_id` | RSU 編號 | RSU 索引值 (0-24) |
| `corridorActive` | 廊道管理是否開啟 | RSU `enableCorridor` 布林值 |
| `mzActive` | Mix Zone 是否正在運行 | RSU `mzActive_` 狀態旗標 |
| `numLaneChanges` | 強制換道累計次數 | 成功執行 `tryForcedLaneChange` 的計數 |
| `numSlowdowns` | 減速指令累計次數 | 成功執行 `boundedSlowDown` 的計數 |
| `numVehiclesAffected` | 受控車輛總量 | 受到任何 RSU 指令影響的車輛次數 |
| `tickLaneChanges` | 本次 Tick 換道數 | 單次 10Hz Tick 內的介入次數 |
| `tickSlowdowns` | 本次 Tick 減速數 | 單次 10Hz Tick 內的介入次數 |
| `evExists` | EV 是否在範圍內 | 1: RSU 偵測到 EV, 0: 無 EV |
| `controlMode` | 當前實驗模式 | 基於 `enableMixZone/enableCorridor` 的字串 |


## 2. RSU 事件日誌 (`BSP10Hz-rsu-new.csv`)
記錄 RSU 內部的狀態變更與 BSM 快照。

| 欄位名稱 | 說明 | 來源/計算方式 |
| :--- | :--- | :--- |
| `subType` | 事件子類型 | 如 `START_MZ`, `ENTER_TS`, `BSM` 等 |
| `mzState` | RSU 狀態機狀態 | 0:INACTIVE, 1:WAIT_READY, 2:BSP_ACTIVE, 3:SILENCE... |
| `pseudoId` | 車輛當前假名 | 來自車輛發送的 BSM 封包 |
| `anonSet` | 匿名集合大小 | 觸發 Mix Zone 時鎖定的成員數量 |
| `sessionId` | 會話唯一的識別碼 | RSU 索引與序列號組合而成 |
| `ready_count` | 已 ACK 的車輛數 | `COMM_READY` 階段收到的回覆數量 |
| `total_members` | 總鎖定成員數 | 會話開始時固定的車輛總數 |
| `quorum_triggered` | 是否由 Quorum 觸發 | 1: 多數車輛 ACK 觸發, 0: 超時觸發 |
| `x, y` | RSU 或觀測位置 | 座標位置 |
| `speed, acl` | 車輛即時動態 | 速度與加速度 |


## 3. 車輛行蹤與隱私日誌 (`trip_full.csv`)
記錄每輛車在每個時間步的詳細數據。

| 欄位名稱 | 說明 | 來源/計算方式 |
| :--- | :--- | :--- |
| `vehIndex` | 車輛 ID | 模擬內部的車輛索引 |
| `inSilence` | 是否處於靜默期 | 1: 正在靜默 (Ts), 0: 正常通訊 |
| `evDist` | 與最近 EV 的距離 | 車輛位置與 EV 位置的歐式距離 |
| `ttc` | 碰撞預警時間 (Time to Collision) | 基於前車距離與相對速度計算 |
| `tts` | 停止預警時間 (Time to Stop) | 車輛停止行駛所需的預計時間 |
| `pseudoId` | 車輛當前使用的假名 | 隨時間變化的假名 ID (如 `PSEU-` 或 `mz_`) |
| `mzActive` | Mix Zone 啟用狀態 | 1: 受 RSU 指令啟動中, 0: 未啟動 |
| `phaseStart, phaseEnd` | BSP 階段時間 | 當前 BSP 循環的預計起始與結束時間 |
| `duration` | 階段持續時長 | $phaseEnd - phaseStart$ |
| `evDist` | 與 EV 的實時距離 | 當前車輛與 EV 的相對距離 |
| `distToLeader` | 與前方引導車距離 | 基於 TraCI 取得的前車間距 |
| `leaderSpeed` | 前方引導車速度 | 前車的即時速率 |


## 4. 隱私評估日誌 (`privacy_metrics.csv`)
專門記錄每個 Mix Zone Session 的隱私收益。

| 欄位名稱 | 說明 | 來源/計算方式 |
| :--- | :--- | :--- |
| `AS_size` | 匿名集合大小 | 該會話鎖定的參與車輛數 (N) |
| `SR` | 猜中率 (Success Rate) | $1 / N$ |
| `entropy` | 隱私熵 (Entropy) | $\log_2(N)$ |
| `duration` | 會話持續時長 | $End\_Time - Start\_Time$ |
