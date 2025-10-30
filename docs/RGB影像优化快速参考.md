# RGB影像优化快速参考

## 🎯 核心方法

**标准差拉伸（Stddev Stretch）**
```
拉伸范围 = [Mean - 2×StdDev, Mean + 2×StdDev]
```

## ✅ 优势

1. **保留95.4%有效数据**（自动去除5%异常值）
2. **各波段独立计算**（保留细节）
3. **保持相对强度**（色彩平衡）
4. **遥感标准方法**（ENVI、ArcGIS默认行为）

## 📊 后端日志示例

优化RGB影像时查看后端输出，应该看到：

```
✅ 策略：标准差拉伸（Mean ± 2*StdDev）+ 保留NoData + 保持色彩平衡

📊 读取影像统计信息（Min, Max, Mean, StdDev）...
   Band 1 (红): Mean=0.2340, StdDev=0.1560
   Band 2 (绿): Mean=0.3120, StdDev=0.1890
   Band 3 (蓝): Mean=0.1980, StdDev=0.1420

🎯 标准差拉伸范围（Mean ± 2*StdDev）:
   Band 1: 0.0010 ~ 0.5460 → 0 ~ 255
   Band 2: 0.0020 ~ 0.6900 → 0 ~ 255
   Band 3: 0.0000 ~ 0.4820 → 0 ~ 255

✅ 标准差拉伸（Mean ± 2σ，遥感标准方法）
✅ 自动去除约5%异常值，保留95%有效数据
✅ 各波段保持相对强度，色彩更自然
```

## 🔍 验证方法

### 方法1：后端日志
查看优化过程中是否显示"标准差拉伸（Mean ± 2*StdDev）"

### 方法2：GDAL检查
```bash
gdalinfo -stats optimized.tif | grep "Mean="

# 期望输出：
#   Band 1: Mean=100-150  ✅ 充分利用0-255
#   Band 2: Mean=100-150  ✅ 充分利用0-255
#   Band 3: Mean=100-150  ✅ 充分利用0-255
```

### 方法3：视觉检查
- ✅ 影像清晰，细节丰富
- ✅ 色彩自然，无严重失真
- ✅ 对比度适中，不过暗或过亮

## 🆚 与其他方案对比

| 方案 | 细节 | 色彩 | 异常值 | 推荐 |
|------|------|------|--------|------|
| 统一范围拉伸 | ❌ | ✅ | ❌ | ⭐ |
| 独立波段拉伸 | ✅ | ❌ | ✅ | ⭐⭐ |
| **标准差拉伸** | ✅ | ✅ | ✅ | ⭐⭐⭐⭐⭐ |

## 🔬 理论基础

### 正态分布

```
        |
        |    ___
        |   /   \      95.4% 数据在 Mean ± 2σ
        |  /     \
        | /       \
        |/         \___
        +---------------
      -2σ  Mean  +2σ
```

### 为什么是2σ？

- **1σ**: 包含68.3%数据（太窄）
- **2σ**: 包含95.4%数据 ✅ **最佳平衡**
- **3σ**: 包含99.7%数据（包含太多异常值）

## 🌍 专业软件对比

| 软件 | 默认方法 | 参数 |
|------|---------|------|
| **ENVI** | 2% Linear Stretch | ≈ Mean ± 2σ |
| **ArcGIS** | Stddev Stretch | n=2 (默认) |
| **GEE** | Custom Viz | min: μ-2σ, max: μ+2σ |
| **我们** | Stddev Stretch | Mean ± 2σ ✅ |

## 📚 详细文档

1. **完整原理**：[RGB影像标准差拉伸优化方案.md](./RGB影像标准差拉伸优化方案.md)
2. **问题修复历程**：[RGB影像优化信息丢失问题修复.md](./RGB影像优化信息丢失问题修复.md)
3. **测试对比**：[RGB影像优化测试对比.md](./RGB影像优化测试对比.md)

## ⚡ 快速使用

1. 上传RGB影像（Float32类型）
2. 点击"优化"按钮
3. 查看后端日志（确认使用标准差拉伸）
4. 检查优化结果

## 🐛 常见问题

### Q: 为什么优化后颜色和原始影像不一样？

A: 标准差拉伸会去除异常值（如云、阴影），这是正常现象。优化后的影像：
- ✅ 细节更清晰
- ✅ 对比度更好
- ✅ 色彩更自然

### Q: 如何重新优化已处理的文件？

A: 
1. 删除优化后的文件（`*_optimized.tif`）
2. 重新上传原始文件
3. 使用新逻辑重新优化

### Q: 所有RGB影像都使用标准差拉伸吗？

A: 是的，标准差拉伸适用于：
- ✅ 反射率数据（0-1）
- ✅ DN值数据（0-255）
- ✅ 16位数据（0-65535）
- ✅ 浮点数据

## 🎓 技术细节

### GDAL命令

```bash
# 步骤1：获取统计信息
gdalinfo -stats input.tif

# 步骤2：应用标准差拉伸
gdal_translate -ot Byte \
  -scale_1 ${mean1-2σ1} ${mean1+2σ1} 0 255 \
  -scale_2 ${mean2-2σ2} ${mean2+2σ2} 0 255 \
  -scale_3 ${mean3-2σ3} ${mean3+2σ3} 0 255 \
  -a_nodata 0 input.tif scaled.tif

# 步骤3：投影转换 + COG优化
gdalwarp -s_srs EPSG:32645 -t_srs EPSG:3857 \
  -of COG -co COMPRESS=JPEG -co QUALITY=85 \
  scaled.tif output.tif
```

### 代码实现

```javascript
// 计算标准差拉伸范围
const calcStddevStretch = (stat) => {
  let stretchMin = stat.mean - 2 * stat.stddev
  let stretchMax = stat.mean + 2 * stat.stddev
  
  // 确保不超出实际范围
  stretchMin = Math.max(stretchMin, stat.min)
  stretchMax = Math.min(stretchMax, stat.max)
  
  return { stretchMin, stretchMax }
}
```

---

**更新日期**: 2025-10-29  
**当前版本**: 标准差拉伸（Mean ± 2σ）  
**推荐指数**: ⭐⭐⭐⭐⭐

