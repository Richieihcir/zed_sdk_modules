# ZED SDK Object Detection 物件偵測模組使用手冊

> 適用版本：ZED SDK 5.x（整理時 API Reference 為 5.4.1）  
> 程式語言：Python／PyZED  
> 本手冊可獨立閱讀。

## 1. 介紹

Object Detection 模組在影像中辨識人、車輛與其他支援類別，並結合深度與位置追蹤，輸出真實尺度的三維資訊：

- 2D 與 3D 邊界框。
- 相對相機或世界座標的位置。
- 尺寸、速度與追蹤 ID。
- 物件遮罩（啟用 segmentation 時）。
- 信心分數與追蹤狀態。

除了內建模型，也可把 YOLO 等外部模型的 2D 結果送入 ZED，由 SDK 補上深度、三維邊界框與追蹤資訊。

## 2. 工作原理

```text
左影像 ─> 神經網路偵測 ─> 2D 邊界框 ─┐
ZED 深度 ────────────────────────────┼─> 3D 位置／尺寸
位置追蹤 ────────────────────────────┼─> 世界座標
時間序列 ────────────────────────────┘─> 速度／穩定 ID
```

### 2.1 偵測

神經網路先在影像平面找出物件類別與 2D 框。模型越精確，通常需要越高的 GPU 運算量與延遲。

### 2.2 深度提升至三維

SDK 取用框內或遮罩內的深度資訊，估算物件中心、尺寸與 3D 邊界框。無效深度、反光表面或過遠目標會降低結果品質。

### 2.3 時序追蹤

開啟 tracking 後，SDK 依空間位置與時間連續性維持物件 ID，並估算速度。使用追蹤前必須先啟用相機位置追蹤。

## 3. 主要參數

### 3.1 初始化參數 `ObjectDetectionParameters`

| 參數 | 功能 | 選擇建議 |
|---|---|---|
| `detection_model` | 選擇偵測模型 | 即時優先選較快模型，精度優先選較大模型 |
| `enable_tracking` | 維持物件 ID、估算速度 | 動態場景建議開啟 |
| `enable_segmentation` | 輸出像素級遮罩 | 需要輪廓時開啟，會增加負載 |
| `max_range` | 最遠偵測距離 | 依場景與深度品質設定 |
| `filtering_mode` | 物件篩選策略 | 依重疊與穩定性需求調整 |
| `instance_module_id` | 模組實例 ID | 同時執行多個偵測器時必須區分 |
| `batch_parameters` | 批次／軌跡分析設定 | 需要離線軌跡資訊時使用 |

### 3.2 執行期參數 `ObjectDetectionRuntimeParameters`

| 參數 | 功能 |
|---|---|
| `detection_confidence_threshold` | 全域最低信心門檻 |
| `object_class_filter` | 只保留指定類別 |
| `object_class_detection_confidence_threshold` | 各類別獨立門檻 |

門檻太低會增加誤報；太高則可能漏掉小型、遮擋或遠距物件。

### 3.3 主要輸出 `Objects`／`ObjectData`

常用欄位包含：

- `id`：追蹤 ID。
- `label`：物件類別。
- `confidence`：偵測信心分數。
- `position`：三維位置。
- `velocity`：三維速度。
- `dimensions`：寬、高、深。
- `bounding_box_2d`、`bounding_box`：2D／3D 邊界框。
- `mask`：分割遮罩。
- `tracking_state`：追蹤品質狀態。

## 4. 標準操作流程

1. 開啟 ZED，設定公尺單位與神經深度模式。
2. 啟用位置追蹤。
3. 設定並啟用物件偵測。
4. 每幀呼叫 `grab()`。
5. 呼叫 `retrieve_objects()`。
6. 篩選 `tracking_state`、距離與信心分數。
7. 結束時依序停用模組並關閉相機。

## 5. Python 完整範例

```python
import math
import pyzed.sl as sl

zed = sl.Camera()
init = sl.InitParameters()
init.coordinate_units = sl.UNIT.METER
init.depth_mode = sl.DEPTH_MODE.NEURAL
init.coordinate_system = sl.COORDINATE_SYSTEM.RIGHT_HANDED_Y_UP

status = zed.open(init)
if status != sl.ERROR_CODE.SUCCESS:
    raise RuntimeError(f"相機開啟失敗：{status}")

tracking = sl.PositionalTrackingParameters()
if zed.enable_positional_tracking(tracking) != sl.ERROR_CODE.SUCCESS:
    zed.close()
    raise RuntimeError("位置追蹤啟用失敗")

params = sl.ObjectDetectionParameters()
params.detection_model = sl.OBJECT_DETECTION_MODEL.MULTI_CLASS_BOX_MEDIUM
params.enable_tracking = True
params.enable_segmentation = True

status = zed.enable_object_detection(params)
if status != sl.ERROR_CODE.SUCCESS:
    zed.disable_positional_tracking()
    zed.close()
    raise RuntimeError(f"物件偵測啟用失敗：{status}")

runtime = sl.RuntimeParameters()
object_runtime = sl.ObjectDetectionRuntimeParameters()
object_runtime.detection_confidence_threshold = 50
objects = sl.Objects()

try:
    while True:
        if zed.grab(runtime) != sl.ERROR_CODE.SUCCESS:
            continue

        status = zed.retrieve_objects(objects, object_runtime)
        if status != sl.ERROR_CODE.SUCCESS or not objects.is_new:
            continue

        for obj in objects.object_list:
            x, y, z = obj.position
            distance = math.sqrt(x * x + y * y + z * z)
            print(
                f"id={obj.id:3d} "
                f"class={obj.label} "
                f"confidence={obj.confidence:.1f}% "
                f"distance={distance:.2f} m "
                f"tracking={obj.tracking_state}"
            )
except KeyboardInterrupt:
    pass
finally:
    zed.disable_object_detection()
    zed.disable_positional_tracking()
    zed.close()
```

首次使用指定 AI 模型時，SDK 可能需要進行模型最佳化，啟動時間會比後續執行長。

## 6. 自訂偵測器流程

外部模型負責產生 2D 框，ZED SDK 負責深度與追蹤：

1. 使用 ZED 擷取影像。
2. 將影像送入自訂模型。
3. 把類別、信心分數、唯一 ID 與 2D 框轉為 `CustomBoxObjectData`。
4. 呼叫 `ingest_custom_box_objects()`。
5. 再用 `retrieve_objects()` 取得 3D 結果。

概念程式碼：

```python
custom_objects = []
for detection in detector_results:
    item = sl.CustomBoxObjectData()
    item.unique_object_id = sl.generate_unique_id()
    item.probability = detection.confidence
    item.label = detection.class_id
    item.bounding_box_2d = detection.zed_box_corners
    item.is_grounded = detection.is_grounded
    custom_objects.append(item)

zed.ingest_custom_box_objects(custom_objects)
zed.retrieve_objects(objects, object_runtime)
```

2D 框角點順序與像素座標必須符合 API 定義；若影像曾縮放或裁切，必須先映射回送入 ZED 的原始影像座標。

## 7. 品質與效能調校

- 降低模型複雜度、關閉 segmentation 或降低輸入解析度可提升 FPS。
- `max_range` 不宜大於場景中可用深度的可靠範圍。
- 追蹤 ID 不等同永久身份；長時間遮擋或離開畫面後可能重新編號。
- 只處理需要的類別，可減少後處理負擔。
- 以 `tracking_state == OK`、距離與信心門檻共同決定是否使用結果。

## 8. 常見問題

| 現象 | 可能原因 | 建議處理 |
|---|---|---|
| 沒有任何結果 | 模型未載入、信心門檻過高 | 檢查啟用狀態並降低門檻 |
| 3D 位置為無效值 | 目標缺少有效深度 | 改善照明、縮短距離或調整深度模式 |
| ID 頻繁切換 | 遮擋、低 FPS、相機姿態不穩 | 提升 FPS、固定相機並啟用位置追蹤 |
| 推論速度過慢 | 模型太大、分割開啟、GPU 不足 | 選較快模型或關閉遮罩 |
| 自訂框位置錯誤 | 縮放座標未映射回原圖 | 檢查 letterbox、裁切與角點順序 |

## 9. GitHub 範例對照

本機 `zed-sdk-source` 可參考：

- `tutorials/tutorial 6 - object detection/python/`
- `object detection/image viewer/python/`
- `object detection/birds eye viewer/python/`
- `object detection/concurrent detections/python/`
- `object detection/custom detector/python/`
- `object detection/multi-camera/python/`

## 10. 官方參考

- [Object Detection 模組](https://docs.stereolabs.com/docs/development/zed-sdk/modules/object-detection)
- [ZED SDK GitHub](https://github.com/stereolabs/zed-sdk)

