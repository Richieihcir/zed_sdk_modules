# ZED SDK Positional Tracking 位置追蹤模組使用手冊

> 整理日期：2026-09-20；對照 ZED SDK API 5.4.1 與官方 GitHub commit `94bac54`。

## 1. 介紹

Positional Tracking 即時估計相機的 6DoF 姿態：位置 `(X,Y,Z)` 與旋轉。適用 VSLAM、機器人里程計、AR/VR、穩定 Object/Body ID，以及 Spatial Mapping。

目前官方文件要求帶 IMU 的立體相機，例如 ZED 2/2i、ZED Mini、ZED X 系列；初代 ZED 與單眼 ZED X One 不支援目前完整模組。

## 2. 原理

```text
Stereo frames → 3D visual features → Visual Odometry ─┐
IMU → acceleration + angular velocity ────────────────┤
Sparse landmarks + loop closure ──────────────────────┤
                                                       ▼
                                         Visual-Inertial SLAM Pose
```

- Stereo Visual Odometry 追蹤連續幀的 3D 特徵。
- IMU 在快速移動和低紋理時提供高頻運動資訊。
- Sparse SLAM Map 保存地標，支援回環閉合和重新定位以減少漂移。

## 3. 座標框架

| Frame | 意義 |
|---|---|
| `REFERENCE_FRAME.WORLD` | 相對啟動原點或 Area Map 的全域姿態 |
| `REFERENCE_FRAME.CAMERA` | 相對上一姿態的增量，適合里程計 |

預設 Pose 位於左相機光心。若需要相機中心、IMU 或機器人 Base Frame，應套用標定外參。

## 4. `PositionalTrackingParameters`

| 參數 | 說明 |
|---|---|
| `initial_world_transform` / `_init_pos` | 自訂起始姿態 |
| `enable_imu_fusion` | 融合 IMU，通常保持啟用 |
| `set_gravity_as_origin` | 以重力對齊世界方向 |
| `set_floor_as_origin` | 偵測地面並把原點置於地板 |
| `enable_area_memory` | 建立稀疏 Area、回環與重新定位 |
| `area_file_path` | 載入既有 `.area` 檔 |
| `enable_pose_smoothing` | 平滑 Pose，但增加延遲 |
| `set_as_static` | 固定相機專用；移動相機不可設錯 |
| `depth_min_range` | Tracking 使用的最小深度 |
| `enable_2d_ground_mode` | 平面導航限制 |
| `mode` | Tracking 演算法模式；依 SDK 版本 |

## 5. API 流程

1. 設定座標系和單位後 `open()`。
2. 建立 `PositionalTrackingParameters`。
3. `enable_positional_tracking()`。
4. 每次成功 `grab()` 後呼叫 `get_position()`。
5. 檢查 `POSITIONAL_TRACKING_STATE`。
6. 取出 Translation、Orientation、Twist、Timestamp 和 Confidence。
7. 結束時 `disable_positional_tracking()` 再 `close()`。

## 6. Python 完整範例

```python
import pyzed.sl as sl

zed = sl.Camera()
init = sl.InitParameters()
init.camera_resolution = sl.RESOLUTION.HD720
init.coordinate_units = sl.UNIT.METER
init.coordinate_system = sl.COORDINATE_SYSTEM.RIGHT_HANDED_Y_UP

err = zed.open(init)
if err > sl.ERROR_CODE.SUCCESS:
    raise RuntimeError(err)

tracking = sl.PositionalTrackingParameters()
tracking.enable_imu_fusion = True
tracking.set_gravity_as_origin = True
tracking.enable_area_memory = True
tracking.enable_pose_smoothing = False

err = zed.enable_positional_tracking(tracking)
if err > sl.ERROR_CODE.SUCCESS:
    zed.close()
    raise RuntimeError(err)

pose = sl.Pose()
translation = sl.Translation()
orientation = sl.Orientation()

try:
    for _ in range(1000):
        if zed.grab() > sl.ERROR_CODE.SUCCESS:
            continue
        state = zed.get_position(pose, sl.REFERENCE_FRAME.WORLD)
        if state == sl.POSITIONAL_TRACKING_STATE.OK:
            xyz = pose.get_translation(translation).get()
            quat = pose.get_orientation(orientation).get()
            print({
                "timestamp_us": pose.timestamp.get_microseconds(),
                "translation_m": list(xyz),
                "quaternion_xyzw": list(quat),
                "twist": list(pose.twist),
            })
finally:
    zed.disable_positional_tracking()
    zed.close()
```

## 7. Area Memory

Area Memory 是 VSLAM 稀疏地標資料，不是稠密 Mesh。可用於：

- 回到已知環境後重新定位。
- 跨工作階段保持一致 World Frame。
- 透過回環降低長距離漂移。

載入 Area 後，Tracking 可能先處於 `SEARCHING`。只有可靠狀態才能交給導航或 Mapping。保存 Area 的 API 名稱和狀態查詢在 SDK 版本間可能不同，應以本機 API Reference 為準。

## 8. 狀態與疑難排解

| 狀態／問題 | 建議 |
|---|---|
| `OK` | 可使用 Pose，仍應監控 Confidence |
| `SEARCHING` | 增加環境特徵、放慢移動、回到已知區域 |
| `FPS_TOO_LOW` | 降低其他 GPU 工作、解析度或模型負載 |
| 漂移過大 | 改善照明、增加回環路徑、啟用 Area Memory、校正相機 |
| 姿態跳動 | 檢查重新定位事件；必要時用 smoothing，但注意延遲 |
| 地面原點錯誤 | 場景需可見穩定地面；不適合時關閉 `set_floor_as_origin` |

## 9. 官方範例與參考

- 本地：`zed-sdk-source/tutorials/tutorial 4 - positional tracking`
- 本地：`zed-sdk-source/positional tracking/positional tracking`
- 本地：`zed-sdk-source/positional tracking/export/fbx`
- [Positional Tracking 官方文件](https://docs.stereolabs.com/docs/development/zed-sdk/modules/positional-tracking)
- [Using the Positional Tracking API](https://docs.stereolabs.com/docs/development/zed-sdk/modules/positional-tracking/using-the-api)
- [GitHub zed-sdk](https://github.com/stereolabs/zed-sdk)

