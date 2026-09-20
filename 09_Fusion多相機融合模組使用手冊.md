# ZED SDK Fusion 多相機融合模組使用手冊

> 適用版本：ZED SDK 5.x（整理時 API Reference 為 5.4.1）  
> 程式語言：Python／PyZED  
> 本手冊可獨立閱讀；實際多相機部署需先完成相機外參與網路規劃。

## 1. 介紹

Fusion 模組用來整合一台或多台 ZED 相機的資料。相機端以 Publisher 發布資料，融合端以 Subscriber 訂閱，並在共同座標系中處理：

- 多相機位置追蹤。
- 多人骨架追蹤。
- 多相機物件偵測。
- 深度與空間映射。
- ZED VIO 與 GNSS 全球定位。

它可以在同一台電腦以共享記憶體執行，也可以透過網路分散到多台電腦。

## 2. 架構與原理

```text
ZED A ─> Camera API ─> Publisher ─┐
ZED B ─> Camera API ─> Publisher ─┼─> Fusion Subscriber
ZED C ─> Camera API ─> Publisher ─┘      │
                                          ├─ 時間同步
相機外參／設定檔 ────────────────────────┤
                                          └─ 共同座標下的融合輸出
```

### 2.1 Publisher／Subscriber

Publisher 在每台相機所連接的程序中擷取資料並發布；Fusion 程序依 `CameraIdentifier` 與 `CommunicationParameters` 訂閱資料。

### 2.2 空間校準

每台相機必須有相對共同原點的位姿。外參誤差會直接造成重影、骨架錯位、物件重複或地圖斷裂。

### 2.3 時間同步

融合器會將不同資料流對齊。網路延遲、幀率不一致與時鐘偏差都可能降低結果穩定性。

## 3. 本機與網路模式

| 模式 | 傳輸方式 | 優點 | 注意事項 |
|---|---|---|---|
| 同機多相機 | Shared Memory | 延遲低、設定較簡單 | USB/GPU/CPU 頻寬集中 |
| 多機分散式 | Local Network | 可分散運算與 USB 負載 | 需規劃網路、IP、時鐘與防火牆 |

同一台主機可使用：

```python
communication = sl.CommunicationParameters()
communication.set_for_shared_memory()
```

網路模式則依 SDK API 設定 IP 與連接埠，Publisher 與 Subscriber 必須使用一致的通信設定。

## 4. 設定檔概念

官方多相機範例通常使用 JSON 設定每台相機：

- 相機序號或識別碼。
- 通信方式、IP 與連接埠。
- 相機在共同座標系的平移與旋轉。
- 是否覆寫重力方向。

概念結構如下；實際欄位名稱應以 SDK 範例附帶設定檔為準：

```json
{
  "serial_number": 12345678,
  "communication": {
    "type": "LOCAL_NETWORK",
    "ip": "192.168.1.20",
    "port": 30000
  },
  "world": {
    "translation": [0.0, 1.8, 0.0],
    "rotation": [0.0, 0.0, 0.0]
  },
  "override_gravity": false
}
```

## 5. 主要參數

### 5.1 `InitFusionParameters`

| 參數 | 功能 | 建議 |
|---|---|---|
| `coordinate_units` | 融合輸出的距離單位 | 所有 Publisher 必須一致 |
| `coordinate_system` | 共同座標系 | 所有相機與外參一致 |
| `verbose` | 顯示詳細資訊 | 整合初期開啟 |
| `output_performance_metrics` | 輸出效能與同步指標 | 驗證部署時開啟 |
| `maximum_working_resolution` | 限制融合處理解析度 | GPU 記憶體不足時調低 |

### 5.2 訂閱參數

| 項目 | 說明 |
|---|---|
| `CameraIdentifier` | 通常使用相機序號唯一識別資料流 |
| `CommunicationParameters` | Shared Memory 或網路連線設定 |
| `Transform` | 相機相對共同世界原點的位姿 |

### 5.3 模組專用參數

Fusion 初始化後，還需為目標功能啟用對應參數，例如：

- `PositionalTrackingFusionParameters`
- `BodyTrackingFusionParameters`
- 各版本提供的空間映射或其他融合參數

## 6. 標準操作流程

### Publisher 端

1. 依序號開啟相機。
2. 啟用位置追蹤及需要的 AI 模組。
3. 設定 `CommunicationParameters`。
4. 呼叫 `start_publishing()`。
5. 持續呼叫 `grab()` 產生資料。

### Fusion 端

1. 初始化 `sl.Fusion()`。
2. 讀取每台相機的通信設定與外參。
3. 對每台相機呼叫 `subscribe()`。
4. 啟用所需 Fusion 功能。
5. 迴圈呼叫 `process()`。
6. 取得融合結果與效能指標。

## 7. Python 多相機人體融合骨架

以下程式假設各相機 Publisher 已經啟動，且 `camera_entries` 已由設定檔解析完成。

```python
import pyzed.sl as sl


def create_camera_entries():
    """依部署設定回傳 (serial, communication, world_pose) 清單。"""
    entries = []

    # 同機相機範例：把序號替換成實際值，並為每台相機設定正確外參。
    for serial in (12345678, 87654321):
        communication = sl.CommunicationParameters()
        communication.set_for_shared_memory()

        world_pose = sl.Transform()
        world_pose.set_identity()
        entries.append((serial, communication, world_pose))

    return entries


fusion = sl.Fusion()
init = sl.InitFusionParameters()
init.coordinate_units = sl.UNIT.METER
init.coordinate_system = sl.COORDINATE_SYSTEM.RIGHT_HANDED_Y_UP
init.output_performance_metrics = True
init.verbose = True

status = fusion.init(init)
if status != sl.FUSION_ERROR_CODE.SUCCESS:
    raise RuntimeError(f"Fusion 初始化失敗：{status}")

subscribed = []
for serial, communication, world_pose in create_camera_entries():
    camera_id = sl.CameraIdentifier(serial)
    status = fusion.subscribe(camera_id, communication, world_pose)
    if status == sl.FUSION_ERROR_CODE.SUCCESS:
        subscribed.append(camera_id)
    else:
        print(f"相機 {serial} 訂閱失敗：{status}")

if not subscribed:
    fusion.close()
    raise RuntimeError("沒有可用的相機資料流")

tracking_params = sl.PositionalTrackingFusionParameters()
if fusion.enable_positionnal_tracking(tracking_params) != sl.FUSION_ERROR_CODE.SUCCESS:
    fusion.close()
    raise RuntimeError("Fusion 位置追蹤啟用失敗")

body_params = sl.BodyTrackingFusionParameters()
body_params.enable_tracking = True
body_params.enable_body_fitting = True

if fusion.enable_body_tracking(body_params) != sl.FUSION_ERROR_CODE.SUCCESS:
    fusion.disable_positionnal_tracking()
    fusion.close()
    raise RuntimeError("Fusion 人體追蹤啟用失敗")

bodies = sl.Bodies()
body_runtime = sl.BodyTrackingFusionRuntimeParameters()

try:
    while True:
        if fusion.process() != sl.FUSION_ERROR_CODE.SUCCESS:
            continue

        status = fusion.retrieve_bodies(bodies, body_runtime)
        if status != sl.FUSION_ERROR_CODE.SUCCESS or not bodies.is_new:
            continue

        for body in bodies.body_list:
            print(f"person={body.id}, position={body.position}")
except KeyboardInterrupt:
    pass
finally:
    fusion.disable_body_tracking()
    fusion.disable_positionnal_tracking()
    fusion.close()
```

> 不同 ZED SDK 小版本的 Fusion runtime 類別或欄位可能調整。執行前請用本機 Python 自動完成或 API Reference 核對名稱。

## 8. 外參驗證方法

1. 將一個靜態目標放在兩台以上相機的重疊視野。
2. 分別查看各相機在共同世界座標下的目標位置。
3. 比較融合前後的位置差異與重投影。
4. 從平移、旋轉單位、旋轉順序及座標軸方向逐項排查。
5. 固定所有支架後再做最終標定；任何相機移動都需重新校準。

## 9. 效能與網路診斷

- 為每台相機監控實際 FPS、掉幀、延遲與時間差。
- 網路模式優先使用有線 Gigabit 或更高頻寬連線。
- 避免相機影像、深度與其他大流量資料共用擁塞網段。
- GPU 記憶體不足時，降低模型、解析度、相機數或工作解析度。
- 先用兩台相機驗證，再逐台擴充，較容易定位問題。

## 10. 常見問題

| 現象 | 可能原因 | 建議處理 |
|---|---|---|
| Fusion 找不到相機 | Publisher 未啟動、IP/port/序號錯誤 | 逐一確認資料流與通信設定 |
| 同一人產生多個骨架 | 外參或同步誤差、重疊視野不足 | 重新校準並檢查效能指標 |
| 地圖出現重影 | 相機外參不準或支架移動 | 重新標定並固定相機 |
| 網路相機間歇掉線 | 頻寬不足、防火牆或封包遺失 | 使用有線網路並檢查 port |
| 結果延遲持續增加 | 處理速度低於輸入速度 | 降低負載並檢查每個 Publisher FPS |
| 座標方向錯誤 | Publisher、Fusion、外參座標系不一致 | 統一單位與座標軸定義 |

## 11. GitHub 範例對照

本機 `zed-sdk-source` 可參考：

- `depth sensing/fusion/python/`
- `body tracking/multi-camera/python/`
- `object detection/multi-camera/python/`
- `spatial mapping/multi camera/python/`
- `global localization/`

## 12. 官方參考

- [Fusion 模組](https://docs.stereolabs.com/docs/development/zed-sdk/modules/fusion)
- [ZED SDK GitHub](https://github.com/stereolabs/zed-sdk)

