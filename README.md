# 5G RRU Site Thermal Advisor

> 用真實當地氣候資料優化 5G RRU 戶外基站散熱設計，避免 over-design。
> Use real local climate data to optimize 5G RRU outdoor site thermal design and avoid over-engineering.

[繁體中文](#中文說明) · [English](#english)

---

## 中文說明

### 為什麼需要這個工具？

在 5G RRU（Remote Radio Unit）的熱流設計中，業界通常以「環境溫度 55 °C、無風（純自然對流）」作為散熱設計的最嚴苛假設。這個假設保護性很高，但也造成兩個常見問題：

1. **散熱器體積過大**：實際 90 % 以上的安裝環境，環境溫度遠低於 55 °C，且大多時候有 1～4 m/s 的風速可供混合對流；以 worst-case 設計會導致鰭片面積、體積、重量被過度堆疊。
2. **鰭片方向錯失強制對流增益**：現場安裝時若沒有特別考量當地盛行風向，鰭片走向經常垂直於主風向，反而阻擋氣流，浪費掉一個免費的散熱輔助。

本工具的核心想法是：

- **抓取站點當地真實的歷史氣候資料**（近 5 年 hourly 溫度、風速、風向、太陽輻射），讓散熱工程師看到 P95 環境溫度、年累積 > 55 °C 小時數、主導風向頻率等具體數字，作為「合理化設計包絡」的依據。
- **依據主導風向給出鰭片走向與機體方位建議**，並考量天線扇區方位 ±45° 約束。
- **量化散熱餘裕回收與體積節省幅度**，讓客戶與工程師有共同的討論基礎。

### 主要功能

- **真實互動地圖**（Leaflet + CartoDB Dark）— 點擊或搜尋地址即可選定站點
- **Open-Meteo 歷史氣候 API** — 免費、無需 API key、CC-BY 4.0 授權
- **Power Law 風速高度修正** — 依據地形粗糙度與遮蔽係數
- **環境溫度直方圖** — 顯示 P50 / P95 / Max 與 55 °C 設計線的對比
- **風玫瑰圖** — 16 方位 × 5 風速級的最熱季風頻率分佈
- **鰭片方向羅盤** — SVG 視覺化建議的鰭片走向與機體方位
- **散熱餘裕回收計算器** — 輸入 Tj_max / θ_JC / 功耗，估算可節省的散熱器體積百分比
- **多站點比較** — 儲存多個站點並匯出 CSV
- **PDF 報告匯出** — 給客戶會議用
- **三種角色視圖**：
  - 熱流工程師（完整 dashboard，含 margin recovery、多站比較）
  - 客戶（精簡版，強調體積節省與商業價值）
  - 現場工程師（行動裝置友善，大型羅盤 + 安裝 QR Code）
- **中英雙語切換**

### 線上試用

部署到 GitHub Pages 後，直接在瀏覽器開啟即可。預設會自動載入「台北 101」站點作為 demo。

### 本地開發

無 build step。直接在瀏覽器開啟 `index.html` 即可（建議用 local server 避免 CORS）：

```bash
python3 -m http.server 8000
# 然後開啟 http://localhost:8000
```

### 部署到 GitHub Pages

1. Fork 或 clone 此 repo
2. 在 GitHub repo Settings → Pages → Source 選擇 `GitHub Actions`
3. Push 到 `main` 分支，`.github/workflows/pages.yml` 會自動部署
4. 部署完成後，工具會在 `https://<your-username>.github.io/<repo-name>/` 上線

### 計算方法詳細說明

請參考 [`docs/methodology.md`](docs/methodology.md)，內含所有公式、係數、假設與資料源 attribution。

### 技術 Stack

- **無 build step**：純 HTML + CSS + Vanilla JavaScript
- **Leaflet 1.9** — 互動式地圖
- **Chart.js 4.4** — 溫度直方圖
- **jsPDF + html2canvas** — PDF 報告匯出
- **QRCode.js** — 現場安裝二維碼
- **IBM Plex Sans / Mono** — 工程感字體
- **CartoDB Dark Matter** — 暗色地圖底圖
- **Open-Meteo Historical Weather API** — 氣候資料源

### 重要免責聲明

⚠️ **本工具是「佈建前的初步評估」，不是 CFD 模擬替代品。**

- 風速 Power Law 修正只考慮地形粗糙度與簡化遮蔽係數，無法模擬真實的局部繞流、亂流。
- 強制對流增益係數 `h_forced ≈ h_natural × √(1 + V/0.5)` 是非常粗糙的數量級估算。
- 體積節省百分比是線性近似，實際散熱器設計仍需 FloTHERM / Icepak / FloEFD 驗證關鍵 worst-case。
- Open-Meteo 為再分析資料（reanalysis），與站點實際微氣候會因建物、植被、海陸風變化而有偏差。

### 授權

MIT License — 你可以自由修改、商用、分發。但天氣資料的使用須遵守 Open-Meteo 的 CC-BY 4.0 授權，請保留 attribution。

---

## English

### Why this tool?

Traditional 5G RRU thermal design uses "ambient = 55 °C, V = 0 m/s" as the worst-case envelope. While safe, this leads to two recurring issues:

1. **Heat sinks are oversized.** Most installation sites spend 90 %+ of their time well below 55 °C with 1–4 m/s of wind available for mixed convection. Designing for the worst case stacks fin area, volume, and weight unnecessarily.
2. **Field installations miss the forced-convection bonus.** Without explicitly aligning fins to local prevailing wind, technicians often install RRUs with fins perpendicular to the dominant flow, wasting a free cooling assist.

This tool addresses both by:

- Pulling **real 5-year hourly historical climate data** for any site (temperature, wind speed/direction, solar radiation) so thermal engineers see actual P95 ambient, hours/year above 55 °C, and prevailing wind frequency.
- Recommending **fin orientation and unit azimuth** based on the prevailing wind, constrained by the antenna sector ±45°.
- Quantifying **thermal margin recovered** and **estimated heat sink volume reduction** so engineers and customers have a shared basis for design relaxation.

### Features

- Interactive map (Leaflet + CartoDB Dark)
- Open-Meteo historical climate API (free, no API key)
- Power Law wind correction by terrain roughness + shielding
- Temperature histogram with P50/P95/Max markers vs 55 °C design line
- Wind rose (16 directions × 5 speed bins, hot-season filtered)
- SVG fin orientation compass
- Thermal margin recovery & volume saving estimator
- Multi-site comparison + CSV export
- PDF report export
- 3 role views: Thermal Engineer / Customer / Field Engineer
- Bilingual (中文 / English)

### Quick Start

```bash
python3 -m http.server 8000
# open http://localhost:8000
```

### Deploy to GitHub Pages

1. Fork/clone this repo
2. Settings → Pages → Source = `GitHub Actions`
3. Push to `main` — `.github/workflows/pages.yml` deploys automatically

### Methodology

See [`docs/methodology.md`](docs/methodology.md) for all formulas, coefficients, assumptions, and data source attribution.

### Disclaimer

⚠️ This is a **preliminary site assessment tool**, not a CFD substitute.

- Power Law correction is a simplified terrain model.
- Forced convection boost is a rough order-of-magnitude estimate.
- Volume reduction is a linear approximation.
- Final thermal design must be verified with FloTHERM/Icepak/FloEFD.

### License

MIT License. Climate data via Open-Meteo (CC-BY 4.0) — please retain attribution.

---

## Acknowledgments

- [Open-Meteo](https://open-meteo.com/) — historical weather API
- [Leaflet](https://leafletjs.com/) — interactive maps
- [Chart.js](https://www.chartjs.org/) — charts
- [CARTO](https://carto.com/basemaps/) — dark basemap tiles
- IBM Plex font family
