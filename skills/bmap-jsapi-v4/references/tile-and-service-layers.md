# 瓦片与地图服务图层

## 何时读取

接入自有 BD09MC 瓦片、第三方 XYZ/TMS、WMS、WMTS、实时路况，或排查瓦片 URL、层级、Y 轴、投影与 CORS 问题时读取。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 数据源 | 正式 v4 选择 | 关键约束 |
|---|---|---|
| 自有 BD09MC 瓦片 | `BMap.TileLayer` | 覆盖 `getTilesUrl` 拼 URL；`tileUrlTemplate` 的 X/Y 语义与实际替换相反 |
| 第三方 XYZ/TMS 栅格 | `BMap.RasterTileLayer` | 默认 `EPSG:3857`；TMS 用 `{-y}` 或 `{reverseY}` |
| WMS GetMap | `BMap.WMSLayer` | 当前实现只接受 `EPSG:3857`，`LAYERS` 由业务提供 |
| WMTS GetTile | `BMap.WMTSLayer` | 可配置 TileMatrixSet 与 TileMatrix/TileCol/TileRow 模板 |
| 百度实时路况 | `BMap.TrafficLayer` | 用统一 Layer API 挂载和移除 |

JSAPI 4.0 还公开 `BMap.XYZLayer` 与 `BMap.PanoramaCoverageLayer`，但本页不根据类名推测它们的配置。标准 XYZ/TMS 使用接口更明确的 `BMap.RasterTileLayer`；地图上的全景覆盖范围优先使用 `BMap.PanoramaCoverageLayer`，并以在线影像覆盖结果为准。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const center = new BMap.Point(116.404, 39.915);
map.centerAndZoom(center, 12);

const rasterLayer = new BMap.RasterTileLayer({
  url: 'https://tiles.example.com/{z}/{x}/{y}.png',
  projection: 'EPSG:3857',
  minZoom: 3,
  maxZoom: 18,
  opacity: 0.85,
});
map.addLayer(rasterLayer);

function dispose() {
  map.removeLayer(rasterLayer);
  map.destroy();
}
```

示例域名只用于展示 URL 结构，不代表真实瓦片服务

自有 Tile、WMS、WMTS 与路况的构造和移除可以共用同一所有权闭环。下面三个 `.example.com` 地址及图层名必须替换为业务服务；WMS/WMTS 的鉴权参数也由业务配置提供：

```javascript
const ownTileLayer = new BMap.TileLayer({ transparentPng: true });
ownTileLayer.getTilesUrl = (tileCoord, zoom) =>
  `https://tiles.example.com/bd09mc/${zoom}/${tileCoord.x}/${tileCoord.y}.png`;
const wmsLayer = new BMap.WMSLayer({
  url: 'https://maps.example.com/geoserver/wms',
  params: {
    LAYERS: 'workspace:business_layer',
    VERSION: '1.1.1',
    FORMAT: 'image/png',
    TRANSPARENT: 'true',
  },
  minZoom: 3,
  maxZoom: 18,
});
const businessWmtsParams = {
  Layer: 'workspace:business_layer',
  Style: '',
  TileMatrixSet: 'EPSG:3857',
  Format: 'image/png',
};
const wmtsLayer = new BMap.WMTSLayer({
  url: 'https://maps.example.com/geoserver/gwc/service/wmts',
  params: { ...businessWmtsParams },
  minZoom: 3,
  maxZoom: 18,
});
// 4.0 分支默认开启自动刷新（60s）；示例用无参构造沿用默认值。
const trafficLayer = new BMap.TrafficLayer();

map.addLayer(ownTileLayer);
map.addLayer(wmsLayer);
map.addLayer(wmtsLayer);
map.addLayer(trafficLayer);

// clearCache() 需要图层已经挂到 Map。
ownTileLayer.setZIndex(5);
console.log(ownTileLayer.isTransparentPng());
ownTileLayer.clearCache();

function removeServiceLayers() {
  map.removeLayer(trafficLayer);
  map.removeLayer(wmtsLayer);
  map.removeLayer(wmsLayer);
  map.removeLayer(ownTileLayer);
}
```

WMTS 构造实现会直接修改传入的 params：补默认字段，并删除 TileMatrix/TileCol/TileRow 模板项。需要复用业务参数时，每次构造都像示例一样传入浅拷贝，不能复用已被构造器消费的同一对象。示例里的 `TileMatrixSet: 'EPSG:3857'` 是按业务服务填的值，不传时的默认值是 `'GoogleMapsCompatible'`，实际取值必须对照服务的 Capabilities 文档。

替换服务地址后，在浏览器 Network 面板逐项确认请求 URL、服务鉴权、响应格式与 CORS；`removeServiceLayers()` 应在最终 `map.destroy()` 之前执行。

## 核心 API

`BMap.TileLayer` 面向已使用 BD09MC 的自定义瓦片。它公开 `getTilesUrl`、`setZIndex`、掩膜和缓存方法，挂载与卸载统一走 Map 的 `addLayer`/`removeLayer`。业务代码覆盖 `getTilesUrl(tileCoord, zoom)`，显式把 `tileCoord.x`、`tileCoord.y` 映射到业务服务的列/行参数。

`isTransparentPng()` 读取透明 PNG 配置；`setZIndex()` 调整层级；`clearCache()` 清理当前瓦片与图标缓存。其中 `clearCache()` 与掩膜方法必须在图层挂载之后调用，所以整体按“构造 → `map.addLayer` → 动态调整”的顺序最稳妥。`clearCache()` 只用于确实需要重新取瓦片的场景，不是隐藏或卸载图层的替代品。

TileLayer 掩膜接收 Boundary 服务返回的坐标点串或字符串数组。不要直接把行政区名称传给 `addBoundary()`：它只把每个字符串交给 Polygon 构造，不会去查询行政区边界。需要行政区边界先用 [Boundary](administrative-district.md) 查坐标串。

在 4.0 下构造 `BMap.Boundary` 会在控制台打印弃用告警，提示改用 `BMap.DistrictLayer`；下面示例仍用 Boundary 取轮廓串，是因为掩膜需要的是坐标字符串，看到这条告警属于预期行为。

```javascript
const boundaryService = new BMap.Boundary();
let boundaryRequestActive = true;

boundaryService.get('北京市', (result) => {
  if (!boundaryRequestActive || !result) return;
  ownTileLayer.clearBoundary();
  ownTileLayer.addBoundary(result.boundaries);
});

function clearTileBoundary() {
  boundaryRequestActive = false;
  ownTileLayer.clearBoundary();
}
```

`setZIndexTop()` 只有在图层挂到地图之后才存在，构造后立刻调用会抛 TypeError。必须在 `map.addLayer()` 之后调用，并在浏览器里确认实际层级。

`tileUrlTemplate` 的 X/Y 语义与直觉相反：4.0 实际把 `{X}` 替换为行、`{Y}` 替换为列。不要依赖直觉；优先使用上面的 `getTilesUrl` 回调，并按实际服务请求验证行列方向。

TileLayer 的 opts 可省略：构造器开头就做了 `this.opts = opts || {}`，末尾读的是归一化后的 `this.opts.tileLoadFunction`，`new BMap.TileLayer()` 不会报错。示例写 `new BMap.TileLayer({})` 只是为了显式表达这里可以传配置。

`BMap.TrafficLayer` 继承 TileLayer，并提供 `setColors`、`setEdge`（路况配色没有对外常量，`BMAP_TRAFFICE_STATUS_*` 那组常量属于驾车路线结果，不是图层配色）。4.0 分支会按 `autoRefresh` / `refreshInterval` 创建自动刷新 timer：`autoRefresh` 默认 `true`、`refreshInterval` 默认 60000，且 `refreshInterval` 会被夹紧到不小于 10000 毫秒。

**TrafficLayer 天生是页面级单实例**：它的原型就是一个已经构造好的图层对象本身，因此 `map`、瓦片缓存、`zIndex`、刷新 timer 等状态在所有 `new BMap.TrafficLayer()` 之间共享，构造参数写的是这份共享状态。示例使用无参构造并只创建一个实例；Map 继续存活而不再展示路况时，保存实例并调用 `map.removeLayer(trafficLayer)`。

`setAutoRefresh()` 与 `setRefreshInterval()` 可以在挂载后修改刷新策略，但它们改的是上面那份共享状态——一处调用会影响页面上所有 TrafficLayer 用法。正式代码优先在构造时确定刷新配置。

`BMap.RasterTileLayer` 是第三方 XYZ/TMS 的公开入口。字符串模板支持 `{x}`、`{y}`、`{z}`、`{s}`、`{-y}`、`{reverseY}`（替换时大小写不敏感），也可传 `(x, y, z) => string`。`projection` 只写 `EPSG:3857` 或 `BD09MC`，其他值会告警并回退到 `EPSG:3857`。地图显示范围是 `minZoom..maxZoom`；实际请求的数据层级是 `floor(mapZoom) + spanLevel`，`spanLevel` 只负责请求 z 的偏移。

`BMap.WMSLayer` 自动组装 `SERVICE=WMS`、`REQUEST=GetMap`、版本（默认 `1.3.0`）、格式以及 BBOX/WIDTH/HEIGHT；WMS 1.1.1 使用 SRS，其余版本使用 CRS。它把用户参数合并到一份新对象上，因此**不会**改动业务传入的 params；但 BBOX、WIDTH、HEIGHT 由内部占位符驱动，业务传了也会被静默丢弃（只有缺少 `LAYERS` 才会 `console.warn`）。`BMap.WMTSLayer` 自动补充 GetTile 默认参数，并把 TileMatrix、TileCol、TileRow 拼入模板；它直接使用传入的 params 对象，这个过程会原地补字段并删除模板项，所以复用同一份配置时传浅拷贝。两者都交给 Map 的统一 `addLayer`/`removeLayer` 管理。

`startLevel`：不存在这个选项。层级控制使用 `minZoom`、`maxZoom`，需要请求层级偏移时使用 `spanLevel`。

## 事件或回调

TileLayer 实例会派发图层级 `tilesloadstart` / `tilesloadend`，可用于观察一轮瓦片加载的开始和结束；Map 会派发单瓦片的 `tileloaded` / `tileloaderror`。需要动态 URL 时使用 `BMap.RasterTileLayer` 的 `url` 回调，它接收 x、y、z 并返回 URL。不要把这些事件写成泛化的 `load`、`tileload` 或 `error`。

网络失败应在浏览器 Network 面板观察实际瓦片请求。CORS 是浏览器对跨源响应的约束，不是各图层类的开关；若图片、Canvas 或自定义加载链需要跨域许可，应由瓦片服务正确返回响应头，并在目标浏览器真实验证。

## 资源清理

- 保存每个图层实例，以便传给 `map.removeLayer(layer)`。
- 先停止业务自己创建的轮询、AbortController 或请求队列，再移除图层。
- 使用 Boundary 异步设置掩膜时，先让 active token 失效并调用 `clearBoundary()`，防止迟到回调修改已卸载图层。
- TrafficLayer 作为页面级共享配置只创建一个；不再展示路况时显式调用 `map.removeLayer(trafficLayer)`。4.0 分支的自动刷新 timer 有两条清理路径：`removeLayer` 触发的图层 `remove()` 会清掉它，`map.destroy()` 里也有一次兜底 `clearInterval`。仍然建议显式 `removeLayer`，因为只有它会同时解除图层与 Map 的关联。
- 只在图层仍挂载且确实需要强制重载时调用 `clearCache()`；它不是移除的替代品。
- 最后调用 `map.destroy()` 释放 WebGL 与地图资源。

## 常见错误

按以下顺序排查空白、错位或重复瓦片：

1. **请求 URL 可访问性**：从 Network 复制一个实际 URL，确认状态码、鉴权参数、Content-Type 和返回内容。
2. **层级**：可用的层级选项是 `minZoom`、`maxZoom` 和 `spanLevel`，没有 `startLevel`。确认地图 zoom 落在 `minZoom..maxZoom`，再确认服务收到的请求 z 等于 `floor(mapZoom) + spanLevel`。
3. **`{y}` 与反向 Y**：XYZ 使用 `{y}`，TMS 使用 `{-y}` 或 `{reverseY}`；公式为 `2^z - y - 1`。
4. **数据坐标系**：第三方标准瓦片通常从 `EPSG:3857` 接入；已经是百度墨卡托的数据才选 `BD09MC`。
5. **CORS**：请求成功不等于浏览器允许后续读取或绘制；检查响应头和控制台安全错误。
6. **样式/透明度**：检查 `opacity`、`zIndex`、掩膜、服务端透明背景和图层覆盖顺序。

其他高频错误：

- 把 `BMap.XYZLayer` 当成 `RasterTileLayer` 的同义名称；标准 XYZ/TMS 优先使用后者。
- 把 `'北京市'` 直接传给 `addBoundary()`，假定图层会自行查询行政区坐标。
- 在 `map.addLayer()` 前调用 `clearCache()`、掩膜方法或 `setZIndexTop()`，它们都要等挂载后才能工作。
- 给 TileLayer 写 RasterTileLayer 的小写占位符：TileLayer 只认 `{X}`、`{Y}`、`{Z}`（区分大小写，且每个占位符只替换第一次出现），RasterTileLayer 的替换才是大小写不敏感的。
- 只凭 TileLayer 模板字母认定 X/Y 分别是横向/纵向坐标，忽略当前实现的相反替换。
- WMS 手工重复传 BBOX/WIDTH/HEIGHT（会被静默丢弃），或遗漏业务必需的 LAYERS。
- 创建多个 TrafficLayer 并假定 autoRefresh、zIndex、瓦片缓存等状态彼此独立：它们共享同一份状态。
- 复用同一个 WMTS params 对象创建多个图层，忽略构造器会原地补字段和删除模板项。
- 只隐藏/清缓存，不调用 `map.removeLayer` 和 `map.destroy`。
