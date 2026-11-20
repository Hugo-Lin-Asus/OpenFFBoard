# NTU Racing Team(NTURacingTeam)GitHub 組織總覽

> 範圍:組織底下全部 44 個公開儲存庫。
> 方法:概覽式分析,依據 README、目錄結構、檔案類型與更新時間;其中 `nturt_vcu` 有讀到程式碼層級。`zephyr` fork 未與上游比對,多數專案未逐檔細讀。
> 分析日期:2026-11-18。

---

## 1. 一句話看懂整個組織

這是一支電動方程式賽車隊的軟硬體全家桶:**自製逆變器 → CAN/CANopen 車載網路 → Zephyr 韌體 VCU → RPi 上的儀表板與記錄 → PCB 設計庫**。2023–2024 年以 ROS 2 為控制中心(EP4/EP5),2025 年起把控制環搬到 Zephyr + MCU(EP6/EP7)。

---

## 2. 系統架構(依現行 EP6/EP7 世代)

```
                      ┌────────── 車上 CAN 網路 (500 kbps) ──────────┐
 BMS ── 4× 逆變器 ── VCU (nturt_vcu) ── 前/後盒 ── IMU ── GPS
        │                │
        │          Zephyr fork (v4.4.2) + nturt_zephyr_common
        │          zbus 訊息匯流排 / SMF 狀態機 / err 錯誤分級
        │
 逆變器韌體:nturt_INV_STM32H743ZI (STM32H743, FOC/SVPWM)
            Inverter_TI (舊, TI C2000)

                      ┌────────────── CAN ──────────────┐
 RPi / CM4:  RPI_Desktop (systemd: can、can logging、web 伺服器)
             Dashboard_Games (觸控儀表板 + CAN 監控)
             gps_tracker / IMU_RPI (GPS、IMU 轉 CAN)

 硬體:nturt_kicad (EP4–EP7) ← nturt_kicad_EP7 (子模組)
 環境:zephyr_workspace、docker
```

---

## 3. 世代演進

| 世代 | 年份 | 控制核心 | CAN 定義 | 主要專案 |
|---|---|---|---|---|
| EP4 | 2023 | STM32 HAL + FreeRTOS 前/後盒;RPi 上跑 ROS 2 | DBC + coderdbc | `nturt_stm32_*`、`nturt_ros` 系列、`ROS_server` |
| EP5 | 2024 | 同上,後盒改 Zephyr (NUCLEO-G474RE) | DBC | `nturt_ep5_rear_box`、`canopennode` |
| EP6 | 2025 | **Zephyr VCU** + 自製 STM32H743 逆變器 | CANopen | `nturt_vcu`、`nturt_zephyr_common`、`nturt_INV_STM32H743ZI` |
| EP7 | 2026 | 持續更新,Zephyr 升到 v4.4 | CANopen | `nturt_kicad_EP7`、`Dashboard_Games`、`RPI_Desktop` |

**轉折依據**:`nturt_test` 實測 ROS 2 控制延遲約 60 ms;micro-ROS 可到 2 ms 但吃掉 99% CPU,因此控制環移入 MCU。

---

## 4. 44 個儲存庫分類索引

### A. 現行韌體(Zephyr 世代)
| 專案 | 說明 | 更新 |
|---|---|---|
| `nturt_vcu` | 整車控制單元。Zephyr + zbus,SMF 階層式狀態機(RTD 流程、ERROR 狀態),控制演算法由 Simulink 產生 C 程式,CANopen 物件字典,儀表板與遙測 | 2026-11 |
| `nturt_zephyr_common` | 共用 Zephyr 模組庫:drivers、subsys (canbus/cmd/err/msg/sensing/sys/telemetry/usb)、自訂 board、tests、Sphinx 文件 | 2026-08 |
| `zephyr` | 隊上 fork,v4.4.2 | 2026-08 |
| `canopennode` | CANopenNode 的 Zephyr 版 fork | 2024-07 |
| `nturt_INV_STM32H743ZI` | 自製逆變器韌體:FOC/SVPWM、電流取樣、sin/cos 與 AS5048A 編碼器、FDCAN、SD 卡記錄 | 2025-10 |
| `zephyr_workspace` | VS Code + Docker 的 Zephyr 開發環境 | 2026-07 |

### B. 車上 Linux(RPi / CM4)
| 專案 | 說明 | 更新 |
|---|---|---|
| `RPI_Desktop` | RPi 部署結構:5 個 systemd 服務(can、canGPSRTK、canlogging、RPIserver),FastAPI + WebSocket 即時網頁儀表板;`GUI-dev` 子模組指向 `Ktliu-Tyler/rpi_can_monitor` | 2025-12 |
| `Dashboard_Games` | CM4 + 2.8 吋觸控屏,Pygame;CAN 監控、馬達監控、小遊戲;附 DBC | 2026-08 |
| `gps_tracker` | ROS 2 GPS(NTRIP/RTK)轉 CAN、CSV 記錄、地圖視覺化、網頁儀表板;附完整 CAN 協定文件 | 2025-08 |
| `IMU_RPI` | RPi 讀 IMU 並以 CAN 送出(0x425–0x429) | 2026-03 |
| `docker` | 統一 ROS 環境映像與 `nturt_docker` 工具(host/jetson/rpi) | 2025-03 |

### C. ROS 2 軟體棧(EP4/EP5,多已停更)
| 專案 | 說明 |
|---|---|
| `nturt_ros` | 元套件,以 12 個子模組管理下列套件 |
| `nturt_can_parser` | SocketCAN ↔ ROS 2 橋接 |
| `ros2_socketcan` | Autoware 版 fork,關閉本地 loopback |
| `nturt_can_config` | DBC + 程式碼產生腳本(CAN 單一真相來源) |
| `c-coderdbc` | DBC → C 產生器 fork,加入收訊逾時檢查與週期送出 |
| `nturt_ros_interface` | 自訂 msg / srv |
| `nturt_realtime_utils` | 即時控制工具 |
| `nturt_bag_recorder` | ros bag 錄製 |
| `nturt_led_controller` | 4 顆 LED 狀態燈 |
| `nturt_screen_controller` | 儀表板螢幕(Dear ImGui) |
| `nturt_nmea_navsat_driver` | NMEA GPS 驅動 + NTRIP/RTK |
| `nturt_push_to_control_tower` | 車況推送到遠端 |
| `nturt_radio_communicator` | 無線電通訊(未完成) |
| `nturt_rpi_deployer` | Docker 化部署 ROS 到 RPi |
| `ROS_server` | 「控制塔」遠端監控網站 |
| `nturt_test` | 控制延遲與整合測試 |

### D. 較舊的 STM32 / 嵌入式
| 專案 | 說明 |
|---|---|
| `nturt_stm32_front_box` | EP4 前盒 / VCU,NUCLEO-H723ZG |
| `nturt_stm32_rear_box` | EP4 後盒,NUCLEO-G431KB |
| `nturt_stm32_module` | OO 風格 C 模組庫,googletest + CI |
| `nturt_ep5_rear_box` | EP5 後盒,Zephyr,NUCLEO-G474RE |
| `Inverter_TI` | 舊逆變器,TI C2000 F2837xS + DRV8305 |
| `bms` | BMS 原型,LTC6811 + Arduino + MCP2515 |
| `STM_Library` | 2021 Keil STM32 函式庫(已封存) |
| `door-opener` | 車庫捲門遙控開門器,STM32F103 |

### E. 硬體設計
| 專案 | 說明 |
|---|---|
| `nturt_kicad` | KiCad 庫:EP4、EP5、EP6、EP7、driverless_car |
| `nturt_kicad_EP7` | EP7 子模組:電控組與動力組 |

### F. 資料、模擬與其他
| 專案 | 說明 |
|---|---|
| `Data-Analysis` | MATLAB 賽車數據分析 |
| `FSDS` | Formula Student Driverless 模擬器 fork(ROS 2 用),AirSim + UE4 |
| `nturt_nms1` / `nturt_nms1_firmware` | Nano Model Simplification mk.1:縮小比例模型車,ROS 2 + micro-ROS(README 皆為 TBD) |

### G. 教學
| 專案 | 說明 |
|---|---|
| `git_homework` | git 課程作業 |
| `2021_summer_project` | 2021 暑期專案(已封存) |
| `nturt_introduction_to_MCU_with_STM32` | 空殼 |

---

## 5. 建議閱讀順序(新人)

1. `zephyr_workspace` → 建好開發環境
2. `nturt_zephyr_common`(`doc/` 的 Sphinx 文件)→ 理解共用函式庫
3. `nturt_vcu` → 看 `prj.conf`、`Kconfig`、`src/ctrl/states`、`src/ctrl/ctrl.c`
4. `nturt_INV_STM32H743ZI` → 逆變器端
5. `RPI_Desktop` + `Dashboard_Games` → 車上 Linux 端
6. `nturt_kicad_EP7` → 硬體

---

## 6. 風險與問題清單

### 高優先
1. **多個專案把虛擬環境或建置產物提交進 git**,造成儲存庫肥大、無法移植:
   - `RPI_Desktop`:`env/`、`laptopENV/`(約 2,400 個 `.pyc`、37 個 `.exe`),另有 `CanServer.log` 與 vim `.swp` 檔
   - `IMU_RPI`:`lib/` 約 6,100 個檔案
   - `Inverter_TI`:`.obj`、`.out`、`.map`
   - `STM_Library`、`bms`(`.rar`/`.exe`)、`gps_tracker`(`build/`、`install/`、`log/`)
2. **`RPI_Desktop` 的程式碼寫死 `/home/pi/Desktop/RPI_Desktop/...` 路徑**,且有對外網頁伺服器(FastAPI + CORS 中介層),請確認 CORS 與驗證設定。`ROS_server` 若仍在運行,同樣需要稽核。

### 中優先
3. **CAN 協定有多份定義**:`nturt_can_config` 的 DBC、`Dashboard_Games/dbc`、`gps_tracker` 的 Markdown/CSV、韌體內的 CANopen 物件字典。建議以單一來源為準,自動產生其他格式。
4. **`nturt_vcu` 的並行與安全細節**(詳見前述 VCU 分析):
   - 控制迴圈鎖逾時只記警告、無降級保護
   - `volatile double` 參數的 getter 沒有加鎖,在 Cortex-M 上可能撕裂讀取
   - 倒檔「車速為零」檢查被註解掉,目前只靠煞車 > 50%
5. **舊世代專案沒有標示封存**:`nturt_ros` 及其子模組、`nturt_stm32_*`、`Inverter_TI`、`ROS_server`,容易被新人誤用。

### 低優先
6. 多數 README 是 TBD 或空的(`nturt_nms1*`、`nturt_stm32_module`、`nturt_realtime_utils`、`RPI_Desktop`)。
7. 中英文文件混用,沒有統一入口。
8. `nturt_radio_communicator` 有未完成 TODO(資料傳輸無可靠性保證)。

---

## 7. 尚未涵蓋

- `zephyr` fork 與上游的差異
- `nturt_INV_STM32H743ZI` 的 FOC 演算法細節
- `nturt_zephyr_common` 各子系統的程式碼
- `nturt_kicad*` 的 PCB 內容
- `GUI-dev` 子模組(`Ktliu-Tyler/rpi_can_monitor`)
- `ROS_server` 與 `RPI_Desktop` 的資安審查

---

## 8. EP7 現行專案深入分析(2026-11 補充)

### 8.1 EP7 組成
`nturt_kicad_EP7`(硬體)、`nturt_zephyr_common`(含 `ep7_vcu` 板)、`nturt_vcu`、`RPI_Desktop`、`Dashboard_Games`,基礎是 `zephyr` fork 與 `zephyr_workspace`。

**硬體(依 KiCad 檔字串推斷,未開 KiCad 驗證):**
- 動力組:自製分立式逆變器(`INV_Discrete_*`:控制、隔離、功率、電流感測),閘極驅動 Infineon 1ED3122MC12H,TO-247-4 封裝、LEM CKSR 電流感測、TDK CeraLink 直流鏈電容;`INV_Boost_*`(用途不明);`DIY_MGJ6` 隔離閘極電源;`MOT_EncTemp`(TI LDC5072,2026-10-15 才 V1.0);`ACC_BMSslave`。
- 電控組:`VCU_CM4`/`VCU_CM5`(Raspberry Pi 運算模組載板)、`Dash_Screen`、`Steering_Panel`、`DIY_IMU`(BNO085)、`Tire_Temp`(MLX90640)、`PCAN_PRO`、`DEV_UPS`。

**`ep7_vcu` 板(STM32H743ZI):** FDCAN1/2、SD 卡 FatFS、USB、RTC、蜂鳴器、輸入電壓監控;感測器為方向盤、APPS1/2、BSE1/2、4 個懸吊位移。相較 EP6,VCU 板上移除了煞車燈、RTD 燈、CANopen LED、IMU、5V 電流感測與 SPI 顯示器,這些功能改由儀表板與其他板子負責,IMU 改走 CANopen。

### 8.2 CAN 位元率:已確認一致(1 Mbit/s)
| 位置 | 設定 |
|---|---|
| `ep7_vcu.dts` fdcan1/fdcan2 | 1,000,000 bps,取樣點 80% |
| `RPI_Desktop/scripts/start_can.sh` | can0/can1 皆 1,000,000 |
| `nturt_zephyr_common` CANopenNode | `CANOPENNODE_BITRATE` 預設 1000 kbit/s |
| `nturt_INV_STM32H743ZI` | FDCAN 核心時脈 30 MHz(.ioc 的 `RCC.FDCANFreq_Value`),預分頻 10 → 3 MHz;每位元 3 tq → 1 Mbit/s |

先前引用的 500 kbps 來自舊的 `nturt_ep5_rear_box` 文件,已不適用於 EP7。

**小問題:** 逆變器端每位元只有 3 個時間量子(同步 1 + Seg1 1 + Seg2 1),取樣點約 66.7%,與 VCU 的 80% 不同,解析度也低。短線纜可運作,但線長增加或雜訊大時裕度較小,建議改成 10 tq 以上(例如預分頻 3、Seg1 7、Seg2 2 → 80%)。

### 8.3 CAN 協定:逆變器端不是 CANopen 協定棧
`nturt_INV_STM32H743ZI` 用自訂 ID 方案(`canid.h`):狀態 0x190、狀態機 0x290、溫度 0x390、控制 0x210、設定 0x110、心跳 0x710、參數 0x730,各加上 `MOT_ID`(1–4)。只有 TX 標頭與一個 RX 過濾器(控制幀),沒有物件字典或 SDO。

VCU 端用 CANopen,透過 `src/canopen/msg.c` 把物件字典項目(如 0x2111、0x2113)聚合成 zbus 訊息。DBC(`NTUR_EP6_260617.dbc`)則同時涵蓋兩者:VCU 是 CANopen 風格 ID(0x181 Cockpit、0x701 心跳),逆變器是上述自訂 ID(INV1–4 的 Status、狀態機、溫度、TI_Log、心跳、PID_Feedback,以及 VCU 發出的 INVCMD1–4)。所以現行協定是「CANopen 與自訂 ID 混合」,需靠 DBC 與物件字典兩份文件才完整。**我沒有驗證 VCU 的 PDO 對映是否真的接收這些自訂 ID。**

### 8.4 逆變器保護電路(`INV_Discrete_Controller`)
依圖紙架構與零件(未追完整網表):
- `Isense_OCP`:U/V/W 相電流與直流電流經比較器(LM393)比較門檻,輸出 `OC&OT_Falut`。
- `OCPnode`、`OTPnode`、`TsenseNode`:過電流、過溫比較節點,各自帶 `GateEnablePull`。
- `SR_Latch`:由 74AUP1G00(NAND)與 74AUP1G07 組成的閂鎖,有 S、R、Q、!Q 四個端子。
- `PWM_Enable`:共 5 個實例,每個把 PWM 與 `enable` 閘控後輸出。
- 頂層網路:`GATE_EN`/`!GATE_EN`、`OCPFLTN`、`GATE_ENC`、`NRST`。

設計意圖:過電流或過溫由類比比較器直接觸發閂鎖並關閉閘極致能,不經過 MCU 軟體,**這是正確的高壓安全設計方向**。限制:我沒有追蹤閂鎖的重置路徑(誰可以清除)、比較器門檻是否由電位器設定(`threshold` 為 R_Potentiometer)、也沒有看到實測或 FMEA 文件。

### 8.5 `RPI_Desktop` 網頁伺服器資安檢查(`CAN_web/CanServer.py`)
| 嚴重度 | 問題 | 位置 |
|---|---|---|
| **高** | **儲存型 XSS。** 儀表板前端把收到的資料(`data.gps.lat`、`inv.name` 等)直接用模板字串寫入 `innerHTML`,沒有跳脫;任何人可 POST 惡意欄位,透過 WebSocket 廣播到所有觀看者的瀏覽器執行。 | 前端 `updateDashboard`(約第 307–413 行) |
| **高** | **`POST /api/can-data` 無驗證。** 能連到 8000 埠的任何人都能注入假遙測並廣播給所有觀看者,與上一項組合成完整攻擊鏈。 | 第 103 行 |
| 中 | 預設綁定 `0.0.0.0:8000`(`HOST` 環境變數預設值),所有網卡都暴露,包括比賽現場的 WiFi。 | 第 437 行附近 |
| 中 | CORS 為 `allow_origins=["*"]` 且 `allow_credentials=True`;WebSocket 不檢查 Origin,任何網頁都能跨站讀取即時資料或發送 POST。程式註解也寫著「生產環境請調整」。 | 第 50–55 行 |
| 低 | 輸入為 `Dict[str, Any]`,沒有大小上限或結構驗證;每筆都廣播並寫日誌,可被拿來耗盡資源。 | 第 103 行 |
| 低 | 客戶端寫死 Tailscale IP `http://100.97.75.73:8000`,明碼 HTTP;好在流量走 Tailscale 隧道。IP 屬私有位址空間,但仍建議移出程式碼。 | `CanClient.py` |
| 低 | `requirements.txt` 是 `pip freeze` 輸出,含舊版套件(例如 `cryptography==38.0.4`),需另行盤點漏洞。 | `requirements.txt` |

沒有發現寫死的密碼或金鑰(以常見關鍵字掃描,結果為空)。伺服器只做顯示,沒有看到回寫到 CAN 的路徑,所以**不會直接控制車輛**,風險集中在資訊正確性與觀看者瀏覽器。

**建議修法(依優先順序):**
1. 前端改用 `textContent`,或在寫入前跳脫 HTML。
2. `/api/can-data` 加上簡單的共享金鑰標頭或限定來源 IP(Tailscale 網段)。
3. 預設綁定改為 `127.0.0.1` 或 Tailscale 介面;CORS 限定實際來源,並把 `allow_credentials` 關掉。
4. 檢查 WebSocket 的 `Origin`。
5. 以 pydantic 模型限制欄位型別與大小。

---

## 9. 韌體核心深入分析:馬達控制 / 通訊 / 核心 / 儀表板(2026-11 補充)

> 方法:四條平行的靜態讀碼分析(沒有上機或實測)。標「已核對」的是我事後回頭讀原始碼確認過的項目;其餘為分析過程的結論,尚未獨立複核。「未驗證」代表需要實機、示波器或網表才能確認。
> 範圍:`nturt_INV_STM32H743ZI`、`nturt_vcu`、`nturt_zephyr_common`、`Dashboard_Games`、`RPI_Desktop`。

### 9.1 最優先處理清單

| # | 嚴重度 | 項目 | 位置 | 狀態 |
|---|---|---|---|---|
| 1 | 高 | **RPDO 映射複製貼上錯誤**:FR 狀態 RPDO 的第 3 個物件指向 FL 轉速(`0x2113:2`,應為 `0x2123:2`);RL 的指向 FR 轉速(`0x2123:2`,應為 `0x2133:2`)。結果 FL、FR 轉速被覆蓋,**RL 轉速永遠不更新**。會餵錯誤輪速給循跡控制與感測器融合。 | `nturt_vcu/src/canopen/objdict/OD.c:363,396` | 已核對 |
| 2 | 高 | **逆變器 RMS 過流保護永遠不會觸發**:`RMS_sum` 是 `uint32_t`,累加 10000 筆 0.01 A 單位的平方;門檻 `36000000`(對應 60 A)比較的是 `RMS_sum/10000`,左邊最大約 429,496,條件不可能成立。實際約 6.5 A RMS 就會溢位。 | `main.c:119,1071-1087`、`config.h:31` | 已核對 |
| 3 | 高 | **FL/FR 逆變器心跳消費者被停用**(時間為 0),只有 RL/RR 為 500 ms。前輪逆變器失聯時 VCU 沒有錯誤,且 RPDO 無逾時,舊狀態一直留著。 | `OD.c:204` | 已核對 |
| 4 | 高 | **APPS 兩顆感測器偏差檢查被關閉**:`dev-tolerance` 被註解,預設為停用。等同缺少 Formula Student「10% / 100 ms」APPS 合理性規則(只剩單顆超出範圍檢查)。 | `nturt_vcu/app.overlay:49` | 已核對 |
| 5 | 高 | **RTD 蜂鳴器被註解**:RTD 不會發聲。若比賽規則要求 RTD 聲響則不合規。 | `src/dashboard/peripherals.c:172` | 已核對 |
| 6 | 高 | **RTD_FORCED 繞過所有檢查**:可從 ERROR 狀態進入 RTD_SOUND,不需踩煞車。進入 RUNNING_OK 後,已存在的錯誤因為只在「置位/清除事件」觸發,不會重新評估,可能帶著 ERROR 級錯誤行駛(靜態推論)。 | `states/states.c:178`、`ctrl.c:272`、`normal.c:209` | 未實測 |
| 7 | 高 | **逆變器 CAN 收幀緩衝區 6 位元組**,HAL 最多會複製 8 位元組;過濾器為 `DUAL`,第二個 ID 是 0x7FF,所以 ID 0x7FF 的幀也會被收進來,可蓋掉堆疊。 | `main.c:1169`、`1372-1374` | 已核對 |
| 8 | 高 | **逆變器扭矩斜坡無效**:`max_ramp = 1/FREQ/RAMP_TIME_DERATE` 為整數除法,結果為 0;且限幅後的 `last_percent` 立刻被 `percent_torque_requested` 覆蓋。扭矩指令會瞬間套用。 | `main.c:207,769-770`、`config.h:69,94` | 已核對 |
| 9 | 高 | **疑似 ABBA 死結(儀表板鎖 ↔ 控制鎖)**:儀表板持鎖呼叫 `ctrl_ctrl_change_gear()`(以 `K_FOREVER` 取控制鎖),同時控制緒持控制鎖發佈 `msg_ctrl_states`,儀表板監聽器又取儀表板鎖。同按左右鍵撞上 10 ms 週期可能永久卡住控制迴圈。 | `normal.c:193,221`、`ctrl.c:123,175,193` | 依鎖順序推論,未驗證 |
| 10 | 高 | **逆變器低壓故障 ISR 內有 `f_sync` 與無限迴圈**,優先權與控制 ISR 同為 0,無法搶占,PWM 停在最後占空比;看門狗模組未啟用。 | `main.c:1149-1153`、`gpio.c:128`、`tim.c:248` | 未驗證 |

### 9.2 馬達控制(`nturt_INV_STM32H743ZI`)

**架構:** FOC + 中點箝位 SVPWM,電流 PI 輸出 Uq/Ud,含母線電壓補償;TIM1 中央對齊,`FREQ=23000`,ISR 每兩個更新事件執行一次。Id 目標固定為 0,**沒有弱磁**,也沒有 ωL 解耦與反電動勢前饋。

**需注意:**
- **ADC 未與 PWM 同步**:三個 ADC 都是軟體啟動、連續轉換、DMA 循環,TIM1 未輸出 TRGO。ISR 讀的是「最新值」而非中心點取樣,電流量測會帶入開關雜訊與相位誤差(中)。
- **PI 積分幾乎沒作用**:`QKP=2.4`、`QKI=1.5`,Ki/Kp ≈ 0.6 s⁻¹;被註解掉的 MATLAB 調法比值約 320 s⁻¹。目前近似純 P 控制,穩態電流誤差未驗證(中)。
- **軟體電流限制是死碼**:`IqOC_controller_output` 算出後從未使用,`max_derate` 也未使用。扭矩命令 `torque_command/1000` 沒有限幅(範圍約 ±32.7),只剩 72 A 的瞬時跳脫兜底;而滿額指令是 70 A,裕度僅 2 A,滿扭矩加上漣波可能誤跳。
- **沒有母線過壓/欠壓保護**:只在啟用瞬間檢查是否 ≥ 20 V。
- **逆變器過溫保護被註解**(`main.c:1506`),只剩馬達溫度保護;馬達低於 5 ℃ 也會跳錯誤。需確認硬體 OTP 是否獨立足夠。
- **故障清除不驗證條件**:一個 CAN 位元即回 READY;硬體 SR 閂鎖的重置路徑在韌體中看不到(未驗證)。
- **PWM 死區為 0、TIM1 break 輸入關閉**:防直通是否完全靠閘極驅動器的硬體死區需確認(未驗證)。
- **編碼器**:用 ADC2 四通道差分 sin/cos 做 `atan2`;AS5048A 驅動整檔被註解;**沒有 sin/cos 的偏移/增益/相位校正**,誤差會變成扭矩漣波。`pole_pairs=1` 但轉速公式寫死 `/4`,換 EP7 的 LDC5072 感測板時極易出錯。
- **零位與電流標定**:`ZERO_ELECTRIC_ANGLE`、`MOT_CURR` 只定義了 CAL、RR、RL;選 `MOT_FL`/`MOT_FR` 會編譯失敗(未驗證),代表前輪逆變器的標定值不在這個倉庫。`CAL_ZERO_ANGLE` 路徑已損壞(以 2 個參數呼叫 4 參數函式)。
- **電流換算係數疑點**:`DCAPLSB` 註解隱含 ADC 滿量程 2048,但 ADC 是 12 位元(4096),若公式正確現值差 2 倍;所有電流閾值都依賴它(待用實測電流校驗)。
- **調參封包**(CAN 0x200)可改 PID 與零位,RUNNING 時被擋,但無範圍檢查、無 `MOT_ID` 過濾(廣播同時改全部)、只存 RAM。`pid.c:26` 在 P=0 時產生 NaN。
- **日誌**:`index_to_sd` 無上限,SD 寫入卡頓超過約 156 ms 會讀出緩衝區範圍;`speed_rad`、`angle_pll` 從未賦值卻被記錄。
- **ISR 內雙精度數學**(`sin`、`cos`、`sqrt`、`fmod`)與每週期讀 RTC,耗時未量測(`TIMING` 巨集關閉)。

### 9.3 通訊(CANopen 與逆變器介面)

**VCU 的 CANopen:** VCU 為 node 0x01;節點表為 BMS 0x10、逆變器 0x11–0x14、RPi 0x04、IMU 0x08、GPS 0x09。心跳生產 100 ms;RPDO 為 `transmissionType=0xFE`、`eventTimer=0`,**沒有逐 PDO 逾時**。aggregation 的 `K_MSEC(100)/(80)/(30)` 是「發佈週期 / 最小間隔 / watermark」,**不是失聯逾時**:到了 watermark 資料沒更新完也照發,所以下游不會得知資料是舊的。

**VCU↔逆變器介面核對結果(一致的部分):** 控制幀 ID(VCU TPDO 0x211–0x214 ↔ 逆變器過濾 `0x210+MOT_ID`)、小端序、控制字位元(enable = bit3、fault reset = bit5)、扭矩縮放(×1000 / 額定 20 Nm)、狀態幀 0x19x 與 RPDO 0x191–0x194。方向由 VCU 的 `PARAM_MOTOR_DIR`(FL +1、FR −1、RL −1、RR +1)負責,逆變器端 `dir` 寫死為 1,逆變器 Iq 的符號關係未驗證。

**失聯行為:**
- 逆變器收不到控制幀:約 1 秒後關閘極並回 READY(`main.c:1096` 用 `==` 比較,只在計數剛好等於 `freq` 的那一個週期觸發)。建議縮短到 50–100 ms,並讓逆變器監看 VCU 心跳 / NMT。
- VCU 收不到逆變器心跳:RL/RR 500 ms 後 `hb_inv_*`(ERROR)進入 ERROR 狀態;FL/FR 無偵測(見 9.1 第 3 項)。
- **BMS 失聯不會被偵測**:0x1016 沒有 node 0x10,且 `hb_acc` 僅為 WARN。

**DBC 與實作的不一致:**
- 扭矩回授縮放:DBC 為 0.025 Nm/count;VCU 為 0.001×20 = 0.02;逆變器實送的是 `Iq/max_current×1000`(電流比例而非扭矩)。
- `INV_STATUS`:DBC 把 bit 1–3 當 3 位元列舉,韌體是三個獨立旗標;狀態幀第 2 位元組 `error_state` 在 DBC 中沒有定義。
- BMS 心跳 DBC 寫 DLC 8,CANopen 規範為 1 位元組。檔名為 EP6,是否已更新到 EP7 未驗證。

**其他:** `control` 遙測群組被停用(TPDO 0x1803 的 COB-ID 為 `0xC0000480`,bit31 = 無效,映射全空),slip ratio、yaw rate 實際不送出。`inv.c:129-134` 在**任何**逆變器錯誤事件時立刻對**四顆**逆變器送 fault reset,持續性故障可能被反覆清除。`MOT_CAL` 建置的 `MOT_ID=0` 會與 BMS(0x190、0x710)撞 ID;逆變器 ID 要靠改 `config.h` 重編,寫錯無法在執行期察覺。

**總線負載估算:** 約 175–215 kbit/s,即 1 Mbit/s 的 18–22%,餘裕充足。扭矩指令(0x21x)優先權低於狀態與心跳(0x19x、0x181、0x281),負載低時無影響,日後增加流量需重新檢視。

### 9.4 VCU 核心與安全邏輯

**狀態機:** 命令為 PEDAL / PEDAL_CLEAR(RTD_BLINK ↔ RTD_STEADY)、RTD(→ RTD_SOUND)、RTD_FINISH(→ RUNNING)、RTD_FORCED、ERROR / FATAL_ERROR、ERROR_RUNNING / CLEARED_RUNNING、DISABLE。扭矩非零只可能在 `RUNNING_OK` 且油門 > 0;RUNNING_ERROR 時輸入與輸出扭矩都歸零,逆變器 ENABLE 位綁在 RUNNING_OK,這部分設計合理。

**安全稽核重點:**
- **煞車/油門合理性**符合 EV4.7 概念(25% 啟動、5% 釋放,Kconfig 預設開啟)。
- **感測器錯誤不閂鎖**:APPS1/2、BSE1/2 個別錯誤為 INFO,只有聚合後的 ACCEL/BRAKE(ERROR)進 ERROR;感測器恢復就自動清除。
- **緊急停止只是軟體命令**(`ctrl.c:277-278`),訊息翻回自動清除;獨立的硬體 shutdown circuit 是否存在**未驗證**。
- **錯誤事件可能漏掉**:`states/err.c:72-74` 的 `@todo`,轉換進行中時錯誤事件被丟棄,不會事後補評估。
- **倒檔**只檢查煞車 ≥ 50%(車速條件被註解),反向扭矩限 10。
- **BMS 通訊遺失容忍 60 秒且僅 WARN**。

**執行緒與資料競爭:**
- zbus listener 無鎖寫入 `g_ctx`(`ctrl.c:255-266`),控制緒在鎖內讀;多欄位結構可能被撕裂(例如油門與煞車不同步)。
- 控制迴圈用 `k_sleep(10ms)`,週期為 10 ms 加執行時間而有漂移;拿不到鎖(5 ms)只記警告並**跳過整個週期**,沿用最後的扭矩指令。VCU 端有無輸出逾時 / 看門狗**未驗證**。
- `volatile double` 參數 getter 無鎖,在 Cortex-M7 上 64 位元讀取不保證不被撕裂。

**命令與參數:** `ctrl param set` **沒有範圍檢查**且立刻寫入快閃(`shell.c:182-198`),可設負值或超過 20 Nm 並在斷電後保留(Simulink 端是否再飽和未驗證);只有儀表板路徑有 `CLAMP`。USB shell 可手動強制狀態轉換、置位/清除錯誤,無驗證(需實體存取)。

### 9.5 儀表板

**韌體端:** 三種模式 NORMAL / SETTING / TEST,`RTD_SOUND` 與 `RUNNING` 期間禁止切換。**沒有獨立執行緒**,全部由 zbus 監聽器與 input callback 在發佈者執行緒內同步執行。11 顆 74HC595 串接(SPI4 5 MHz,每次約 18 µs,只在輸出值改變時才送出,不易閃爍)。

**需注意:**
- 資料中斷時**顯示凍結在最後一個值**,沒有逾時標示;只有逆變器心跳錯誤才顯示 "Er"。CAN bus-off、SD 卡故障、低壓(`pow_low`)沒有駕駛端指示(未驗證)。
- **RTD 鍵長按直接 `sys_reset()`**,沒有狀態檢查(`peripherals.c:111`),行駛中誤觸會重啟 VCU。
- 設定模式**每按一格(含長按連發)就寫一次快閃**(`setting.c:222,233,243`),快閃磨損;亮度可設為 0,連 "Er" 都看不到。
- `dashboard.c:262` 的 `static char buf[20]` 與 `accel_rgb/brake_rgb` 在多執行緒共用且未加鎖;`normal.c:363-364` 在已加鎖後 `default: return;` 未解鎖(目前被篩選器擋住,屬潛在問題)。
- 74HC595 位元順序是否與實體接線一致、開機初值、OE 控制:未驗證。

**RPi 端(`Dashboard_Games`):** CAN 讀取是 daemon 執行緒,與 pygame 主迴圈以 lock 分離,斷線每 2 秒重連;資料逾時 1 秒會標示年齡,**這點做得好**。只讀不送,不影響車輛。預設只用 can1、無核心過濾器,全部交給純 Python 解碼(CM4 負載未驗證)。`game.py:174` 把 "can1" 寫死在文字中。沒有看到開機自啟。`GUI-dev` 子模組為空,`canlogging-v4.py` 未能分析。

### 9.6 建議驗證計畫

**單元 / 模擬(不需硬體):**
1. `ztest` + `native_sim`:完整狀態轉換表,**特別是 RTD_FORCED 從 ERROR 進入**後的行為。
2. 為 OD.c 的 PDO 映射寫一致性檢查腳本(RPDO 目標物件應與節點一致),可直接抓到第 1 項類型的錯誤。
3. 逆變器端把 RMS 保護與扭矩斜坡抽成純函式,在主機上做單元測試(溢位、整數除法)。

**HIL / 實機:**
4. APPS 單顆斷線、兩顆偏差 > 10% 持續 100 ms、油門加煞車同踩。
5. 控制鎖逾時後扭矩是否停留;逆變器收不到控制幀(量測實際關斷時間);FL/FR 逆變器心跳中斷;BMS 失聯。
6. 持續性逆變器故障下的 fault reset 迴圈;`param set` 極端值斷電重啟。
7. 示波器:ADC 取樣時間點與 PWM 相位;ISR 實際耗時(開啟 `TIMING`);確認 TIM1 實際時脈(若 240 MHz,PWM 約 23.4 kHz,與 `Ts=1/23000` 約差 1.7%)。
8. 用實測電流校驗 `ACAPLSB`/`DCAPLSB`;確認硬體 SR 閂鎖重置路徑與 `GATE_EN` 在 ISR 凍結時是否仍有效。
9. CAN bus-off 與總線錯誤下的 VCU / 逆變器 / 儀表板行為。

**仍待深入:** `ctrl_inv_fault_reset` 在行駛中的行為、EP7 實車是否使用這份 OD.c(是否另有 EDS 產生)、OD 是否被持久化覆蓋、`ws2812` 48 顆(spi2 + DMA)的用途、`Inverter_TI` 的舊控制是否仍有參考價值。
