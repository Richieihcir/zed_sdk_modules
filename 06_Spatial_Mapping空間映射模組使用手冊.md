# ZED SDK 空間映射使用手冊

> 適用主題：StereoLabs ZED SDK Spatial Mapping（3D Reconstruction）  
> 文件語言：繁體中文  
> 整理日期：2026-09-20  
> 對照版本：ZED SDK API 5.4.1；`stereolabs/zed-sdk` commit `94bac54b237815053cd876435bb5b2c913603655`（2026-06-18）

## 目錄

1. [介紹](#1-介紹)
2. [支援條件與準備工作](#2-支援條件與準備工作)
3. [運作原理](#3-運作原理)
4. [核心 API 流程](#4-核心-api-流程)
5. [參數說明](#5-參數說明)
6. [輸出格式與後處理](#6-輸出格式與後處理)
7. [Python 完整範例：一次性建圖](#7-python-完整範例一次性建圖)
8. [C++ 範例：非同步即時更新](#8-c-範例非同步即時更新)
9. [官方 GitHub 範例解讀](#9-官方-github-範例解讀)
10. [參數配置範例](#10-參數配置範例)
11. [掃描操作建議](#11-掃描操作建議)
12. [錯誤狀態與疑難排解](#12-錯誤狀態與疑難排解)
13. [參考資料](#13-參考資料)

---

## 1. 介紹

ZED SDK 的空間映射（Spatial Mapping，也稱 3D Reconstruction）會把相機取得的彩色影像、雙目深度與位置追蹤結果持續融合，建立環境的三維地圖。典型用途包括：

- 機器人避障、路徑規劃與導航。
- 室內／室外環境數位化。
- AR/MR 中的遮擋、碰撞與虛實融合。
- 建立可供 MeshLab、Blender 或其他 3D 工具使用的模型。

ZED SDK 可輸出兩種地圖：

| 類型 | ZED SDK 類別 | 內容 | 適合用途 |
|---|---|---|---|
| 三角網格 | `sl::Mesh` / `sl.Mesh` | 頂點、三角面與分塊；可濾波及貼紋理 | 碰撞模型、場景表面、AR/MR、3D 模型 |
| 融合點雲 | `sl::FusedPointCloud` / `sl.FusedPointCloud` | 帶顏色的三維點 | 量測、視覺化、後續點雲處理 |

融合點雲的每個點帶有色彩資訊，通常比未貼圖的 Mesh 需要更多記憶體。Mesh 的紋理必須在建圖時先啟用影像保存，之後才能套用。

## 2. 支援條件與準備工作

### 2.1 相機與硬體

依目前官方文件，本模組需要帶 IMU 的立體相機，例如：

- ZED 2 / ZED 2i
- ZED Mini
- ZED X / ZED X Mini / ZED X Nano

不支援：

- 初代 ZED：沒有內建 IMU。
- ZED X One GS / 4K / S：屬於單眼相機。

ZED SDK 需要相容的 NVIDIA GPU 與 CUDA。正式部署前，請依所安裝 SDK 版本核對官方系統需求；SDK、CUDA 與顯示卡驅動版本必須相容。

### 2.2 軟體

- 安裝 ZED SDK。
- Python：安裝與該 SDK 配套的 PyZED（匯入名稱為 `pyzed.sl`）。
- C++：安裝 CMake、C++17 編譯器，以及範例需要的 CUDA；即時 OpenGL 範例另需 OpenCV、OpenGL、GLUT 與 GLEW。
- 建議先以 ZED Explorer / ZED Depth Viewer 檢查影像、深度和追蹤品質。

### 2.3 取得官方範例

```bash
git clone https://github.com/stereolabs/zed-sdk.git
```

本手冊對應的主要目錄：

```text
zed-sdk/
├─ tutorials/tutorial 5 - spatial mapping/
│  ├─ cpp/main.cpp
│  └─ python/spatial_mapping.py
└─ spatial mapping/spatial mapping/
   ├─ cpp/src/main.cpp
   └─ python/spatial_mapping.py
```

前者是最小教學；後者包含即時顯示、非同步更新、SVO／串流輸入與更多參數。

## 3. 運作原理

### 3.1 資料流程

```text
左／右相機影像
      │
      ├─ 雙目匹配 → 每幀深度圖／3D 點
      │
      └─ 影像特徵 + IMU → 視覺慣性位置追蹤 → 相機在世界座標系的姿態
                                      │
深度 + 姿態 ─────────────────────────┘
      │
      ▼
空間融合：將連續觀測整合到固定的 World Frame
      │
      ├─ Mesh：頂點 + 三角面 + Chunk
      └─ Fused Point Cloud：彩色 3D 點
```

空間映射並不是單張深度圖的直接輸出。每次 `grab()` 成功後，SDK 會在背景把新的影像、深度和相機姿態融合到既有地圖，因此必須先啟用位置追蹤。

### 3.2 世界座標系

地圖被建立在固定的世界座標系（World Frame）中。`InitParameters.coordinate_system` 決定軸向慣例，`coordinate_units` 決定所有距離的單位。若選擇：

```text
RIGHT_HANDED_Y_UP + METER
```

則結果適合多數 OpenGL／三維工具：Y 軸向上，距離以公尺表示。

位置追蹤若啟用 Area Memory 並載入先前的 Area 檔，可在不同工作階段重新定位，使地圖維持在同一物理位置。注意：Area Memory 儲存的是重定位資訊，不等於直接保存完整 Mesh；三維地圖仍應另行匯出。

### 3.3 解析度、範圍與資源的關係

- `resolution_meter` 越小，體素／幾何越密，細節越多，但記憶體和運算量越高。
- `range_meter` 越大，一次可納入更遠的深度，但遠距深度誤差也更大。
- 高解析度加長距離會快速放大需處理的空間體積，是最容易造成低 FPS 或記憶體不足的組合。
- 官方原則是：使用能滿足應用需求的最低密度與最短距離。

### 3.4 Chunk 與非同步更新

SDK 把地圖切成多個 Chunk。即時視覺化通常不必每次重傳整張地圖，可設定 `use_chunk_only = true`，再使用：

1. `requestSpatialMapAsync()` 發出背景擷取要求。
2. `getSpatialMapRequestStatusAsync()` 輪詢是否完成。
3. `retrieveSpatialMapAsync(map)` 取得更新結果。

這可減少即時更新成本，但要求過於頻繁仍會拖慢建圖。官方進階範例約每 0.5 秒請求一次；目前 GitHub C++ 顯示範例則在符合條件時以約 100 ms 間隔嘗試更新，實際專案應按 GPU 負載調整。

## 4. 核心 API 流程

無論 C++ 或 Python，建圖生命週期皆為：

1. 建立相機物件。
2. 設定解析度、深度模式、座標系與單位。
3. `open()` 開啟相機／SVO／串流。
4. `enablePositionalTracking()` / `enable_positional_tracking()`。
5. 設定 `SpatialMappingParameters`。
6. `enableSpatialMapping()` / `enable_spatial_mapping()`。
7. 反覆呼叫 `grab()`；SDK 在背景融合地圖。
8. 檢查 `getSpatialMappingState()`。
9. 一次性或非同步擷取地圖。
10. 若為 Mesh，可進行 `filter()` 與 `applyTexture()`。
11. `save()` 匯出。
12. 依序關閉 Spatial Mapping、Positional Tracking，最後關閉相機。

關鍵規則：

- 一定要先啟用位置追蹤，再啟用空間映射。
- 一定要在 `disableSpatialMapping()` 之前擷取並保存地圖；停用後不能再取圖。
- 結束時先停空間映射，再停位置追蹤。

## 5. 參數說明

### 5.1 相機初始化參數

| 參數 | 建議／範例 | 影響 |
|---|---|---|
| `camera_resolution` | `HD720` 或相機適用的 60 FPS 模式 | 較高 FPS 通常可提升位置追蹤穩定性；官方文件建議 HD720 60 FPS |
| `depth_mode` | `NEURAL` | 影響深度品質、速度與 GPU 負載；官方進階範例使用 NEURAL |
| `coordinate_system` | `RIGHT_HANDED_Y_UP` | 決定輸出地圖軸向 |
| `coordinate_units` | `METER` | 決定 API 中距離數值的單位 |
| `depth_maximum_distance` | 例如 `8.0` | 深度模組的最遠計算距離；與 mapping range 並非同一參數 |
| `input` | 相機、`.svo/.svo2` 或網路串流 | SVO 適合可重複測試與調參 |

### 5.2 `SpatialMappingParameters`

以下以 ZED SDK 5.4.1 API 為基準。

| 參數 | 預設值 | 說明與調整方向 |
|---|---:|---|
| `resolution_meter` | `0.05` m | 幾何解析度。官方概述允許約 0.01–0.12 m；越小越細緻、資源需求越高 |
| `range_meter` | `0` | 建圖深度範圍。`0` 表示 SDK 依解析度與相機狀態自動計算；可與相機的 `depth_maximum_distance` 不同 |
| `map_type` | `MESH` | `MESH` 或 `FUSED_POINT_CLOUD` |
| `max_memory_usage` | `2048` MB | Meshing 使用的最大 CPU 記憶體；不是 GPU VRAM 限額 |
| `save_texture` | `false` | 建圖期間保留影像供 Mesh 貼圖；增加記憶體，只適用 Mesh |
| `use_chunk_only` | `false` | `true` 可提升分塊更新效能；`false` 保持 Mesh 與內部分塊資料一致 |
| `reverse_vertex_order` | `false` | 反轉三角形頂點順序，用於修正前／背面剔除；只適用 Mesh |
| `stability_counter` | `0` | 同一穩定 3D 點需被觀察幾次才整合；`0` 由 SDK 依解析度自動決定。調高較穩但建立較慢 |
| `disparity_std` | `0.3` px | 視差雜訊標準差假設；深度很準可設 `<0.1`，雜訊大可設 `>0.5` |
| `decay` | `1.0` | 當前深度與歷史深度的融合權重；`1` 完整融合，`0` 丟棄過去資料、只整合當前深度 |
| `enable_forget_past` | `false` | 啟用局部地圖，忘記遠離相機的舊區域，以限制記憶體與漂移；保留半徑門檻約為 mapping range 的 1.5 倍 |

#### 解析度預設

概念文件給出的典型值：

| 預設 | 典型解析度 | 用途 |
|---|---:|---|
| `HIGH` | 2 cm | 小範圍、細節較多；最耗資源 |
| `MEDIUM` | 5 cm | 品質與效能平衡 |
| `LOW` | 8 cm | 大範圍、戶外或碰撞網格 |

也可直接設定數值，例如 `resolution_meter = 0.03` 代表 3 cm。

#### 距離預設與版本差異

概念頁面使用：

- `NEAR`：約 3.5 m
- `MEDIUM`：約 5 m
- `FAR`：約 10 m

但 ZED SDK 5.4.1 C++ API 參考列出的列舉名稱為 `SHORT`、`MEDIUM`、`LONG`、`AUTO`。不同 SDK 版本的名稱可能不同，請以本機 SDK 標頭／Python 列舉為準。跨版本程式可優先使用 `MEDIUM`，或直接設定 `range_meter`。

官方概述指出可設定約 2–20 m；目前「Using the API」頁面則說預設範圍被限制在約 10 m。這代表有效上限會受 SDK 版本、相機與深度設定影響，應以當前 API 的 `allowed_range`、回傳錯誤及實測為準。

### 5.3 執行期深度參數

| 參數 | 官方範例 | 說明 |
|---|---:|---|
| `RuntimeParameters.confidence_threshold` | C++ 進階範例 `30`；Python 進階範例 `50` | 過濾低可信度深度。值改變會影響密度與雜訊；應按場景實測 |

### 5.4 Mesh 濾波

完成擷取後可使用 `MeshFilterParameters`。三種常見預設：

| 濾波 | 行為 |
|---|---|
| `LOW` | 主要填洞並清除異常值，保留較多幾何 |
| `MEDIUM` | 中度簡化 |
| `HIGH` | 較強簡化，面數更少 |

濾波只適用 Mesh，不適用 Fused Point Cloud。

## 6. 輸出格式與後處理

### 6.1 Mesh

常見做法：

```text
extractWholeSpatialMap(mesh)
→ mesh.filter(...)
→ mesh.applyTexture(...)（若 save_texture=true）
→ mesh.save("mesh.obj")
```

- OBJ：適合一般三維工具；貼圖通常會搭配 MTL 與影像檔。
- PLY：適合保存幾何或點雲；官方進階 C++ 範例以 PLY 保存。

### 6.2 Fused Point Cloud

建立時將 `map_type` 設為 `FUSED_POINT_CLOUD`，容器也必須改用 `FusedPointCloud`。不要對它呼叫 Mesh 專用的 `filter()` 或 `applyTexture()`。

## 7. Python 完整範例：一次性建圖

以下程式是依官方 Tutorial 與 Sample 重整的最小可用版本。它擷取 500 個成功幀，輸出 `mesh.obj`，並確保失敗時仍正確清理資源。

```python
import sys
import pyzed.sl as sl


def check(code, operation):
    if code > sl.ERROR_CODE.SUCCESS:
        raise RuntimeError(f"{operation} 失敗：{code!r}")


def main():
    zed = sl.Camera()
    mapping_enabled = False
    tracking_enabled = False

    init = sl.InitParameters()
    init.camera_resolution = sl.RESOLUTION.HD720
    init.depth_mode = sl.DEPTH_MODE.NEURAL
    init.coordinate_system = sl.COORDINATE_SYSTEM.RIGHT_HANDED_Y_UP
    init.coordinate_units = sl.UNIT.METER
    init.depth_maximum_distance = 8.0

    try:
        check(zed.open(init), "開啟相機")

        tracking = sl.PositionalTrackingParameters()
        check(zed.enable_positional_tracking(tracking), "啟用位置追蹤")
        tracking_enabled = True

        mapping = sl.SpatialMappingParameters(
            resolution=sl.MAPPING_RESOLUTION.MEDIUM,
            mapping_range=sl.MAPPING_RANGE.MEDIUM,
            max_memory_usage=2048,
            save_texture=False,
            use_chunk_only=False,
            reverse_vertex_order=False,
            map_type=sl.SPATIAL_MAP_TYPE.MESH,
        )
        # 0 讓 SDK 依解析度選擇穩定度；若場景雜訊大可再調高。
        mapping.stability_counter = 0

        check(zed.enable_spatial_mapping(mapping), "啟用空間映射")
        mapping_enabled = True

        runtime = sl.RuntimeParameters()
        runtime.confidence_threshold = 50

        captured = 0
        while captured < 500:
            if zed.grab(runtime) <= sl.ERROR_CODE.SUCCESS:
                state = zed.get_spatial_mapping_state()
                print(
                    f"\r成功幀：{captured + 1:3d}/500，映射狀態：{state!r}",
                    end="",
                    flush=True,
                )
                captured += 1

        print("\n正在擷取完整 Mesh……")
        mesh = sl.Mesh()
        check(zed.extract_whole_spatial_map(mesh), "擷取完整 Mesh")

        filter_params = sl.MeshFilterParameters()
        filter_params.set(sl.MESH_FILTER.LOW)
        mesh.filter(filter_params)

        if not mesh.save("mesh.obj"):
            raise RuntimeError("無法保存 mesh.obj")
        print("完成：mesh.obj")

    finally:
        # 必須先停 Mapping，再停 Tracking。
        if mapping_enabled:
            zed.disable_spatial_mapping()
        if tracking_enabled:
            zed.disable_positional_tracking()
        zed.close()


if __name__ == "__main__":
    try:
        main()
    except Exception as exc:
        print(f"錯誤：{exc}", file=sys.stderr)
        sys.exit(1)
```

執行：

```bash
python spatial_mapping_minimal.py
```

若要改成融合點雲：

```python
mapping.map_type = sl.SPATIAL_MAP_TYPE.FUSED_POINT_CLOUD
spatial_map = sl.FusedPointCloud()
```

然後把 `extract_whole_spatial_map(mesh)` 的容器改成 `spatial_map`，不要執行 Mesh 濾波或貼圖。

## 8. C++ 範例：非同步即時更新

下列片段著重於連續取回 Mesh。它假設相機已開啟、位置追蹤已啟用。

```cpp
#include <sl/Camera.hpp>
#include <chrono>
#include <iostream>

sl::SpatialMappingParameters mapping;
mapping.map_type = sl::SpatialMappingParameters::SPATIAL_MAP_TYPE::MESH;
mapping.set(sl::SpatialMappingParameters::MAPPING_RESOLUTION::MEDIUM);
mapping.set(sl::SpatialMappingParameters::MAPPING_RANGE::MEDIUM);
mapping.use_chunk_only = true;
mapping.max_memory_usage = 2048;
mapping.save_texture = false;
mapping.stability_counter = 0;

auto err = zed.enableSpatialMapping(mapping);
if (err > sl::ERROR_CODE::SUCCESS) {
    std::cerr << "enableSpatialMapping failed: " << err << '\n';
    return EXIT_FAILURE;
}

sl::Mesh mesh;
sl::RuntimeParameters runtime;
runtime.confidence_threshold = 30;

bool request_pending = false;
auto last_request = std::chrono::steady_clock::now();

while (application_is_running) {
    if (zed.grab(runtime) > sl::ERROR_CODE::SUCCESS)
        continue;

    const auto state = zed.getSpatialMappingState();
    if (state == sl::SPATIAL_MAPPING_STATE::FPS_TOO_LOW)
        std::cerr << "警告：建圖 FPS 過低\n";
    if (state == sl::SPATIAL_MAPPING_STATE::NOT_ENOUGH_MEMORY)
        std::cerr << "警告：建圖記憶體不足\n";

    const auto now = std::chrono::steady_clock::now();
    const auto elapsed =
        std::chrono::duration_cast<std::chrono::milliseconds>(now - last_request);

    if (!request_pending && elapsed.count() >= 500) {
        zed.requestSpatialMapAsync();
        request_pending = true;
        last_request = now;
    }

    if (request_pending &&
        zed.getSpatialMapRequestStatusAsync() <= sl::ERROR_CODE::SUCCESS) {
        zed.retrieveSpatialMapAsync(mesh);
        request_pending = false;
        // 在這裡把已更新的 chunks 交給 renderer 或碰撞系統。
    }
}

// 若要保存完整最終結果，先做阻塞式完整擷取。
zed.extractWholeSpatialMap(mesh);
mesh.filter(sl::MeshFilterParameters::MESH_FILTER::LOW);
mesh.save("mesh.obj");

zed.disableSpatialMapping();
zed.disablePositionalTracking();
zed.close();
```

### 8.1 建置官方最小 C++ Tutorial

Linux：

```bash
cd "zed-sdk/tutorials/tutorial 5 - spatial mapping/cpp"
mkdir build
cd build
cmake ..
cmake --build . --config Release
```

Windows：以 CMake GUI 或命令列產生 Visual Studio x64 專案，並用 Release 組態編譯。若建置完整 OpenGL Sample，還須提供 OpenCV、GLUT、GLEW 與 OpenGL。

## 9. 官方 GitHub 範例解讀

### 9.1 Tutorial 5：最小生命週期

官方 Tutorial 的 C++ 與 Python 版本均採下列流程：

```text
開相機 → 啟用 Tracking → 啟用 Mapping
→ 成功 grab 500 幀
→ extractWholeSpatialMap
→ Mesh filter
→ 保存 OBJ
→ 依序停用 Mapping、Tracking、Camera
```

它最適合用來驗證 SDK 安裝與硬體是否可正常建圖，但沒有即時視覺化，也沒有使用非同步擷取。

### 9.2 完整 Spatial Mapping Sample

目前 GitHub 完整 C++ 範例的重要設定：

```cpp
init_parameters.depth_mode = DEPTH_MODE::NEURAL;
spatial_mapping_parameters.set(MAPPING_RESOLUTION::MEDIUM);
spatial_mapping_parameters.set(MAPPING_RANGE::MEDIUM);
spatial_mapping_parameters.use_chunk_only = true;
spatial_mapping_parameters.stability_counter = 10;
spatial_mapping_parameters.decay = 0.1;
runtime_parameters.confidence_threshold = 30;
```

它會等待位置追蹤達到 `OK` 後才啟用 Mapping，並使用非同步要求／取回流程更新 OpenGL 顯示。此做法比「一開 Tracking 就立即建圖」更適合正式應用。

Python 完整範例另提供：

- `--input_svo_file`：從 `.svo` / `.svo2` 重播。
- `--ip_address`：從 IP 或 `IP:port` 串流讀取。
- `--resolution`：選擇 HD2K、HD1200、HD1080、HD720、SVGA 或 VGA。
- `--build_mesh`：指定 Mesh；未指定時範例建立 Fused Point Cloud。
- 空白鍵啟用／停止建圖，停止時擷取並保存結果。

執行範例：

```bash
# 有線相機，建立融合點雲
python spatial_mapping.py

# 有線相機，建立 Mesh
python spatial_mapping.py --build_mesh

# 使用 SVO 重播，建立 Mesh
python spatial_mapping.py --input_svo_file recording.svo2 --build_mesh

# 使用網路串流
python spatial_mapping.py --ip_address 192.168.1.10:30000
```

命令列中不可同時指定 SVO 與 IP 來源。

### 9.3 範例碼中的注意事項

- GitHub Python 完整範例尾端有重複的 `free()` / `close()` 清理碼；自行整合時應使用 `try/finally`，只清理一次。
- 部分 README 保留舊 API 名稱或小型拼字錯誤，例如舊文中的 `extractWholeMesh`；目前範例程式與新文件使用 `extractWholeSpatialMap` / `extract_whole_spatial_map`。
- 官方進階範例的參數是展示用途，不是所有場景的最佳值。特別是 `stability_counter=10` 與 `decay=0.1` 會明顯改變融合速度與歷史權重。

## 10. 參數配置範例

### 10.1 室內精細掃描

```text
resolution: HIGH 或 0.02–0.03 m
range: SHORT/NEAR 或約 3–4 m
map_type: MESH
save_texture: true（需要外觀時）
stability_counter: 先用 0 自動，再按雜訊微調
更新頻率: 較低，避免與高密度建圖搶資源
```

適合小房間或物件，但應預留較多記憶體並緩慢移動相機。

### 10.2 機器人碰撞網格

```text
resolution: LOW，約 0.08 m
range: MEDIUM
map_type: MESH
save_texture: false
use_chunk_only: true
filter: MEDIUM 或 HIGH
```

碰撞通常不需要細緻紋理；較粗解析度能換取快速更新與較低記憶體。

### 10.3 大範圍彩色點雲

```text
resolution: MEDIUM 或 LOW
range: LONG/FAR（先評估遠距誤差）
map_type: FUSED_POINT_CLOUD
save_texture: false
max_memory_usage: 依 RAM 提升
enable_forget_past: 若只需局部地圖可設 true
```

融合點雲帶顏色，長時間建圖時要特別注意記憶體。

## 11. 掃描操作建議

- 開始前先確認深度圖穩定；玻璃、鏡面、反光金屬、純黑區與低紋理白牆都可能降低品質。
- 相機要平順移動，避免快速旋轉、強烈運動模糊與突然遮住鏡頭。
- 保持場景有足夠光線與可追蹤紋理。
- 從近距離、幾何清楚的區域開始，再逐步擴展。
- 讓相鄰視角有重疊，並適時回看已掃區域，降低追蹤漂移。
- 不要長時間只對著平面牆壁或重複紋理。
- 先以 SVO 錄製同一段路徑，再離線測試不同參數，可大幅提高調參可重現性。

## 12. 錯誤狀態與疑難排解

### 12.1 常見 Mapping 狀態

| 狀態 | 意義 | 建議 |
|---|---|---|
| `OK` | 正常整合新資料 | 持續監控 FPS 與 Tracking 狀態 |
| `FPS_TOO_LOW` | 系統無法維持一致建圖所需幀率；停止整合新資料，但仍可擷取現有模型 | 降低解析度／範圍、減少非同步擷取頻率、降低其他 GPU 工作、提高相機 FPS |
| `NOT_ENOUGH_MEMORY` | 達到記憶體限制；停止整合，但仍可擷取現有模型 | 降低密度／範圍、關閉紋理、提高合理的記憶體上限，或啟用局部忘記策略 |
| `NOT_ENABLED` | 尚未啟用或已停用 Mapping | 檢查呼叫順序與 `enableSpatialMapping()` 回傳值 |

### 12.2 地圖破碎或重影

可能原因：位置追蹤不穩、移動過快、深度雜訊、場景有動態物體。

建議：

- 等 Tracking 進入 `OK` 再啟用 Mapping。
- 改善照明並降低移動速度。
- 縮短 mapping range。
- 調整 `confidence_threshold`、`stability_counter` 與 `disparity_std`。
- 移除大量移動物體，或只在環境較靜態時建圖。

### 12.3 紋理無法輸出

檢查：

- `map_type` 必須是 `MESH`。
- 建圖開始前必須設 `save_texture = true`。
- 完整擷取後必須呼叫 `applyTexture()` / `apply_texture()`。
- 保存格式與輸出目錄必須能同時容納模型、材質與紋理影像。

### 12.4 非同步結果沒有更新

檢查是否完成完整三步驟：

```text
request → status == SUCCESS → retrieve
```

一次請求尚未完成前不要重複發送；即時渲染器若使用 Chunk，也要正確更新已變更分塊。

### 12.5 API 名稱不一致

ZED SDK 版本之間可能改名。遇到編譯／屬性錯誤時：

1. 先查本機 SDK 隨附標頭或 Python `dir(sl.MAPPING_RANGE)`。
2. 再查與本機版本相同的 API Reference。
3. 不要直接混用舊 README、最新版網頁與不同版本的 Python wheel。

## 13. 參考資料

### 官方文件

- [Spatial Mapping Overview](https://docs.stereolabs.com/docs/development/zed-sdk/modules/spatial-mapping)
- [Using the Spatial Mapping API](https://docs.stereolabs.com/docs/development/zed-sdk/modules/spatial-mapping/using-the-api)
- [Spatial Mapping Tutorial](https://docs.stereolabs.com/docs/tutorials/spatial-mapping)
- [ZED SDK C++ API 5.4.1（LLM-friendly Markdown）](https://www.stereolabs.com/docs/api/llms-api-cpp.txt)
- [ZED SDK Python API（LLM-friendly Markdown）](https://www.stereolabs.com/docs/api/llms-api-python.txt)
- [ZED SDK Downloads](https://www.stereolabs.com/developers/release/)

### GitHub

- [stereolabs/zed-sdk](https://github.com/stereolabs/zed-sdk)
- [Tutorial 5：Spatial Mapping](https://github.com/stereolabs/zed-sdk/tree/master/tutorials/tutorial%205%20-%20spatial%20mapping)
- [完整 Spatial Mapping Sample](https://github.com/stereolabs/zed-sdk/tree/master/spatial%20mapping/spatial%20mapping)
- [C++ 完整範例 main.cpp](https://github.com/stereolabs/zed-sdk/blob/master/spatial%20mapping/spatial%20mapping/cpp/src/main.cpp)
- [Python 完整範例 spatial_mapping.py](https://github.com/stereolabs/zed-sdk/blob/master/spatial%20mapping/spatial%20mapping/python/spatial_mapping.py)

---

## 快速檢查表

```text
[ ] 相機是支援 IMU 的立體 ZED 型號
[ ] ZED SDK、CUDA、GPU 驅動版本相容
[ ] 深度與位置追蹤正常
[ ] 座標系與單位符合下游系統
[ ] 先 Tracking，後 Mapping
[ ] resolution / range 沒有超出資源預算
[ ] Mesh 與 FusedPointCloud 使用正確容器
[ ] 紋理在建圖前啟用
[ ] 完整結果在停用 Mapping 前擷取
[ ] 先停 Mapping，再停 Tracking
```
