# ZED SDK Body Tracking 人體追蹤模組使用手冊

> 適用版本：ZED SDK 5.x（整理時 API Reference 為 5.4.1）  
> 程式語言：Python／PyZED  
> 本手冊可獨立閱讀。

## 1. 介紹

Body Tracking 模組辨識人體並輸出二維與三維骨架、人體位置、速度、追蹤 ID 與可選的人體遮罩。它適合：

- 人機互動與手勢分析。
- 運動姿態與人體工學研究。
- 機器人避障與人員追蹤。
- 虛擬製作、動畫與多人空間分析。

SDK 支援不同骨架格式，例如 `BODY_18`、`BODY_34` 與 `BODY_38`。關節點是模型估測的語意關鍵點，不應直接視為醫療等級的真實解剖關節中心。

## 2. 工作原理

```text
左影像 ─> 人體神經網路 ─> 2D 關節 ─┐
ZED 深度 ──────────────────────────┼─> 3D 關節與人體位置
位置追蹤 ──────────────────────────┼─> 世界座標
時間序列 ──────────────────────────┘─> 人體 ID 與速度
                         └───────────> 骨架擬合
```

### 2.1 人體與關節偵測

神經網路從影像估測人體框與關節點。解析度、遮擋、人物尺寸和姿勢會影響關節信心分數。

### 2.2 深度與三維骨架

SDK 使用雙目深度將 2D 關節轉為真實尺度的 3D 座標。深度失效時，部分關節可能得到低可信度或無效座標。

### 2.3 追蹤與骨架擬合

追蹤用來維持人物 ID 並平滑時間序列。`enable_body_fitting` 會加入人體運動學約束，使骨架更連續，但可能增加延遲與計算量。

## 3. 主要參數

### 3.1 初始化參數 `BodyTrackingParameters`

| 參數 | 功能 | 建議 |
|---|---|---|
| `detection_model` | 人體偵測模型 | 在速度與精度間取捨 |
| `body_format` | 骨架關節格式 | 依下游資料格式選 `BODY_18/34/38` |
| `enable_tracking` | 維持人物 ID 與速度 | 連續追蹤建議開啟 |
| `enable_body_fitting` | 套用骨架運動學擬合 | 動畫與姿態平滑可開啟 |
| `enable_segmentation` | 人體遮罩 | 需要輪廓時開啟 |
| `max_range` | 最遠追蹤距離 | 依有效深度範圍設定 |
| `instance_module_id` | 模組實例 ID | 多實例並行時區分 |

### 3.2 執行期參數 `BodyTrackingRuntimeParameters`

| 參數 | 功能 |
|---|---|
| `detection_confidence_threshold` | 人體偵測最低信心分數 |
| `minimum_keypoints_threshold` | 一個人體至少需要的有效關節條件，依版本提供 |
| `skeleton_smoothing` | 骨架時間平滑程度，依版本提供 |

實際欄位可能隨 SDK 版本調整，部署前應以本機 API Reference 與自動完成結果確認。

### 3.3 主要輸出 `Bodies`／`BodyData`

- `id`：人物追蹤 ID。
- `position`、`velocity`：人體中心位置與速度。
- `keypoint_2d`、`keypoint`：2D／3D 關節座標。
- `keypoint_confidence`：各關節信心分數。
- `bounding_box_2d`、`bounding_box`：2D／3D 人體框。
- `local_position_per_joint`：相對父關節的位置。
- `local_orientation_per_joint`：各關節局部旋轉。
- `global_root_orientation`：根節點方向。
- `tracking_state`：人物追蹤狀態。

## 4. 標準操作流程

1. 開啟相機並啟用適合的深度模式。
2. 啟用位置追蹤。
3. 設定骨架格式、模型與 tracking／fitting。
4. 啟用人體追蹤。
5. 每幀擷取後呼叫 `retrieve_bodies()`。
6. 依人物與關節信心分數篩選資料。
7. 結束時停用人體追蹤、位置追蹤並關閉相機。

## 5. Python 完整範例

```python
import math
import pyzed.sl as sl

zed = sl.Camera()
init = sl.InitParameters()
init.coordinate_units = sl.UNIT.METER
init.coordinate_system = sl.COORDINATE_SYSTEM.RIGHT_HANDED_Y_UP
init.depth_mode = sl.DEPTH_MODE.NEURAL

status = zed.open(init)
if status != sl.ERROR_CODE.SUCCESS:
    raise RuntimeError(f"相機開啟失敗：{status}")

tracking = sl.PositionalTrackingParameters()
if zed.enable_positional_tracking(tracking) != sl.ERROR_CODE.SUCCESS:
    zed.close()
    raise RuntimeError("位置追蹤啟用失敗")

params = sl.BodyTrackingParameters()
params.detection_model = sl.BODY_TRACKING_MODEL.HUMAN_BODY_MEDIUM
params.body_format = sl.BODY_FORMAT.BODY_38
params.enable_tracking = True
params.enable_body_fitting = True
params.enable_segmentation = False

status = zed.enable_body_tracking(params)
if status != sl.ERROR_CODE.SUCCESS:
    zed.disable_positional_tracking()
    zed.close()
    raise RuntimeError(f"人體追蹤啟用失敗：{status}")

runtime = sl.RuntimeParameters()
body_runtime = sl.BodyTrackingRuntimeParameters()
body_runtime.detection_confidence_threshold = 40
bodies = sl.Bodies()

try:
    while True:
        if zed.grab(runtime) != sl.ERROR_CODE.SUCCESS:
            continue

        status = zed.retrieve_bodies(bodies, body_runtime)
        if status != sl.ERROR_CODE.SUCCESS or not bodies.is_new:
            continue

        for body in bodies.body_list:
            x, y, z = body.position
            distance = math.sqrt(x * x + y * y + z * z)

            valid_joint_count = sum(
                1 for confidence in body.keypoint_confidence
                if confidence >= 50
            )

            print(
                f"person={body.id:3d} "
                f"distance={distance:.2f} m "
                f"valid_joints={valid_joint_count} "
                f"tracking={body.tracking_state}"
            )
except KeyboardInterrupt:
    pass
finally:
    zed.disable_body_tracking()
    zed.disable_positional_tracking()
    zed.close()
```

## 6. 關節資料使用範例

下游演算法不應只判斷人物的整體 confidence，也需逐關節檢查：

```python
minimum_joint_confidence = 50

for joint_index, point_3d in enumerate(body.keypoint):
    confidence = body.keypoint_confidence[joint_index]
    if confidence < minimum_joint_confidence:
        continue

    x, y, z = point_3d
    if not all(map(lambda value: value == value, (x, y, z))):
        continue  # 排除 NaN

    process_joint(joint_index, x, y, z, confidence)
```

若要計算關節角度，需先依所選 `BODY_FORMAT` 建立正確的骨架拓樸，並對低信心、遮擋及瞬間跳點做時間濾波。

## 7. 品質與安全注意事項

- 關節被身體、家具或其他人遮擋時，位置可能由模型推測而非直接觀測。
- 快速動作會受曝光時間、影像模糊與幀率影響。
- 服裝寬鬆、逆光、人物太小或畫面邊緣都可能降低品質。
- 動作分析應保存信心分數，避免只保存處理後的單一角度。
- 醫療、職安或控制用途需另行驗證精度、延遲與失效模式。
- 收集人體資料時應遵守隱私與個資規範。

## 8. 常見問題

| 現象 | 可能原因 | 建議處理 |
|---|---|---|
| 骨架抖動 | 關節信心低、深度雜訊 | 開啟 fitting、提高門檻並做時間濾波 |
| 人物 ID 交換 | 多人交錯或長時間遮擋 | 提高幀率並加入應用層身份策略 |
| 腳部關節不穩 | 腳被裁切或地面反射 | 調整相機視角並保留完整人體 |
| 3D 關節為 NaN | 關節缺少有效深度 | 檢查距離、照明和深度模式 |
| 執行速度不足 | 高精度模型、BODY_38、fitting 負載高 | 選較快模型或較少關節格式 |

## 9. GitHub 範例對照

本機 `zed-sdk-source` 可參考：

- `tutorials/tutorial 8 - body tracking/python/`
- `body tracking/body tracking/python/`
- `body tracking/multi-camera/python/`
- `body tracking/export/JSON/`
- `body tracking/export/FBX/`
- `body tracking/integrations/`

## 10. 官方參考

- [Body Tracking 模組](https://docs.stereolabs.com/docs/development/zed-sdk/modules/body-tracking)
- [ZED SDK GitHub](https://github.com/stereolabs/zed-sdk)

