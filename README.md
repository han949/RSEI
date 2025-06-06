# 基于GEE与Landsat的RSEI生态遥感指数计算指南

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
```
# 示例代码

以下是JavaScript代码示例：

```javascript
//说明：注意我这里用的是年尺度的RSEI计算，故采用的均值合成而不是中值合成
// 导入研究区
var geometry = ee.FeatureCollection("users/RaySpaniare/jiujiang");
// 加载 Landsat 8/9 数据集
var landsat = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')  // Landsat 8
  .merge(ee.ImageCollection('LANDSAT/LC09/C02/T1_L2'))    // Landsat 9
  .filterDate('2023-01-01', '2023-12-31')
  .filterBounds(geometry)
  .filter(ee.Filter.lt('CLOUD_COVER', 1));
// 计算各个指数
function calculateIndices(image) {
  // 应用缩放因子
  var optical = image.select(['SR_B2', 'SR_B3', 'SR_B4', 'SR_B5', 'SR_B6', 'SR_B7'])
    .multiply(0.0000275).add(-0.2);  // 光学波段的缩放
  var thermal = image.select(['ST_B10'])
    .multiply(0.00341802).add(149.0);  // 热红外波段的缩放
  // 重命名波段以便计算
  optical = optical.select(
    ['SR_B2', 'SR_B3', 'SR_B4', 'SR_B5', 'SR_B6', 'SR_B7'],
    ['Blue', 'Green', 'Red', 'NIR', 'SWIR1', 'SWIR2']
  );
  // 湿度指数 - Tasseled Cap Wetness for Landsat 8/9 OLI
  var wetness = optical.expression(
    'Blue * 0.1511 + Green * 0.1973 + Red * 0.3283 + NIR * 0.3407 + SWIR1 * (-0.7117) + SWIR2 * (-0.4559)',
    {
      'Blue': optical.select('Blue'),
      'Green': optical.select('Green'),
      'Red': optical.select('Red'),
      'NIR': optical.select('NIR'),
      'SWIR1': optical.select('SWIR1'),
      'SWIR2': optical.select('SWIR2')
    }
  ).rename('wetness');
  // 绿度指数 - NDVI (-1 到 1)
  var ndvi = optical.normalizedDifference(['NIR', 'Red'])
    .rename('greenness');
  // 干度指数 - NDBSI
  // 1. 计算 SI (Soil Index)
  var si = optical.expression(
    '((SWIR1 + Red) - (NIR + Blue)) / ((SWIR1 + Red) + (NIR + Blue))',
    {
      'SWIR1': optical.select('SWIR1'),
      'Red': optical.select('Red'),
      'NIR': optical.select('NIR'),
      'Blue': optical.select('Blue')
    }
  );
  // 2. 计算 IBI (Index-based Built-up Index)
  var ibi = optical.expression(
    '(2 * SWIR1 / (SWIR1 + NIR) - (NIR / (NIR + Red) + Green / (Green + SWIR1))) / ' +
    '(2 * SWIR1 / (SWIR1 + NIR) + (NIR / (NIR + Red) + Green / (Green + SWIR1)))',
    {
      'SWIR1': optical.select('SWIR1'),
      'NIR': optical.select('NIR'),
      'Red': optical.select('Red'),
      'Green': optical.select('Green')
    }
  );
  // 3. 计算 NDBSI (Normalized Difference Bare Soil Index)
  var ndbsi = si.add(ibi).divide(2)
    .rename('dryness');
  // 热度指数 - 使用地表温度
  var lst = thermal.select(['ST_B10'])
    .subtract(273.15)  // 转换为摄氏度
    .rename('heat');
  return image.addBands([wetness, ndvi, ndbsi, lst]);
}
// 对影像集应用指数计算并获取均值影像
var withIndices = landsat.map(calculateIndices);
// 分别获取四个指标的均值影像
var greenness = withIndices.select('greenness').mean();
var wetness = withIndices.select('wetness').mean();
var heat = withIndices.select('heat').mean();
var dryness = withIndices.select('dryness').mean();
// 直接使用研究区实际边界
var region = geometry;
// 设置投影信息和计算参数
var projection = 'EPSG:32648';  // WGS84 UTM Zone 48N
var scale = 30;  // Landsat 的空间分辨率
// 将四个指标标准化用于PCA计算
function standardizeForPCA(image) {
  var stats = image.reduceRegion({
    reducer: ee.Reducer.minMax(),
    geometry: region,
    scale: scale,
    maxPixels: 1e13
  });
  var min = ee.Number(stats.values().get(0));  // 获取最小值
  var max = ee.Number(stats.values().get(1));  // 获取最大值
  return image.subtract(min).divide(max.subtract(min));
}
// 标准化四个指标
var greenness_std = standardizeForPCA(greenness);
var wetness_std = standardizeForPCA(wetness);
var heat_std = standardizeForPCA(heat);
var dryness_std = standardizeForPCA(dryness);
// 组合标准化后的指标用于PCA
var compositeImage = ee.Image.cat([greenness_std, wetness_std, heat_std, dryness_std]);
// PCA 计算函数
function calculatePCA(image) {
  var scale = 30;
  var bandNames = image.bandNames();
  // 计算均值
  var meanDict = image.reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: region,
    scale: scale,
    maxPixels: 1e13
  });
  // 中心化
  var means = ee.Image.constant(meanDict.values(bandNames));
  var centered = image.subtract(means);
  // 转换为数组
  var arrays = centered.toArray();
  // 计算协方差矩阵
  var covar = arrays.reduceRegion({
    reducer: ee.Reducer.centeredCovariance(),
    geometry: region,
    scale: scale,
    maxPixels: 1e13
  });
  // 特征值分解
  var covarArray = ee.Array(covar.get('array'));
  var eigens = covarArray.eigen();

  // 获取特征向量
  var eigenVectors = eigens.slice(1, 1);

  // 获取PC1的系数（第一列）
  var ndviCoef = eigenVectors.get([0, 0]);    // 绿度
  var wetCoef = eigenVectors.get([1, 0]);     // 湿度
  var lstCoef = eigenVectors.get([2, 0]);     // 热度
  var ndbsiCoef = eigenVectors.get([3, 0]);   // 干度

  // 创建调整矩阵 - 同样按照绿度、湿度、热度、干度的顺序
  var adjustMatrix = ee.Array([
    [ee.Number(ndviCoef).lt(0).multiply(2).subtract(1), 0, 0, 0],   // 绿度系数调整
    [0, ee.Number(wetCoef).lt(0).multiply(2).subtract(1), 0, 0],    // 湿度系数调整
    [0, 0, ee.Number(lstCoef).gt(0).multiply(2).subtract(1), 0],    // 热度系数调整
    [0, 0, 0, ee.Number(ndbsiCoef).gt(0).multiply(2).subtract(1)]   // 干度系数调整
  ]);

  // 调整特征向量
  var adjustedEigenVectors = eigenVectors.matrixMultiply(adjustMatrix);
  // 将调整后的特征向量转换为图像格式
  var eigenImage = ee.Image(adjustedEigenVectors);
  // 计算主成分
  var arrayImage = arrays.toArray(1);
  var principalComponents = eigenImage.matrixMultiply(arrayImage);
  return principalComponents
    .arrayProject([0])
    .arrayFlatten([['PC1', 'PC2', 'PC3', 'PC4']]);
}
// 计算主成分（使用标准化后的数据）
var principalComponents = calculatePCA(compositeImage);
// 计算 RSEI₀（直接使用PC1，因为已经调整了方向）
var rsei0 = principalComponents.select('PC1');
// 标准化到0-1范围
var rsei = rsei0.unitScale(
  rsei0.reduceRegion({
    reducer: ee.Reducer.min(),
    geometry: region,
    scale: scale,
    maxPixels: 1e13
  }).values().get(0),
  rsei0.reduceRegion({
    reducer: ee.Reducer.max(),
    geometry: region,
    scale: scale,
    maxPixels: 1e13
  }).values().get(0)
);
// 导出四个原始指标
Export.image.toDrive({
  image: greenness.clip(region),
  description: 'RSEI_greenness_raw',
  folder: 'RSEI_Results',
  scale: scale,
  crs: projection,
  region: region.geometry().bounds(),
  maxPixels: 1e13
});
Export.image.toDrive({
  image: wetness.clip(region),
  description: 'RSEI_wetness_raw',
  folder: 'RSEI_Results',
  scale: scale,
  crs: projection,
  region: region.geometry().bounds(),
  maxPixels: 1e13
});
Export.image.toDrive({
  image: heat.clip(region),
  description: 'RSEI_heat_raw',
  folder: 'RSEI_Results',
  scale: scale,
  crs: projection,
  region: region.geometry().bounds(),
  maxPixels: 1e13
});
Export.image.toDrive({
  image: dryness.clip(region),
  description: 'RSEI_dryness_raw',
  folder: 'RSEI_Results',
  scale: scale,
  crs: projection,
  region: region.geometry().bounds(),
  maxPixels: 1e13
});
// 导出 RSEI 结果（原有代码）
Export.image.toDrive({
  image: rsei.clip(region),
  description: 'RSEI_final',
  folder: 'RSEI_Results',
  scale: scale,
  crs: projection,
  region: region.geometry().bounds(),
  maxPixels: 1e13
});
// 计算湿度指数的实际范围用于显示
wetness.reduceRegion({
  reducer: ee.Reducer.minMax(),
  geometry: region,
  scale: scale,
  maxPixels: 1e13
}).evaluate(function (result) {
  // 使用计算得到的实际范围来显示湿度
  Map.addLayer(wetness.clip(region), {
    min: result.wetness_min,
    max: result.wetness_max,
    palette: ['#FFE4B5', '#0000FF']
  }, 'Wetness');
});
// 其他图层显示保持不变
Map.addLayer(greenness.clip(region), {
  min: -1,
  max: 1,
  palette: ['white', 'green']
}, 'Greenness (NDVI)');
Map.addLayer(heat.clip(region), {
  min: 0,
  max: 50,
  palette: ['blue', 'yellow', 'red']
}, 'Heat (LST °C)');
Map.addLayer(dryness.clip(region), {
  min: -1,
  max: 1,
  palette: ['#006400', '#8B4513']
}, 'Dryness (NDBSI)');
// 最后显示RSEI（原有代码）
Map.addLayer(rsei.clip(region), {
  min: 0,
  max: 1,
  palette: ['red', 'yellow', 'green']
}, 'RSEI');
// 设置地图视图范围
Map.centerObject(region, 9);
// 结束
```
### 仓库地址：https://github.com/han949/RSEI
###### 本文借鉴了RaySpaniare的博客，欢迎大家进一步完善。



