# OpenFFBoard 調教檢查清單

> 基於韌體 v1.17.1 原始碼閱讀整理。
> - 預設值取自原始碼,實機值可能因 Flash 已存設定而不同。
> - 「症狀與調整方向」是依程式邏輯推論,**尚未在硬體上驗證**,請當作起點。
> - 指令的類別前綴(例如 `axis.`)未實測確認,請以 Configurator 送出的字串為準。

## 調教順序

1. 安全設定(`power` 壓低、固定馬達、拆輪圈)
2. TMC 基礎:極對數、`cpr`、編碼器方向,再 `fullCalibration`
3. TMC 電流環:`pidautotune`
4. Axis:範圍、方向、增益
5. 濾波與取樣率
6. Effects 增益
7. 上遊戲驗證,最後 `save`

一次只改一個變數,每次改完都要重測。

## A. TMC4671 電流環(馬達層)

| 參數(指令) | 預設值 | 太低的症狀 | 太高的症狀 | 調整方向 |
|---|---|---|---|---|
| `torqueP` / `fluxP` | 700 | 扭矩響應慢、手感黏 | 高頻嘯叫、發熱、抖動 | 先跑 `pidautotune`;手動每次改 10 到 20% |
| `torqueI` / `fluxI` | 800 | 扭矩有穩態誤差 | 過衝、低頻振盪 | 同上;autotune 以過衝超過 4% 為停止條件 |
| `pidautotune` | 測試電流 `bangInitPower` = 5000 | 小電流調出的值在大電流可能偏樂觀 | 轉子被鎖住,沒有反電動勢 | 在實際使用的電源電壓下執行;換電壓後重跑 |
| `pwmfreq` | 約 24.4 kHz(`pwmcnt` = 4095,時脈 25 MHz) | 電流漣波大、可聽噪音 | 開關損耗與驅動板發熱增加 | 改後必須重新確認 torque/flux 濾波截止頻率 |
| `torqueFilter_mode/f/q` | 關閉(`none`);啟用時預設 1000 Hz、Q 50 | 有高頻雜訊時沒濾掉 | 延遲增加、力道變鈍 | 低通 Q 以 `q/100` 計,陷波/峰值以 `q/10` 計,輸入同數字效果不同 |
| flux 濾波(固定) | 350 Hz 低通,Q 0.7 | — | — | 不可調;校準與 autotune 時會暫時關閉 |
| `pwmlim`、`acttrq` | — | — | — | 測試期間先壓到額定的 10 到 20% |
| `poles`、`cpr`、`encdir`、`encpol` | — | 錯了後面全部沒意義 | — | 先確認,再 `fullCalibration` |
| `seqpi` | 開 | — | — | 一般保持預設 |
| 速度/位置 PI | 速度 P 256 I 0;位置 P 128 I 0 | — | — | FFB 走扭矩模式,通常不用動;autotune 不會調這兩個 |

## B. Axis(軸層)

| 參數(指令) | 預設值 | 太低的症狀 | 太高的症狀 | 調整方向 |
|---|---|---|---|---|
| `power` | 5000(約 15%) | 力道出不來 | 超出馬達或電源能力、有安全風險 | 同時是軟體總增益與 TMC 硬體扭矩上限,逐步加 |
| `fxratio` | 204(約 80%) | 遊戲力道偏弱 | 限位器沒有餘裕、頂到 `power` 時被截斷 | 255 = 100%;遊戲力道上限 ≈ `power` × `fxratio` |
| `degrees` | 900 | 範圍太小時限位器過早介入 | 範圍太大時轉不到頭 | 依車種調整 |
| `esgain` | 127 | 限位器軟、容易衝過 | 限位器撞擊感強、可能反彈 | 與 `power` 一起調 |
| `idlespring` | 127 | 無 FFB 時方向盤不回中 | 回中太強 | FFB 未啟動時才作用 |
| `axisdamper` | 30 | 放手後回彈、振盪 | 手感黏、轉動費力 | 內部截止約 60 Hz,與遊戲阻尼是兩組獨立效果 |
| `axisfriction` | 0 | 沒有靜摩擦感 | 轉動卡頓 | 低速時有漸增曲線避免零速抖動 |
| `axisinertia` | 0 | 方向盤過輕飄 | 反應遲鈍 | 內部截止約 20 Hz;低解析度編碼器容易放大雜訊 |
| `maxspeed` | 0(關閉) | 限速過早介入、力道被吃掉 | 方向盤失控時沒有保護 | 啟用後為 PI 結構(P 0.3、I 0.03),只減力不反向 |
| `maxtorquerate` | 0(關閉) | 限制太嚴時響應鈍 | 沒有抑制突變 | 可緩解電流環抖動或 USB 資料突變造成的衝擊 |
| `expo` | 0(關閉) | — | — | 設定 −127 到 127,對應指數約 0.28 到 3.54 |
| `invert` | 開 | 方向反了 | — | 馬達與編碼器方向不同時切換 |
| `filterProfile_id` | 1(中) | 檔位低:較平滑但延遲大 | 檔位高:低延遲但雜訊被放大 | 高解析度編碼器選高檔,低解析度選低檔 |
| 速度濾波(依檔位) | 0:40 Hz、1:70 Hz、2:120 Hz、3:180 Hz,Q 均 0.55 | — | — | 作者註記「TODO tune」;第 4 檔是預留 |
| 加速度濾波 | 0:40、1:55、2:70 Hz(Q 0.30);3:120 Hz(Q 0.55) | — | — | 加速度用未濾波速度差分,雜訊比速度大 |
| 力道淡入 | 1 秒 | — | — | 馬達就緒時自動啟動,避免突然衝擊 |

## C. Effects(遊戲效果層)

| 參數(指令) | 預設值 | 太低的症狀 | 太高的症狀 | 調整方向 |
|---|---|---|---|---|
| 全域增益(遊戲送 `0x0D`) | 255 | 整體偏弱 | — | 由遊戲或驅動設定 |
| `spring` | 64 | 回中弱 | 回中過強 | 縮放常數 16 |
| `damper` | 64 | 阻尼感弱、容易擺盪 | 手感黏 | 縮放常數 4 |
| `friction` | 254 | 路面質感弱 | 阻力大 | 縮放常數 1 |
| `inertia` | 127 | 缺乏重量感 | 遲鈍 | 縮放常數 2 |
| `filterProfile_id`(Effects) | 0(預設) | — | — | 0 為預設、1 為自訂;自訂前改 `*_f`、`*_q` |
| `damper_f` / `damper_q` | 30 Hz / 40 | 阻尼延遲大 | 雜訊進入力道 | 低通濾波 |
| `friction_f` / `friction_q` | 50 Hz / 20 | 同上 | 同上 | |
| `inertia_f` / `inertia_q` | 15 Hz / 20 | 同上 | 同上 | |
| `filterCfFreq` / `filterCfQ` | 500 Hz / 70 | 恆力變遲鈍 | 遊戲低頻更新造成階梯感 | 遊戲只以 60 到 100 Hz 更新時可調低 |
| `frictionPctSpeedToRampup` | 25 | 零速附近有突變 | 低速摩擦太弱 | 單位是最大速度的百分比 |
| 效果池上限 | 40 個 | — | 滿了之後新效果被拒絕,力道莫名消失 | 用 `effects` 觀察用量 |

Q 值在程式內都乘以 0.01 換算。

## D. HID / 取樣率層

| 參數(指令) | 預設值 | 說明與影響 | 調整方向 |
|---|---|---|---|
| `hidsendspd` | 全速 USB:模式 0(1 kHz);高速 USB:模式 3(1 kHz) | 決定 FFB 更新頻率,也決定速度/加速度估算的取樣率 | 改了之後所有濾波器會依新頻率重算,需重新確認手感 |
| 可選頻率 | 高速 USB:8k/4k/2k/1k/500/250/125 Hz | | 頻率高延遲低,但 CPU 與 SPI 負載增加 |
| `HID_BINTERVAL` | 1(1000 Hz) | 全速 USB 的輪詢上限 | 韌體編譯期常數 |
| `hidrate` | 唯讀 | 遊戲實際的 HID 更新率 | 判斷遊戲送力道的頻率 |
| `cfrate` | 唯讀 | 恆力實際更新率 | 與 `filterCfFreq` 搭配參考 |
| `estop` | — | 緊急停止 | 測試時務必確認可用 |
| `save` | — | 把設定寫入 Flash | 改完要記得執行,否則斷電後恢復 |

## E. 症狀速查

| 症狀 | 先檢查 | 再檢查 |
|---|---|---|
| 高頻嘯叫、馬達發熱 | `torqueP` 過大、`pwmfreq` | torque 濾波、編碼器雜訊 |
| 低頻振盪或過衝 | `torqueI` 過大 | `axisdamper` 太低、`maxtorquerate` |
| 扭矩響應慢、手感黏 | `torqueP` 太低、濾波太強 | `damper` 增益太高 |
| 方向盤放手後來回擺盪 | `axisdamper`、阻尼增益 | 速度濾波檔位 |
| 力道出不來 | `power`、`fxratio` | 遊戲內增益、全域增益 |
| 力道偶爾消失 | 效果池滿(40) | 遊戲是否不釋放效果 |
| 恆力有階梯感 | `cfrate` 偏低 | `filterCfFreq` 調低 |
| 低速時抖動或顫動 | 編碼器解析度與雜訊、`filterProfile_id` | `axisfriction`、`frictionPctSpeedToRampup` |
| 撞限位太硬或太軟 | `esgain` | `power` |
| 開機或換電壓後怪異 | 重跑 `fullCalibration` | 重跑 `pidautotune` |
| 改過頻率後手感變了 | `hidsendspd`、`pwmfreq` | 濾波截止頻率是否被連動改變 |

## 已知注意事項

- `TMC4671::getFlux()` 使用 `&&` 而非 `&`,只會回傳 0 或 1(尚未修正)。
- 速度與加速度濾波的作者註解標示「尚未完全調整」,預設值可能不是最佳。
- 自動調整只測 flux,並把同一組值套到 torque;這是簡化做法,不是頻域分析。
- 調教時固定馬達、拆掉輪圈、手遠離轉軸,保持可隨時斷電並設扭矩上限。

## 來源檔案

| 層 | 檔案 |
|---|---|
| TMC4671 | `Firmware/FFBoard/UserExtensions/Src/TMC4671.cpp`、`Inc/TMC4671.h` |
| Axis | `Firmware/FFBoard/Src/Axis.cpp`、`Inc/Axis.h` |
| Effects | `Firmware/FFBoard/Src/EffectsCalculator.cpp`、`Inc/EffectsCalculator.h` |
| HID / PID | `Firmware/FFBoard/Src/HidFFB.cpp`、`Inc/ffb_defs.h`、`UserExtensions/Src/FFBHIDMain.cpp` |
