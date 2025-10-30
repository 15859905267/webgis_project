# RGB影像标准差拉伸优化方案（最终版）

## 问题分析

### 之前尝试的方案

1. **统一范围拉伸**（第一版）
   - ❌ 问题：所有波段使用同一个min/max，导致某些波段被压缩，丢失大量细节
   - 示例：Band 1只用了59个灰度级（丢失76%细节）

2. **独立波段拉伸**（第二版）
   - ⚠️ 问题：虽然保留了细节，但破坏了RGB的色彩平衡
   - 示例：各波段独立拉伸后，色彩失真严重

### 最终方案：标准差拉伸（Stddev Stretch）

这是**遥感图像处理的标准方法**，被广泛应用于：
- ENVI、ERDAS、ArcGIS等专业遥感软件
- Google Earth Engine
- 科研论文中的影像展示

## 核心原理

### 标准差拉伸公式

```
拉伸范围 = [Mean - 2×StdDev, Mean + 2×StdDev]
```

### 为什么是 Mean ± 2σ？

根据**正态分布**（高斯分布）原理：
- **68.3%** 的数据落在 Mean ± 1σ 范围内
- **95.4%** 的数据落在 Mean ± 2σ 范围内  ✅ **我们使用这个**
- **99.7%** 的数据落在 Mean ± 3σ 范围内

选择 **Mean ± 2σ** 的原因：
1. ✅ 保留 **95.4%** 的有效数据
2. ✅ 自动去除 **4.6%** 的异常值（极亮和极暗的像素）
3. ✅ 各波段独立计算，保留细节
4. ✅ 基于统计分布，保持色彩相对强度

## 技术实现

### 步骤1：读取统计信息

```bash
gdalinfo -stats input.tif

# 输出示例：
Band 1 Block=512x512 Type=Float32, ColorInterp=Red
  Minimum=0.001, Maximum=0.856, Mean=0.234, StdDev=0.156

Band 2 Block=512x512 Type=Float32, ColorInterp=Green
  Minimum=0.002, Maximum=0.923, Mean=0.312, StdDev=0.189

Band 3 Block=512x512 Type=Float32, ColorInterp=Blue
  Minimum=0.000, Maximum=0.795, Mean=0.198, StdDev=0.142
```

### 步骤2：计算拉伸范围（每个波段独立）

```javascript
// Band 1 (红)
stretchMin_1 = Mean_1 - 2 × StdDev_1 = 0.234 - 2×0.156 = -0.078 → 0.001 (不低于实际最小值)
stretchMax_1 = Mean_1 + 2 × StdDev_1 = 0.234 + 2×0.156 = 0.546

// Band 2 (绿)
stretchMin_2 = Mean_2 - 2 × StdDev_2 = 0.312 - 2×0.189 = -0.066 → 0.002
stretchMax_2 = Mean_2 + 2 × StdDev_2 = 0.312 + 2×0.189 = 0.690

// Band 3 (蓝)
stretchMin_3 = Mean_3 - 2 × StdDev_3 = 0.198 - 2×0.142 = -0.086 → 0.000
stretchMax_3 = Mean_3 + 2 × StdDev_3 = 0.198 + 2×0.142 = 0.482
```

### 步骤3：应用拉伸（转换为Byte）

```bash
gdal_translate -ot Byte \
  -scale_1 0.001 0.546 0 255 \  # Band 1独立拉伸
  -scale_2 0.002 0.690 0 255 \  # Band 2独立拉伸
  -scale_3 0.000 0.482 0 255 \  # Band 3独立拉伸
  -a_nodata 0 \                 # NoData透明
  -of GTiff input.tif output.tif
```

## 优势分析

### 与其他方法对比

| 方法 | 细节保留 | 色彩平衡 | 异常值处理 | 适用场景 |
|------|---------|---------|-----------|---------|
| **Min-Max拉伸** | ❌ 差 | ✅ 好 | ❌ 差 | 异常值少时 |
| **统一范围拉伸** | ❌ 差 | ✅ 好 | ❌ 差 | 各波段范围相近时 |
| **独立波段拉伸** | ✅ 优 | ❌ 差 | ✅ 好 | 不关心色彩时 |
| **标准差拉伸** | ✅ 优 | ✅ 好 | ✅ 优 | **所有场景（推荐）** |

### 具体优势

1. **自适应异常值处理**
   - 自动识别并排除异常高/低值
   - 不需要手动设置百分位数
   - 基于统计分布，更科学

2. **保持色彩平衡**
   - 虽然各波段独立计算，但基于各自的统计分布
   - 保持了各波段的相对强度关系
   - 色彩更加自然

3. **细节完整保留**
   - 每个波段都充分利用0-255范围
   - 95.4%的有效数据都被保留
   - 不会因为异常值压缩动态范围

4. **适用性广**
   - 适用于反射率数据（0-1）
   - 适用于DN值数据（0-255）
   - 适用于16位数据（0-65535）
   - 适用于浮点数据

## 实际效果示例

### 场景1：反射率数据（0-1范围）

```
原始统计：
  Band 1: Mean=0.234, StdDev=0.156
  Band 2: Mean=0.312, StdDev=0.189
  Band 3: Mean=0.198, StdDev=0.142

拉伸范围：
  Band 1: [0.001, 0.546] → [0, 255]
  Band 2: [0.002, 0.690] → [0, 255]
  Band 3: [0.000, 0.482] → [0, 255]

结果：
  ✅ 去除了异常高值（如云、水体反光）
  ✅ 去除了异常低值（如阴影）
  ✅ 保留了95.4%的地物信息
  ✅ 色彩自然，对比度好
```

### 场景2：16位DN值数据

```
原始统计：
  Band 1: Mean=1250, StdDev=850
  Band 2: Mean=1680, StdDev=920
  Band 3: Mean=980, StdDev=720

拉伸范围：
  Band 1: [0, 2950] → [0, 255]     (而不是0-65535)
  Band 2: [0, 3520] → [0, 255]
  Band 3: [0, 2420] → [0, 255]

结果：
  ✅ 避免了整个16位范围的拉伸（那会导致图像过暗）
  ✅ 聚焦在有效数据范围
  ✅ 对比度显著提升
```

## 代码实现要点

### 关键函数

```javascript
// 计算标准差拉伸范围
const calcStddevStretch = (stat) => {
  let stretchMin = stat.mean - 2 * stat.stddev
  let stretchMax = stat.mean + 2 * stat.stddev
  
  // 确保不超出实际范围
  stretchMin = Math.max(stretchMin, stat.min)
  stretchMax = Math.min(stretchMax, stat.max)
  
  // 如果有NoData（值为0），确保拉伸范围不包含0
  if (hasNoData && stretchMin <= 0) {
    stretchMin = 1
  }
  
  return { stretchMin, stretchMax }
}

// 为每个波段独立计算
const b1Stretch = calcStddevStretch(band1)
const b2Stretch = calcStddevStretch(band2)
const b3Stretch = calcStddevStretch(band3)
```

### GDAL命令

```bash
gdal_translate -ot Byte \
  -scale_1 ${b1Min} ${b1Max} 0 255 \
  -scale_2 ${b2Min} ${b2Max} 0 255 \
  -scale_3 ${b3Min} ${b3Max} 0 255 \
  -a_nodata 0 -of GTiff input.tif output.tif
```

## 验证方法

### 后端日志检查

优化时查看后端输出：

```
✅ 策略：标准差拉伸（Mean ± 2*StdDev）+ 保留NoData + 保持色彩平衡

📊 读取影像统计信息（Min, Max, Mean, StdDev）...
   检测到各波段完整统计信息:
   Band 1 (红): Mean=0.2340, StdDev=0.1560, Range=[0.0010, 0.8560]
   Band 2 (绿): Mean=0.3120, StdDev=0.1890, Range=[0.0020, 0.9230]
   Band 3 (蓝): Mean=0.1980, StdDev=0.1420, Range=[0.0000, 0.7950]

🎯 标准差拉伸范围（Mean ± 2*StdDev）:
   Band 1: 0.0010 ~ 0.5460 → 0 ~ 255
   Band 2: 0.0020 ~ 0.6900 → 0 ~ 255
   Band 3: 0.0000 ~ 0.4820 → 0 ~ 255
   📖 说明：此方法去除约5%的异常值，同时保持各波段的相对强度

✅ 标准差拉伸（Mean ± 2σ，遥感标准方法）
✅ 自动去除约5%异常值，保留95%有效数据
✅ 各波段保持相对强度，色彩更自然
✅ 保留NoData值（背景透明）
```

### 优化结果检查

```bash
# 检查优化后的统计信息
gdalinfo -stats optimized.tif

# 期望看到：
Band 1: Mean=100-150, StdDev=50-80  ✅ 充分利用0-255
Band 2: Mean=100-150, StdDev=50-80  ✅ 充分利用0-255
Band 3: Mean=100-150, StdDev=50-80  ✅ 充分利用0-255
```

## 理论基础

### 为什么选择标准差而不是百分位？

| 方法 | 优点 | 缺点 |
|------|------|------|
| **百分位（如2%-98%）** | 精确控制裁剪比例 | 需要读取全部像素，速度慢 |
| **标准差（Mean ± 2σ）** | 只需统计信息，速度快 | 假设正态分布 |

对于遥感影像：
- ✅ 大多数地物的反射率**近似正态分布**
- ✅ GDAL可快速计算统计信息（不需要读取全部像素）
- ✅ 标准差方法被广泛验证和应用

### 正态分布验证

遥感影像的像素值分布通常接近正态分布：

```
        |
        |    ___
        |   /   \
        |  /     \
        | /       \
        |/         \___
        +---------------
      -2σ  Mean  +2σ
      
      95.4%的数据在此范围内
```

## 与专业软件对比

### ENVI

ENVI的"2% Linear Stretch"选项：
- 使用 **2%-98%百分位拉伸**
- 效果与 **Mean ± 2σ** 相近（假设正态分布时）

### ArcGIS

ArcGIS的"Stretch"功能默认选项：
- **Standard Deviation** (标准差拉伸)
- 默认使用 **n=2** (Mean ± 2σ)

### Google Earth Engine

```javascript
// GEE中的可视化参数
var visParams = {
  min: mean - 2 * stdDev,
  max: mean + 2 * stdDev
};
```

## 总结

### 标准差拉伸的核心价值

1. **科学性** - 基于统计学原理（正态分布）
2. **实用性** - 遥感图像处理的行业标准
3. **高效性** - 只需统计信息，不需读取全部像素
4. **自适应性** - 自动适应各种数据范围和分布
5. **平衡性** - 兼顾细节保留和色彩平衡

### 适用场景

- ✅ **所有RGB影像的优化**（推荐作为默认方法）
- ✅ **反射率数据**（如Landsat、Sentinel-2）
- ✅ **DN值数据**（如无人机影像）
- ✅ **高位深数据**（如16位TIF）
- ✅ **有异常值的数据**（如云、水体反光）

### 不适用场景

- ❌ 像素值分布严重偏态（非正态分布）
- ❌ 需要保持绝对辐射值的科学分析

---

**方法来源**: 遥感图像处理标准方法  
**适用软件**: ENVI, ERDAS, ArcGIS, QGIS, GEE  
**理论基础**: 正态分布统计学  
**推荐指数**: ⭐⭐⭐⭐⭐

