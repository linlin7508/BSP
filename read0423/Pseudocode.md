# V2X EV Priority & Privacy System - Paper Resources

這份文件為論文寫作提供了演算法虛擬碼 (Pseudocodes)、實驗配置表格版型，以及實驗結果預期表格，確保您的論文在論述與量化指標上具備最高完整性。

---

## 1. Algorithms (Pseudocode)

### Algorithm 1: Main Control Loop (RSU 10Hz Tick)
```text
Algorithm 1: Continuous RSU Control Evaluation
Input: Current Tick t, List of Vehicles V, RSU Coverage R
Output: Control Signals & Privacy Status

1: Reset per-tick tracking variables (affectedCount, laneChanges, forcedStops...)
2: evExists ← False, bestEvDist ← ∞, bestEvVid ← -1
3: for each vehicle v ∈ V in range R do
4:     if v.isLeader() then
5:         evExists ← True
6:         if distance(v, RSU) < bestEvDist then
7:             bestEvVid ← v.id
8:             bestEvDist ← distance(v, RSU)
9: 
10: if evExists then
11:     evState ← EXTRACT_STATE(bestEvVid)
12:     // Execute continuous corridor maintenance
13:     EXECUTE_HOLD(evState, enableCorridor)
14:
15: // Privacy System State Machine (only advances if Mix Zone is offline or evaluating)
16: if not mzActive then
17:     if enableMixZone then
18:         EVALUATE_PRIVACY_TRIGGER()
19: else
20:     ADVANCE_MIX_ZONE_FSM(t)
21:
22: LOG_PAPER_METRICS(t, evState, variables)
23: SCHEDULE_NEXT_TICK(t + 0.1s)
```

### Algorithm 2: Tier 1 (EV Bypass & Gradient Clearing)
```text
Algorithm 2: Tier 1 Gradient Corridor Clearing
Input: EV State evState, Traffic Interface tci, Center RSU
Output: Modifies Speed Mode & Leader Velocity

1: if not evState.isValid then return
2: distToRSU ← distance(evState, RSU)
3: inControlZone ← (distToRSU < 150m) // Bounded control segment
4:
5: // Release control if EV leaves intersection area
6: if inControlZone then
7:     ev.setSpeedMode(0) // Disable collision avoidance constraints
8:     ev.setLaneChangeMode(FORCED)
9: else
10:    ev.restoreDefaultModes()
11:
12: dynamicDist ← MAX(20, MIN(evState.speed × 5.0, 120))
13: leader ← ev.getLeaderNearest(dynamicDist)
14:
15: if leader.exists and inControlZone then
16:     if leader.dist < 20m then
17:         // Extreme proximity: Emergency Stop
18:         blocker.setSpeed(0.0) within 1.5s
19:     else if leader.dist < 40m then
20:         // Escape Zone: Force forward acceleration
21:         blocker.setSpeed(MAX(10.0, evState.speed)) within 1.0s
22:     else
23:         // Buffer Zone: Smooth yield
24:         blocker.setSpeed(MAX(2.0, evState.speed × 0.5)) within 2.0s
```

### Algorithm 3: Tier 2 (Corridor Predictive Lane Clearing)
```text
Algorithm 3: Tier 2 Predictive Lane Clearing
Input: Vehicle v, EV State evState
Output: Lane Change Directives

1: targetLane ← -1
2: t_arrival ← distance(v, evState) / MAX(1.0, evState.speed)
3:
4: if v.lane == evState.lane and v.road == evState.road then
5:     predictiveDanger ← (t_arrival < 6.0s) // 6 seconds look-ahead
6:
7:     if predictiveDanger then
8:         targetLane ← FIND_SUITABLE_LANE(v)
9:         if targetLane ≠ -1 and NOT_IN_COOLDOWN(v) then
10:            v.forceChangeLane(targetLane)
11:            v.lastLaneChangeTick ← CURRENT_TIME
12:            UPDATE_INTERVENTION_COUNTER(LaneChange)
13:            return
14:
15: // Fallback geometric corridor clearing
16: APPLY_GEOMETRIC_CLEARING(v, evState)
```

### Algorithm 4: Dynamic Distance Calculation
*Note: We seamlessly embedded the Dynamic Distance execution directly inside Algorithm 2.*
```text
Algorithm 4: Dynamic Reaction Distance
Input: Current EV Speed (v_ev)
Output: Target reaction range (d_reaction)

1: t_react ← 3.0 // System reaction time (s)
2: t_buffer ← 2.0 // Safety buffer time (s)
3: 
4: d_req ← v_ev * (t_react + t_buffer)
5: d_reaction ← CLAMP(d_req, 20.0, 120.0) // Bound by physical constraints
6: return d_reaction
```

---

## 2. Experiment Tables

### Table 1: Simulation Parameters
| Parameter | Value |
| :--- | :--- |
| Simulation Framework | OMNeT++ 5.6, Veins 5.2, SUMO 1.x |
| Control Tick Rate | 10 Hz (0.1s update interval) |
| Road Topology | Urban Intersections |
| Speed Limit | 50 km/h |
| EV Broadcast Interval | 1 Hz |
| Traffic Demand | High / Medium |
| Mix Zone Radius | 120m |

### Table 2: Evaluated Operational Configurations (Cases)
| Case | Tier 1 (Bypass) | Tier 2 (Corridor) | Mix Zone (Privacy) |
| :--- | :--- | :--- | :--- |
| **Baseline** | ❌ OFF | ❌ OFF | ❌ OFF |
| **Tier 1 Only** | ✅ ON | ❌ OFF | ❌ OFF |
| **Full System** | ✅ ON | ✅ ON | ✅ ON |

### Table 3: Emergency Vehicle Performance Metrics
| Case | Average Travel Time (s) | Avg Speed (m/s) | Forced Stop Count | Minimum Speed (m/s) |
| :--- | :--- | :--- | :--- | :--- |
| **Baseline** | 152.0 | 15.49 | 2 | 0.0 |
| **Tier 1 Only** | 152.0 | 15.49 | 2 | 0.0 |
| **Full System** | 234.0 | 17.87 | 5 | 0.0 |


### Table 4: Automated Traffic Intervention Overheads
| Case | Affected Vehicles/Tick | Lane Changes Executed | Forced Target Stops | Lane Intrusions Avoided |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 1 Only** | 0.0001 | 0 | 0 | 0 |
| **Full System** | 0.0013 | 9 | 0 | 8 |


---

## 3. Python Plotting Pipeline

您可以使用剛生成的 `results/paper_metrics.csv` 透過以下 Python 程式碼快速繪製圖表：

```python
import pandas as pd
import matplotlib.pyplot as plt

# 讀取剛剛輸出的 CSV (欄位包括 time, ev_speed, dist_to_rsu, numLaneChanges 等)
df = pd.read_csv("results/paper_metrics_Tier 1.csv")

# 只抓取 EV 存在的資料
df_ev = df[df['evExists'] == 1]

# ① EV Speed vs Time (判斷有無卡死)
plt.figure(figsize=(10, 4))
plt.plot(df_ev['time'], df_ev['speed'], label="EV Speed", color="red")
plt.title("EV Speed Profile over Route")
plt.xlabel("Simulation Time (s)")
plt.ylabel("Speed (m/s)")
plt.grid(True)
plt.legend()
plt.savefig("fig1_speed.png")

# ② Distance to Next Intersection vs Time
plt.figure(figsize=(10, 4))
plt.plot(df_ev['time'], df_ev['dist_to_tls'], label="Dist to Next TLS", color="green")
plt.title("Distance to Traffic Signal")
plt.xlabel("Simulation Time (s)")
plt.ylabel("Distance (m)")
plt.grid(True)
plt.legend()
plt.savefig("fig2_dist.png")

# ③ Lane Interventions / Traffic Operations (Tier 2 效果)
plt.figure(figsize=(10, 4))
plt.plot(df_ev['time'], df_ev['numLaneChanges'], label="Lane Changes Triggered", color="blue")
plt.plot(df_ev['time'], df_ev['numForcedStops'], label="Forced Stops", color="orange")
plt.title("System Interventions vs Time")
plt.xlabel("Simulation Time (s)")
plt.ylabel("Count")
plt.grid(True)
plt.legend()
plt.savefig("fig3_interventions.png")
```
