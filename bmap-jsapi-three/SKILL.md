---
name: bmap-jsapi-three
description: 百度地图 JSAPI Three 版 (MapVThree)：基于three.js的Web二三维一体化地图可视化库，支持多源底图加载、三维模型加载、地理数据可视化、自然环境渲染、量测编辑等功能。适用于构建专业的二三维一体化地图、WebGIS、数字孪生等应用。
license: MIT
version: 1.0.0
---

# Mapv-three 开发指南

使用 Mapv-three 构建高性能的 3D 地图和 GIS 应用 - 一个采用 Z-up 坐标系的跨浏览器 WebGL 库。

## 何时适用

在以下场景中参考这些指南：

- 3D 地图编辑和要素绘制
- 地图测量工具开发
- 建筑物、区域等 3D 可视化
- 实时交通数据展示
- 路径追踪动画开发

## 快速参考

### 0. 基础概念

- `reference/common/coordinate-system.md` - 坐标系：Z-up、投影方式
- `reference/common/event-binding.md` - 事件绑定模式

### 1. 初始化

- `reference/initialization.md` - 引擎初始化、资源配置、百度地图适配器

### 2. 数据管理

- `reference/datasource.md` - GeoJSON/JSON/CSV 加载、属性映射、增删改查

### 3. 可视化组件

- `reference/label.md` - 文本/图标/组合标签、碰撞检测、淡入淡出
- `reference/polygon.md` - 2D 填充、3D 拉伸、贴地渲染
- `reference/polyline.md` - 屏幕空间/贴地模式、虚线样式
- `reference/wall.md` - 围栏效果、纹理动画

### 4. 绘制与编辑

- `reference/editor.md` - 多边形/线条/点/圆/矩形绘制与编辑
- `reference/measure.md` - 距离/面积/坐标测量

### 5. 动画与追踪

- `reference/tracker.md` - PathTracker/OrbitTracker/ObjectTracker

### 6. 环境效果

- `reference/sky-weather.md` - DynamicSky/PhysicalSky、云层、天气系统


## 注意事项

- **坐标系**：Z-up（X-东、Y-北、Z-上），与 Three.js 默认 Y-up 不同
- **MeasureType**：使用 `mapvthree.Editor.MeasureType`

## 如何使用

请阅读各个参考文件以获取详细说明和代码示例。每个参考文件包含：功能说明、代码示例、API 参数、注意事项。
