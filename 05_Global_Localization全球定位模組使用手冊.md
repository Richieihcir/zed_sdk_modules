# ZED SDK Global Localization 全球定位模組使用手冊

> 適用版本：ZED SDK 5.x（整理時 API Reference 為 5.4.1）  
> 程式語言：Python／PyZED  
> 本手冊可獨立閱讀；範例中的 GNSS 資料來源需依實際接收器替換。

## 1. 介紹

Global Localization 將 ZED 相機的視覺慣性里程計（VIO）與 GNSS 或 RTK 定位融合，讓系統同時取得：

- 高更新率、短時間平滑的相機姿態。
- 經緯度與海拔等全球座標。
- GNSS 短暫遮蔽時的連續定位。
- GNSS 恢復後對長時間漂移的修正。

典型應用包括戶外機器人、自主載具、測繪、巡檢與跨區域導航。

## 2. 工作原理

### 2.1 兩種定位來源

ZED VIO 利用雙目影像與 IMU 估測相機的六自由度運動，局部精度高，但長時間可能累積漂移。GNSS 提供絕對地理位置，不會產生同類型的累積漂移，但更新率較低，且會受建築物、樹蔭與多路徑效應影響。

Fusion 模組依時間戳、量測協方差與校準狀態綜合兩者：

```text
ZED 影像 + IMU ──> VIO 局部姿態 ──┐
                                    ├─> Fusion ─> 全球姿態
GNSS / RTK ───────> 經緯高與誤差 ──┘
```

### 2.2 時間同步

每筆 GNSS 資料必須使用與影像時間軸一致的時間戳。時間偏差會使系統把「不同時刻的位置」當成同一時刻融合，造成抖動、延遲或校準失敗。

### 2.3 VIO／GNSS 校準

系統需要估測局部 VIO 座標系與地理座標系的關係。啟動後應讓載具產生足夠的平移與轉向；只原地靜止通常無法得到穩定校準。

### 2.4 不確定度加權

GNSS 協方差描述量測可信度。協方差過小會讓融合器過度相信品質不佳的 GNSS；過大則幾乎無法利用 GNSS 約束漂移。

## 3. 主要參數

### 3.1 相機與追蹤參數

| 類別 | 參數 | 說明 | 建議 |
|---|---|---|---|
| `sl.InitParameters` | `coordinate_units` | ZED 座標單位 | 與 Fusion 一致，常用 `METER` |
| `sl.InitParameters` | `coordinate_system` | 座標系統 | 與 Fusion 一致 |
| `sl.PositionalTrackingParameters` | `enable_imu_fusion` | 融合相機與 IMU | 有 IMU 的機型建議開啟 |
| `sl.PositionalTrackingParameters` | `set_gravity_as_origin` | 以重力校正世界座標方向 | 一般戶外定位建議開啟 |

### 3.2 Fusion 初始化參數

| 類別 | 參數 | 說明 |
|---|---|---|
| `sl.InitFusionParameters` | `coordinate_units` | Fusion 輸出單位 |
| `sl.InitFusionParameters` | `coordinate_system` | Fusion 使用的座標系 |
| `sl.InitFusionParameters` | `verbose` | 顯示詳細診斷訊息 |
| `sl.InitFusionParameters` | `output_performance_metrics` | 啟用效能統計 |

### 3.3 GNSS 資料欄位

| 欄位 | 單位／格式 | 說明 |
|---|---|---|
| `ts` | 奈秒時間戳 | 必須與 ZED 資料時間對齊 |
| 緯度、經度 | 度或弧度 | `set_coordinates` 的最後一個參數決定格式 |
| 海拔 | 公尺 | 需確認是橢球高或海拔高 |
| `position_covariances` | 3×3 展平陣列 | ENU 座標下的位置協方差 |

若輸入為一般 GNSS 的度數，需使用：

```python
gnss.set_coordinates(latitude_deg, longitude_deg, altitude_m, False)
```

其中 `False` 表示輸入不是弧度。常見的對角協方差可寫成：

```python
[eph**2, 0, 0,
 0, eph**2, 0,
 0, 0, epv**2]
```

## 4. 標準操作流程

1. 開啟 ZED，設定單位與座標系。
2. 啟用相機位置追蹤。
3. 建立通信參數，讓相機發布資料。
4. 初始化 Fusion，並訂閱相機。
5. 在 Fusion 端啟用位置追蹤與 GNSS 融合。
6. 持續擷取相機影格，並注入帶時間戳的 GNSS 資料。
7. 呼叫 `process()`，讀取融合後姿態與校準狀態。

> PyZED API 的方法名稱保留 `enable_positionnal_tracking()` 拼字（`positionnal` 有兩個 n），請依安裝版本的 API 使用。

## 5. Python 完整範例骨架

```python
import time
import pyzed.sl as sl


def read_gnss_sample():
    """替換成實際 GNSS 驅動。

    回傳：latitude_deg, longitude_deg, altitude_m,
          horizontal_sigma_m, vertical_sigma_m, timestamp_ns
    沒有新資料時回傳 None。
    """
    return None


zed = sl.Camera()
init = sl.InitParameters()
init.coordinate_units = sl.UNIT.METER
init.coordinate_system = sl.COORDINATE_SYSTEM.RIGHT_HANDED_Y_UP

if zed.open(init) != sl.ERROR_CODE.SUCCESS:
    raise RuntimeError("無法開啟 ZED")

tracking = sl.PositionalTrackingParameters()
tracking.enable_imu_fusion = True
tracking.set_gravity_as_origin = True
if zed.enable_positional_tracking(tracking) != sl.ERROR_CODE.SUCCESS:
    zed.close()
    raise RuntimeError("無法啟用位置追蹤")

communication = sl.CommunicationParameters()
communication.set_for_shared_memory()
if zed.start_publishing(communication) != sl.ERROR_CODE.SUCCESS:
    zed.close()
    raise RuntimeError("無法發布相機資料")

fusion = sl.Fusion()
fusion_init = sl.InitFusionParameters()
fusion_init.coordinate_units = sl.UNIT.METER
fusion_init.coordinate_system = sl.COORDINATE_SYSTEM.RIGHT_HANDED_Y_UP
fusion_init.output_performance_metrics = True

if fusion.init(fusion_init) != sl.FUSION_ERROR_CODE.SUCCESS:
    zed.close()
    raise RuntimeError("Fusion 初始化失敗")

camera_id = sl.CameraIdentifier(zed.get_camera_information().serial_number)
pose = sl.Transform()
if fusion.subscribe(camera_id, communication, pose) != sl.FUSION_ERROR_CODE.SUCCESS:
    fusion.close()
    zed.close()
    raise RuntimeError("Fusion 訂閱相機失敗")

fusion_tracking = sl.PositionalTrackingFusionParameters()
fusion_tracking.enable_GNSS_fusion = True
if fusion.enable_positionnal_tracking(fusion_tracking) != sl.FUSION_ERROR_CODE.SUCCESS:
    fusion.close()
    zed.close()
    raise RuntimeError("無法啟用 Fusion 位置追蹤")

runtime = sl.RuntimeParameters()
fused_pose = sl.Pose()

try:
    while True:
        if zed.grab(runtime) != sl.ERROR_CODE.SUCCESS:
            continue

        sample = read_gnss_sample()
        if sample is not None:
            lat, lon, alt, eph, epv, timestamp_ns = sample

            gnss = sl.GNSSData()
            gnss.ts.set_nanoseconds(timestamp_ns)
            gnss.set_coordinates(lat, lon, alt, False)  # False：輸入為度
            gnss.position_covariances = [
                eph * eph, 0.0, 0.0,
                0.0, eph * eph, 0.0,
                0.0, 0.0, epv * epv,
            ]
            fusion.ingest_gnss_data(gnss)

        if fusion.process() == sl.FUSION_ERROR_CODE.SUCCESS:
            state = fusion.get_position(fused_pose)
            if state == sl.POSITIONAL_TRACKING_STATE.OK:
                xyz = fused_pose.get_translation().get()
                print(f"融合位置：x={xyz[0]:.3f}, y={xyz[1]:.3f}, z={xyz[2]:.3f}")

        time.sleep(0.001)
except KeyboardInterrupt:
    pass
finally:
    fusion.disable_positionnal_tracking()
    fusion.close()
    zed.stop_publishing()
    zed.disable_positional_tracking()
    zed.close()
```

## 6. 實作注意事項

- GNSS 天線與相機原點不重合時，應量測並設定兩者的槓桿臂偏移。
- 確認 GNSS 時間基準是 UTC、GPS time 或系統 monotonic time，並在驅動層完成轉換。
- RTK Fixed、Float 與一般單點定位的協方差不可使用相同固定值。
- 測試路線應包含直線、轉彎與足夠位移，以利外參與方向校準。
- 相機與 Fusion 的座標系及單位必須完全一致。

## 7. 常見問題

| 現象 | 可能原因 | 建議處理 |
|---|---|---|
| 校準長時間無法收斂 | 運動不足、時間戳錯誤、GNSS 品質差 | 增加轉彎與位移，檢查時間基準及協方差 |
| 融合位置週期性跳動 | GNSS 多路徑、協方差過小 | 依接收器品質動態設定協方差 |
| 方向明顯錯誤 | 座標系不一致或初始化運動不足 | 統一座標系並重新校準 |
| GNSS 遮蔽後漂移 | VIO 特徵不足、曝光或震動問題 | 改善視野、固定相機並調整影像品質 |
| 高度偏移固定 | 橢球高與正高混用 | 明確統一高度定義 |

## 8. GitHub 範例對照

本機 `zed-sdk-source` 可參考：

- `tutorials/tutorial 9 - recording external data/python/`
- `global localization/live/python/`
- `global localization/recording/python/`
- `global localization/playback/python/`
- `global localization/map server/`

## 9. 官方參考

- [Global Localization 模組](https://docs.stereolabs.com/docs/development/zed-sdk/modules/global-localization)
- [ZED SDK GitHub](https://github.com/stereolabs/zed-sdk)

