# Label 组件使用指南

## 概述

Label 用于渲染文本标签、图标标签或组合标签，支持批量管理大量标签数据。

## 类型

| 类型 | 值 | 说明 |
|------|-----|------|
| 纯图标 | `'icon'` | 仅显示图标 |
| 纯文本 | `'text'` | 仅显示文本 |
| 图标+文本 | `'icontext'` | 同时显示图标和文本 |

## 基础用法

### 纯文本标签

```javascript
const textLabel = engine.add(new mapvthree.Label({
    type: 'text',
    textSize: 16,
    textFillStyle: 'rgb(255, 8, 8)',
    textStrokeStyle: 'rgba(0, 0, 0, 1)',
    textStrokeWidth: 2,
}));

const data = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: { text: '北京市' }
}]);

data.defineAttribute('text', 'text');
textLabel.dataSource = data;
```

### 纯图标标签

```javascript
const iconLabel = engine.add(new mapvthree.Label({
    type: 'icon',
    vertexIcons: true,
    iconWidth: 40,
    iconHeight: 40,
}));

const data = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: { icon: 'assets/icons/marker.png' }
}]);

data.defineAttribute('icon', 'icon');
iconLabel.dataSource = data;
```

### 图标+文本组合

```javascript
const comboLabel = engine.add(new mapvthree.Label({
    type: 'icontext',
    vertexIcons: true,
    textSize: 16,
    iconWidth: 40,
    iconHeight: 40,
    padding: [2, 2],
}));

const data = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: { icon: 'assets/icons/marker.png', text: '地点名称' }
}]);

data.defineAttributes({
    icon: 'icon',
    text: 'text'
});
comboLabel.dataSource = data;
```

## 构造参数

### 基础参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | string | `'icon'` | `'icon'` / `'text'` / `'icontext'` |
| `flat` | boolean | `false` | 贴地渲染 |
| `vertexIcons` | boolean | `false` | 数据中携带 icon URL |
| `opacity` | number | `1` | 透明度（0-1） |
| `keepSize` | boolean | `true` | 保持固定屏幕大小 |
| `depthTest` | boolean | `true` | 深度测试 |

### 文字样式参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `textSize` | number | `16` | 文字大小（像素） |
| `textFamily` | string | `'sans-serif'` | 字体 |
| `textWeight` | string | `'400'` | 字重（`'400'`/`'700'`） |
| `textFillStyle` | string/array | `[255,255,255]` | 文字颜色 |
| `textStrokeStyle` | string/array | `[0,0,0]` | 描边颜色 |
| `textStrokeWidth` | number | `0` | 描边宽度 |
| `textAnchor` | string | `'center'` | 锚点位置 |
| `textAlign` | string | `'center'` | 文本对齐（`'left'`/`'center'`/`'right'`） |
| `textOffset` | array | `[0, 0]` | 文字偏移 `[x, y]` |
| `textPadding` | array | `[0, 2]` | 文字内边距 `[x, y]` |
| `maxWidth` | number | - | 文本最大宽度（超出换行） |

**textAnchor 可选值：**
- `'center'`（默认）
- `'left'` / `'right'` / `'top'` / `'bottom'`
- `'top-left'` / `'top-right'` / `'bottom-left'` / `'bottom-right'`

### 图标样式参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `iconWidth` | number | `40` | 图标宽度 |
| `iconHeight` | number | `40` | 图标高度 |
| `mapSrc` | string | - | 默认图标 URL |
| `useIconScale` | boolean | `false` | 使用图标原始宽高比 |

### 布局参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `padding` | array | `[2, 2]` | 图标与文字间距 `[x, y]` |
| `offset` | array | `[0, 0]` | 坐标偏移 |
| `pixelOffset` | array | `[0, 0]` | 像素偏移 |
| `rotateZ` | number | `0` | 旋转角度（弧度） |

### 淡入淡出参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enableFade` | boolean | `false` | 启用淡入淡出 |
| `fadeDuration` | number | `300` | 动画时长（毫秒） |

## 高级功能

### 数据级样式覆盖

通过数据属性覆盖全局样式：

```javascript
const data = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: {
        text: '自定义样式',
        textSize: 20,
        textFillStyle: 'rgb(255, 0, 0)',
        icon: 'assets/icon.png',
        iconSize: [50, 50],
        rotateZ: 0.78,  // 弧度
    }
}]);

data.defineAttributes({
    text: 'text',
    textSize: 'textSize',
    textFillStyle: 'textFillStyle',
    icon: 'icon',
    iconSize: 'iconSize',
    rotateZ: 'rotateZ',
});
```

**支持的数据级属性：**
- `text`, `textSize`, `textFillStyle`, `textStrokeStyle`, `textStrokeWidth`
- `textAnchor`, `textOffset`, `textWeight`, `textFamily`
- `icon`, `iconSize`, `iconOpacity`
- `offset`, `rotateZ`, `maxWidth`

### 淡入淡出效果

```javascript
const label = engine.add(new mapvthree.Label({
    type: 'text',
    enableFade: true,
    fadeDuration: 500,
}));

// 数据必须包含 id 字段
const data = mapvthree.GeoJSONDataSource.fromGeoJSON([{
    type: 'Feature',
    properties: { id: 'point_001', text: '北京市' },
    geometry: { type: 'Point', coordinates: [116.404, 39.915] }
}]);
```

### 碰撞检测

通过 `engine.rendering.collision` 配置：

```javascript
const label = engine.add(new mapvthree.Label({
    type: 'text',
    textSize: 14,
}));

// 添加到碰撞检测系统
engine.rendering.collision.add(label, {
    margin: [5, 5]  // 碰撞边距 [x, y]
}, 'poi-labels');   // 分组名称（可选）

// 移除碰撞检测
engine.rendering.collision.remove(label);
```

### 贴地渲染

```javascript
const label = engine.add(new mapvthree.Label({
    type: 'text',
    flat: true,
}));
```

### 事件交互

```javascript
label.addEventListener('click', (e) => {
    console.log('点击了标签');
    console.log('数据索引:', e.entity.index);
    console.log('原始数据:', e.entity.value);
});

label.addEventListener('mouseover', (e) => {
    console.log('鼠标悬停:', e.entity.value.text);
});
```

**事件对象结构：**

| 属性 | 类型 | 说明 |
|------|------|------|
| `e.entity.index` | number | 数据索引 |
| `e.entity.value` | object | 原始数据对象 |
| `e.point` | Vector3 | 点击的 3D 坐标 |
| `e.distance` | number | 与相机的距离 |

**支持的事件类型：**
- `click`, `dblclick`, `rightclick`
- `mouseover`, `mouseout`, `mousemove`
- `mouseenter`, `mouseleave`

## 实际应用

### POI 标注

```javascript
const poiLabel = engine.add(new mapvthree.Label({
    type: 'icontext',
    vertexIcons: true,
    textSize: 14,
    textFillStyle: '#333',
    textStrokeStyle: '#fff',
    textStrokeWidth: 2,
    iconWidth: 32,
    iconHeight: 32,
}));

// 启用碰撞检测避免重叠
engine.rendering.collision.add(poiLabel, { margin: [10, 10] });
```

### 实时数据更新

```javascript
const dataLabel = engine.add(new mapvthree.Label({
    type: 'text',
    textSize: 16,
    enableFade: true,
    fadeDuration: 300,
}));

// 动态更新数据
setInterval(() => {
    const value = Math.floor(Math.random() * 1000);
    dataSource.setAttributeValues('sensor_001', {
        text: `流量: ${value}/s`,
        textFillStyle: value > 500 ? 'rgb(255, 0, 0)' : 'rgb(0, 255, 0)',
    });
}, 2000);
```

## 注意事项

1. **图标路径**：`vertexIcons: true` 时，数据中的 icon 必须是有效 URL
2. **淡入淡出**：数据必须包含 `id` 属性才能正确跟踪状态
3. **碰撞检测**：重叠的标签会被自动隐藏
4. **性能**：大量标签建议启用碰撞检测减少渲染数量
5. **坐标系**：使用 Z-up 坐标系
