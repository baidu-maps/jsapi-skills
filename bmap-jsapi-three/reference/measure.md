# Measure 测量工具使用指南

## 概述

Measure 是专门用于地图测量的工具类，继承自 Editor，提供距离测量、面积测量、点坐标测量功能。默认显示测量标签，支持连续测量和自定义标签样式。

## 测量类型

通过 `mapvthree.Editor.MeasureType` 访问：

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| 距离测量 | `Editor.MeasureType.DISTANCE` | 测量线条长度 |
| 面积测量 | `Editor.MeasureType.AREA` | 测量多边形面积 |
| 点测量 | `Editor.MeasureType.POINT` | 测量点坐标 |

## 基本用法

```javascript
import * as mapvthree from '@baidumap/mapv-three';

// 距离测量
const distanceMeasure = engine.add(new mapvthree.Measure({
    type: mapvthree.Editor.MeasureType.DISTANCE
}));

// 面积测量
const areaMeasure = engine.add(new mapvthree.Measure({
    type: mapvthree.Editor.MeasureType.AREA
}));

// 点坐标测量
const pointMeasure = engine.add(new mapvthree.Measure({
    type: mapvthree.Editor.MeasureType.POINT
}));

// 开始测量
distanceMeasure.start();

// 停止测量
distanceMeasure.stop();
```

## 构造参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | MeasureType | `DISTANCE` | 测量类型 |
| `enableMidpointHandles` | boolean | `true` | 显示中点控制点 |
| `renderOptions` | object | 见下方 | 渲染选项 |

**renderOptions 默认值：**
```javascript
{
    depthTest: false,
    transparent: true,
    renderOrder: 100
}
```

> Measure 内部自动设置 `showLabel: true` 和 `singleMode: true`，无需手动配置。

## 距离测量

```javascript
const measure = engine.add(new mapvthree.Measure({
    type: mapvthree.Editor.MeasureType.DISTANCE
}));

// 设置样式
measure.setStyle({
    strokeColor: '#ffff00',
    strokeWidth: 3,
    strokeOpacity: 1
});

// 自定义距离格式
measure.setDistanceFormatter((distance) => {
    return distance < 1000
        ? `${distance.toFixed(1)} 米`
        : `${(distance / 1000).toFixed(2)} 公里`;
});

// 开始测量
measure.start();
```

## 面积测量

```javascript
const areaMeasure = engine.add(new mapvthree.Measure({
    type: mapvthree.Editor.MeasureType.AREA
}));

// 设置样式
areaMeasure.setStyle({
    fillColor: '#52c41a',
    fillOpacity: 0.3,
    strokeColor: '#52c41a',
    strokeWidth: 2
});

// 自定义面积格式
areaMeasure.setAreaFormatter((area) => {
    return area < 10000
        ? `${area.toFixed(1)} 平方米`
        : `${(area / 10000).toFixed(2)} 公顷`;
});

areaMeasure.start();
```

## 点坐标测量

```javascript
const pointMeasure = engine.add(new mapvthree.Measure({
    type: mapvthree.Editor.MeasureType.POINT
}));

// 设置样式
pointMeasure.setStyle({
    fillColor: '#ff4d4f',
    pointRadius: 8,
    fillOpacity: 1
});

// 自定义坐标格式
pointMeasure.setPointFormatter((point) => {
    return `经度: ${point[0].toFixed(6)}\n纬度: ${point[1].toFixed(6)}`;
});

pointMeasure.start();
```

## 连续测量

```javascript
// 启用连续测量模式
measure.start({ continuous: true });

// 停止连续测量
measure.stop();
```

## 切换测量类型

```javascript
measure.setType(mapvthree.Editor.MeasureType.AREA);
measure.setType(mapvthree.Editor.MeasureType.DISTANCE);
measure.setType(mapvthree.Editor.MeasureType.POINT);
```

## 清除与获取结果

```javascript
// 清除所有
measure.clear();

// 清除特定类型
measure.clearByType(mapvthree.Editor.MeasureType.DISTANCE);

// 获取所有结果
const all = measure.getMeasurements();

// 获取特定类型结果
const distances = measure.getMeasurementsByType(mapvthree.Editor.MeasureType.DISTANCE);
```

## 自定义标签渲染

```javascript
measure.setLabelRenderer((value) => {
    const div = document.createElement('div');
    div.innerText = value.attributes.text;
    div.style.cssText = `
        background: rgba(0,0,0,0.8);
        color: #fff;
        padding: 4px 8px;
        border-radius: 4px;
        font-size: 12px;
    `;

    // 主标签特殊样式
    if (value.attributes.isMain) {
        div.style.fontWeight = 'bold';
    }

    return div;
});
```

**value.attributes 属性：**
- `text` - 标签文本内容
- `type` - 测量类型（'distance'/'area'/'point'）
- `isMain` - 是否为主标签

## 编辑测量结果

```javascript
// 启用编辑
measure.enableEdit();

// 编辑特定要素
measure.enableEdit('feature-id');

// 使用过滤函数
measure.enableEdit(feature => feature.properties?.measureType === 'distance');

// 退出编辑
measure.disableEdit();
```

## 事件

```javascript
// 测量完成
measure.addEventListener('created', (e) => {
    console.log('测量完成');
});

// 开始测量
measure.addEventListener('start', () => {
    console.log('开始测量');
});

// 测量过程中实时更新
measure.addEventListener('update', (e) => {
    console.log('当前值:', e);
});
```

## 只读属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `isDrawing` | boolean | 是否正在绘制 |
| `isEditing` | boolean | 是否正在编辑 |
| `measureType` | string | 当前测量类型 |

## 完整示例

```javascript
import * as mapvthree from '@baidumap/mapv-three';

// 创建引擎
const engine = new mapvthree.Engine(container, {
    rendering: { enableAnimationLoop: true }
});
engine.map.setCenter([116.404, 39.915]);
engine.map.setZoom(16);

// 创建测量工具
const measure = engine.add(new mapvthree.Measure({
    type: mapvthree.Editor.MeasureType.DISTANCE
}));

// 配置格式化
measure.setDistanceFormatter(d => d < 1000 ? `${d.toFixed(1)} m` : `${(d/1000).toFixed(2)} km`);
measure.setAreaFormatter(a => a < 10000 ? `${a.toFixed(1)} m²` : `${(a/10000).toFixed(2)} ha`);
measure.setPointFormatter(p => `${p[0].toFixed(6)}, ${p[1].toFixed(6)}`);

// 自定义标签
measure.setLabelRenderer((value) => {
    const div = document.createElement('div');
    div.innerText = value.attributes.text;
    div.style.cssText = 'background:rgba(0,0,0,0.8);color:#fff;padding:4px 8px;border-radius:4px;';
    if (value.attributes.isMain) div.style.fontWeight = 'bold';
    return div;
});

// 监听事件
measure.addEventListener('created', () => console.log('测量完成'));

// 开始测量
measure.start();

// 切换类型
// measure.setType(mapvthree.Editor.MeasureType.AREA);

// 清理
// measure.clear();
// engine.remove(measure);
```

## 最佳实践

### 用户体验优化

```javascript
// 视觉反馈
measure.addEventListener('start', () => {
    document.body.style.cursor = 'crosshair';
});

measure.addEventListener('created', () => {
    document.body.style.cursor = 'default';
});

// 键盘快捷键
document.addEventListener('keydown', (e) => {
    if (e.key === 'Escape') measure.stop();
});
```

### 统一样式配置

```javascript
const MEASURE_STYLES = {
    distance: { strokeColor: '#1890ff', strokeWidth: 3 },
    area: { fillColor: '#52c41a', fillOpacity: 0.3, strokeColor: '#52c41a', strokeWidth: 2 },
    point: { fillColor: '#ff4d4f', pointRadius: 8 }
};

measure.setStyle(MEASURE_STYLES.distance);
```

## 注意事项

1. **坐标系**：使用 Z-up 坐标系
2. **内存管理**：组件销毁时调用 `engine.remove(measure)`
3. **样式设置**：测量标签默认显示，可通过 `setLabelRenderer` 自定义
4. **精度控制**：通过格式化函数控制显示精度