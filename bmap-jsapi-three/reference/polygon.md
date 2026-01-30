# Polygon 多边形组件使用指南

## 概述

Polygon 用于创建填充多边形，支持 2D 平面渲染、3D 拉伸和贴地渲染。

## 核心特性

- 基础多边形填充
- Z轴拉伸（3D 建筑效果）
- 贴地渲染
- 纹理贴图
- 顶点颜色
- 带洞多边形

## 基础使用

```javascript
const polygon = engine.add(new mapvthree.Polygon({
    color: '#ff0000',
    opacity: 0.8
}));

polygon.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: {
        type: 'Polygon',
        coordinates: [[
            [116.404, 39.915],
            [116.405, 39.915],
            [116.405, 39.916],
            [116.404, 39.916],
            [116.404, 39.915]  // 必须闭合
        ]]
    }
}]);
```

## 3D 拉伸

```javascript
// 基础拉伸
const building = engine.add(new mapvthree.Polygon({
    extrude: true,
    extrudeValue: 100,     // 拉伸高度 100 米
    color: '#4488ff'
}));

// 数据驱动高度
const buildings = engine.add(new mapvthree.Polygon({
    extrude: true,
    vertexColors: true
}));

const data = mapvthree.GeoJSONDataSource.fromGeoJSON(geojson);
data.defineAttributes({
    color: 'color',
    height: 'buildingHeight'
});
buildings.dataSource = data;
```

## 贴地渲染

```javascript
const groundPolygon = engine.add(new mapvthree.Polygon({
    isGroundPrimitive: true,
    color: 'rgba(255, 0, 0, 0.5)'
}));

// 动态贴地多边形
const dynamicGround = engine.add(new mapvthree.Polygon({
    isGroundPrimitive: true,
    isDynamic: true
}));
```

## 样式配置

```javascript
const polygon = engine.add(new mapvthree.Polygon({
    color: '#ff0000',           // 填充颜色
    opacity: 0.8,               // 透明度
    emissive: '#ff4400',        // 自发光颜色
    vertexColors: true,         // 启用顶点颜色
    mapSrc: 'texture.jpg',      // 纹理贴图
    mapScale: 1.0,              // 纹理缩放
    zOffset: 50,                // 高度抬升
}));
```

## 顶点颜色

```javascript
const polygon = engine.add(new mapvthree.Polygon({
    extrude: true,
    vertexColors: true
}));

const data = mapvthree.GeoJSONDataSource.fromGeoJSON([
    { /* ... */, properties: { color: [255, 0, 0] } },
    { /* ... */, properties: { color: [0, 255, 0] } }
]);

data.defineAttribute('color', 'color');
polygon.dataSource = data;
```

## 构造参数

### 基础参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `color` | string | `'#ffffff'` | 填充颜色 |
| `opacity` | number | `1` | 透明度 |
| `emissive` | string | `'#000000'` | 自发光颜色 |
| `vertexColors` | boolean | `false` | 使用顶点颜色（需数据映射） |

### 拉伸参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `extrude` | boolean | `false` | 是否拉伸 |
| `extrudeValue` | number | `100` | 拉伸高度（米） |
| `enableBottomFace` | boolean | `true` | 是否封闭底面 |
| `vertexHeights` | boolean | `false` | 从数据中读取高度 |
| `perPositionHeight` | boolean | `true` | 使用顶点高度 |

### 高度与偏移

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `zOffset` | number | `0` | 高度抬升（米） |
| `normalOffset` | number | `0` | 沿法线方向抬升 |

### 纹理参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `mapSrc` | string | `''` | 纹理贴图路径 |
| `mapScale` | number | `1` | 纹理缩放 |

### 贴地渲染

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `isGroundPrimitive` | boolean | `false` | 贴地渲染 |
| `isDynamic` | boolean | `false` | 支持动态更新 |

### 渲染参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `transparent` | boolean | `false` | 透明效果 |
| `depthWrite` | boolean | `true` | 深度写入 |
| `renderOrder` | number | `0` | 渲染顺序 |
| `useAO` | boolean | `false` | 环境光遮蔽 |
| `concaveIntensity` | number | `0.2` | 凹面强度 |
| `heightIntensity` | number | `0.4` | 高度强度 |

## 数据属性映射

通过 `dataSource.defineAttribute()` 映射数据属性：

| 属性 | 说明 | 示例 |
|------|------|------|
| `color` | 多边形颜色 | `defineAttribute('color', 'fillColor')` |
| `height` | 高度值（需 `vertexHeights: true`） | `defineAttribute('height', 'buildingHeight')` |
| `uvRotation` | UV 旋转角度（弧度） | `defineAttribute('uvRotation', () => Math.PI / 2)` |

```javascript
data.defineAttributes({
    color: (attrs) => {
        switch (attrs.type) {
            case 'residential': return [200, 200, 200];
            case 'commercial': return [100, 150, 255];
            default: return [180, 180, 180];
        }
    },
    height: 'building_height'
});
```

## 几何体属性

通过 `polygon.geometry` 访问：

```javascript
polygon.geometry.useUV = true;           // 启用 UV 坐标
polygon.geometry.sideUVReversed = true;  // 侧面 UV 反向
polygon.geometry.sideUVNormalized = true; // 侧面 UV 归一化
```

## 材质属性

通过 `polygon.material` 访问：

```javascript
polygon.material.heightExtrusion = 100;  // 动态设置高度拉伸值
```

## 带洞多边形

```javascript
const polygonWithHole = {
    type: 'Feature',
    geometry: {
        type: 'Polygon',
        coordinates: [
            // 外环
            [[116.404, 39.915], [116.405, 39.915], [116.405, 39.916], [116.404, 39.916], [116.404, 39.915]],
            // 内环（洞）
            [[116.4042, 39.9152], [116.4048, 39.9152], [116.4048, 39.9158], [116.4042, 39.9158], [116.4042, 39.9152]]
        ]
    }
};
```

## 实际应用

### 城市建筑可视化

```javascript
const buildings = await mapvthree.GeoJSONDataSource.fromURL('buildings.geojson');

buildings.defineAttributes({
    color: (attrs) => {
        switch (attrs.type) {
            case 'residential': return [200, 200, 200];
            case 'commercial': return [100, 150, 255];
            default: return [180, 180, 180];
        }
    },
    height: 'building_height'
});

const layer = engine.add(new mapvthree.Polygon({
    extrude: true,
    vertexColors: true,
    enableBottomFace: false
}));
layer.dataSource = buildings;
```

### 区域高亮

```javascript
const highlight = engine.add(new mapvthree.Polygon({
    isGroundPrimitive: true,
    color: 'rgba(255, 255, 0, 0.6)'
}));
highlight.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON(areaData);
```

## 资源清理

```javascript
// 移除图层
engine.remove(polygon);

// 释放资源
polygon.dispose();
```

## 最佳实践

1. **性能优化**：关闭底面渲染 `enableBottomFace: false`
2. **合并相同样式**：使用单个 Polygon 对象 + FeatureCollection
3. **透明度设置**：区域填充 0.6-0.8，建筑 0.8-1.0
4. **坐标闭合**：多边形首尾点必须相同
5. **AO 效果**：使用 `useAO: true` 增强立体感

## 常见问题

**Q: 多边形没有显示？**
- 检查坐标是否闭合
- 检查 dataSource 是否正确设置
- 检查坐标是否在地图视野范围内

**Q: 拉伸效果不明显？**
- 确保 `extrude: true`
- 检查 `extrudeValue` 值是否足够大
- 从侧面观察（正上方看不到拉伸）

**Q: 顶点颜色不生效？**
- 确保 `vertexColors: true`
- 检查属性映射 `data.defineAttribute('color', 'color')`
