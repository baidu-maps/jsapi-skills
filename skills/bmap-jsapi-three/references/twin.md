# Twin 孪生车流

## 概述

Twin 是 MapV-Three 的实时孪生车流可视化系统，支持大规模车辆的实时位置更新、模型渲染、动画播放和交互操作。适用于智慧交通、数字孪生城市、自动驾驶仿真等场景。

主要特性：
- 支持多种车型模型（写实风格/极简风格）
- 实时数据推送与插值动画
- 车辆追踪（外部视角/车内视角）
- 车辆颜色自定义
- 附加物体（标签、特效点、DOM 元素等）

> 模拟车流请参考 `mock-twin.md`。

## 基础用法

```javascript
// 创建 Twin 实例
const twin = engine.add(new mapvthree.Twin({
    delay: 1000,
    keepSize: true,
    modelConfig: {
        2: mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.CAR,
        3: mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.TRUCK,
    },
}));

// 配置数据处理器
twin.dataProvider
    .process('time', 'timestamp')
    .process('point', item => [item.lon, item.lat, item.altitude])
    .process('dir', item => (90 - item.heading) / 180 * Math.PI)
    .process('modelType', item => item.vehicleType)
    .process('color', item => mapvthree.Twin.REALISTIC_TEMPLATE_COLOR[item.colorName]);

// 推送实时数据
twin.push([
    { id: 'car_001', timestamp: Date.now(), lon: 116.404, lat: 39.915, altitude: 0, heading: 90, vehicleType: 2, colorName: 'white' },
    { id: 'car_002', timestamp: Date.now(), lon: 116.405, lat: 39.916, altitude: 0, heading: 45, vehicleType: 3, colorName: 'blue' },
]);
```

## 构造参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `modelConfig` | object | - | 模型配置，key 为模型类型编号，value 为模型地址（GLB/GLTF） |
| `delay` | number | `2000` | 数据刷新时间间隔（毫秒），控制插值动画的平滑度 |
| `objects` | Array | `[]` | 车辆附加物体数组（如 TwinLabel、EffectPoint、DOMPoint 等） |
| `objectAttributes` | object | `{}` | 附加物体所需数据属性映射，如 `{ text: 'speed' }` |
| `extraDir` | number | `0` | 模型额外角度偏移 |
| `keepSize` | boolean | `false` | 模型是否保持像素大小不变（不随缩放变化） |
| `maxScale` | number | `20` | 模型最大缩放比例 |
| `enableColorList` | Array | `['body']` | 支持颜色设置的模型部件列表 |
| `autoClearTrack` | boolean | `true` | 数据重置时是否自动清除车辆追踪 |

## DataProvider 数据处理器

DataProvider 用于将原始数据转换为 Twin 所需的标准格式，支持链式调用。

```javascript
twin.dataProvider
    .process('time', 'timestamp')                              // 直接映射字段
    .process('point', item => [item.lon, item.lat, item.alt])  // 自定义转换函数
    .process('dir', item => (90 - item.heading) / 180 * Math.PI)
    .process('modelType', item => item.vehicleType)
    .process('color', item => mapvthree.Twin.REALISTIC_TEMPLATE_COLOR[item.colorName])
    .process('scale', item => [1, 1, 1])
    .process('speed', 'speed');
```

### 标准属性列表

| 属性名 | 类型 | 必需 | 说明 |
|--------|------|------|------|
| `time` | number | 是 | 时间戳（毫秒） |
| `point` | Array | 是 | 地理坐标 `[经度, 纬度, 高度]` |
| `dir` | number | 是 | 方向角（弧度），0 = 东，PI/2 = 北 |
| `modelType` | number | 是 | 模型类型编号（对应 modelConfig 的 key） |
| `color` | string | 否 | 车辆颜色值 |
| `scale` | Array | 否 | 缩放比例 `[x, y, z]` |
| `speed` | number | 否 | 速度（可用于附加物体显示） |

> **注意**：`process` 的第二个参数可以是字符串（直接复制源数据中的同名属性），也可以是函数（自定义转换逻辑）。

## 方法

### 数据管理

| 方法 | 参数 | 说明 |
|------|------|------|
| `push(data)` | Array | 推送一帧车辆数据（每个元素需包含 `id`），触发实时更新 |
| `reset()` | - | 重置所有状态和数据 |

### 播放控制

| 方法 | 说明 |
|------|------|
| `pause()` | 暂停车流动画 |
| `start()` | 开始或继续播放 |

### 车辆追踪

```javascript
// 追踪指定车辆（外部视角）
twin.trackById(carId, {
    radius: 100,     // 视角距离（米）
    pitch: 1.2,      // 俯仰角（弧度）
    height: 50,      // 高度（米）
});

// 车内视角追踪
twin.trackById(carId, {
    inside: true,
    extraDir: 0,     // 额外角度偏移
});

// 清除追踪
twin.clearTrack();
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | any | - | 车辆 ID |
| `option.radius` | number | `100` | 视角距离（米） |
| `option.pitch` | number | `1.2` | 俯仰角（弧度） |
| `option.height` | number | - | 高度（米） |
| `option.inside` | boolean | `false` | 是否车内视角 |
| `option.extraDir` | number | - | 额外角度偏移 |

### 视觉控制

```javascript
// 按 ID 设置车辆颜色
twin.setColorById('car_001', '#ff0000');

// 按模型类型控制可见性
twin.setVisibleByType(2, false);         // 隐藏所有类型为 2 的车辆
twin.setObjectVisibleByType(3, false);   // 隐藏类型为 3 的车辆的附加物体
twin.setObjectVisible(false);            // 隐藏所有附加物体

// 控制整体可见性
twin.visible = false;
```

### 查询方法

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `getModelInstance(modelType)` | Object3D | 获取指定模型类型的实例 |
| `getEntityByIntersection(intersection)` | object | 通过射线交集获取实体数据 |
| `getCurrentBuffers()` | object | 获取当前缓冲区数据 |

### 资源管理

| 方法 | 说明 |
|------|------|
| `modelClear()` | 清除模型资源 |
| `objectsClear()` | 清除附加物体 |
| `dispose()` | 完全释放 Twin 对象 |

## 事件

```javascript
// 点击事件
twin.receiveRaycast = true;  // 开启射线检测
twin.addEventListener('click', (e) => {
    console.log('点击的车辆:', e.clickInfo);
});

// 每帧更新事件
twin.addEventListener('ticking', (e) => {
    console.log('当前帧数据:', e.buffers);
});

// 模型加载完成事件
twin.addEventListener('modelLoaded', (e) => {
    console.log('模型映射:', e.modelMap);
});
```

| 事件 | 触发时机 | 回调参数 |
|------|----------|----------|
| `click` | 点击车辆 | `e.clickInfo` - 车辆数据 |
| `ticking` | 每帧更新 | `e.buffers` - 缓冲区数据 |
| `modelLoaded` | 模型加载完成 | `e.modelMap` - 模型映射 |

> **注意**：使用点击事件前需设置 `twin.receiveRaycast = true`。

## 内置模型模板

### 写实风格

```javascript
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.CAR        // 轿车
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.BUS        // 公交车
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.TRUCK      // 卡车
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.TAXI       // 出租车
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.SUV        // SUV
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.MPV        // MPV
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.MAN        // 行人（男）
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.WOMAN      // 行人（女）
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.BICYCLE    // 自行车
mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.MOTORCYCLE // 摩托车
// ... 20+ 种车型
```

### 极简风格

```javascript
mapvthree.Twin.MINIMALIST_TEMPLATE_MODEL.CAR              // 极简轿车
mapvthree.Twin.MINIMALIST_TEMPLATE_MODEL.MAN_ANIMATE      // 带动画行人（男）
mapvthree.Twin.MINIMALIST_TEMPLATE_MODEL.WOMAN_ANIMATE    // 带动画行人（女）
mapvthree.Twin.MINIMALIST_TEMPLATE_MODEL.BICYCLE_ANIMATE  // 带动画自行车
// ... 包含所有写实模型 + 动画版本
```

### 后端服务类型映射

```javascript
// 自动映射后端 type 值（3-27）到对应模型
const modelConfig = mapvthree.Twin.SERVICE_TEMPLATE_MODEL;
```

### 内置颜色

```javascript
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.white    // 白色 #F4F8FC
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.black    // 黑色 #25272F
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.gray     // 灰色 #6F7580
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.blue     // 蓝色 #284CC7
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.red      // 红色 #D11800
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.green    // 绿色 #3FA765
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.brown    // 棕色 #CC7C42
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.yellow   // 黄色 #DBA100
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.orange   // 橙色 #D14D00
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.purple   // 紫色 #8100D1
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.cyanl    // 青色 #00BEA5（注意拼写为 cyanl）
mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.pink     // 粉色 #D37AD0
```

> 也可直接使用任意 CSS 颜色值，如 `'#ff0000'`、`'rgb(255, 0, 0)'`。

## 附加物体

Twin 支持为每辆车添加附加物体，如标签（Label）、特效点（EffectPoint）、DOM 元素（DOMPoint）等。

```javascript
// 创建车辆标签（使用 Label）
const label = engine.add(new mapvthree.Label({
    type: 'icontext',
    mapSrc: 'website_assets/images/speed-panel.png',
    iconWidth: 141,
    iconHeight: 54,
    textSize: 14,
    textFillStyle: '#fff',
    textStrokeStyle: 'rgba(0, 0, 0, 0.8)',
    textStrokeWidth: 2,
    pixelOffset: [0, 40],
    keepSize: true,
    depthTest: false,
    transparent: true,
}));

// 创建特效点
const effectPoint = engine.add(new mapvthree.EffectPoint({
    color: 'rgba(210, 160, 57, 1.0)',
    size: 30,
    type: 'Wave',
    duration: 1000,
}));

// 创建 DOM 覆盖物
const domPoint = engine.add(new mapvthree.DOMPoint({
    offset: [-270, -200],
}));
domPoint.renderItem = (feature) => {
    const node = document.createElement('div');
    node.innerText = feature.speed + ' km/h';
    return node;
};

// 将附加物体绑定到 Twin
const twin = engine.add(new mapvthree.Twin({
    modelConfig: { ... },
    objects: [label, effectPoint, domPoint],
    objectAttributes: {
        'text': 'speed',        // 标签文本绑定到 speed 属性
    },
}));

// 数据处理中提供速度数据
twin.dataProvider
    .process('speed', item => item.speed);
```

> 附加物体的详细参数请参考 `label.md`、`effect-point.md` 等对应文档。

## 完整示例

```javascript
const engine = new mapvthree.Engine(document.getElementById('map_container'), {
    rendering: { enableAnimationLoop: true },
});
engine.map.lookAt([113.314, 23.516], { range: 500 });

// 添加天空
engine.add(new mapvthree.DynamicSky());

// 创建标签（使用 Label）
const label = engine.add(new mapvthree.Label({
    type: 'icontext',
    mapSrc: 'website_assets/images/speed-panel.png',
    iconWidth: 141,
    iconHeight: 54,
    textSize: 14,
    textFillStyle: '#fff',
    textStrokeStyle: 'rgba(0, 0, 0, 0.8)',
    textStrokeWidth: 2,
    pixelOffset: [0, 40],
    keepSize: true,
    depthTest: false,
    transparent: true,
}));

// 创建 Twin
const twin = engine.add(new mapvthree.Twin({
    delay: 1000,
    keepSize: true,
    modelConfig: {
        2: mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.CAR,
        3: mapvthree.Twin.REALISTIC_TEMPLATE_MODEL.TRUCK,
    },
    objects: [label],
    objectAttributes: { 'text': 'speed' },
}));

twin.receiveRaycast = true;
twin.addEventListener('click', e => {
    console.log('点击车辆:', e.clickInfo);
});

// 配置数据处理
twin.dataProvider
    .process('time', 'timestamp')
    .process('point', item => [item.x, item.y, item.z])
    .process('dir', item => (90 - item.heading) / 180 * Math.PI)
    .process('modelType', item => item.type < 5 ? 2 : 3)
    .process('color', item => mapvthree.Twin.REALISTIC_TEMPLATE_COLOR.white)
    .process('speed', item => Math.round(item.speed));

// WebSocket 实时数据推送
const socket = io('ws://your-server:8899/');
socket.on('vehicle_data', e => {
    const data = JSON.parse(e.tracks);
    data.forEach(item => { item.timestamp = e.timestamp; });
    twin.push(data);
});
```

## 资源清理

```javascript
twin.reset();       // 重置数据状态
twin.modelClear();  // 清除模型
twin.objectsClear();// 清除附加物体
twin.dispose();     // 完全释放
engine.remove(twin);
```
