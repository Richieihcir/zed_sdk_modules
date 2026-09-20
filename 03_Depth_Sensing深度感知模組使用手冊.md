# ZED SDK Depth Sensing 深度感知模組使用手冊

> 整理日期：2026-09-20；對照 ZED SDK API 5.4.1 與官方 GitHub commit `94bac54`。

## 1. 介紹

Depth Sensing 以 ZED 左右影像估計每個像素的深度，可輸出 32-bit Depth、Confidence、表面法向量及帶色彩的 Point Cloud。適用於距離量測、避障、3D 視覺、AR 和上層 AI 模組。

## 2. 原理

雙目三角測量的簡化關係：

```text
Z = f × B / disparity
```

`f` 是焦距、`B` 是基線，disparity 是左右對應像素的水平位移。距離越遠，視差越小，深度誤差通常以距離平方增加。無紋理、反光、透明、過曝或被遮擋區域可能產生無效值。

`MEASURE.DEPTH` 是沿相機 Z 軸的距離；Point Cloud `(X,Y,Z)` 的向量長度才是相機到該點的歐氏距離。

## 3. 參數

### 3.1 `InitParameters`

| 參數 | 說明 |
|---|---|
| `depth_mode` | `PERFORMANCE`、`QUALITY`、`ULTRA`、`NEURAL`、`NEURAL_PLUS`；`NONE` 關閉；5.4 有實驗性 `CUSTOM` |
| `depth_minimum_distance` | 最近有效距離 |
| `depth_maximum_distance` | 最遠深度，縮短可排除不需要的遠距資料 |
| `coordinate_units` | 公尺、毫米等 |
| `coordinate_system` | Point Cloud 與 Normals 軸向 |

### 3.2 `RuntimeParameters`

| 參數 | 說明 |
|---|---|
| `enable_depth` | 本幀是否計算深度 |
| `confidence_threshold` | 過濾低可信度 Depth |
| `texture_confidence_threshold` | 依紋理可信度過濾 |
| `remove_saturated_areas` | 排除過曝飽和區 |
| `measure3D_reference_frame` | 3D Measure 使用 Camera 或 World Frame |

### 3.3 常用 Measure

| Measure | 內容 |
|---|---|
| `DEPTH` | 每像素 32-bit 浮點 Z 深度 |
| `XYZ` | 3D 座標 |
| `XYZRGBA` / `XYZBGRA` | 3D 座標與封裝色彩 |
| `CONFIDENCE` | 深度可信度 |
| `NORMALS` | 表面法向量 |

`retrieve_image(..., VIEW.DEPTH)` 是 8-bit 顯示結果，不得用於量測。

## 4. API 流程

1. 設定 `depth_mode`、單位和範圍。
2. `open()` 相機。
3. 設定 `RuntimeParameters`。
4. 成功 `grab()`。
5. 以 `retrieve_measure()` 取得 Depth、Point Cloud 或 Normals。
6. 檢查 `get_value()` 回傳碼與數值是否 finite。

## 5. Python 完整範例

```python
import math
import pyzed.sl as sl

zed = sl.Camera()
init = sl.InitParameters()
init.depth_mode = sl.DEPTH_MODE.NEURAL
init.coordinate_units = sl.UNIT.METER
init.coordinate_system = sl.COORDINATE_SYSTEM.RIGHT_HANDED_Y_UP
init.depth_maximum_distance = 15.0

err = zed.open(init)
if err > sl.ERROR_CODE.SUCCESS:
    raise RuntimeError(err)

runtime = sl.RuntimeParameters()
runtime.confidence_threshold = 50
image = sl.Mat()
depth = sl.Mat()
cloud = sl.Mat()

try:
    for _ in range(100):
        if zed.grab(runtime) > sl.ERROR_CODE.SUCCESS:
            continue

        zed.retrieve_image(image, sl.VIEW.LEFT)
        zed.retrieve_measure(depth, sl.MEASURE.DEPTH)
        zed.retrieve_measure(cloud, sl.MEASURE.XYZRGBA)

        x, y = image.get_width() // 2, image.get_height() // 2
        depth_err, z = depth.get_value(x, y)
        point_err, p = cloud.get_value(x, y)

        if depth_err <= sl.ERROR_CODE.SUCCESS and math.isfinite(float(z)):
            print("Z depth:", z, "m")

        if point_err <= sl.ERROR_CODE.SUCCESS:
            xyz = [float(v) for v in p[:3]]
            if all(math.isfinite(v) for v in xyz):
                distance = math.sqrt(sum(v * v for v in xyz))
                print("3D point:", xyz, "distance:", distance, "m")
finally:
    zed.close()
```

## 6. 降解析度與 GPU 取回

```python
small = sl.Resolution(640, 360)
point_cloud_gpu = sl.Mat()
if zed.grab(runtime) <= sl.ERROR_CODE.SUCCESS:
    zed.retrieve_measure(
        point_cloud_gpu,
        sl.MEASURE.XYZRGBA,
        sl.MEM.GPU,
        small,
    )
```

重複使用同一 `sl.Mat`。若後續演算法在 CUDA 上，保留在 GPU 可避免昂貴拷貝。

## 7. 參數選擇

| 情境 | 建議 |
|---|---|
| 即時避障 | PERFORMANCE/NEURAL、限制最大距離、降低 Retrieve Resolution |
| 精細量測 | ULTRA/NEURAL_PLUS、穩定光線、縮短距離、提高可信度過濾 |
| AI 3D 偵測 | NEURAL、Meter 單位、Camera/World Frame 與應用一致 |
| 無需 Depth | `DEPTH_MODE.NONE`，減少 GPU 使用 |

## 8. 疑難排解

| 問題 | 建議 |
|---|---|
| NaN/Inf 過多 | 改善紋理與照明、避開玻璃反光、縮短距離、檢查曝光 |
| 遠距抖動 | 深度誤差隨距離增加；縮短 range、使用時間／空間濾波 |
| 畫面可見但量不到 | 確認取的是 `MEASURE.DEPTH`，不是顯示用 `VIEW.DEPTH` |
| FPS 太低 | 降深度模式／解析度，避免不必要的多種 Measure 取回 |
| 3D 軸方向不符 | 統一 `coordinate_system`，勿在下游猜測軸向 |

## 9. 官方範例與參考

- 本地：`zed-sdk-source/tutorials/tutorial 3 - depth sensing`
- 本地：`zed-sdk-source/depth sensing/depth sensing`
- 本地：`zed-sdk-source/depth sensing/export`
- 本地：`zed-sdk-source/depth sensing/voxel point cloud`
- 本地：`zed-sdk-source/depth sensing/custom depth`
- [Depth Sensing 官方文件](https://docs.stereolabs.com/docs/development/zed-sdk/modules/depth-sensing)
- [Using the Depth API](https://docs.stereolabs.com/docs/development/zed-sdk/modules/depth-sensing/using-the-api)
- [GitHub zed-sdk](https://github.com/stereolabs/zed-sdk)

