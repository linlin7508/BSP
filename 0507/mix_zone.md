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
| `sirenWaveSpeed` | 100 | m/s | 警笛感知/傳播速度 (Tier 1) |
| `reactionDelayRange` | 0.3 - 0.8 | s | 駕駛人隨機反應延遲 (Tier 1) |

## Tier 1 寫實化避讓模型 (Semi-Realistic Siren Model)

為了避免「所有車輛同時避讓」的完美作弊現象，系統實作了基於物理傳播與生理反應的延遲模型：

1.  **波傳播延遲 (Wave Propagation)**：避讓指令隨距離擴散，$Delay_{wave} = Distance / 100.0$。這模擬了警笛聲傳播或駕駛人感知的過程，使得近處車輛先反應，遠處車輛後反應。
2.  **反應延遲 (Reaction Delay)**：每位駕駛隨機產生 $0.3 \sim 0.8$ 秒的反應時間，增加模擬的隨機性與真實度。
3.  **弱推動與安全限制**：
    -   **紅燈推動**：僅限於路口 30m 內，使用 `SpeedMode(7)` (Collision Aware) 以 $40\%$ 最高速緩慢推進，避免暴衝。
    -   **弱強制換道**：使用 `LaneChangeMode(512)`，在保證不撞車的前提下嘗試偏移。
    -   **防補位 (Anti-refill)**：避讓後進入 3 秒冷卻期，防止車輛立即切回原車道。


## 實測結果表 (Simulation Results)

以下為優化後 Full System 模擬中，EV 經過各個 RSU 時的統計數據：

| RSU | Approach Time (s) | Leave Time (s) | Unique Other Vehicles (In 120m) | Max Simultaneous Others |
| --- | --- | --- | --- | --- |
| RSU[3] | 155.0 | 165.0 | 3 | 6 |
| RSU[4] | 165.0 | 171.0 | 1 | 3 |
| RSU[7] | 124.0 | 136.0 | 7 | 20 |
| RSU[8] | 136.0 | 155.0 | 8 | 20 |
| RSU[11] | 95.0 | 108.0 | 9 | 20 |
| RSU[12] | 109.0 | 124.0 | 6 | 18 |
| RSU[15] | 67.0 | 81.0 | 9 | 29 |
| RSU[16] | 81.0 | 95.0 | 9 | 29 |
| RSU[20] | 61.0 | 66.0 | 0 | 0 |

## 全系統性能對比評估 (Overall Performance Evaluation)

### 4.1 場景 A：車禍堵塞 (事故封閉，單線道瓶頸)
*場景描述：模擬事故導致主幹道縮減，測試 RSU 如何引導 EV 穿透物理瓶頸。*

| 測試情境 (Case) | 總旅行時間 (s) | 平均速度 (m/s) | 總停止時間 (s) | 速度標準差 (平順度) |
| :--- | :--- | :--- | :--- | :--- |
| Baseline (基準) | 152.0 | 15.49 | 17.0 | 9.85 |
| Tier 1 Only (僅警笛) | 152.0 | 15.49 | 17.0 | 9.85 |
| **Full System (全域協作)** | **110.0** | **20.80** | **0.0** | **5.37** |

### 4.2 場景 B：高密度擁堵 (無事故，10Hz 高頻偵測)
*場景描述：測試 100 輛背景車造成的極端交通壓力，對比局部避讓與全域調度的極限。*

| 測試情境 (Case) | 總旅行時間 (s) | 平均速度 (m/s) | 停止次數 (Stops) | 速度標準差 (平順度) |
| :--- | :--- | :--- | :--- | :--- |
| Baseline (基準) | 157.9 | 14.76 | 243 | 9.87 |
| Tier 1 Only (僅警笛) | 153.0 | 15.20 | 226 | 8.42 |
| **Full System (全域協作)** | **107.3** | **21.70** | **0.0** | **5.77** |

**科學結論**：透過 **「車道保留 (Lane Reservation)」** 與 **「對向借道 (Opposite-lane Driving)」** 的協同作用，EV 在納入 **「波傳播延遲」** 與 **「反應延遲」** 等寫實約束後，仍成功在超高密度車流中達成了 **零停頓 (Zero-stop)** 通行，旅行時間縮短了 **27.6%**。這證明了 V2X 協作式控制在應對極端交通壓力時，遠優於傳統的警笛 (Tier 1) 避讓模式。

## 5. Tier 1 介入效能深度分析 (Scientific Activity Analysis)

本分析旨在量化 Tier 1 (STML) 模式下的 RSU 介入活動，並對比「事故封閉」與「正常擁堵」兩種極端場景下的表現。

### 5.1 實驗場景 A：車禍堵塞 (事故封閉，單線道瓶頸)
*場景描述：模擬 73s-123s 間發生事故，導致主幹道完全封鎖或縮減為單線道。*

| 指標 (Metric) | Baseline (基準) | Tier 1 (STML) | 效能表現 |
| :--- | :--- | :--- | :--- |
| 總旅行時間 (s) | 152.0 | 152.0 | **無顯著提升** |
| 受影響車次 (Affected) | 0 | 232 | RSU 有介入但無法穿透瓶頸 |
| 停止次數 (Stops) | 高 | 高 | 被物理障礙完全阻擋 |

**結論**：在「物理斷路」情況下，單純的局部避讓（Tier 1）無法解決問題，因為前方車輛即便想讓路也無處可去。

### 5.2 實驗場景 B：高密度擁堵 (無事故，10Hz 高頻偵測)
*場景描述：移除人為事故，純粹測試 100 輛背景車輛造成的高密度流量，並開啟 RSU 10Hz (0.1s) 偵測。*

| 指標 (Metric) | Baseline (基準) | Tier 1 (STML) | 改善程度 (Gain) |
| :--- | :--- | :--- | :--- |
| **總旅行時間 (s)** | 157.9 | **153.0** | **-4.9s (↓3.1%)** |
| **平均速度 (m/s)** | 14.76 | 15.20 | +0.44 m/s |
| **停止次數 (Stops)** | 243 | 226 | -17 次停止 |
| 受影響車次 (Affected) | 0 | 404 | 介入強度提升 1.7x |
| RSU 觸發換道 (LC) | 0 | 80 | 顯著引發避讓行為 |

### 5.3 科學觀察與結論 (Scientific Findings)

1. **偵測頻率的重要性**: 將 RSU 偵測與 BSM 頻率由 1Hz 提升至 10Hz 後，Tier 1 的介入次數從 232 提升至 404。這證明了「看見」EV 是所有優化邏輯的前提。
2. **避讓行為的量化**: 在無事故場景下，Tier 1 成功引發了 **80 次強制換道**，這直接減少了 EV 的停止次數（由 243 降至 226），並節省了約 5 秒的旅行時間。
3. **性能瓶頸 (The Density Wall)**: 即便有 80 次換道介入，EV 仍面臨 226 次停止。這量化證明了「局部避讓」在極高密度下存在極限。
4. **全域協作的優勢**: 對比 **Full System (107.3s, 0 Stops)**，可以看出全域調度（Corridor Control）與無視規則（SpeedMode 31）才是穿透「高密度牆」的最終解決方案。
