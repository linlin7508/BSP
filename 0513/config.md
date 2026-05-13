# 模擬配置定義表 (Simulation Configuration Definitions)

本文檔定義了 `omnetpp.ini` 中各個 Config 的用途及其對應的數據輸出文件，用於論文數據整理與實驗複現。

## 1. 核心性能實驗 (Performance Metrics)
這組實驗主要用於評估緊急車輛 (EV) 的通行效率與對交通流的干擾。

| 配置名稱 (Config Name) | 系統定義 (Definition) | 主要功能 (Key Features) | 輸出文件 (Output CSV) |
| :--- | :--- | :--- | :--- |
| **BSP10Hz** | **Full System** | 啟用全部功能：Bypass (繞行), Corridor (走廊清空), Mix Zone (隱私)。 | `paper_metrics_full.csv` |
| **BSP10Hz_Tier1** | **Tier 1 Only** | 僅啟用基礎繞行 (SpeedMode 7)，車輛僅在 EV 接近時做簡單避讓。 | `paper_metrics_tier1.csv` |
| **BSP10Hz_Baseline** | **Baseline** | 基準對照，無任何 RSU 干預，交通流按 SUMO 預設運行。 | `paper_metrics_baseline.csv` |
| **Baseline_NoMZ** | **Safety Baseline** | 用於安全邊界分析，重複跑 100 次以取得統計平均值，禁用 Mix Zone。 | `paper_metrics_common.csv` |

---

## 2. 隱私策略實驗 (Privacy Strategies)
這組實驗用於對比不同隱私保護方案的有效性。

| 配置名稱 (Config Name) | 策略類型 (Strategy) | 邏輯描述 (Logic) | 輸出文件 (Privacy CSV) |
| :--- | :--- | :--- | :--- |
| **Privacy_NoChange** | Strategy 0 | **No Change**: 車輛始終使用靜態 ID (Worst Case)。 | `privacy_events_nochange.csv` |
| **Privacy_Periodic** | Strategy 1 | **Periodic Sync**: 全體車輛每 5秒 強制同步更換 ID，無靜默期。 | `privacy_events_periodic.csv` |
| **Privacy_Proposed** | Strategy 2 | **Proposed Full**: 觸發式 Mix Zone + $\theta$ 閾值 + 1秒 靜默期 (Ts)。 | `privacy_events_proposed.csv` |

---

## 3. 敏感度分析 (Sensitivity Analysis)
針對提議系統 (Proposed System) 的參數進行微調測試。

### A. $\theta$ 閾值影響 (Theta Impact)
| 配置名稱 (Config Name) | $\theta$ 數值 | 用途 | 輸出文件 (Privacy CSV) |
| :--- | :--- | :--- | :--- |
| **Privacy_Proposed_Theta2** | 2 | 最小匿名集合大小。 | `privacy_events_proposed_theta2.csv` |
| **Privacy_Proposed_Theta3** | 3 | 中等匿名要求。 | `privacy_events_proposed_theta3.csv` |
| **Privacy_Proposed_Theta4** | 4 | 較高匿名要求（可能導致觸發頻率下降）。 | `privacy_events_proposed_theta4.csv` |
| **Privacy_Proposed_Theta5** | 5 | 最高匿名要求。 | `privacy_events_proposed_theta5.csv` |

### B. 車流密度影響 (Traffic Density)
| 配置名稱 (Config Name) | 密度等級 | 車輛總數 (Vehicles) | 輸出文件 (Privacy CSV) |
| :--- | :--- | :--- | :--- |
| **Privacy_Proposed_DensityLow** | Low | 20 (需配合 .rou.xml) | `privacy_events_proposed_low.csv` |
| **Privacy_Proposed_DensityMedium**| Medium | 50 (預設) | `privacy_events_proposed_medium.csv` |
| **Privacy_Proposed_DensityHigh** | High | 100 (需配合 .rou.xml) | `privacy_events_proposed_high.csv` |
| **Privacy_NoChange_DensityLow** | Low | 20 (基準對照) | `privacy_events_nochange_low.csv` |
| **Privacy_NoChange_DensityHigh** | High | 100 (基準對照) | `privacy_events_nochange_high.csv` |

---

## 數據保存說明 (Data Persistence)
1. **.sca / .vec 文件**：OMNeT++ 引擎會自動按 `ConfigName-RunNumber` 命名，**不會互相覆蓋**。
2. **CSV 報告**：
   - 所有的 `paper_metrics_*.csv` 與 `privacy_events_*.csv` 均已在 `omnetpp.ini` 中進行了分路徑處理，**不同配置之間不會互相覆蓋**。
   - 同一個配置若跑多次（Run 0, Run 1...），代碼會以 **Append (追加)** 模式寫入同一文件，方便後續用 Python/MATLAB 一次性讀取整組數據。
