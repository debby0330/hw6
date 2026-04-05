# Week 6 Prediction Shootout
# 第 6 週：空間預測對決

> 「同一套工具，不同的風暴，不同的答案。這就是為什麼參數調整很重要。」

---

## 專案簡介

本專案針對兩個性質截然不同的降雨事件，在花蓮縣 + 宜蘭縣範圍內，
比較四種空間內插方法的表現，並深入分析 Kriging 的不確定性估計能力。

| 事件 | 說明 | 日期 | 有效測站 | 最大雨量 |
|------|------|------|---------|---------|
| 事件 1 | 楊柳颱風 (Typhoon Yagi) | 2025-08-12 | 34 站 | 29 mm |
| 事件 2 | 0728 豪雨事件 | 2025-07-28 | 154 站 | 171.5 mm |

---

## 資料夾結構

```
hw6/
├── data/
│   ├── rain_20250812/
│   │   └── rain_20250812.csv        # 楊柳颱風雨量資料（CoLife / 中央氣象署）
│   └── rain_20250728/
│       └── rain_20250728.csv        # 0728 豪雨雨量資料（CoLife / 中央氣象署）
├── output/                          # 所有輸出檔案（自動建立）
│   ├── variogram_comparison.png
│   ├── event1_interpolation_comparison.png
│   ├── event2_interpolation_comparison.png
│   ├── event1_kriging_vs_rf_diff.png
│   ├── event2_kriging_vs_rf_diff.png
│   ├── event1_sigma_map.png
│   ├── event2_sigma_map.png
│   ├── kriging_rainfall.tif
│   ├── kriging_variance.tif
│   └── rf_rainfall.tif
├── Week6_Shootout.ipynb             # 主分析 Notebook
└── README.md                        # 本文件
```

---

## 環境需求

Python 3.10 以上，安裝以下套件：

```bash
pip install numpy pandas matplotlib pyproj scipy pykrige scikit-learn rasterio
```

---

## 使用方法

1. 打開 `Week6_Shootout.ipynb`
2. 修改 **A0 第一格** 的路徑設定：

```python
PATH_E1 = r'C:\Users\user\CascadeProjects\hw6\data\rain_20250812\rain_20250812.csv'
PATH_E2 = r'C:\Users\user\CascadeProjects\hw6\data\rain_20250728\rain_20250728.csv'
OUT_DIR = r'C:\Users\user\CascadeProjects\hw6\output'
```

3. `Kernel → Restart & Run All`
4. 所有圖表與 GeoTIFF 自動輸出至 `OUT_DIR`

---

## 分析流程

```
CSV 原始資料
    │
    ▼
A0  篩選花蓮+宜蘭 → 過濾 -998/0 → 每站取 Past24hr 最大值 → 轉換 EPSG:3826
    │
    ▼
A1  Variogram 分析
    ├─ 手動計算實驗 Variogram（15 個 lag bins）
    ├─ 擬合 Spherical vs Exponential（pykrige）
    ├─ SSR 比較選出最佳模型
    └─ 跨事件參數比較（Sill / Range / Nugget）
    │
    ▼
A2  四種內插方法（1000m 網格，EPSG:3826）
    ├─ Nearest Neighbor（scipy NearestNDInterpolator）
    ├─ IDW（手動 cdist，power=2）
    ├─ Ordinary Kriging（pykrige，最佳 Variogram 模型）
    └─ Random Forest（scikit-learn，n=200，min_samples_leaf=3）
    │
    ▼
A3  不確定性分析
    ├─ Sigma Map：Kriging variance → 標準差（Blues colormap）
    └─ Kriging vs RF 差異圖（RdBu_r colormap）
    │
    ▼
A4  GeoTIFF 輸出（事件 2，EPSG:3826）
    ├─ kriging_rainfall.tif
    ├─ kriging_variance.tif
    └─ rf_rainfall.tif
    │
    ▼
A5  跨事件綜合比較表 + 加分問答
```

---

## 輸出檔案說明

| 檔案 | 說明 |
|------|------|
| `variogram_comparison.png` | 兩事件 Variogram 比較圖（Spherical vs Exponential） |
| `event1_interpolation_comparison.png` | 楊柳颱風四種方法 2×2 並列圖 |
| `event2_interpolation_comparison.png` | 0728 豪雨四種方法 2×2 並列圖 |
| `event1_kriging_vs_rf_diff.png` | 楊柳颱風 Kriging − RF 差異圖 |
| `event2_kriging_vs_rf_diff.png` | 0728 豪雨 Kriging − RF 差異圖 |
| `event1_sigma_map.png` | 楊柳颱風 Kriging 預測標準差分布圖 |
| `event2_sigma_map.png` | 0728 豪雨 Kriging 預測標準差分布圖 |
| `kriging_rainfall.tif` | 0728 豪雨 Kriging 插值結果（EPSG:3826，float32） |
| `kriging_variance.tif` | 0728 豪雨 Kriging Variance（EPSG:3826，float32） |
| `rf_rainfall.tif` | 0728 豪雨 Random Forest 插值結果（EPSG:3826，float32） |

> **注意**：GeoTIFF 已做 `np.flipud()` y 軸翻轉，可直接在 QGIS / ArcGIS 中正確顯示。

---

## 主要發現

### Variogram 比較（口訣：Sill 看天氣、Nugget 看儀器、Range 看地理）

| 參數 | 楊柳颱風 | 0728 豪雨 | 原因 |
|------|---------|---------|------|
| **Sill** | 較低 | 極高 | 豪雨強度變異遠大於颱風 |
| **Range** | 長（~138 km） | 短（~18 km） | 颱風環流大；對流豪雨局部叢聚 |
| **Nugget** | 偏高 | ≈ 0 | 颱風測站稀疏（34站）；豪雨密集（154站） |
| **最佳模型** | Exponential | Exponential | 兩者皆符合漸近式空間衰減 |

### Kriging vs Random Forest

- **測站稀疏（楊柳颱風）**：Kriging 明顯較優，RF 訓練點不足容易失準
- **測站密集（0728 豪雨）**：兩者差距縮小，RF 可捕捉局部非線性規律
- **不確定性估計**：Kriging 提供理論保證的 Sigma Map；RF 無法直接輸出可靠的空間不確定性
- **防災決策建議**：優先使用 Kriging，Sigma Map 是判斷「哪裡需要保守決策」的關鍵工具

---

## 資料來源

- **雨量資料**：[CoLife 歷史資料庫](https://history.colife.org.tw/) → 氣象 → 中央氣象署_雨量站
- **事件查詢**：[全民防災 e 點通](https://bear.emic.gov.tw/MY2/disasterInfo)

---

## AI 使用聲明

| 項目 | 說明 |
|------|------|
| 輔助工具 | Claude AI (claude-sonnet-4-6) |
| AI 協助內容 | 資料解析流程、EPSG:3826 座標轉換、四種內插方法程式架構、GeoTIFF 輸出 |
| 學生自行完成 | Variogram 參數解讀、不確定性分析文字、跨事件比較分析、所有圖表結果確認 |
