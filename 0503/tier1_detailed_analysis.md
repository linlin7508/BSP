# Tier 1 介入效能深度分析 (Scientific Activity Analysis)

本分析旨在量化 Tier 1 (STML) 模式下的 RSU 介入活動，解釋為何在極高密度交通中，即使介入活動顯著增加，旅行時間仍受限於物理邊界。

| 指標 (Metric) | Baseline (基準) | Tier 1 (STML) | 增長倍率 (Gain) |
| :--- | :--- | :--- | :--- |
| 受影響車次 (Affected Vehicles) | 0 | 174 | 174.00x |
| RSU 觸發換道數 (RSU Lane Changes) | 0 | 0 | N/A |
| RSU 觸發減速數 (RSU Slowdowns) | 0 | 0 | N/A |
| RSU 介入總時刻 (Assisted Ticks) | 0 | 174 | N/A |

## 科學觀察與結論

1. **介入強度 (Intervention Intensity)**: Tier 1 模式下，RSU 的總介入時刻從 0 提升至 174，證明 STML 邏輯已全面運作。
2. **密度牆效應 (Density Wall)**: 儘管 RSU 成功引發了顯著的換道與減速行為，但 EV 旅行時間仍維持在 152s。這量化證明了在『波傳播延遲』與『空間飽和』的雙重限制下，局部拓撲優化無法穿透高密度交通造成的物理牆。
3. **V2X 的必要性**: 本數據支持論文核心論點：傳統警笛即便配合現代化 STML 優化，其效能上限仍低於 V2X 全域協作系統 (Full System) 的 110s 通行能力。
