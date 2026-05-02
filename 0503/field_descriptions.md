# CSV 欄位說明（examples/easy）

此資料夾的輸出主要來自 RSU 的 10Hz tick logging 與 BSM logging。

---

## 1) `results/paper_metrics_full.csv`

每個 tick 會輸出一列，用於 paper-level 評估（EV 效能 + 系統介入成本）。

| 欄位 | 說明 |
| --- | --- |
| `time` | `simTime()`（秒） |
| `rsu_id` | RSU index |
| `ev_id` | EV 的 external id（找不到時可能為空字串） |
| `speed` | EV 速度（m/s） |
| `position_x`, `position_y` | EV 座標 |
| `lane` | EV lane index（若取不到則 -1） |
| `accel` | EV 加速度（m/s^2） |
| `is_stopped` | EV 本 tick 是否視為停住（speed < 0.1） |
| `stop_count` | EV 累計停住次數（edge detection） |
| `min_speed` | EV 最低速度（若無資料則 -1） |
| `dist_to_rsu` | EV 到 RSU center 的距離（m） |
| `dist_to_tls` | EV 到下一個 TLS 的距離（m），取不到則 -1 |
| `evExists` | 本 tick 是否偵測到 EV |
| `evLatched` | 目前與 `evExists` 相同（保留欄位） |
| `evAssisted` | 本 tick 是否有套用任何 EV assist / Tier1 規則（1/0） |
| `corridorActive` | `enableCorridor` 是否開啟（Full System） |
| `mzActive` | Mix Zone 是否正在運作 |
| `dynamicDist` | EV assist 動態距離/視窗（目前作為 runtime 診斷用） |
| `controlMode` | 文字模式：Baseline / Tier1_Only / Tier_1_and_2 / Full_System 等 |
| `numVehiclesAffected` | 本 tick 介入到的車輛數（累加計數器） |
| `numLaneChanges` | 本 tick 記錄到的變道數（累加計數器） |
| `numForcedStops` | 本 tick 強制停車數（Tier1 版本通常應趨近 0） |
| `numSlowdowns` | 本 tick 減速介入數 |
| `laneIntrusionCount` | 本 tick lane reservation 相關干預（主要屬於 Full System） |

> 註：Tier1 更新後的設計目標是「不全停、不強制清道」，因此 `numForcedStops` 在 Tier1_Only 情境下理論上應大幅下降。

---

## 2) RSU / Vehicle BSM 相關 CSV

若你有開啟對應 logging（例如 BSM row），常見欄位意義如下：

| 欄位 | 說明 |
| --- | --- |
| `subType` | 事件子類型（例如 `START_MZ`, `ENTER_TS`, `BSM`） |
| `mzState` | Mix Zone FSM state（INACTIVE/WAIT_READY/BSP_ACTIVE/SILENCE/DRAINING） |
| `pseudoId` | 車輛目前使用的 pseudonym |
| `anonSet` | Mix Zone 匿名集合大小（若非 MZ 期間可能為 -1） |
| `sessionId` | Mix Zone session id（若非 MZ 期間可能為 -1） |
| `x`, `y` | 車輛座標 |
| `speed`, `acl` | 車輛速度與加速度 |

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

---

## 2026-05-03 更新：Tier1 指標對照（重要）

為了對齊新版程式碼（Tier1 = deterministic bias、且與 corridor 隔離），以下幾個欄位解讀請以「**每個 tick（10Hz）**」為準：

- `numVehiclesAffected`, `numLaneChanges`, `numForcedStops`, `numSlowdowns`, `laneIntrusionCount`：都是 **本 tick 的計數**（不是累積總量）。
- Tier1 only（`enableCorridor=0, enableTier1=1`）的設計目標：
  - `numForcedStops` 應趨近 0（不做全域強制停車）
  - `laneIntrusionCount` 應顯著降低（lane reservation 主要屬於 corridor/Full System）
  - `numSlowdowns`、`numLaneChanges` 反映「偏好」介入（是否成功仍由 SUMO 決定）
field_descriptions.md
