# ZED SDK Camera 相機模組使用手冊

> 整理日期：2026-09-20；對照 ZED SDK API 5.4.1 與官方 GitHub commit `94bac54`。

## 1. 介紹

Camera 模組是所有 ZED 應用程式的入口，負責開啟實體相機、SVO/SVO2 檔或網路串流，擷取左右影像、控制曝光與白平衡、讀取校正資訊，以及錄製、播放和串流影像。

典型用途包括機器視覺、錄影、遠端串流、離線演算法測試，以及為 Depth、Tracking、Object Detection 等上層模組提供同步影像。

## 2. 原理

```text
Live Camera / SVO / Network Stream
              │
       Camera.open(init)
              │
       Camera.grab(runtime)  ← 推進一個同步時間點
              │
       retrieve_image(view)  ← 取回指定左／右、彩色／灰階影像
```

`InitParameters` 在 `open()` 前建立處理管線；開啟後不可任意更改解析度或 FPS。每次成功 `grab()` 後，影像、深度、姿態和 AI 結果才屬於同一幀。

SVO 主要保存原始未整流影像、時間戳和 Sensor 資料。播放時由當前 SDK 重新執行整流、Depth、Tracking 與 AI，可用同一份錄影反覆比較演算法參數。

## 3. 主要參數

### 3.1 `InitParameters`

| 參數 | 說明 |
|---|---|
| `camera_resolution` | `AUTO`、HD2K、HD1200、HD1080、HD720、SVGA、VGA；可用模式依型號 |
| `camera_fps` | 相機 FPS，必須受所選解析度支援 |
| `input` | Live、SVO 或 Stream；使用 `set_from_svo_file()`、`set_from_stream()` 設定 |
| `sdk_verbose` | SDK 診斷訊息等級 |
| `depth_mode` | 是否計算深度；純影像工作可用 `NONE` 節省資源 |
| `coordinate_units` | 3D 輸出的公尺、毫米等單位 |
| `coordinate_system` | 3D 模組共用的座標軸慣例 |

### 3.2 `VIDEO_SETTINGS`

| 設定 | 功能 |
|---|---|
| `BRIGHTNESS`、`CONTRAST`、`HUE`、`SATURATION`、`SHARPNESS` | 影像外觀 |
| `GAIN`、`EXPOSURE` | 感光增益、曝光；`-1` 通常恢復 Auto |
| `WHITEBALANCE_TEMPERATURE` | 色溫；`-1` 恢復 Auto |
| `LED_STATUS` | 支援型號的狀態 LED |
| `AEC_AGC_ROI` | 自動曝光／增益計算區域 |

### 3.3 常用 View

`VIEW.LEFT`、`RIGHT`、`LEFT_UNRECTIFIED`、`RIGHT_UNRECTIFIED`、`LEFT_GRAY`、`RIGHT_GRAY`。`VIEW.DEPTH` 是顯示用影像，不能取代真正的 32-bit Depth Measure。

## 4. API 流程

1. 建立 `sl.Camera()`。
2. 設定 `sl.InitParameters()`。
3. 呼叫 `open()` 並檢查錯誤。
4. 迴圈中呼叫 `grab()`。
5. 成功後呼叫 `retrieve_image()`。
6. 重複使用同一個 `sl.Mat`，避免每幀重新配置。
7. 停用 Recording／Streaming 等功能後呼叫 `close()`。

## 5. Python 完整範例

```python
import pyzed.sl as sl

zed = sl.Camera()
init = sl.InitParameters()
init.camera_resolution = sl.RESOLUTION.HD1080
init.camera_fps = 30
init.depth_mode = sl.DEPTH_MODE.NONE
init.sdk_verbose = 1

err = zed.open(init)
if err > sl.ERROR_CODE.SUCCESS:
    raise RuntimeError(f"open failed: {err!r}")

# 手動曝光與白平衡；傳入 -1 可恢復 Auto。
zed.set_camera_settings(sl.VIDEO_SETTINGS.EXPOSURE, 50)
zed.set_camera_settings(sl.VIDEO_SETTINGS.WHITEBALANCE_TEMPERATURE, 4600)

image = sl.Mat()
runtime = sl.RuntimeParameters()
try:
    for index in range(100):
        if zed.grab(runtime) <= sl.ERROR_CODE.SUCCESS:
            zed.retrieve_image(image, sl.VIEW.LEFT, sl.MEM.CPU)
            frame = image.get_data()  # NumPy BGRA view
            timestamp = zed.get_timestamp(sl.TIME_REFERENCE.IMAGE)
            print(index, frame.shape, timestamp.get_microseconds())
finally:
    image.free(sl.MEM.CPU)
    zed.close()
```

## 6. SVO 錄製與播放

```python
# 已完成 zed.open() 後：
recording = sl.RecordingParameters()
recording.video_filename = "capture.svo2"
recording.compression_mode = sl.SVO_COMPRESSION_MODE.H264
if zed.enable_recording(recording) > sl.ERROR_CODE.SUCCESS:
    raise RuntimeError("enable recording failed")

for _ in range(300):
    if zed.grab() > sl.ERROR_CODE.SUCCESS:
        break
zed.disable_recording()
```

播放：

```python
init = sl.InitParameters()
init.set_from_svo_file("capture.svo2")
zed = sl.Camera()
zed.open(init)
while True:
    err = zed.grab()
    if err == sl.ERROR_CODE.END_OF_SVOFILE_REACHED:
        break
    if err <= sl.ERROR_CODE.SUCCESS:
        zed.retrieve_image(image, sl.VIEW.LEFT)
zed.close()
```

## 7. 疑難排解

| 問題 | 建議 |
|---|---|
| `open()` 失敗 | 檢查 USB/GMSL 連線、裝置占用、SDK/Driver、解析度與 FPS |
| FPS 不符設定 | 該型號可能不支援該解析度/FPS 組合；查 `CameraInformation` |
| 影像過曝或閃爍 | 使用 Auto，或同時調整 Exposure/Gain；必要時設定 AEC/AGC ROI |
| SVO 播放到尾端 | 正常回傳 `END_OF_SVOFILE_REACHED`；要循環可重設 SVO position |
| CPU/GPU 複製過慢 | 降低 Retrieve Resolution，或讓 CUDA 消費端使用 `MEM.GPU` |

## 8. 官方範例與參考

- 本地：`zed-sdk-source/tutorials/tutorial 1 - hello ZED`
- 本地：`zed-sdk-source/tutorials/tutorial 2 - image capture`
- 本地：`zed-sdk-source/camera control`
- 本地：`zed-sdk-source/camera streaming`
- [Camera 官方文件](https://docs.stereolabs.com/docs/development/zed-sdk/modules/camera)
- [Using the Video API](https://docs.stereolabs.com/docs/development/zed-sdk/modules/camera/using-the-api)
- [GitHub zed-sdk](https://github.com/stereolabs/zed-sdk)

