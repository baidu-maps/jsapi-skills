# Tracker 追踪器使用指南

## 概述

MapV Three 提供四种追踪器用于实现路径动画、轨道运动和对象跟踪效果。

## 追踪器类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| `PathTracker` | 沿路径追踪 | 车辆行驶、轨迹回放 |
| `OrbitTracker` | 轨道追踪 | 环绕动画、建筑展示 |
| `HorizontalOrbitTracker` | 水平旋转追踪 | 水平面仰角环绕 |
| `ObjectTracker` | 对象追踪 | 跟随目标、视角锁定 |

## 通用特性

### 生命周期控制

```javascript
tracker.start(options);  // 开始动画
tracker.pause();         // 暂停，返回当前状态
tracker.stop();          // 停止，触发 onFinish
```

### 状态查询

| 属性 | 类型 | 说明 |
|------|------|------|
| `isRunning` | boolean | 是否运行中 |
| `isPaused` | boolean | 是否暂停 |
| `currentState` | object | 当前状态：`{point, hpr, direction}` |

### 回调函数

```javascript
tracker.onStart = () => console.log('开始');
tracker.onFinish = () => console.log('结束');
tracker.onUpdate = (state) => {
    // state: {point: [lng,lat,alt], hpr: {heading,pitch,roll}, direction}
    console.log(state.point);
};
```

## PathTracker - 路径追踪

```javascript
const tracker = engine.add(new mapvthree.PathTracker());

tracker.track = [
    [116.368264, 39.176959, 38],
    [116.370264, 39.178959, 40],
    [116.372264, 39.180959, 42],
];

tracker.start({
    duration: 10000,
    pitch: 60,
    range: 100
});
```

### 路径数据格式

```javascript
// 坐标数组
tracker.track = [[lng1, lat1, alt1], [lng2, lat2, alt2]];

// GeoJSON
tracker.track = {
    type: 'Feature',
    geometry: { type: 'LineString', coordinates: [...] }
};

// 帧信息（支持关键帧模式）
tracker.track = [
    { x: 116.368, y: 39.177, z: 38, pitch: 0, yaw: 0, speed: 50, time: 0 }
];
```

### 让模型沿路径移动

```javascript
const vehicle = engine.add(new mapvthree.SimpleModel({ url: 'car.glb' }));

const tracker = engine.add(new mapvthree.PathTracker());
tracker.track = routeCoordinates;
tracker.object = vehicle;

tracker.start({ duration: 30000, range: 200, pitch: 45 });
```

### 视图模式

| 值 | 说明 |
|------|------|
| `'follow'` | 跟随不锁定方向 |
| `'lock'` | 锁定路径切线方向 |
| `'unlock'` | 相机不跟随 |
| `'keyFrame'` | 关键帧模式 |
| `'activeFrame'` | 活跃帧模式（支持 speed） |

### 轨迹插值

```javascript
tracker.pointHandle = 'curve';              // 平滑曲线插值
tracker.interpolateDirectThreshold = 30;    // 插值直接阈值
tracker.interpolateDirectThresholdPercent = 0.1;  // 插值百分比
```

## OrbitTracker - 环绕追踪

```javascript
const tracker = engine.add(new mapvthree.OrbitTracker());

tracker.start({
    center: [116.404, 39.915, 0],
    radius: 200,
    duration: 15000,
    startAngle: 0,
    endAngle: 360,
    keepRunning: true
});
```

### 围绕对象旋转

```javascript
const model = engine.add(new mapvthree.SimpleModel({ url: 'building.glb' }));

tracker.start({
    object: model,
    radius: 100,
    duration: 10000,
    pitch: 30,
    keepRunning: true
});
```

### 配置选项

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `center` | Array | - | 地理坐标 [lng, lat, alt] |
| `projectedCenter` | Array | - | 投影坐标 [x, y, z] |
| `object` | Object3D | - | 围绕的对象（与 center 二选一） |
| `radius` | number | `100` | 旋转半径（米） |
| `height` | number | `0` | 高度偏移（米） |
| `startAngle` | number | `0` | 起始角度（度） |
| `endAngle` | number | `360` | 结束角度（度） |
| `duration` | number | `10000` | 持续时间（毫秒） |
| `keepRunning` | boolean | `true` | 持续运行 |
| `loopMode` | string | `'repeat'` | 循环模式 |
| `direction` | string | `'normal'` | 动画方向 |
| `useWorldAxis` | boolean | `false` | 使用世界轴 |

**loopMode 可选值：** `'repeat'`, `'reverse'`, `'alternate'`

**direction 可选值：** `'normal'`, `'reverse'`, `'alternate'`, `'alternate-reverse'`

## HorizontalOrbitTracker - 水平旋转追踪

提供更直观的水平面仰角参数：

```javascript
const tracker = engine.add(new mapvthree.HorizontalOrbitTracker());

tracker.start({
    center: [116.404, 39.915, 0],
    range: 200,    // 观察点到目标的空间距离
    angle: 30,     // 相对水平面的仰角（度）
    duration: 10000
});
```

## ObjectTracker - 对象追踪

```javascript
const vehicle = engine.add(new mapvthree.SimpleModel({ url: 'car.glb' }));
const tracker = engine.add(new mapvthree.ObjectTracker());

tracker.track(vehicle, {
    range: 100,
    pitch: 60,
    heading: 0,
    lock: true
});
```

### 支持的对象类型

- `Object3D` - Three.js 对象
- `Vector3` - 三维向量
- `[x, y, z]` - 坐标数组
- `instanced entity` - 实例化对象

### 配置选项

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `lock` | boolean | `true` | 锁定视角 |
| `range` / `radius` | number | - | 观察距离（米） |
| `pitch` | number | `0` | 俯仰角（度） |
| `heading` | number | `0` | 方位角（度） |
| `height` | number | - | 高度偏移 |
| `extraDir` | number | - | 方向修正角度 |
| `duration` | number | `0` | 持续时间（0=持续追踪） |
| `easing` | string | `'linear'` | 缓动函数 |

### 特有回调

```javascript
tracker.onTrackFrame = (lastState, currentState) => {
    // lastState: 上一帧状态（首帧为 null）
    // currentState: {point: [lng,lat,alt], hpr: {heading,pitch,roll}}
};
```

### 实时追踪移动对象

```javascript
const car = engine.add(new mapvthree.SimpleModel({ url: 'car.glb' }));
const tracker = engine.add(new mapvthree.ObjectTracker());

tracker.track(car, { range: 80, pitch: 50, lock: true });

// 模拟车辆移动
setInterval(() => {
    car.position.fromArray(
        engine.map.projectArrayCoordinate(route[index++])
    );
}, 100);
```

## 通用 start() 选项

| 选项 | 类型 | 默认值 | 说明 |
|-----|------|-------|------|
| `duration` | number | `1000` | 持续时间（毫秒）|
| `heading` | number | `0` | 方位角（度）|
| `pitch` | number | `60` | 俯仰角（度）|
| `range` | number | `100` | 距离范围（米）|
| `easing` | string/function | `'linear'` | 缓动函数 |
| `repeatCount` | number | `1` | 重复次数 |
| `delay` | number | `0` | 延迟启动（毫秒）|
| `keepRunning` | boolean | `false` | 持续运行 |
| `direction` | string | `'normal'` | 动画方向 |

## 缓动函数

**内置缓动：**
- `'linear'` - 线性
- `'ease-in'` - 缓入（t²）
- `'ease-out'` - 缓出
- `'ease-in-out'` - 缓入缓出

**自定义函数：**
```javascript
tracker.start({
    easing: (t) => t * t * t,  // 自定义三次方缓动
    duration: 5000
});
```

## 组合使用

```javascript
// 先环绕展示，再沿路径漫游
const orbitTracker = engine.add(new mapvthree.OrbitTracker());
const pathTracker = engine.add(new mapvthree.PathTracker());

orbitTracker.onFinish = () => {
    pathTracker.track = routeData;
    pathTracker.start({ duration: 30000, pitch: 60 });
};

orbitTracker.start({
    center: [116.404, 39.915, 0],
    radius: 200,
    duration: 10000,
    repeatCount: 2
});
```

## 资源清理

```javascript
tracker.stop();
engine.remove(tracker);
```

## 常见问题

**Q: 路径不平滑？**
```javascript
tracker.pointHandle = 'curve';
tracker.interpolateDirectThreshold = 30;
```

**Q: 如何实现倒放？**
```javascript
tracker.start({ direction: 'reverse', ...options });
```

**Q: 如何平滑过渡到追踪状态？**
```javascript
engine.map.flyTo({ center: target, duration: 2000 })
    .then(() => tracker.start(options));
```

**Q: OrbitTracker 与 HorizontalOrbitTracker 区别？**
- `OrbitTracker`：使用 `radius` + `height` 控制位置
- `HorizontalOrbitTracker`：使用 `range` + `angle`（仰角）控制，更直观
