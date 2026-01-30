# Editor 编辑器使用指南

## 概述

Editor 是 MapV Three 的地图编辑器组件，提供要素绘制、编辑、测量功能。支持多边形、线、点、圆、矩形等几何类型。

## 绘制类型

```javascript
mapvthree.Editor.DrawerType = {
    POLYGON: 'polygon',      // 多边形
    LINE: 'line',            // 线段
    POINT: 'point',          // 点
    CIRCLE: 'circle',        // 圆形
    RECTANGLE: 'rectangle'   // 矩形
}
```

## 基本用法

### 初始化

```javascript
const editor = engine.add(new mapvthree.Editor({
    type: mapvthree.Editor.DrawerType.POLYGON,
    enableMidpointHandles: true,   // 启用中点标记（可在边上添加顶点）
    singleMode: false,             // 单要素模式（绘制新要素时自动清除旧要素）
    showLabel: false,              // 显示测量标签
    renderOptions: {
        depthTest: false,
        transparent: true,
        renderOrder: 0
    }
}));
```

### 构造函数参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| type | string | 'polygon' | 默认绘制类型 |
| enableMidpointHandles | boolean | false | 启用中点标记 |
| singleMode | boolean | false | 单要素模式 |
| showLabel | boolean | false | 显示测量标签 |
| continuousDrawing | boolean | false | 连续绘制模式 |
| renderOptions | object | - | 渲染选项 |

### 绘制要素

```javascript
// 1. 设置绘制类型
editor.type = mapvthree.Editor.DrawerType.POLYGON;

// 2. 设置样式
editor.setStyle({
    fillColor: '#ff0000',
    fillOpacity: 0.5,
    strokeColor: '#ffffff',
    strokeWidth: 2
});

// 3. 开始绘制
editor.start({ continuous: false });

// 4. 停止绘制
editor.stop();
```

## 样式配置

### 样式属性

| 属性 | 类型 | 说明 | 适用类型 |
|------|------|------|----------|
| fillColor | string | 填充颜色 | polygon, circle, rectangle, point |
| fillOpacity | number | 填充透明度 (0-1) | polygon, circle, rectangle, point |
| strokeColor | string | 边框颜色 | 全部 |
| strokeWidth | number | 边框宽度（像素） | 全部 |
| strokeOpacity | number | 边框透明度 (0-1) | 全部 |
| pointRadius | number | 点半径 | point |

### 设置样式

```javascript
// 为所有类型设置默认样式
editor.setStyle({
    fillColor: '#ff0000',
    fillOpacity: 0.5,
    strokeColor: '#ffffff',
    strokeWidth: 2
});

// 为特定类型设置样式
editor.setStyle({
    strokeColor: '#00ff00',
    strokeWidth: 3
}, mapvthree.Editor.DrawerType.LINE);

// 获取样式
const style = editor.getStyle(mapvthree.Editor.DrawerType.POLYGON);
```

### 更新要素样式

```javascript
// 更新单个要素样式（合并）
editor.updateFeatureStyle('feature-id', {
    fillColor: '#00ff00'
});

// 替换要素样式
editor.updateFeatureStyle('feature-id', {
    fillColor: '#0000ff',
    strokeColor: '#ffffff'
}, true);  // true 表示完全替换
```

## 编辑功能

### 启用编辑

```javascript
// 编辑所有要素
editor.enableEdit();

// 编辑指定 ID 的要素
editor.enableEdit('feature-id');

// 使用过滤函数
editor.enableEdit((feature) => feature.geometry.type === 'Polygon');
```

### 关闭编辑

```javascript
editor.disableEdit();
```

## 数据管理

### 导出数据

```javascript
// 导出所有数据（GeoJSON 格式）
const allData = editor.exportData();

// 导出特定类型
const polygonData = editor.exportData(mapvthree.Editor.DrawerType.POLYGON);
```

### 导入数据

```javascript
const geojson = {
    type: 'FeatureCollection',
    features: [{
        type: 'Feature',
        id: 'polygon-1',
        geometry: {
            type: 'Polygon',
            coordinates: [[[116.516, 39.799], [116.517, 39.799], [116.517, 39.800], [116.516, 39.799]]]
        },
        properties: {}
    }]
};

// 导入数据
editor.importData(geojson, {
    clear: true,      // 清除现有数据
    fitBounds: true   // 自动调整视图
});
```

### 删除数据

```javascript
// 删除所有要素
editor.delete();

// 删除特定类型
editor.delete(mapvthree.Editor.DrawerType.POLYGON);

// 根据 ID 删除
editor.deleteById('feature-id');
editor.deleteById(['feature-1', 'feature-2']);
```

## 可见性控制

```javascript
// 显示/隐藏所有要素
editor.show();
editor.hide();

// 显示/隐藏特定类型
editor.show(mapvthree.Editor.DrawerType.POLYGON);
editor.hide(mapvthree.Editor.DrawerType.LINE);
```

## 状态查询

```javascript
// 是否正在绘制
if (editor.isDrawing) {
    console.log('绘制中');
}

// 是否正在编辑
if (editor.isEditing) {
    console.log('编辑中');
}
```

## 事件系统

### 支持的事件

| 事件 | 参数 | 说明 |
|------|------|------|
| start | `{drawerType, options}` | 开始绘制 |
| created | `{feature}` | 要素创建完成 |
| update | `{feature}` | 要素更新 |
| delete | - | 要素删除 |
| featureClick | `{featureId}` | 点击要素 |

### 事件监听

```javascript
editor.addEventListener('created', (e) => {
    console.log('要素创建:', e.feature);
});

editor.addEventListener('update', (e) => {
    console.log('要素更新:', e.feature);
});

editor.addEventListener('delete', () => {
    console.log('要素删除');
});

editor.addEventListener('featureClick', (e) => {
    console.log('点击要素:', e.featureId);
    editor.enableEdit(e.featureId);
});
```

## 完整示例

```javascript
// 初始化引擎
const engine = new mapvthree.Engine(document.getElementById('map'), {
    map: { center: [116.516, 39.799], zoom: 18 }
});

engine.add(new mapvthree.MapView({
    imageryProvider: new mapvthree.BingImageryTileProvider()
}));

// 创建编辑器
const editor = engine.add(new mapvthree.Editor({
    type: mapvthree.Editor.DrawerType.POLYGON,
    enableMidpointHandles: true
}));

// 设置样式
editor.setStyle({
    fillColor: '#ff0000',
    fillOpacity: 0.5,
    strokeColor: '#ffffff',
    strokeWidth: 2
});

// 监听事件
editor.addEventListener('created', () => {
    const data = editor.exportData();
    console.log('当前数据:', JSON.stringify(data, null, 2));
});

editor.addEventListener('featureClick', (e) => {
    editor.enableEdit(e.featureId);
});

// 开始绘制
function startDraw(type) {
    editor.type = type;
    editor.start({ continuous: false });
}

// 停止绘制
function stopDraw() {
    editor.stop();
}

// 清空
function clearAll() {
    editor.delete();
}

// 销毁
function destroy() {
    engine.remove(editor);
}
```

## 注意事项

1. **坐标系**：使用 Z-up 坐标系，坐标格式为 `[经度, 纬度]`

2. **性能优化**：
   - 大量要素时使用 `show()`/`hide()` 而非反复添加删除
   - 批量删除使用数组形式 `deleteById([...])`

3. **内存管理**：
   - 组件销毁时调用 `engine.remove(editor)`

4. **事件处理**：
   - 频繁触发的 `update` 事件建议使用防抖处理
