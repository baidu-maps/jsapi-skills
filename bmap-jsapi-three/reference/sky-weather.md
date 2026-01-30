# 天空和天气系统使用指南

## 概述

MapV Three 提供完整的天空和天气模拟系统，从基础静态天空到动态大气渲染效果。

## 天空类型选择

| 类型 | 性能 | 效果 | 适用场景 |
|------|------|------|----------|
| `EmptySky` | 最佳 | 仅光照 | 不需要天空背景 |
| `DefaultSky` | 高 | 简单渐变 | 移动端/低性能设备 |
| `StaticSky` | 中 | 静态贴图 | 预设天气效果 |
| `DynamicSky` | 低 | 动态云层 | 桌面应用 |

## 时间系统

所有天空类都支持时间控制：

```javascript
// 时间以秒为单位，从午夜开始
sky.time = 3600 * 14.5;  // 下午2:30
```

## 天气预设

- `clear` - 晴天
- `partlyCloudy` - 局部多云
- `cloudy` - 多云
- `overcast` - 阴天
- `foggy` - 雾天
- `rainy` - 雨天
- `snowy` - 雪天
- `stormy` - 暴雨天
- `thunderstorm` - 雷阵雨

## EmptySky - 空白天空

仅提供光照系统，无天空背景：

```javascript
const sky = engine.add(new mapvthree.EmptySky({ time: 3600 * 10 }));
sky.sunLightIntensity = 2.5;
sky.skyLightIntensity = 1.2;
```

## DefaultSky - 渐变天空

简单渐变效果，性能最佳：

```javascript
const sky = engine.add(new mapvthree.DefaultSky());
sky.color = new THREE.Color(0x87ceeb);
sky.highColor = new THREE.Color(0x67ceeb);
```

## StaticSky - 静态天空

预设天气和时间效果：

```javascript
const sky = engine.add(new mapvthree.StaticSky());
sky.time = 3600 * 17;
sky.onWeatherChanged('cloudy');
```

## DynamicSky - 动态天空

动态大气效果和云层：

```javascript
const sky = engine.add(new mapvthree.DynamicSky());
sky.affectWorld = true;
sky.enableAtmospherePass = true;
sky.enableCloudsPass = true;
sky.cloudsCoverage = 0.6;
sky.cloudsSpeed = 1.0;
```

### DynamicSky 配置参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `affectWorld` | boolean | true | 影响场景环境反射 |
| `enableAtmospherePass` | boolean | true | 大气层后处理 |
| `enableCloudsPass` | boolean | true | 体积云渲染 |
| `cloudsCoverage` | number | 0.05-0.8 | 云层覆盖率 |
| `cloudsSpeed` | number | 1.0 | 云层移动速度 |

## DynamicWeather - 天气系统

配合 DynamicSky 使用：

```javascript
const sky = engine.add(new mapvthree.DynamicSky());
const weather = engine.add(new mapvthree.DynamicWeather(sky));

weather.transitionDuration = 3000;  // 过渡时间
weather.weather = 'rainy';

// 监听天气变化
weather.addWeatherChangedListener((type) => {
    console.log('天气变为:', type);
});
```

## 完整示例

```javascript
import * as mapvthree from '@baidumap/mapv-three';

const engine = new mapvthree.Engine(container, {
    rendering: { enableAnimationLoop: true },
    map: { projection: 'ecef' }
});

// 添加动态天空
const sky = engine.add(new mapvthree.DynamicSky());
sky.enableAtmospherePass = true;
sky.enableCloudsPass = true;
sky.cloudsCoverage = 0.3;

// 设置时间
engine.clock.currentTime = new Date('2024-06-21T16:00:00');

// 添加天气系统
const weather = engine.add(new mapvthree.DynamicWeather(sky));
weather.weather = 'partlyCloudy';

// 定时切换天气
const weathers = ['clear', 'partlyCloudy', 'cloudy', 'rainy'];
let index = 0;
setInterval(() => {
    weather.weather = weathers[index];
    index = (index + 1) % weathers.length;
}, 5000);
```

## 性能优化

```javascript
// 根据设备性能调整
if (isMobile) {
    // 移动端使用 DefaultSky
    const sky = engine.add(new mapvthree.DefaultSky());
} else {
    // 桌面端使用 DynamicSky
    const sky = engine.add(new mapvthree.DynamicSky());
}

// 高空时禁用云层
engine.map.addEventListener('zoom_changed', () => {
    const zoom = engine.map.getZoom();
    sky.enableCloudsPass = zoom > 8;
});
```

## 最佳实践

1. **设备适配**：移动端用 `DefaultSky`，桌面用 `DynamicSky`
2. **过渡时间**：天气过渡至少 2 秒
3. **资源清理**：不使用时调用 `dispose()`

## 常见问题

**Q: 天空效果卡顿？**
- 使用较低级别的天空类型（DefaultSky 或 StaticSky）
- 禁用 `enableCloudsPass`

**Q: 环境反射不正确？**
- 确保 `affectWorld: true`
