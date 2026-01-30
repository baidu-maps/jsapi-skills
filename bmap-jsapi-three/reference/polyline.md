# Polyline 折线组件使用指南

## 概述

Polyline 是 Mapv-three 中用于创建折线的核心组件。它支持两种渲染模式：
- **普通折线模式**（屏幕空间线）：线条宽度在屏幕上保持恒定，适合道路、路径等
- **贴地折线模式**（XY平面拉伸）：线条宽度在世界空间中固定，适合地面标线、管线等

支持实线、虚线、纹理、动画等丰富的视觉效果，是地图可视化中最常用的组件之一。

## 基础使用

### 创建普通折线（屏幕空间）

```javascript
// 创建屏幕空间折线，线宽在屏幕上恒定
const polyline = engine.add(new mapvthree.Polyline({
    flat: false,           // false 表示屏幕空间模式
    color: '#ff0000',      // 红色
    lineWidth: 4,          // 线宽（像素）
    opacity: 1
}));

// 设置数据源
const data = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'LineString',
        coordinates: [
            [116.404, 39.915],
            [116.405, 39.920],
            [116.406, 39.918]
        ]
    }
}]);
polyline.dataSource = data;
```

### 创建贴地折线（世界空间）

```javascript
// 创建贴地折线，线宽在世界空间中固定
const fatLine = engine.add(new mapvthree.Polyline({
    flat: true,            // true 表示贴地模式
    color: '#00ff00',      // 绿色
    lineWidth: 5,          // 线宽（米）
    opacity: 0.8
}));

fatLine.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'LineString',
        coordinates: [
            [116.404, 39.915],
            [116.405, 39.920]
        ]
    }
}]);
```

## 两种模式对比

| 特性 | 普通折线 (flat: false) | 贴地折线 (flat: true) |
|------|----------------------|---------------------|
| 渲染方式 | 屏幕空间线条 | XY平面拉伸 |
| 线宽单位 | 像素 | 米（世界坐标） |
| 缩放行为 | 线宽不随缩放改变 | 线宽随缩放改变 |
| 适用场景 | 道路、轨迹、边界线 | 地面标线、管线、河流 |
| 性能 | 较好 | 一般 |
| 3D效果 | 无厚度感 | 有实际宽度 |

## 样式配置

### 颜色和透明度

```javascript
const polyline = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#ff0000',      // 线条颜色
    opacity: 0.8,          // 透明度（0-1）
    alphaTest: 0.1         // 透明度测试阈值，低于此值的像素被剔除
}));
```

### 线宽设置

```javascript
// 普通折线：线宽单位为像素
const screenLine = engine.add(new mapvthree.Polyline({
    flat: false,
    lineWidth: 3           // 3像素宽
}));

// 贴地折线：线宽单位为米
const worldLine = engine.add(new mapvthree.Polyline({
    flat: true,
    lineWidth: 10          // 10米宽
}));
```

### 虚线样式

Polyline 支持灵活的虚线配置：

```javascript
const dashedLine = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#ffff00',
    lineWidth: 4,
    dashed: true,          // 启用虚线
    dashArray: 20,         // 虚线总长度（实线+空隙）
    dashRatio: 0.5,        // 实线占比（0.5表示一半实线，一半空隙）
    dashOffset: 0          // 虚线起始偏移
}));
```

**虚线参数说明：**
- `dashArray`: 虚线一个周期的总长度
- `dashRatio`: 实线部分占周期的比例（0-1）
- `dashOffset`: 虚线起点偏移，可用于创建动画效果

### 顶点颜色

为线条的不同部分设置不同颜色：

```javascript
const colorfulLine = engine.add(new mapvthree.Polyline({
    flat: false,
    vertexColors: true,    // 启用顶点颜色
    lineWidth: 5
}));

// 数据中携带颜色信息
const data = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    properties: {
        color: '#ff0000'   // 这段线为红色
    },
    geometry: {
        type: 'LineString',
        coordinates: [[116.404, 39.915], [116.405, 39.920]]
    }
}]);
colorfulLine.dataSource = data;
```

### 纹理贴图

为折线添加纹理效果：

```javascript
const texturedLine = engine.add(new mapvthree.Polyline({
    flat: false,
    lineWidth: 6,
    map: 'path/to/texture.png',  // 纹理路径
    opacity: 0.9
}));
```

### 高度设置

为折线设置高度，创建立体效果：

```javascript
const elevatedLine = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#00ffff',
    lineWidth: 4,
    height: 100            // 线条高度100米
}));
```

### 自发光效果

```javascript
const glowingLine = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#00ff00',
    emissive: '#00ff00',   // 自发光颜色，与主色相同会更亮
    lineWidth: 5,
    opacity: 0.9
}));
```

## 贝塞尔曲线

Polyline 支持自动将折线转换为平滑的贝塞尔曲线：

```javascript
const curveLine = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#ff00ff',
    lineWidth: 4,
    isCurve: true          // 自动生成贝塞尔曲线
}));

// 即使只提供几个关键点，也会生成平滑曲线
curveLine.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'LineString',
        coordinates: [
            [116.404, 39.915],
            [116.406, 39.920],
            [116.408, 39.916]
        ]
    }
}]);
```

## 实际应用场景

### 场景1: 道路网络

创建道路网络，使用屏幕空间折线：

```javascript
// 主干道
const mainRoad = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#ffaa00',
    lineWidth: 6,
    opacity: 0.9
}));

// 次干道
const secondaryRoad = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#ffdd00',
    lineWidth: 4,
    opacity: 0.8
}));

// 设置道路数据
mainRoad.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON(mainRoadData);
secondaryRoad.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON(secondaryRoadData);
```

### 场景2: 轨迹回放

创建带虚线效果的运动轨迹：

```javascript
const trajectory = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#00ffff',
    lineWidth: 3,
    dashed: true,
    dashArray: 15,
    dashRatio: 0.6,
    height: 50             // 轨迹离地50米
}));

// 设置轨迹数据
trajectory.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'LineString',
        coordinates: trajectoryPoints  // GPS轨迹点
    }
}]);
```

### 场景3: 地面标线

使用贴地模式创建地面标线：

```javascript
const groundMarking = engine.add(new mapvthree.Polyline({
    flat: true,            // 贴地模式
    color: '#ffffff',
    lineWidth: 2,          // 2米宽
    opacity: 0.9,
    dashed: true,
    dashArray: 5,
    dashRatio: 0.5         // 虚线标线
}));

groundMarking.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'LineString',
        coordinates: markingCoordinates
    }
}]);
```

### 场景4: 管线网络

创建地下管线网络可视化：

```javascript
// 供水管线
const waterPipe = engine.add(new mapvthree.Polyline({
    flat: true,
    color: '#0066ff',
    lineWidth: 3,
    height: -2,            // 地下2米
    opacity: 0.7
}));

// 燃气管线
const gasPipe = engine.add(new mapvthree.Polyline({
    flat: true,
    color: '#ffaa00',
    lineWidth: 2,
    height: -3,            // 地下3米
    opacity: 0.7
}));

waterPipe.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON(waterPipeData);
gasPipe.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON(gasPipeData);
```

### 场景5: 河流水系

使用贝塞尔曲线创建平滑的河流：

```javascript
const river = engine.add(new mapvthree.Polyline({
    flat: true,
    color: '#4488ff',
    lineWidth: 50,         // 50米宽的河流
    opacity: 0.6,
    isCurve: true,         // 生成平滑曲线
    height: -1             // 略低于地面
}));

// 只需提供关键控制点，系统会自动生成平滑曲线
river.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'LineString',
        coordinates: [
            [116.404, 39.915],
            [116.410, 39.920],
            [116.415, 39.918],
            [116.420, 39.922]
        ]
    }
}]);
```

### 场景6: 边界线

创建带自发光效果的区域边界：

```javascript
const boundary = engine.add(new mapvthree.Polyline({
    flat: false,
    color: '#ff0000',
    emissive: '#ff0000',
    lineWidth: 5,
    dashed: true,
    dashArray: 20,
    dashRatio: 0.7,
    height: 20
}));

boundary.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'LineString',
        coordinates: boundaryCoordinates
    }
}]);
```

### 场景7: 多彩路径

使用顶点颜色创建彩虹路径：

```javascript
const rainbowPath = engine.add(new mapvthree.Polyline({
    flat: false,
    vertexColors: true,
    lineWidth: 6
}));

// 多段不同颜色的线段
const colorSegments = [
    { color: '#ff0000', coords: [[116.404, 39.915], [116.405, 39.916]] },
    { color: '#ff7700', coords: [[116.405, 39.916], [116.406, 39.917]] },
    { color: '#ffff00', coords: [[116.406, 39.917], [116.407, 39.918]] },
    { color: '#00ff00', coords: [[116.407, 39.918], [116.408, 39.919]] },
    { color: '#0000ff', coords: [[116.408, 39.919], [116.409, 39.920]] }
];

const features = colorSegments.map(seg => ({
    type: 'Feature',
    properties: { color: seg.color },
    geometry: {
        type: 'LineString',
        coordinates: seg.coords
    }
}));

rainbowPath.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON(features);
```

## 构造参数

### 通用参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `flat` | boolean | `false` | 渲染模式：false=屏幕空间，true=贴地 |
| `color` | string | `'#00ffff'` | 线条颜色 |
| `lineWidth` | number | `4` | 线宽（屏幕空间=像素，贴地=米） |
| `height` | number | `0` | 线条高度（米） |
| `opacity` | number | `1` | 透明度 |
| `alphaTest` | number | `0` | 透明度测试阈值 |
| `vertexColors` | boolean | `false` | 使用顶点颜色 |
| `emissive` | string | - | 自发光颜色 |
| `map` | string | - | 纹理贴图路径 |
| `isCurve` | boolean | `false` | 自动生成贝塞尔曲线 |

### 虚线参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `dashed` | boolean | `false` | 启用虚线 |
| `dashArray` | number | `20` | 虚线周期长度 |
| `dashOffset` | number | `0` | 虚线起始偏移 |
| `dashRatio` | number | `0.5` | 实线占比（0-1） |

### 贴地模式特有参数（flat=true）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `keepSize` | boolean | `true` | 保持像素宽度 |
| `vertexWidths` | boolean | `false` | 使用顶点宽度 |
| `mapSrc` | string | - | 纹理贴图路径 |
| `mapGap` | number | `50` | 纹理间隔倍数 |
| `lineJoin` | string | `'bevel'` | 拐角样式：'bevel'/'round'/'miter' |
| `lineCap` | string | `'butt'` | 端点样式：'butt'/'round'/'square' |
| `miterLimit` | number | - | 拐角长度限制 |
| `raycastBuffer` | number | `0` | 拾取缓冲区 |

### 贴地模式动画参数（flat=true）

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enableAnimation` | boolean | `false` | 启用动画 |
| `animationSpeed` | number | `1` | 动画速度 |
| `animationTailType` | number | `1` | 尾迹类型：1=比例，2=固定长度 |
| `animationTailRatio` | number | `0.2` | 尾迹长度比例 |
| `animationTailLength` | number | `100` | 尾迹固定长度 |
| `animationIdle` | number | `1000` | 动画空闲时间（ms） |
| `enableAnimationChaos` | boolean | `false` | 不规则动画效果 |

## 数据属性映射

通过 `dataSource.defineAttribute()` 映射：

| 属性 | 说明 | 条件 |
|------|------|------|
| `color` | 线段颜色 | 需 `vertexColors: true` |
| `lineWidth` | 线段宽度 | 需 `vertexWidths: true`（贴地模式） |

```javascript
data.defineAttribute('color', 'strokeColor');
```

## 最佳实践

1. **模式选择**：
   - 大量线条用 `flat: false`（性能更好）
   - 需要世界空间宽度或内置动画用 `flat: true`

2. **性能优化**：
   - 相同样式的线合并为 MultiLineString
   - 虚线比实线消耗更多资源

3. **线宽建议**：
   - 屏幕空间：2-6 像素
   - 贴地模式：1-20 米

4. **数据组织**：
   ```javascript
   // 推荐：MultiLineString 合并相同样式
   {
       type: 'Feature',
       geometry: {
           type: 'MultiLineString',
           coordinates: [
               [[116.404, 39.915], [116.405, 39.920]],
               [[116.406, 39.918], [116.407, 39.922]]
           ]
       }
   }
   ```

## 常见问题

**Q: 折线没有显示？**
- 检查 dataSource 是否正确设置
- 检查坐标是否在当前视野范围内
- 检查 lineWidth 是否设置过小
- 检查 opacity 是否为 0

**Q: 屏幕空间线和贴地线如何选择？**
- 道路、轨迹、UI线条 → 使用 `flat: false`（屏幕空间）
- 地面标线、管线、河流 → 使用 `flat: true`（贴地模式）

**Q: 虚线不连续或显示异常？**
- 调整 `dashArray` 和 `dashRatio` 参数
- 确保 `dashed: true` 已设置
- 检查线条是否太短，短线段虚线效果不明显

**Q: 贝塞尔曲线不够平滑？**
- 增加控制点数量
- 检查控制点分布是否均匀
- 确保 `isCurve: true` 已设置

**Q: 线条颜色不正确？**
- 如果使用 `vertexColors: true`，颜色从数据的 properties.color 读取
- 如果使用纹理贴图，color 会与纹理混合
- emissive 自发光会叠加在 color 之上

**Q: 贴地线宽度看起来不对？**
- 贴地模式的 lineWidth 单位是米（世界坐标）
- 缩放地图时，线宽会随之变化
- 如需固定视觉宽度，使用屏幕空间模式（`flat: false`）

## 动画效果

### 虚线流动动画（屏幕空间）

`dashOffset` 支持直接设置，可实现流动效果：

```javascript
const line = engine.add(new mapvthree.Polyline({
    flat: false,
    dashed: true,
    dashArray: 20,
    dashRatio: 0.5
}));

// dashOffset 可直接修改
let offset = 0;
function animate() {
    offset += 0.5;
    line.dashOffset = offset % 20;
    requestAnimationFrame(animate);
}
animate();
```

### 内置动画（贴地模式）

贴地模式支持更丰富的内置动画：

```javascript
const animatedLine = engine.add(new mapvthree.Polyline({
    flat: true,
    color: '#00ffff',
    lineWidth: 10,
    enableAnimation: true,
    animationSpeed: 2,
    animationTailType: 1,
    animationTailRatio: 0.3,
    animationIdle: 500
}));
```

## 资源清理

```javascript
engine.remove(polyline);
polyline.dispose();
```

## 相关组件

- **Wall** - 墙体组件，适合创建垂直围栏、边界墙
- **Polygon** - 多边形组件，适合创建面状区域
- **Path** - 路径动画组件，支持对象沿路径运动
- **Editor** - 地图编辑器，可用于交互式绘制折线
