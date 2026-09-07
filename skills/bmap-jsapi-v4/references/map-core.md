# Map 核心能力

## 何时读取

创建 `BMap.Map`、初始化或调整视野、切换旋转/倾斜、设置个性化底图样式，或销毁地图时读取。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

- 单点初始化：`centerAndZoom(point, zoom)`。
- 只改中心或级别：`setCenter` / `setZoom`；按坐标或像素平移：`panTo` / `panBy`。
- 读取当前视野：`getCenter` / `getZoom` / `getBounds`。
- 多点自动入镜：`setViewport(points, options)`；只计算最佳视野用 `getViewport`。
- 交互与范围：优先在构造参数或 `setOptions` 中配置交互，用 `setMinZoom` / `setMaxZoom` 约束级别，用 `restrictBounds` 限制可视范围。
- 底图类型：用 `setMapType(BMAP_NORMAL_MAP)` 等公开常量切换。
- 3D 视角：`setHeading` 控制朝向，`setTilt` 控制倾角。
- 个性化底图：`setMapStyle({ styleId })` 或 `setMapStyle({ styleJson })`。
- 控件与面板配色：`setTheme('dark')`，见 [UI 版本与主题](ui-version-and-theme.md)。
- 容器恢复显示或布局改变后尺寸不对：调用 `checkResize()`，见 [性能与排障](performance-and-troubleshooting.md)。
- 页面卸载：先拆业务资源，再调用 `destroy()`。

## 最小可运行示例

```javascript
const beijing = new BMap.Point(116.404, 39.915);
const tianjin = new BMap.Point(117.2, 39.084);
const map = new BMap.Map('map', {
  center: beijing,
  zoom: 12,
  minZoom: 5,
  maxZoom: 18,
  enableWheelZoom: true,
});

map.setMapType(BMAP_NORMAL_MAP);
map.setViewport([beijing, tianjin], {
  margins: [40, 40, 40, 40],
  enableAnimation: false,
});
map.setHeading(30, { noAnimation: true });
map.setTilt(45, { noAnimation: true });
map.setMapStyle({ styleId: 'YOUR_STYLE_ID' });

const onMoveEnd = () => {
  console.log(map.getCenter(), map.getZoom());
};
map.addEventListener('moveend', onMoveEnd);

function disposeMap() {
  map.removeEventListener('moveend', onMoveEnd);
  map.destroy();
}
```

此 JavaScript 片段假设页面已经按 [项目接入](project-setup.md) 加载 JSAPI，且存在非零尺寸的 `#map` 容器。4.0 默认已开启滚轮缩放，示例只是演示可在构造参数中显式表达业务配置；运行期间需要修改时可调用 `map.setOptions({ enableWheelZoom: false })`。

## 核心 API

| 目的 | API | 关键约束 |
|---|---|---|
| 构造地图 | `new BMap.Map(idOrElement, options?)` | 4.0 支持在 options 里直接给 `center` / `zoom`；未指定时会落到默认视野 |
| 单点/城市视野 | `centerAndZoom(center, zoom, options?)` | `center` 支持 `Point` 或城市名重载 |
| 中心点 | `setCenter(center, options?)` / `getCenter()` | 城市名重载依赖在线地理编码；确定性初始化优先传 `Point` |
| 缩放级别 | `setZoom(zoom, options?)` / `getZoom()` | 相对缩放可基于当前 `getZoom()` 计算后再设置 |
| 平移 | `panTo(point, options?)` / `panBy(x, y, options?)` | 前者移动到目标坐标，后者按屏幕像素平移 |
| 当前范围 | `getBounds()` | 返回当前视野的地理范围 |
| 多点入镜 | `setViewport(pointsOrViewport, options?)` / `getViewport(pointsOrBounds, options?)` | `setViewport` 修改地图；`getViewport` 只计算最佳中心和级别 |
| 朝向 | `setHeading(degrees, options?)` / `getHeading()` / `resetHeading(options?)` | 入参会按 360° 归一化 |
| 倾斜 | `setTilt(degrees, options?)` | 建议按 0–73 使用；运行时的真实截断值更宽，见下文 |
| 缩放范围 | `setMinZoom` / `setMaxZoom` | 未显式设置时，WebGL 渲染下上限为 21，下限受自适应最小级别影响；先设置范围，再初始化或调整视野 |
| 交互 | 构造参数 / `setOptions(options)` | 新代码优先集中配置；旧的成对 `enable*` / `disable*` 方法仅用于兼容 |
| 可视范围 | `restrictBounds(bounds)` | 限制地图可浏览范围，并可能抬高实际最小级别 |
| 底图类型 | `setMapType(mapType)` / `getMapType()` | 使用公开 `BMAP_*_MAP` 常量 |
| 样式 | `setMapStyle(config)` | `styleId` 与 `styleJson` 二选一为常见用法 |
| 底图显示项 | `setDisplayOptions(options)` | 控制 POI、覆盖物、图层、建筑、道路、室内与天空等显示项 |
| 尺寸重算 | `checkResize()` | 隐藏容器恢复显示或自动适配关闭后手动同步尺寸 |
| 释放 | `destroy()` | 最后调用 |

倾斜角有三个互不相同的限值，写业务代码时别把它们混成一个数：

- `setTilt()` 的实现会把输入截断到不高于 87；实际交互可达到的最大倾角仍受当前地图状态限制，应通过 `getCurrentMaxTilt()` 判断。
- 用户拖拽/手势的上限是动态的，由 `getCurrentMaxTilt()` 给出：常规情况最高 73，地球底图下只有 50，缩放级别较低时会一路降到 0。要判断当前能倾斜多少，读 `getCurrentMaxTilt()`，不要写死 73。
- 构造时的 `options.tilt` 被硬钳制在 0–73。

结论：业务上按 0–73 设计，需要精确边界时用 `getCurrentMaxTilt()` 查询。倾斜与旋转只在 WebGL 渲染下有效。

缩放级别范围不由版本决定。`minZoom` / `maxZoom` 默认不设限，未显式设置时 WebGL 渲染下的上限是 21；下限名义上是 3，但容器自适应最小级别默认开启，会把下限抬高到「世界刚好铺满容器」，所以实际最小级别经常大于 3。需要确定范围就显式调用 `setMinZoom` / `setMaxZoom`，并用 `getMinZoom()` / `getMaxZoom()` 读回实际生效值。

## 4.0 的默认行为

写业务代码前先按这张表确认默认值，避免为了“打开”一个本来就开着的能力而多写调用：

| 配置 | 4.0 默认值 | 说明 |
|---|---|---|
| `options.center` / `options.zoom` | 未指定时落到默认视野 | 构造时即可设定视野，另支持 `options.heading` / `options.tilt` |
| `centerAndZoom` | 非必需 | 未调用时展示默认视野，Map 构造后即处于已就绪状态 |
| 滚轮缩放 | 开启 | 不需要调用 `enableScrollWheelZoom()`；要禁用才调 `disableScrollWheelZoom()` |
| 拖拽、惯性拖拽 | 开启 | 旋转、倾斜及其手势同样默认开启 |
| 双击缩放、连续缩放 | 开启 | 左键双击放大、右键双击缩小 |
| 键盘操作 | 关闭 | 需要时调用 `enableKeyboard()` |
| 容器自适应 | 开启 | 尺寸变化自动 resize；缩放时是否保持中心点默认关闭 |
| 缩放级别范围 | 未设限 | `minZoom` / `maxZoom` 默认不设限；WebGL 渲染下上限取 21，下限受自适应最小级别影响，常大于 3 |
| 自适应最小级别 | 开启 | 会把实际可用的最小级别抬高到「世界刚好铺满容器」 |
| `firsttilesloaded` / `firsttileloaded` | 每个地图实例只派发一次 | 两个事件名互为别名，成对派发 |
| `displayOptions.indoor` | 关闭 | 开启后会自动展示楼层选择控件 |
| `enableIconClick` | 关闭 | 开启后允许点击底图标注；点击信息从 Map 的 `click` 事件参数读取 |
| `enableIconInfoWindow` | 关闭 | 置 `true` 会强制把 `enableIconClick` 也打开 |
| `enableIconHighlight` | 开启 | 关闭后只保留指针变化与选中标注常驻 |
| 默认控件 | 不自动显示 | 控件按 [控件与右键菜单](controls-and-context-menu.md) 显式 `addControl` |

“未调用 `centerAndZoom` 时展示默认视野”指的是北京天安门（`116.404, 39.915`）附近、zoom 12。这是 4.0 有意提供的兜底行为，不是初始化失败；但正式项目仍应显式设置业务视野，不要依赖这个默认值。

这些是 `apiVersion=4.0` 的当前默认行为；UI 样式与主题相关的默认值见 [UI 版本与主题](ui-version-and-theme.md)。

图层的添加与移除在 4.0 统一为 `map.addLayer(layer)` / `map.removeLayer(layer)`。`addTileLayer`、`removeTileLayer`、`addGeoJSONLayer`、`addDistrictLayer`、`addCustomHtmlLayer`、`addNormalLayer` 及各自的 remove 版本仍然可用，但会打印过时提示。个性化样式统一使用 `setMapStyle()`；`setMapStyleV2()` 仍由实现保留，但新代码无需绕过统一入口。

新代码优先通过构造参数一次性表达交互配置，运行期间统一用 `setOptions()` 修改，例如 `map.setOptions({ enableDragging: false, enableWheelZoom: false })`。`enable/disableDragging`、`enable/disableScrollWheelZoom`、`enable/disableContinuousZoom`、`enable/disableDoubleClickZoom` 与 `enable/disableKeyboard` 等成对方法仍由运行时保留，但在 4.0 源码中已标记为过时，不应作为新代码主线。

## 事件或回调

常用 Map 事件有 `moveend`、`zoomend`、`tilesloaded` 与 `resize`。监听器必须保存函数引用，移除时传入同一个引用。

`centerAndZoom`、`setHeading`、`setTilt` 的 options 可提供 `callback`；回调表达的是本次视野动画结束，不代表在线瓦片、样式或服务数据都已成功。

## 资源清理

```javascript
map.removeEventListener('moveend', onMoveEnd);
map.destroy();
```

如果该 Map 还挂载控件、覆盖物、图层或动画，先按各模块 reference 移除或停止它们，再销毁地图。

## 常见错误

- 只构造 `BMap.Map`，没有显式调用 `centerAndZoom`，导致页面停留在默认的北京 / zoom 12，而不是业务视野。
- 把 `getHeading()` 的带符号返回值误认为一定等于 `setHeading()` 的 0–360 入参。
- 将 `setViewport` 当作纯计算；只想计算时应核对 `getViewport`。
- 把固定数字当成当前视野一定可达的最大倾角，而不读取 `getCurrentMaxTilt()`。
- 把「WebGL 最小级别是 3」当成恒定值，忽略默认开启的自适应最小级别会把它抬高。
- 在 4.0 项目里为了“打开滚轮缩放”而调用 `enableScrollWheelZoom()`，却没意识到它本来就是开启的；真正需要的往往是 `disableScrollWheelZoom()`。
- 组件卸载后仍保留监听器或回调并继续访问已销毁地图。
