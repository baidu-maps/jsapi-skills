# TileMask 瓦片掩膜

## 概述

TileMask 通过 Stencil Buffer 实现对瓦片图层的区域裁剪渲染。指定一个 GeoJSON 多边形区域作为掩膜范围，只显示该区域内的瓦片内容。

适用场景：
- 按行政区划边界裁剪地图显示
- 聚焦特定区域的地图可视化

支持的目标：
- `MapView` - 自动提取内部所有 TileProvider
- 单个 `TileProvider` - 需要 `supportsMask = true`（如 `BaiduVectorTileProvider`、`BaiduLaneVectorTileProvider`）

## 基础用法

```javascript
// 准备掩膜区域（GeoJSON Polygon）
const geojson = {
    type: 'FeatureCollection',
    features: [{
        type: 'Feature',
        geometry: {
            type: 'Polygon',
            coordinates: [[[116.1, 39.7], [116.7, 39.7], [116.7, 40.1], [116.1, 40.1], [116.1, 39.7]]]
        }
    }]
};

// 添加百度矢量地图
const mapView = engine.add(new mapvthree.MapView({
    vectorTileProvider: new mapvthree.BaiduVectorTileProvider(),
}));

// 创建掩膜，targets 传入 MapView
const tileMask = engine.add(new mapvthree.TileMask({
    targets: [mapView],
    region: geojson,
}));
```

## 构造参数

```javascript
new mapvthree.TileMask(options)
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `options.targets` | Array | - | **必需**。要应用掩膜的目标列表，支持 MapView 或 TileProvider |
| `options.region` | Object/GeoJSONDataSource | - | **必需**。掩膜区域，GeoJSON 格式或 GeoJSONDataSource 实例 |

## 属性

| 属性 | 类型 | 读写 | 说明 |
|------|------|------|------|
| `enabled` | boolean | 读写 | 是否启用掩膜 |
| `region` | Object/GeoJSONDataSource | 读写 | 掩膜区域，设置后立即生效 |
| `targets` | Array | 只读 | 当前目标 provider 列表（返回副本） |

## 方法

| 方法 | 说明 |
|------|------|
| `addTarget(target)` | 添加掩膜目标（MapView 或 TileProvider） |
| `removeTarget(target)` | 移除掩膜目标 |
| `addObject(object, targetProvider?)` | 添加需要掩膜的 3D 对象，可选 targetProvider 限定掩膜范围 |
| `removeObject(object)` | 移除需要掩膜的 3D 对象 |
| `addMaterial(material, targetProvider?)` | 添加需要掩膜的材质（支持单个或数组），可选 targetProvider 限定掩膜范围 |
| `removeMaterial(material)` | 移除需要掩膜的材质 |
| `dispose()` | 释放引用资源（targets、dataSource 等） |

## 动态控制

```javascript
// 启用/禁用掩膜
tileMask.enabled = false;
tileMask.enabled = true;

// 动态更新掩膜区域
tileMask.region = {
    type: 'Feature',
    geometry: {
        type: 'Polygon',
        coordinates: [[[111.5, 27.3], [112.0, 27.3], [112.0, 27.7], [111.5, 27.7], [111.5, 27.3]]]
    }
};

// 动态添加/移除目标
tileMask.addTarget(anotherVectorMap);
tileMask.removeTarget(anotherVectorMap);
```

## 完整示例

```javascript
const engine = new mapvthree.Engine(document.getElementById('map_container'), {
    rendering: { enableAnimationLoop: true },
});
engine.map.setCenter([111.917, 27.518]);
engine.map.setZoom(14);

// 添加天空
engine.add(new mapvthree.DynamicSky());

// 添加百度矢量地图
const mapView = engine.add(new mapvthree.MapView({
    vectorTileProvider: new mapvthree.BaiduVectorTileProvider(),
}));

// 加载行政区划边界（GeoJSON Polygon）
const response = await fetch('data/hunan_border.json');
const borderGeoJSON = await response.json();

// 创建掩膜
const tileMask = engine.add(new mapvthree.TileMask({
    targets: [mapView],
    region: borderGeoJSON,
}));

// 交互控制
document.getElementById('btn-enable').onclick = () => {
    tileMask.enabled = true;
};
document.getElementById('btn-disable').onclick = () => {
    tileMask.enabled = false;
};
```

## 注意事项

- `targets` 中的 TileProvider 需要 `supportsMask = true` 才能被掩膜。目前内置支持的有 `BaiduVectorTileProvider` 和 `BaiduLaneVectorTileProvider`。
- `region` 的 GeoJSON 必须是 Polygon 或 MultiPolygon 类型。如果数据是 LineString，需先转换为闭合的 Polygon。
- TileMask 必须通过 `engine.add()` 添加到场景中才能生效。
- 设置 `region` 后掩膜区域会立即更新并重新渲染。

## 资源清理

```javascript
engine.remove(tileMask);
tileMask.dispose();
```

> **注意**：应先调用 `engine.remove()` 再调用 `dispose()`。
