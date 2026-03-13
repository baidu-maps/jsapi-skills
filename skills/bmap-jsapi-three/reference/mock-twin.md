# MockTwin 模拟车流

## 概述

MockTwin 是基于 Twin 的模拟车流生成器，通过线路数据自动生成沿路径行驶的车辆，支持车型/颜色概率配置、速度和间距动态调整。适用于快速搭建车流演示、压力测试和场景预览。

## 基础用法

```javascript
// 准备路线数据（地理坐标数组）
const routeCoordinates = [[116.404, 39.915], [116.405, 39.916], ...];

// 创建模拟车流
const mockTwin = engine.add(new mapvthree.MockTwin(
    {
        delay: 1000,
        modelConfig: mapvthree.Twin.SERVICE_TEMPLATE_MODEL,
    },
    routeCoordinates,
    {
        color: {
            'red': 0.1,
            'blue': 0.3,
            'yellow': 0.3,
            'pink': 0.2,
            '#333': 0.1,
        },
        modelType: {
            1: 0.1,
            2: 0.3,
            3: 0.3,
            4: 0.2,
            5: 0.1,
        },
    }
));

// 启动模拟
mockTwin.start({
    gap: 10,           // 车辆间距（米）
    speed: 80,         // 行驶速度（km/h）
    initialCount: 500, // 初始车辆数
});
```

## 构造参数

```javascript
new mapvthree.MockTwin(twinConfig, routeData, probabilityConfig)
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `twinConfig` | object | Twin 的配置选项（参考 `twin.md` 构造参数），默认 `{}` |
| `routeData` | Array | **必需**。路线坐标数组 `[[x,y], ...]` |
| `probabilityConfig` | object | 概率配置，包含 `color` 和 `modelType` 子对象，默认 `{}`（随机选择） |

**probabilityConfig 格式：**

```javascript
{
    color: {
        'red': 0.3,      // 颜色名称: 出现概率
        'blue': 0.5,
        '#333': 0.2,
    },
    modelType: {
        1: 0.2,          // 模型类型编号: 出现概率
        2: 0.5,
        3: 0.3,
    },
}
```

> **说明**：颜色名称会通过 `Twin.REALISTIC_TEMPLATE_COLOR` 映射为实际颜色值；也可直接使用十六进制颜色值（如 `'#333'`）。概率值之和建议为 1。

## 方法

| 方法 | 说明 |
|------|------|
| `start(options)` | 启动模拟车流 |
| `clear()` | 清除所有模拟车辆并停止定时器 |
| `dispose()` | 释放所有资源（调用 clear + 释放内部 Twin） |

### start(options)

| 参数 | 类型 | 说明 |
|------|------|------|
| `gap` | number | **必需**。车辆间距（米） |
| `speed` | number | **必需**。行驶速度（km/h） |
| `initialCount` | number | 初始车辆数，不传或设为 0 则车辆按间隔时间逐辆生成 |

> **注意**：`gap` 和 `speed` 没有默认值保护，不传递会导致计算异常。

## 属性

```javascript
mockTwin.speed = 60;                           // 动态修改速度（km/h）
mockTwin.gap = 5;                              // 动态修改间距（米）
mockTwin.probabilityConfig.color.red = 0.5;    // 修改颜色概率
mockTwin.probabilityConfig.modelType[2] = 0.6; // 修改车型概率

// 读取当前值
console.log(mockTwin.speed);                   // 60
console.log(mockTwin.gap);                     // 5
console.log(mockTwin.data);                    // 当前路线数据
```

> **注意**：`data` 属性的 setter 当前仅存储值，**不会重新初始化路线**。如需更换路线，应销毁当前实例并创建新的 MockTwin。

### visibleCallback

通过覆写 `visibleCallback` 方法控制模拟车流的显隐：

```javascript
mockTwin.visibleCallback = () => {
    return engine.map.getZoom() > 18;
};
```

> **已知限制**：源码中存在 `onBeforeSceneRender` 方法重复定义，可能导致 `visibleCallback` 不被自动调用。建议同时使用 `mockTwin.visible` 属性手动控制显隐。

## 完整示例

```javascript
const engine = new mapvthree.Engine(document.getElementById('map_container'), {
    rendering: { enableAnimationLoop: true },
});
engine.map.setCenter([116.404, 39.915]);
engine.map.setZoom(19.5);

// 加载路线数据
const response = await fetch('data/json/routes.json');
const routeData = await response.json();
const coordinates = routeData.map(v => engine.map.unprojectArrayCoordinate(v));

// 可视化路线（可选）
const line = engine.add(new mapvthree.FatLine());
line.dataSource = mapvthree.GeoJSONDataSource.fromGeoJSON([
    { geometry: { type: 'LineString', coordinates } },
]);

// 创建模拟车流
const mockTwin = engine.add(new mapvthree.MockTwin(
    {
        delay: 1000,
        modelConfig: mapvthree.Twin.SERVICE_TEMPLATE_MODEL,
    },
    coordinates,
    {
        color: { 'red': 0.1, 'blue': 0.3, 'yellow': 0.3, 'pink': 0.2, '#333': 0.1 },
        modelType: { 1: 0.1, 2: 0.3, 3: 0.3, 4: 0.2, 5: 0.1 },
    }
));

// 根据缩放级别自动显隐
mockTwin.visibleCallback = () => engine.map.getZoom() > 18;

// 启动
mockTwin.start({ gap: 10, speed: 80, initialCount: 500 });

// 动态调整
mockTwin.speed = 120;
mockTwin.gap = 5;

// 清除后重新生成
mockTwin.clear();
mockTwin.start({ gap: 5, speed: 60, initialCount: 1000 });
```

## 注意事项

- MockTwin 必须通过 `engine.add()` 添加到场景，内部依赖 `afterAddToEngine` 进行初始化。
- 未提供 `probabilityConfig` 时，颜色从 `Twin.REALISTIC_TEMPLATE_COLOR` 中随机选取，车型从 `modelConfig` 的 key 中随机选取。
- 概率值之和建议为 1；若不等于 1，部分随机值可能无法命中任何选项。
- `clear()` 后可再次调用 `start()` 重启模拟；`dispose()` 后不应再使用该实例。

## 资源清理

```javascript
mockTwin.clear();   // 清除车辆、停止定时器（可再次 start）
mockTwin.dispose(); // 释放所有资源（不可再使用）
engine.remove(mockTwin);
```

> 务必调用 `clear()` 或 `dispose()` 以停止内部定时器，避免内存泄漏。
