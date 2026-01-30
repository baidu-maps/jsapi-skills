# DataSource 数据源使用指南

## 概述

DataSource 是 Mapv-three 中管理地理数据的核心组件。它负责将各种格式的原始数据转换为可视化组件可渲染的标准格式。

## 数据源类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **GeoJSONDataSource** | 处理标准 GeoJSON 格式（推荐） | 大多数地理可视化 |
| **JSONDataSource** | 处理通用 JSON 格式 | 非标准格式数据 |
| **CSVDataSource** | 处理 CSV 表格数据 | 点数据、表格导入 |
| **DataSource** | 基础类，手动创建数据项 | 动态生成、实时更新 |

## GeoJSONDataSource（推荐）

### 创建数据源

```javascript
// 方式1: 从 GeoJSON 对象创建
const data = mapvthree.GeoJSONDataSource.fromGeoJSON({
    type: 'FeatureCollection',
    features: [
        {
            type: 'Feature',
            geometry: { type: 'Point', coordinates: [116.39, 39.9] },
            properties: { name: '北京', value: 100 }
        }
    ]
});

// 方式2: 从 URL 加载
const data = await mapvthree.GeoJSONDataSource.fromURL('path/to/data.geojson');
```

### 绑定到可视化组件

```javascript
const layer = engine.add(new mapvthree.Polygon({ color: 'red' }));
layer.dataSource = data;
```

### 支持的几何类型

- `Point` / `MultiPoint`
- `LineString` / `MultiLineString`
- `Polygon` / `MultiPolygon`

## JSONDataSource

适用于非标准 JSON 格式，支持自定义坐标字段和解析函数。

```javascript
// 从 JSON 对象创建
const jsonData = [
    { lng: 116.39, lat: 39.9, name: '北京', value: 100 },
    { lng: 121.47, lat: 31.23, name: '上海', value: 90 }
];

const data = mapvthree.JSONDataSource.fromJSON(jsonData, {
    parseCoordinates: (item) => ({
        type: 'Point',
        coordinates: [item.lng, item.lat]
    })
});

// 从 URL 加载
const data = await mapvthree.JSONDataSource.fromURL('data.json', {
    parseCoordinates: (item) => ({
        type: 'Point',
        coordinates: [item.lng, item.lat]
    })
});
```

**配置选项：**

| 选项 | 类型 | 说明 |
|------|------|------|
| coordinatesKey | string | 坐标字段名，默认 'coordinates' |
| parseCoordinates | Function | 自定义坐标解析函数 |
| parseFeature | Function | 自定义特征解析函数 |

## CSVDataSource

适用于 CSV 表格数据，继承自 JSONDataSource。

```javascript
// 从 CSV 字符串创建
const csvString = `lng,lat,name,value
116.39,39.9,北京,100
121.47,31.23,上海,90`;

const data = mapvthree.CSVDataSource.fromCSVString(csvString, {
    parseCoordinates: (item) => ({
        type: 'Point',
        coordinates: [parseFloat(item.lng), parseFloat(item.lat)]
    })
});

// 从 URL 加载
const data = await mapvthree.CSVDataSource.fromURL('data.csv', {
    parseCoordinates: (item) => ({
        type: 'Point',
        coordinates: [parseFloat(item.lng), parseFloat(item.lat)]
    })
});
```

## 属性映射

将数据属性映射到渲染属性（着色器 attribute）。

```javascript
// 批量定义属性映射
// 格式: { 渲染属性名: 数据属性名 或 映射函数 }
data.defineAttributes({
    color: 'colorField',                  // 'colorField' 是数据中的属性名
    size: (attrs) => attrs.value / 10     // 函数映射，attrs 是数据项的 attributes
});

// 单个属性映射
// defineAttribute(渲染属性名, 数据属性名)
data.defineAttribute('height', 'buildingHeight');  // 将数据中的 buildingHeight 映射到渲染的 height

// 条件映射
data.defineAttribute('color', (attrs) => {
    if (attrs.value > 100) return [255, 0, 0];
    return [0, 255, 0];
});
```

**示例数据：**
```javascript
// 假设 GeoJSON 数据如下
{
    type: 'Feature',
    properties: {
        colorField: [255, 0, 0],    // 映射到 color
        value: 50,                   // 通过函数映射到 size (50/10=5)
        buildingHeight: 100          // 映射到 height
    }
}
```

## 数据操作

### 添加数据

```javascript
// 添加单个
data.add(new mapvthree.DataItem([116.39, 39.9], { name: '北京' }));

// 批量添加
data.add([item1, item2, item3]);
```

### 修改数据

```javascript
// 修改属性
data.setAttributeValues('featureId', { color: [255, 0, 0], size: 20 });

// 修改坐标
data.setCoordinates('featureId', [116.40, 39.91]);
```

### 删除数据

```javascript
data.remove('featureId');
data.remove(['id1', 'id2']);
```

### 查询数据

```javascript
const count = data.size;                    // 数据项数量
const item = data.getDataItem(0);           // 获取数据项
const allItems = data.dataItems;            // 所有数据项
```

### 过滤数据

```javascript
// 过滤函数签名: (dataItem, index) => boolean
data.setFilter((dataItem, index) => dataItem.attributes.visible === true);
```

### 导出数据

```javascript
const geojson = data.exportToGeoJSON();
```

## 实际应用示例

### 城市点位可视化

```javascript
const cities = await mapvthree.GeoJSONDataSource.fromURL('cities.geojson');

cities.defineAttributes({
    color: (attrs) => {
        const pop = attrs.population;
        if (pop > 10000000) return [255, 0, 0];
        if (pop > 5000000) return [255, 128, 0];
        return [0, 255, 0];
    },
    size: (attrs) => Math.sqrt(attrs.population) / 100
});

const points = engine.add(new mapvthree.SimplePoint({
    vertexColors: true,
    vertexSizes: true
}));
points.dataSource = cities;
```

### 实时数据更新

```javascript
const data = new mapvthree.DataSource();
data.add([
    new mapvthree.DataItem([116.39, 39.9], { id: 'car1' }),
    new mapvthree.DataItem([121.47, 31.23], { id: 'car2' })
]);

const points = engine.add(new mapvthree.SimplePoint({ size: 10 }));
points.dataSource = data;

// 定时更新位置
setInterval(async () => {
    const positions = await fetch('/api/vehicle-positions').then(r => r.json());
    positions.forEach(pos => {
        data.setCoordinates(pos.id, [pos.lng, pos.lat]);
    });
}, 1000);
```

## 最佳实践

### 性能优化

```javascript
// 批量添加（推荐）
data.add([item1, item2, item3, ...]);

// 批量定义属性（推荐）
data.defineAttributes({ color: 'c', size: 's' });

// 数据过滤减少渲染负担
data.setFilter((dataItem) => dataItem.attributes.visible);
```

### 数据格式选择

1. **GeoJSON**（推荐）- 标准格式，支持所有几何类型
2. **JSON** - 灵活格式，需要自定义解析
3. **CSV** - 适合表格数据和点数据
4. **手动创建** - 适合动态生成和实时更新

## 常见问题

**Q: 数据没有显示？**
- 检查 `layer.dataSource = data` 是否正确设置
- 检查坐标格式（GeoJSON 使用 [经度, 纬度]）
- 检查数据是否在地图视野范围内

**Q: 属性映射不生效？**
- 确保可视化对象启用了相应特性（如 `vertexColors: true`）
- 检查属性名是否正确

**Q: 数据更新后视图没有刷新？**
- 使用数据源的更新方法（`setAttributeValues`, `setCoordinates`）
- 数据源修改后会自动触发更新
