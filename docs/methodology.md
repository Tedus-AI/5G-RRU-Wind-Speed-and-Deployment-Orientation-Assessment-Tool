# Methodology — 計算方法與假設

> 本文件公開所有計算公式、係數、假設與資料源。任何工程師都可以從此追蹤每一個數字怎麼來的。

## 1. 資料源 (Data Source)

### Open-Meteo Historical Weather API

- **API**: <https://archive-api.open-meteo.com/v1/archive>
- **授權**: CC-BY 4.0
- **覆蓋**: 全球，1940 年至今（約 1 個月延遲）
- **解析度**: ~10 km grid（ERA5 reanalysis based）
- **抓取期間**: 預設為截止當前 1 個月前的近 5 年逐時 (hourly) 資料
- **抓取變數**:
  - `temperature_2m` — 2 m 高度氣溫 (°C)
  - `wind_speed_10m` — 10 m 高度風速 (m/s)
  - `wind_direction_10m` — 10 m 高度風向 (deg, 氣象慣例 = 風來自的方向)
  - `shortwave_radiation` — 全天空短波輻射 (W/m²)

### 為什麼選 Open-Meteo？

| 條件 | Open-Meteo | NOAA NCEI | NASA POWER | 商用 API |
|---|---|---|---|---|
| 免費 | ✅ | ✅ | ✅ | ❌ |
| 無需 API key | ✅ | ❌ | ⚠️ | ❌ |
| Hourly 資料 | ✅ | ⚠️ | ⚠️ | ✅ |
| 全球覆蓋 | ✅ | ⚠️ | ✅ | ✅ |
| CORS 友善 | ✅ | ❌ | ⚠️ | ✅ |

Open-Meteo 是唯一同時滿足以上所有條件的選擇，適合純前端 (browser-only) 工具。

## 2. 風速高度修正 (Wind Profile)

使用 **Power Law** 公式：

$$
V_h = V_{10} \times \left( \frac{h}{10} \right)^{\alpha} \times s
$$

| 變數 | 意義 | 單位 |
|---|---|---|
| $V_{10}$ | 氣象站 10 m 標準高度風速 | m/s |
| $V_h$ | 修正後的安裝高度 $h$ 風速 | m/s |
| $h$ | 基地台離地架設高度 | m |
| $\alpha$ | Power Law 指數，取決於地形粗糙度 | — |
| $s$ | 局部遮蔽折減係數 | — |

### α 值（地形粗糙度）

| 地形類型 | α | 來源 |
|---|---|---|
| 海岸 / 開闊海域 | 0.10 | Justus & Mikhail (1976), IEC 61400-12 |
| 開闊地 / 草原 | 0.14 | 經典 Hellmann exponent，open country |
| 郊區 / 一般住宅區 | 0.22 | DNV-RP-C205 |
| 市區 / 高樓密集 | 0.33 | DNV-RP-C205, Davenport class IV |

### s 值（遮蔽係數）

| 情境 | s |
|---|---|
| 無遮蔽 | 1.0 |
| 單向遮蔽（一面牆 / 一棟建物） | 0.7 |
| 多向遮蔽（巷弄、屋頂凹處） | 0.4 |
| 嚴重遮蔽（凹角、機房旁） | 0.2 |

> ⚠️ 遮蔽係數是經驗估計，並非標準。實務上若不確定，建議取保守值。

## 3. 主導風向 (Prevailing Wind)

### 為什麼用「最熱季」而不是「全年」？

散熱設計的瓶頸發生在環境溫度最高的時段。如果取全年平均風向，可能會被冬天強勁的東北季風主導，但夏天實際是西南風——兩者方向相反，鰭片若依全年主導風向設計反而錯誤。

### 計算流程

1. 取近 5 年逐時資料
2. 計算每月平均氣溫
3. 取氣溫最高的 3 個月作為「最熱季」（自動處理南北半球差異）
4. 在最熱季的所有 hourly 樣本中，計算 16 方位 × 5 風速級的頻率分佈
5. 取頻率最高的方位作為主導風向

### 風玫瑰圖

- **方位數**: 16 (每 22.5°)
- **風速分級**: 0–1.5 / 1.5–3 / 3–5 / 5–8 / >8 m/s
- **靜風閾值**: V < 0.5 m/s 視為 calm，從風玫瑰中排除
- **顯示方式**: 堆疊極座標圖，由內向外為風速由低到高

## 4. 鰭片方向建議

### 核心原則

**鰭片應平行於主導風向**，使氣流可以順著鰭片通道流過，而不是被鰭片阻擋。

### 機體方位約束

5G RRU 通常綁定特定的天線扇區指向（例如三扇區基地台的 0° / 120° / 240°），工程師沒有完全的旋轉自由度。本工具假設機體可在 **扇區方位 ±45°** 範圍內微調。

### 演算法

```
for offset in [-45° ... +45°] step 1°:
    candidate_az = (sector_az + offset) mod 360
    diff = min(angle_to(candidate_az, wind_axis), angle_to(candidate_az, wind_axis + 180°))
    track best (smallest diff)

recommended_az = sector_az + best_offset
```

風軸是雙向的（鰭片正反兩面都通），所以同時檢查 wind_axis 和 wind_axis+180°。

### 對齊品質指標

$$
\text{alignment\_quality} = \max\left(0, 1 - \frac{\text{best diff}}{45°}\right)
$$

- > 70 % = GOOD（綠燈）
- 40–70 % = MODERATE（黃燈）
- < 40 % = POOR（紅燈，建議改用純自然對流設計）

## 5. 對流模式判定

```
if prevailing_freq >= 30%:
    mode = "mixed"  // 自然 + 強制對流可納入計算
else:
    mode = "natural"  // 純自然對流為基準，風為 bonus
```

30 % 是經驗門檻：若主導風向頻率太低，代表風向變化大，無法在設計階段穩定依賴強制對流。

## 6. 散熱餘裕回收 (Margin Recovery)

### 輸入

| 變數 | 意義 |
|---|---|
| $T_{j,max}$ | 元件接面最高允許溫度 (°C) |
| $\theta_{JC}$ | 封裝 junction-to-case 熱阻 (K/W) |
| $Q$ | 元件功耗 (W) |

### 原始設計（55 °C / 0 m/s 包絡）

要求散熱器熱阻：

$$
\theta_{CA,\text{orig}} = \frac{T_{j,max} - 55}{Q} - \theta_{JC}
$$

### 實際站點包絡

使用最熱季 P95 環境溫度與修正後風速：

$$
\theta_{CA,\text{real}} = \frac{T_{j,max} - T_{P95,\text{hot}}}{Q} - \theta_{JC}
$$

### 強制對流增益

$$
h_{\text{forced}} \approx h_{\text{natural}} \times \sqrt{1 + \frac{V_h}{0.5}}
$$

這是一個非常粗糙的數量級估算，源自簡化的 Reynolds-Nusselt 關聯式對於低速 (< 5 m/s) 平板流的近似。**僅作工程感數值參考，絕對不能替代 CFD。**

### 體積節省估計

假設散熱器熱阻 $\theta_{HS} \propto 1 / (h \cdot A_{\text{fin}})$，且鰭片面積與體積成線性正比，則：

$$
\frac{V_{\text{new}}}{V_{\text{orig}}} \approx \frac{\theta_{CA,\text{orig}} / \theta_{CA,\text{real}}}{h_{\text{boost}}}
$$

$$
\text{Volume Saving} = 1 - \frac{V_{\text{new}}}{V_{\text{orig}}}
$$

上限 cap 在 70 %，避免不切實際的數字。

## 7. 重要假設與限制

1. **微氣候差異**: Open-Meteo 是 ~10 km 網格再分析，無法反映建物峽谷效應、海陸風、屋頂熱島。
2. **線性近似**: 體積節省公式是非常粗略的線性外推，實務上熱阻與鰭片密度的關係是非線性的。
3. **無熱輻射模型**: 目前未納入太陽直射對機殼表面溫度的影響（僅顯示 W/m² 數據）。未來可加入。
4. **無雨水/濕度模型**: 高濕度會略微影響空氣熱物性，但量級小可忽略。
5. **單一元件**: Margin recovery 只考慮一顆關鍵元件，多熱源 + thermal coupling 仍需 CFD。

## 8. 如何驗證本工具的結果？

建議流程：

1. 用本工具對你目前正在設計的 RRU 案例執行分析
2. 取得「實際站點包絡」（P95 T, Vh）
3. 在 FloTHERM / Icepak 中建立兩組 case：
   - Case A: T_amb = 55 °C, V = 0 m/s（傳統 worst-case）
   - Case B: T_amb = P95 from this tool, V = Vh from this tool, 鰭片方向 = recommended
4. 比對兩個 case 的關鍵元件 Tj
5. 若 Case B 的 Tj 比 Case A 低很多，代表確實有 over-design 空間，可以開始減鰭片
6. 但仍要保留一個 worst-case 驗證，以防極端事件

## 9. 未來增強方向 (Roadmap)

- [ ] 加入太陽輻射對機殼表面溫度的解析模型 (Sol-Air Temperature)
- [ ] 加入雨水蒸發冷卻效應
- [ ] 加入「雪夜」與「沙塵暴」極端事件警示
- [ ] 接 NOAA / 中央氣象局開放資料作為交叉驗證
- [ ] FloTHERM / Icepak power map 上傳介面
- [ ] 多元件 (multi-die package) margin 計算

## 10. 引用 (Citation)

如果你在簡報、技術報告或論文中引用本工具的計算結果，請註明：

> Climate data: Open-Meteo Historical Weather API (CC-BY 4.0)
> Tool: 5G RRU Site Thermal Advisor, MIT License
