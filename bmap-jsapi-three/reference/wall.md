# Wall 墙体组件使用指南

## 概述

Wall 是 Mapv-three 中用于创建立体墙体效果的组件。它可以沿着线路创建无厚度的垂直墙面，广泛应用于围栏、边界线、区域分隔等场景。支持丰富的视觉效果，包括颜色、纹理、透明度和多种动画效果。

## 基础使用

### 创建简单墙体

```javascript
// 创建一个基础墙体
const wall = engine.add(new mapvthree.Wall({
    height: 200,           // 墙体高度（米）
    color: '#00ffff',      // 墙体颜色
    opacity: 0.8           // 透明度
}));

// 设置数据源（使用 GeoJSON LineString）
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
wall.dataSource = data;
```

## 样式配置

### 颜色和透明度

```javascript
const wall = engine.add(new mapvthree.Wall({
    height: 150,
    color: '#ff0000',      // 红色墙体
    opacity: 0.6,          // 整体透明度
    minOpacity: 0.2,       // 最低透明度（用于渐变效果）
    maxOpacity: 0.9        // 最高透明度
}));
```

### 使用纹理贴图

```javascript
const wall = engine.add(new mapvthree.Wall({
    height: 200,
    mapSrc: 'assets/textures/fence.png',  // 推荐使用 mapSrc
    mapScale: 1,
    opacity: 0.8,
    transparent: true,
    depthWrite: false
}));
```

### 顶点颜色

通过数据源为不同路段设置不同颜色：

```javascript
const wall = engine.add(new mapvthree.Wall({
    height: 200,
    vertexColors: true     // 启用顶点颜色
}));

// 在数据中携带颜色信息
const data = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    properties: {
        color: '#ff0000'   // 这段墙体为红色
    },
    geometry: {
        type: 'LineString',
        coordinates: [[116.404, 39.915], [116.405, 39.920]]
    }
}]);
wall.dataSource = data;
```

## 动画效果

Wall 组件支持 4 种动画类型，可创建动态视觉效果。

### 1. 垂直方向动画（推荐）

最常用的动画效果，适合大多数场景：

```javascript
const wall = engine.add(new mapvthree.Wall({
    height: 200,
    color: '#00ffff',
    enableAnimation: true,      // 启用动画
    animationTailType: 3,       // 垂直方向动画
    animationSpeed: 1,          // 动画速度（倍数）
    animationIdle: 1000         // 动画间隔时间（毫秒）
}));
```

### 2. 条纹上升动画

创建多组条纹向上移动的效果：

```javascript
const wall = engine.add(new mapvthree.Wall({
    height: 200,
    color: '#00ffff',
    enableAnimation: true,
    animationTailType: 4,       // 条纹上升动画
    animationSpeed: 2,
    animationRatio: 0.7,        // 实线占比（0-1），0.7表示70%实线，30%虚线
    animationBales: 5           // 显示的条纹组数
}));
```

### 3. 按墙长度比例动画

动画长度根据墙体总长度的比例计算：

```javascript
const wall = engine.add(new mapvthree.Wall({
    height: 200,
    color: '#00ffff',
    enableAnimation: true,
    animationTailType: 1,       // 按比例动画
    animationSpeed: 1.5,
    animationTailRatio: 0.3     // 动画尾迹占墙体长度的30%
}));
```

### 4. 按固定长度动画

动画长度为固定值，不随墙体长度变化：

```javascript
const wall = engine.add(new mapvthree.Wall({
    height: 200,
    color: '#00ffff',
    enableAnimation: true,
    animationTailType: 2,       // 按固定长度动画
    animationSpeed: 1,
    animationTailLength: 100    // 动画尾迹长度100米
}));
```

## 实际应用场景

### 场景1: 区域围栏

```javascript
const fence = engine.add(new mapvthree.Wall({
    height: 150,
    mapSrc: 'assets/textures/fence.png',
    opacity: 0.8,
    transparent: true,
    depthWrite: false,
    enableAnimation: true,
    animationTailType: 3,
    animationSpeed: 1
}));

fence.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'LineString',
        coordinates: [
            [116.404, 39.915],
            [116.405, 39.915],
            [116.405, 39.916],
            [116.404, 39.916],
            [116.404, 39.915]  // 闭合
        ]
    }
}]);
```

### 场景2: 条纹上升动画

```javascript
const warningZone = engine.add(new mapvthree.Wall({
    height: 200,
    color: '#ff0000',
    opacity: 0.7,
    transparent: true,
    enableAnimation: true,
    animationTailType: 4,       // 条纹上升
    animationSpeed: 2,
    animationRatio: 0.5,
    animationBales: 8
}));
```

### 场景4: 多段不同颜色的墙体

使用顶点颜色创建多彩墙体：

```javascript
const colorfulWall = engine.add(new mapvthree.Wall({
    height: 180,
    vertexColors: true,
    enableAnimation: true,
    animationTailType: 3,
    animationSpeed: 1
}));

// 多段数据，每段不同颜色
const multiColorData = mapvthree.GeoJSONDataSource.fromGeoJSON([
    {
        type: 'Feature',
        properties: { color: '#ff0000' },  // 红色段
        geometry: {
            type: 'LineString',
            coordinates: [[116.404, 39.915], [116.405, 39.915]]
        }
    },
    {
        type: 'Feature',
        properties: { color: '#00ff00' },  // 绿色段
        geometry: {
            type: 'LineString',
            coordinates: [[116.405, 39.915], [116.406, 39.915]]
        }
    },
    {
        type: 'Feature',
        properties: { color: '#0000ff' },  // 蓝色段
        geometry: {
            type: 'LineString',
            coordinates: [[116.406, 39.915], [116.407, 39.915]]
        }
    }
]);
colorfulWall.dataSource = multiColorData;
```

## 构造参数

### 基础参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `height` | number | `100` | 墙体高度（米） |
| `color` | string | `'#00ffff'` | 墙体颜色 |
| `emissive` | string/Array | `[0,0,0]` | 自发光颜色 |
| `vertexColors` | boolean | `false` | 使用顶点颜色 |
| `vertexHeights` | boolean | `false` | 使用顶点高度 |

### 纹理参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `mapSrc` | string | - | 纹理贴图路径（推荐） |
| `map` | string | - | 纹理贴图路径 |
| `mapScale` | number/Array | `1` | 纹理缩放系数 |

### 透明度参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `opacity` | number | `1` | 整体透明度 |
| `minOpacity` | number | `0` | 最低透明度（渐变效果） |
| `maxOpacity` | number | `1` | 最高透明度 |
| `transparent` | boolean | - | 启用透明度 |
| `depthWrite` | boolean | - | 深度写入 |

### 动画参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enableAnimation` | boolean | `false` | 启用动画 |
| `animationSpeed` | number | `1` | 动画速度倍数 |
| `animationTailType` | number | `3` | 动画类型（见下方说明） |
| `animationTailRatio` | number | `0.2` | 比例动画的长度比例（type=1） |
| `animationTailLength` | number | `100` | 固定长度动画的长度（type=2） |
| `animationIdle` | number | `1000` | 动画间隔时间（毫秒） |
| `animationRatio` | number | `0.5` | 条纹实线占比（type=4） |
| `animationBales` | number | `5` | 条纹组数（type=4） |

**animationTailType 可选值：**
| 值 | 说明 | 相关参数 |
|---|------|---------|
| `1` | 按比例拖尾 | `animationTailRatio` |
| `2` | 固定长度拖尾 | `animationTailLength` |
| `3` | 垂直方向动画 | - |
| `4` | 条纹上升动画 | `animationRatio`, `animationBales` |

## 数据属性映射

通过 `dataSource.defineAttribute()` 映射：

| 属性 | 说明 | 条件 |
|------|------|------|
| `color` | 墙体颜色 | 需 `vertexColors: true` |
| `height` | 墙体高度 | 需 `vertexHeights: true` |

```javascript
dataSource.defineAttribute('color', p => Math.random() * 0xffffff);
dataSource.defineAttribute('height', 'wallHeight');
```

## 资源清理

```javascript
engine.remove(wall);
wall.dispose();
```

## 最佳实践

1. **性能优化**：静态场景关闭动画，相同样式合并为 MultiLineString
2. **透明度搭配**：建议 opacity 0.6-0.9，配合 `transparent: true` 和 `depthWrite: false`
3. **动画速度**：animationSpeed 建议 0.5-2
4. **闭合路径**：围栏场景确保首尾坐标相同

## 常见问题

**Q: 墙体没有显示？**
- 检查 dataSource 是否正确设置
- 检查坐标是否在当前地图视野范围内
- 检查 height 是否设置过小
- 检查 opacity 是否设置为 0

**Q: 动画效果不明显？**
- 尝试调整 animationSpeed 增加速度
- 检查 enableAnimation 是否设置为 true
- 对于 type=4 的条纹动画，调整 animationBales 和 animationRatio

**Q: 纹理显示异常？**
- 确认纹理路径正确
- 检查纹理文件是否存在
- 尝试调整 mapScale 参数

**Q: 墙体颜色不正确？**
- 如果使用了 vertexColors=true，颜色会从数据的 properties.color 中读取
- 如果使用了纹理贴图，color 参数会与纹理混合，可能影响最终颜色

## 相关组件

- **Polyline** - 折线组件，适合创建道路、路径等线状要素
- **Polygon** - 多边形组件，适合创建面状区域
- **Editor** - 地图编辑器，可用于交互式绘制墙体路径
