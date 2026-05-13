# 實驗配置定義 (Experimental Configurations)

本文件定義了所有實驗配置的變數設置與預期行為。

## 1. 核心性能實驗 (Core Performance)
這組實驗用於驗證 BSP 協議在不同控制等級下對 EV 通行效率與交通安全的影響。

| 配置名稱 (Config Name) | 輔助邏輯 (Assistance) | 隱私策略 (Privacy) | 輸出文件 (Performance CSV) |
| :--- | :--- | :--- | :--- |
| **BSP10Hz** | **Full System**: Bypass + Corridor | Strategy 2 (Proposed) | `paper_metrics_full.csv` |
| **BSP10Hz_Tier1** | **Tier 1 Only**: Siren-based only | Strategy 2 (Proposed) | `paper_metrics_tier1.csv` |
| **BSP10Hz_Baseline** | **Baseline**: No RSU assistance | Strategy 2 (Proposed) | `paper_metrics_baseline.csv` |
| **Baseline_NoMZ** | **Worst Case**: No assistance + No Privacy | Strategy 0 (NoChange) | `paper_metrics_baseline_nomz.csv` |

---

## 2. 隱私策略實驗 (Privacy Strategies)
這組實驗用於對比不同隱私保護方案的有效性。

| 配置名稱 (Config Name) | 策略類型 (Strategy) | 邏輯描述 (Logic) | 輸出文件 (Privacy CSV) |
| :--- | :--- | :--- | :--- |
| **Privacy_NoChange** | Strategy 0 | No Change: 車輛始終使用靜態 ID (Worst Case)。 | `privacy_events_nochange.csv` |
| **Privacy_Periodic** | Strategy 1 | Periodic Sync: 全體車輛每 5秒 強制同步更換 ID，無靜默期。 | `privacy_events_periodic.csv` |
| **Privacy_Proposed** | Strategy 2 | Proposed Full: 觸發式 Mix Zone + $\theta$ 閾值 + 1秒 靜默期 ($T_s$)。 | `privacy_events_proposed.csv` |

### $\theta$ 閾值敏感度分析 (Sensitivity Analysis)
*   `Privacy_Proposed_Theta2` ($\theta=2$): 輸出至 `privacy_events_theta2.csv`
*   `Privacy_Proposed_Theta3` ($\theta=3$): 輸出至 `privacy_events_theta3.csv`
*   `Privacy_Proposed_Theta4` ($\theta=4$): 輸出至 `privacy_events_theta4.csv`

---

## 3. 交通密度實驗 (Traffic Density)
這組實驗用於觀察不同交通壓力下，系統的擴展性與隱私混淆效果。

| 配置名稱 (Config Name) | 密度等級 | 車輛總數 (Vehicles) | 輸出文件 (Privacy CSV) |
| :--- | :--- | :--- | :--- |
| **Privacy_Proposed_DensityLow** | Low | 20 (需配合 .rou.xml) | `privacy_events_proposed_low.csv` |
| **Privacy_Proposed_DensityMedium**| Medium | 50 (預設) | `privacy_events_proposed_medium.csv` |
| **Privacy_Proposed_DensityHigh** | High | 100 (需配合 .rou.xml) | `privacy_events_proposed_high.csv` |
| **Privacy_NoChange_DensityLow** | Low | 20 (基準對照) | `privacy_events_nochange_low.csv` |
| **Privacy_NoChange_DensityHigh** | High | 100 (基準對照) | `privacy_events_nochange_high.csv` |

---

## 4. 攻擊者模型定義 (Attacker Model)
本實驗採用最嚴苛的攻擊者假設，以測試系統的隱私防護邊界。

*   **類型 (Type)**：誠實但好奇的基礎設施 (Honest-but-Curious Infrastructure / RSU)。
*   **能力 (Capabilities)**：
    *   **本地全知 (Local Omniscience)**：攻擊者監聽 RSU 範圍內所有車輛發送的 BSM 訊息。
    *   **軌跡重建 (Trajectory Reconstruction)**：攻擊者會根據時間與空間的連續性，記錄每個車輛 ID 的移動軌跡。
    *   **偽名關聯 (Pseudonym Linking)**：當車輛因 Mix Zone 或週期性更換而變更 ID 時，攻擊者使用 **最近鄰算法 (Nearest Neighbor Algorithm)** 嘗試將新 ID 關聯回舊 ID。
*   **攻擊目標**：突破 Mix Zone 的匿名性，實現對特定車輛的長距離追蹤。
*   **強度備註**：目前攻擊者在低密度場景下極強（TSR 接近 1.0），主要壓力測試點在於 **高車流密度 (High Density)** 下，混淆效果是否能有效降低攻擊者的關聯成功率。

---

## 數據保存說明 (Data Persistence)
1. **.sca / .vec 文件**：OMNeT++ 引擎會自動按 `ConfigName-RunNumber` 命名，**不會互相覆蓋**。
2. **CSV 報告**：
   - 所有的 `paper_metrics_*.csv` 與 `privacy_events_*.csv` 均已在 `omnetpp.ini` 中進行了分路徑處理，**不同配置之間不會互相覆蓋**。
   - 同一個配置若跑多次（Run 0, Run 1...），代碼會以 **Append (追加)** 模式寫入同一文件，方便後續用 Python/MATLAB 一次性讀取整組數據。
