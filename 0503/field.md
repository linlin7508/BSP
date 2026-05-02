# 📄 CSV 欄位說明（統一版本）

本文整理模擬系統輸出的所有 CSV 欄位，並標註不同版本（Full System vs examples/easy / Tier1）之差異。

---

# 1. 📊 `paper_metrics_full.csv`（核心控制與 EV / RSU 指標）

## 🧠 用途

* 系統級效能分析
* RSU 控制行為統計
* EV 行為影響評估
* Tier1 / Full System 對照

---

## 🟦 基礎欄位（所有版本共用）

| 欄位               | 說明                                     |
| ---------------- | -------------------------------------- |
| `time`           | 模擬時間（秒，`simTime()`）                    |
| `rsu_id`         | RSU 編號                                 |
| `controlMode`    | 當前模式（Baseline / Tier1 / Full_System 等） |
| `evExists`       | 是否偵測到 EV（0/1）                          |
| `mzActive`       | Mix Zone 是否啟動                          |
| `corridorActive` | Corridor 系統是否啟用                        |

---

## 🚗 EV 狀態欄位（examples/easy 新增重點）

| 欄位                         | 說明                |
| -------------------------- | ----------------- |
| `ev_id`                    | EV ID（可能為空）       |
| `speed`                    | EV 速度             |
| `accel`                    | EV 加速度            |
| `position_x`, `position_y` | EV 座標             |
| `lane`                     | 車道 index          |
| `is_stopped`               | 是否停止（speed < 0.1） |
| `stop_count`               | 累積停止次數            |
| `min_speed`                | 最低速度              |
| `dist_to_rsu`              | EV → RSU 距離       |
| `dist_to_tls`              | EV → 最近紅綠燈距離      |

---

## ⚙️ RSU 控制行為欄位（Tier1 / Full System）

| 欄位                    | 說明                   |
| --------------------- | -------------------- |
| `evAssisted`          | 本 tick 是否有 EV assist |
| `numVehiclesAffected` | 本 tick 影響車輛數         |
| `numLaneChanges`      | 本 tick lane change 數 |
| `numSlowdowns`        | 本 tick 減速數           |
| `numForcedStops`      | 本 tick 強制停車數         |
| `laneIntrusionCount`  | lane reservation 介入數 |
| `dynamicDist`         | 動態控制距離（debug）        |

---

## 🧪 Tier1 更新重點（2026/05）

### 🟢 設計目標差異

| 指標                   | Tier1 Only     | Full System      |
| -------------------- | -------------- | ---------------- |
| `numForcedStops`     | 🚫 幾乎 0        | ✔ 可能存在           |
| `laneIntrusionCount` | 🚫 很低          | ✔ 高（reservation） |
| `numSlowdowns`       | ✔ soft control | ✔ hard control   |
| `numLaneChanges`     | ✔ bias-based   | ✔ 強制導引           |

---

# 2. 📡 RSU / BSM 日誌（`BSP10Hz-rsu-new.csv`）

## 🧠 用途

* Mix Zone 行為追蹤
* BSM 封包觀測
* 匿名性分析

---

## 📌 事件與狀態欄位

| 欄位          | 說明                              |
| ----------- | ------------------------------- |
| `subType`   | 事件類型（BSM / START_MZ / ENTER_TS） |
| `mzState`   | FSM 狀態（INACTIVE → SILENCE）      |
| `sessionId` | Mix Zone session ID             |
| `pseudoId`  | 車輛 pseudonym                    |

---

## 👥 Mix Zone 專用欄位

| 欄位                 | 說明           |
| ------------------ | ------------ |
| `anonSet`          | 匿名集合大小 N     |
| `total_members`    | session 成員總數 |
| `ready_count`      | ACK 車輛數      |
| `quorum_triggered` | 是否 quorum 觸發 |

---

## 📍 感測資料

| 欄位             | 說明       |
| -------------- | -------- |
| `x`, `y`       | RSU/車輛位置 |
| `speed`, `acl` | 動態狀態     |

---

# 3. 🚘 車輛軌跡（`trip_full.csv`）

## 🧠 用途

* 車輛行為分析
* 隱私 / 混淆分析
* EV interaction tracking

---

## 🟦 車輛識別與狀態

| 欄位          | 說明          |
| ----------- | ----------- |
| `vehIndex`  | 車輛 ID       |
| `pseudoId`  | 假名 ID（可變）   |
| `inSilence` | 是否靜默（Ts）    |
| `mzActive`  | Mix Zone 狀態 |

---

## 🚗 EV interaction

| 欄位       | 說明                |
| -------- | ----------------- |
| `evDist` | 與 EV 距離           |
| `ttc`    | Time to Collision |
| `tts`    | Time to Stop      |

---

## 🚙 車流關係

| 欄位             | 說明   |
| -------------- | ---- |
| `distToLeader` | 前車距離 |
| `leaderSpeed`  | 前車速度 |

---

## ⏱ Phase timing

| 欄位           | 說明       |
| ------------ | -------- |
| `phaseStart` | BSP 起始   |
| `phaseEnd`   | BSP 結束   |
| `duration`   | phase 時長 |

---

# 4. 🔐 隱私評估（`privacy_metrics.csv`）

## 🧠 用途

* Mix Zone privacy quantification
* anonymity effectiveness

---

| 欄位         | 說明                 |
| ---------- | ------------------ |
| `AS_size`  | 匿名集合大小 N           |
| `SR`       | Success Rate = 1/N |
| `entropy`  | log2(N)            |
| `duration` | session 持續時間       |

---

#  版本差異總整理（重點）

## 🆕 examples/easy 新增

* EV 直接 tracking（`ev_id`, `lane`, `accel`）
* road safety metrics（`is_stopped`, `stop_count`）
* infrastructure distance（TLS / RSU）
* Tier1 control breakdown 更細

---

## ⚙️ Full system（原始版）

* 偏 RSU / Mix Zone internal state
* 強調 FSM / quorum / session control
* 車輛資訊較少

---

## 🧠 設計哲學差異

| 系統          | 特性                       |
| ----------- | ------------------------ |
| Full System | 偏「隱私控制系統內部」              |
| Tier1/easy  | 偏「交通行為 + EV impact」      |
| 合併後版本       | 可做論文 evaluation pipeline |

