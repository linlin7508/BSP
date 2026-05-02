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

# 🔄 RSU 智能清理系統 - 版本差異整理

---

# 1. 🧠 系統核心哲學變更（最重要）

## 🟦 舊版（Rerouting & Corridor-first）

### 核心思想

```text id="v8kq2p"
EV = center
RSU = proactive traffic rewriter
```

### 特徵

* 強制清空（force clearing）
* 強制換道
* 高介入交通控制
* 假設：

  > traffic can be fully controlled

---

## 🟩 新版（Trinity Control / Tiered System）

### 核心思想

```text id="x1wq9k"
EV priority + bounded intervention + safety constraint system
```

### 特徵

* 不再全域控制
* 改為三層控制：

  * Lane Reservation
  * Opposite-lane assist
  * Strategic Boost

---

### 🔥 本質差異

| 面向   | 舊版       | 新版                  |
| ---- | -------- | ------------------- |
| 控制哲學 | 強制清空     | 約束式優化               |
| 車流假設 | 可完全重寫    | 只能局部影響              |
| 安全模型 | implicit | explicit constraint |

---

# 2. 🚧 Lane Reservation（重大變更）

---

## 🟨 舊版

### 行為：

* 無顯式 lane lock
* 用：

  * forced lane change
  * slowdown
  * stop

👉 問題：

* 會造成：

  * lane oscillation
  * deadlock risk

---

## 🟩 新版（關鍵升級）

### 機制：

```text id="6q9x0a"
LaneChangeMode(512)
```

### 行為：

* 鎖車道
* 阻止 cut-in
* 建立 "virtual corridor"

---

### 🔥 本質提升

| 項目            | 舊版       | 新版          |
| ------------- | -------- | ----------- |
| 車道穩定性         | 低        | 高           |
| deadlock risk | 存在       | 幾乎消除        |
| EV 前方清空       | reactive | pre-emptive |

---

# 3. 🚗 Opposite-lane Mechanism（重大新增）

---

## 🟨 舊版

* 沒有真正 opposite lane model
* 僅 forward clearing

---

## 🟩 新版（新增核心能力）

### 概念：

```text id="4kzq9r"
EV can temporarily use opposite lane
```

### RSU 行為：

* detect EV intention
* push opposite traffic away

---

### 🔥 影響

| 面向                    | 舊版       | 新版       |
| --------------------- | -------- | -------- |
| deadlock handling     | weak     | strong   |
| intersection conflict | possible | resolved |
| EV flexibility        | limited  | high     |

---

# 4. ⚡ Speed Control Strategy 變更

---

## 🟨 舊版（暴力型）

```text id="9xq1we"
setSpeed(0.1)
setSpeed(max)
forcedStop
```

### 問題：

* shockwave traffic
* rear collision risk
* unrealistic behavior

---

## 🟩 新版（Strategic Boost）

### 改為：

```text id="m2k9rz"
SpeedMode(0) + adaptive max boost
```

---

### 分層控制：

| 距離       | 行為              |
| -------- | --------------- |
| >35m     | no intervention |
| <35m     | boost / push    |
| blocking | lateral escape  |

---

### 🔥 改進

| 項目                | 舊版       | 新版         |
| ----------------- | -------- | ---------- |
| traffic stability | poor     | smooth     |
| safety realism    | low      | high       |
| control type      | discrete | continuous |

---

# 5. 📊 Performance Metrics 差異（非常重要）

---

## 🟨 舊版數據

| 指標                     | 問題   |
| ---------------------- | ---- |
| 152s → 110s            | ✔ OK |
| 0 stop                 | ✔ OK |
| 但：                     |      |
| ❌ intervention cost未解釋 |      |

---

## 🟩 新版數據（加入語意層）

### 新增解釋：

| 指標                | 意義                      |
| ----------------- | ----------------------- |
| affected vehicles | system footprint        |
| lane changes      | control aggressiveness  |
| slowdown count    | soft intervention ratio |

---

### 🔥 差異

| 面向               | 舊版   | 新版     |
| ---------------- | ---- | ------ |
| interpretability | low  | high   |
| paper value      | weak | strong |

---

# 6. 🧩 系統安全模型變更

---

## 🟨 舊版

* safety = implicit
* rely on:

  * forced stop
  * deceleration

---

## 🟩 新版

### explicit safety rules：

* 路口排除 (`:junction skip`)
* EV identity lock (`EV_leader1`)
* TraCI crash prevention

---

### 🔥 改進

| 項目                   | 舊版       | 新版        |
| -------------------- | -------- | --------- |
| simulation stability | medium   | high      |
| reproducibility      | low      | high      |
| crash risk           | possible | minimized |

---

# 7. 📈 數據來源可信度（重大差異）

---

## 🟨 舊版

* metrics：

  * hand collected
  * partially inferred

---

## 🟩 新版

### pipeline：

```text id="c1q8xz"
SUMO logs
→ paper_metrics_full.csv
→ analysis scripts
→ tables
```

---

### 🔥 upgrade

| 項目              | 舊版        | 新版                |
| --------------- | --------- | ----------------- |
| reproducibility | weak      | strong            |
| auditability    | low       | high              |
| research grade  | prototype | publication-ready |

---

# 8. 🧠 系統層級差異總結

---

## 🟦 舊版模型

```text id="v2k9xa"
RSU = traffic rewriter
```

---

## 🟩 新版模型

```text id="p9q1ld"
RSU =
  multi-layer controller:
    - lane reservation layer
    - speed shaping layer
    - opposite lane coordination layer
    - EV priority arbitration
```
