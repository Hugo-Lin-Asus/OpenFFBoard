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
