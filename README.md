# 基于GEE与Landsat的RSEI生态遥感指数计算指南
####  本文借鉴了RaySpaniare的博客，欢迎大家进一步完善。

## 一、前言
本文重点讲解**RSEI生态遥感指数**的计算方法。该指数由徐涵秋教授于2013年提出，至今仍被广泛应用。本文基于GEE平台和Landsat8/9 C2L2数据，解决以下常见问题：
1. ​**PC1的正负判断**：何时使用PC1或1-PC1？
2. ​**计算步骤**：如何从NDVI、Wet、LST、NDBSI生成RSEI？

---

## 二、数据准备
### 2.1 数据源与注意事项
- ​**数据源**：Landsat8/9 C2L2（Collection 2 Level 2）
- ​**关键处理**：
  - ​**缩放因子**：NDVI和LST需通过缩放因子和偏移量校正（参考[USGS指南](https://www.usgs.gov/media/files/landsat-8-9-collection-2-level-2-science-product-guide)）
  - ​**缨帽变换(Wet指数)**：不同传感器系数不同（Landsat7 ETM+/Landsat8 OLI/Modis需分别处理）

### 2.2 指数公式
| 指数    | 计算公式                          | 相关波段                |
|---------|-----------------------------------|-------------------------|
| NDVI    | (NIR - Red) / (NIR + Red)        | B5, B4 (Landsat8/9)     |
| Wet     | Tasseled Cap变换（固定系数法）    | 多波段线性组合          |
| LST     | 热红外波段反演（单窗算法）        | B10/B11 (Landsat8/9)    |
| NDBSI   | (SI + IBI) / 2                    | SI: (B5+B3-B6)/(B5+B3+B6)<br>IBI: [2B5/(B5+B4) - (B2+B6)/(B2+B6)] / [2B5/(B5+B4) + (B2+B6)/(B2+B6)] |

---

## 三、计算流程
### 3.1 标准化处理
对NDVI、Wet、LST、NDBSI进行归一化：
```math
X_{\text{norm}} = \frac{X - X_{\min}}{X_{\max} - X_{\min}}

#


