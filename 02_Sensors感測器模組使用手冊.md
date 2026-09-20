# ZED SDK Sensors 感測器模組使用手冊

> 整理日期：2026-09-20；對照 ZED SDK API 5.4.1 與官方 GitHub commit `94bac54`。

## 1. 介紹

Sensors 模組提供 IMU、磁力計、氣壓計和溫度資料。可用 Sensor 依相機型號而異；程式應先讀取 `sensors_configuration`，不要假設所有 ZED 都有相同硬體。初代 ZED 沒有內建慣性感測器。

## 2. 原理與時間同步

各 Sensor 具有不同取樣率。影像常為 15–120 FPS，IMU 可達數百 Hz，磁力計與氣壓計較慢。`SensorsData` 同時保存多種資料，但一次呼叫中並非每項都更新，因此必須比較各資料自己的 timestamp。

| 時間參考 | 原理 | 適用情境 |
|---|---|---|
| `TIME_REFERENCE.CURRENT` | 回傳呼叫當下最新樣本，不必先 `grab()` | 高頻 IMU 執行緒 |
| `TIME_REFERENCE.IMAGE` | 回傳與目前相機影像同步的資料 | VIO、Sensor/影像配對、錄製 |

```text
Sensor internal threads ──→ CURRENT samples
Camera.grab() ────────────→ IMAGE timestamp ──→ synchronized sensor samples
```

## 3. 資料與參數

| 類別／欄位 | 內容 |
|---|---|
| `get_imu_data()` | 線性加速度、角速度、融合姿態、協方差 |
| `get_magnetometer_data()` | 校正／未校正磁場、Heading、校正狀態 |
| `get_barometer_data()` | 氣壓與相對高度 |
| `get_temperature_data()` | 多個內部溫度讀值 |
| `SensorParameters.sampling_rate` | 最大取樣率 |
| `sensor_range`、`resolution` | 範圍和解析度 |
| `noise_density`、`random_walk` | 濾波器設計所需的噪聲特性 |

官方 Python Tutorial 使用的常見單位：加速度 m/s²、角速度 deg/s、磁場 µT、氣壓 hPa。仍應以 `sensor_unit` 與目前 API 為準。

## 4. API 流程

### 4.1 與影像同步

1. `open()` 相機。
2. 成功 `grab()`。
3. `get_sensors_data(data, TIME_REFERENCE.IMAGE)`。
4. 取出 IMU／磁力計／氣壓計。

### 4.2 高頻讀取

1. `open()` 相機；不需要 Depth 時設 `DEPTH_MODE.NONE`。
2. 不斷呼叫 `get_sensors_data(..., CURRENT)`。
3. 以 timestamp 去重。
4. 迴圈頻率不得低於需要保存的 Sensor 速率。

## 5. Python 完整範例

```python
import time
import pyzed.sl as sl

zed = sl.Camera()
init = sl.InitParameters()
init.depth_mode = sl.DEPTH_MODE.NONE
err = zed.open(init)
if err > sl.ERROR_CODE.SUCCESS:
    raise RuntimeError(err)

info = zed.get_camera_information()
cfg = info.sensors_configuration
print("model:", info.camera_model)
print("IMU rate:", cfg.accelerometer_parameters.sampling_rate)

data = sl.SensorsData()
last_imu = -1
last_mag = -1
last_baro = -1
end_time = time.time() + 5.0

try:
    while time.time() < end_time:
        if zed.get_sensors_data(data, sl.TIME_REFERENCE.CURRENT) > sl.ERROR_CODE.SUCCESS:
            continue

        imu = data.get_imu_data()
        imu_ts = imu.timestamp.get_microseconds()
        if imu_ts > last_imu:
            last_imu = imu_ts
            print("IMU", imu_ts,
                  imu.get_linear_acceleration(),
                  imu.get_angular_velocity(),
                  imu.get_pose().get_orientation().get())

        mag = data.get_magnetometer_data()
        mag_ts = mag.timestamp.get_microseconds()
        if mag_ts > last_mag:
            last_mag = mag_ts
            print("MAG", mag.get_magnetic_field_calibrated())

        baro = data.get_barometer_data()
        baro_ts = baro.timestamp.get_microseconds()
        if baro_ts > last_baro:
            last_baro = baro_ts
            print("BARO", baro.pressure)
finally:
    zed.close()
```

## 6. 影像同步範例

```python
image = sl.Mat()
sensors = sl.SensorsData()
while True:
    if zed.grab() <= sl.ERROR_CODE.SUCCESS:
        zed.retrieve_image(image, sl.VIEW.LEFT)
        if zed.get_sensors_data(sensors, sl.TIME_REFERENCE.IMAGE) <= sl.ERROR_CODE.SUCCESS:
            imu = sensors.get_imu_data()
            save_pair(image, imu, imu.timestamp.get_nanoseconds())
```

## 7. 疑難排解

| 問題 | 建議 |
|---|---|
| Sensor 欄位不可用 | 先檢查 `SensorParameters.is_available` 與相機型號 |
| 樣本重複 | 比較各 Sensor timestamp，不要只比較 `SensorsData` 物件 |
| 漏 IMU 樣本 | 提高 CURRENT 讀取頻率，避免在同一迴圈做慢速 I/O |
| 影像與 IMU 時間不一致 | 使用 IMAGE reference；外部資料需建立明確時鐘轉換 |
| Heading 不穩 | 檢查磁力計校準和附近磁場干擾，不要把磁北直接當真北 |

## 8. 官方範例與參考

- 本地：`zed-sdk-source/tutorials/tutorial 7 - sensor data`
- 本地：`zed-sdk-source/sensors_api/managed`
- 本地：`zed-sdk-source/sensors_api/threaded`
- [Sensors 官方文件](https://docs.stereolabs.com/docs/development/zed-sdk/modules/sensors)
- [Using the Sensors API](https://docs.stereolabs.com/docs/development/zed-sdk/modules/sensors/using-the-api)
- [GitHub zed-sdk](https://github.com/stereolabs/zed-sdk)

