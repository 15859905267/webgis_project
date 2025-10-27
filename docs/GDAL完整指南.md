# GDAL完整配置与故障排除指南

> 本文档整合了GDAL的安装、配置、故障排除和自定义conda环境的所有内容。

---

## 📖 目录

1. [快速开始](#快速开始)
2. [详细安装步骤](#详细安装步骤)
3. [配置项目](#配置项目)
4. [故障排除](#故障排除)
5. [自定义Conda环境](#自定义conda环境)
6. [验证与测试](#验证与测试)

---

## 🚀 快速开始

### 最简3步配置

```bash
# 1. 打开 Anaconda Prompt
# 2. 安装GDAL
conda activate xm  # 或你的环境名
conda install -c conda-forge gdal

# 3. 验证安装
gdalinfo --version
```

然后在 `server/config.js` 中配置你的conda环境名：

```javascript
export default {
  condaEnv: 'xm',  // 改成你的环境名
}
```

---

## 📦 详细安装步骤

### 方法1：在现有环境中安装（推荐）

```bash
# 1. 激活你的环境
conda activate your_env_name

# 2. 安装GDAL
conda install -c conda-forge gdal

# 3. 验证
gdalinfo --version
```

### 方法2：创建新的专用环境

```bash
# 1. 创建新环境
conda create -n gdal_env python=3.9

# 2. 激活环境
conda activate gdal_env

# 3. 安装GDAL
conda install -c conda-forge gdal

# 4. 安装Node.js依赖（如果需要）
conda install nodejs
```

### 方法3：使用系统PATH（不推荐）

如果你希望GDAL在系统PATH中：

```bash
# 1. 安装到base环境
conda activate base
conda install -c conda-forge gdal

# 2. 添加到系统PATH（Windows）
# 添加到环境变量：C:\Users\YourName\anaconda3\Library\bin
```

---

## ⚙️ 配置项目

### 1. 配置Conda环境名

编辑 `server/config.js`：

```javascript
export default {
  // Conda环境名称，如果GDAL安装在conda环境中，请在此处填写环境名称
  // 例如：'base', 'myenv', 'gdal_env'
  // 如果GDAL已添加到系统PATH，可以设置为 null
  condaEnv: 'xm',  // 👈 改成你的环境名
  
  // 其他配置...
}
```

### 2. 在Anaconda Prompt中启动后端

**重要：必须在Anaconda Prompt中启动，而不是普通CMD！**

```bash
# 1. 打开 Anaconda Prompt（不是普通CMD）
# 2. 激活环境
conda activate xm

# 3. 进入项目目录
cd D:\code\前端学习之路\demo\demo07

# 4. 启动后端
cd server
npm run dev
```

如果成功，你会看到：

```
✅ GDAL已安装: GDAL 3.8.0, released 2023/11/16
   使用Conda环境: xm
服务器已启动: http://localhost:8080
```

---

## 🔧 故障排除

### ❌ 问题1：GDAL检测失败

**症状：**
```
❌ GDAL检测失败: Command failed...
服务器未检测到GDAL，请检查配置
```

**原因分析：**

#### 原因1：conda命令不在PATH中

**解决方案：**
- ✅ **在Anaconda Prompt中启动后端**（而不是普通CMD）
- Anaconda Prompt会自动配置conda命令

**验证方法：**
```bash
# 在你的终端中运行
where conda
# 应该显示：C:\Users\...\anaconda3\Scripts\conda.exe

# 如果显示"找不到"，说明你在普通CMD中，需要切换到Anaconda Prompt
```

#### 原因2：环境名称不正确

**解决方案：**
```bash
# 1. 查看所有conda环境
conda env list

# 输出示例：
# base                  *  C:\Users\...\anaconda3
# xm                       C:\Users\...\anaconda3\envs\xm
# myenv                    C:\Users\...\anaconda3\envs\myenv

# 2. 修改 server/config.js 中的 condaEnv 为实际的环境名
```

#### 原因3：GDAL未在该环境中安装

**解决方案：**
```bash
# 1. 激活环境
conda activate xm

# 2. 检查GDAL是否已安装
gdalinfo --version

# 如果显示错误，说明未安装，执行：
conda install -c conda-forge gdal
```

#### 原因4：后端在普通CMD中启动

**解决方案：**
1. 关闭当前CMD窗口
2. 打开 **Anaconda Prompt**
3. 重新启动后端

---

### ❌ 问题2：端口被占用（EADDRINUSE）

**症状：**
```
Error: listen EADDRINUSE: address already in use :::8080
```

**原因：** 8080端口被其他程序占用

**解决方案：**

**方法1：杀死占用进程**
```bash
# Windows
netstat -ano | findstr :8080
taskkill /PID <PID> /F

# 或直接杀死所有Node进程
taskkill /F /IM node.exe
```

**方法2：更改端口**

编辑 `server/app.js`，修改端口号：
```javascript
const PORT = 8081  // 改成其他端口
```

---

### ❌ 问题3：优化超时（timeout exceeded）

**症状：**
```
AxiosError: timeout of 300000ms exceeded
```

**原因：** 大文件（70MB+）优化需要超过5分钟

**解决方案：**

已在最新版本中修复，超时时间已增加到15分钟：
- 前端：`src/api/index.js` → `timeout: 900000`（15分钟）
- 后端：`server/routes/image.js` → `req.setTimeout(15 * 60 * 1000)`

**如果还是超时：**
1. 检查任务管理器，确认 `gdalwarp.exe` 是否在运行
2. 等待更长时间（70MB文件可能需要10-15分钟）
3. 使用手动脚本 `optimize_tif.bat`

---

### ❌ 问题4：文件ID重复

**症状：** 多个文件显示相同的状态

**原因：** 之前的ID生成逻辑有bug

**解决方案：** 已修复（自动找最大ID+1）

如果仍有问题：
```bash
# 手动重置元数据
del public\data\imageData.json
# 重启后端，会自动重新生成
```

---

## 🎯 自定义Conda环境

### 为什么需要配置环境名？

**场景：** 不同开发者的conda环境名称不同

| 开发者 | 环境名 | 配置 |
|--------|--------|------|
| 张三 | `xm` | `condaEnv: 'xm'` |
| 李四 | `base` | `condaEnv: 'base'` |
| 王五 | `gis_env` | `condaEnv: 'gis_env'` |

### 配置步骤

**Step 1：查看你的环境名**
```bash
conda env list
```

**Step 2：修改配置文件**

编辑 `server/config.js`：
```javascript
export default {
  condaEnv: 'your_env_name',  // 改成你的环境名
}
```

**Step 3：重启后端**
```bash
# 在Anaconda Prompt中
conda activate your_env_name
cd server
npm run dev
```

### 团队协作建议

**方案1：使用统一环境名（推荐）**
- 团队约定使用 `gdal_env`
- 每个人都创建这个环境：
  ```bash
  conda create -n gdal_env python=3.9
  conda activate gdal_env
  conda install -c conda-forge gdal
  ```

**方案2：配置文件个性化**
- 将 `server/config.js` 添加到 `.gitignore`
- 每个人使用自己的配置
- 提供 `server/config.example.js` 作为模板

**方案3：使用环境变量**
```javascript
// server/config.js
export default {
  condaEnv: process.env.CONDA_ENV_NAME || 'base',
}
```

```bash
# 每个人在启动前设置
set CONDA_ENV_NAME=xm
npm run dev
```

---

## ✅ 验证与测试

### 1. 检查GDAL安装

```bash
# 激活环境
conda activate xm

# 检查GDAL版本
gdalinfo --version

# 应该显示类似：
# GDAL 3.8.0, released 2023/11/16
```

### 2. 测试GDAL命令

```bash
# 进入数据目录
cd public\data

# 测试gdalinfo（查看文件信息）
gdalinfo 2024_kle_vh_kndvi.tif

# 应该显示文件的详细信息（投影、尺寸等）
```

### 3. 测试后端启动

```bash
# 在Anaconda Prompt中
conda activate xm
cd server
npm run dev

# 应该看到：
# 🔍 开始同步元数据...
# ✅ GDAL已安装: GDAL 3.x.x
# 服务器已启动: http://localhost:8080
```

### 4. 测试优化功能

1. 打开前端：`http://localhost:3000`
2. 进入"影像数据管理"
3. 选择一个未优化的TIF文件
4. 点击"优化TIF"
5. 观察进度条和后端日志

**预期结果：**
- 进度条从 0% → 20% → 70% → 90% → 100%
- 后端显示：
  ```
  🚀 开始优化: xxx.tif
  ⏳ 步骤1/3: 投影转换 + COG格式转换...
  ✅ 投影转换完成
  ⏳ 步骤2/3: 添加金字塔...
  ✅ 金字塔添加完成
  ✅ 优化成功!
  ```

---

## 📚 相关资源

- [GDAL官方文档](https://gdal.org/)
- [Conda官方文档](https://docs.conda.io/)
- [COG格式说明](https://www.cogeo.org/)

---

## 🆘 仍有问题？

### 逐步排查清单

- [ ] 是否在 **Anaconda Prompt** 中启动？
- [ ] 是否激活了正确的conda环境？
- [ ] `conda env list` 中是否有你配置的环境？
- [ ] `gdalinfo --version` 是否能正常显示版本？
- [ ] `server/config.js` 中的 `condaEnv` 是否正确？
- [ ] 后端启动时是否显示"✅ GDAL已安装"？
- [ ] 端口8080是否被占用？

### 获取帮助

如果以上方法都无法解决，请提供：
1. `conda env list` 的输出
2. `gdalinfo --version` 的输出（在conda环境中）
3. 后端启动时的完整日志
4. 操作系统和Anaconda版本

---

> **最后更新：** 2025-10-17  
> **版本：** v3.0 - 整合版

