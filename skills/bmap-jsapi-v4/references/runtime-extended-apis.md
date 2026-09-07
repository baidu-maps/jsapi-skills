# 扩展 API

## 何时读取

需要使用 `BMap.Heatmap`、`BMap.PointLayer`、`BMap.ClusterLayer`、`BMap.TrackLine`、`BMap.Icons` 等 JSAPI 4.0 扩展能力时读取。

## 本页导航

[扩展可视化](#扩展可视化) · [PointLayer 与 ClusterLayer](#pointlayer-与-clusterlayer) · [TrackLine](#trackline) · [事件](#事件或回调) · [其他扩展名称](#其他扩展名称) · [资源清理](#资源清理) · [常见错误](#常见错误)

类型包对部分扩展 API 的覆盖可能滞后；TypeScript 项目需要额外配置时进入 [TypeScript 集成](typescript-integration.md)。

## 扩展可视化

JSAPI 4.0 提供以下扩展能力：

| API | 用途 | 主要公开操作 |
|---|---|---|
| `BMap.Heatmap` | 点权重热力图 | `setData`、`clearData`、`setOptions`、`setGradient`、`setRadius` |
| `BMap.PointLayer` | 支持形状或图标的 WebGL 点图层 | `setData`、`clearData`、`setOptions`、`getItems`、`setEnablePicked`、`hitTest` |
| `BMap.ClusterLayer` | 按屏幕距离聚合点 | `setData`、`clearData`、`setOptions`、`redraw`、`getClusterLayer`、`getSingleLayer` |
| `BMap.TrackLine` | 轨迹绘制、播放与跟随 | `setData`、`start`、`pause`、`resume`、`stop`、`setSpeed`、`setProcess` |
| `BMap.Icons` | 内置 SVG 图标与 Marker Icon 工厂 | 读取图标字符串、`createIcon(svg, options)` |

Heatmap、PointLayer、ClusterLayer 与 TrackLine 使用 `map.addLayer(layer)` / `map.removeLayer(layer)`；`BMap.Icons.createIcon()` 的结果交给 Marker。数据解析、WebGL 渲染和交互结果仍需在有效 AK 页面验证。

## Heatmap 示例

```javascript
const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 13);

const heatmap = new BMap.Heatmap({
  size: 28,
  min: 0,
  max: 100,
  weightField: 'count',
});

heatmap.setData({
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: { count: 80 },
  }],
});

map.addLayer(heatmap);

function dispose() {
  map.removeLayer(heatmap);
  map.destroy();
}
```

`size` 默认 20，`unit` 只接受 `'px'` 或 `'m'`，`weightField` 默认 `'count'`。`setGradient()` 更新渐变，`setRadius()` 更新热力核半径。

## PointLayer 与 ClusterLayer

两者接收点 GeoJSON。`PointLayer` 未配置 `icon` 时绘制几何图元，配置 `icon` 后使用图标模式；拾取要显式设置 `enablePicked: true`。`ClusterLayer` 的核心参数是 `clusterRadius`、`clusterMinPoints`、`clusterMinZoom`、`clusterMaxZoom` 和 `fitViewOnClick`。

```javascript
const pointLayer = new BMap.PointLayer({
  shape: 'circle',
  size: 18,
  fillColor: '#1677ff',
  strokeColor: '#ffffff',
  strokeWeight: 2,
  enablePicked: true,
});

pointLayer.setData({
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: { id: 'point-1', name: '示例点' },
  }],
});

const onPointClick = (event) => {
  console.log(event.value, event.pixel, event.latLng);
};
pointLayer.addEventListener('click', onPointClick);
map.addLayer(pointLayer);

function removePointLayer() {
  pointLayer.removeEventListener('click', onPointClick);
  pointLayer.clearData();
  map.removeLayer(pointLayer);
}
```

`getItems()` 返回当前逐点缓存；`setEnablePicked(false)` 可关闭拾取，`hitTest(x, y)` 可按容器像素主动命中测试，未命中或实现尚未就绪时返回 `null`。

```javascript
const clusterLayer = new BMap.ClusterLayer({
  clusterRadius: 60,
  clusterMinPoints: 2,
  fitViewOnClick: true,
  singleStyle: {
    shape: 'circle',
    size: 14,
    fillColor: '#1677ff',
  },
});

clusterLayer.setData({
  type: 'FeatureCollection',
  features: [
    { type: 'Feature', geometry: { type: 'Point', coordinates: [116.404, 39.915] }, properties: { id: 1 } },
    { type: 'Feature', geometry: { type: 'Point', coordinates: [116.406, 39.916] }, properties: { id: 2 } },
  ],
});

map.addLayer(clusterLayer);

const onClusterClick = (event) => {
  console.log(event.value.isCluster, event.value.pointCount, event.value.properties);
};
const onClusterChange = (event) => {
  console.log(event.value.singles, event.value.clusters, event.value.zoom);
};
clusterLayer.addEventListener('click', onClusterClick);
clusterLayer.addEventListener('change', onClusterChange);

function removeClusterLayer() {
  clusterLayer.removeEventListener('change', onClusterChange);
  clusterLayer.removeEventListener('click', onClusterClick);
  clusterLayer.clearData();
  map.removeLayer(clusterLayer);
}
```

## TrackLine

`TrackLine` 接收单条 LineString Feature。播放状态通过 `start/pause/resume/stop` 控制；`setProcess()` 的取值范围是 0–1，`setSpeed()` 要求正数。

```javascript
const trackLine = new BMap.TrackLine({
  color: '#1677ff',
  width: 5,
  passedColor: '#999999',
  duration: 20,
  speedMode: 0,
  autoStart: true,
});

trackLine.setData({
  type: 'Feature',
  properties: { id: 'route-1' },
  geometry: {
    type: 'LineString',
    coordinates: [[116.392, 39.906], [116.418, 39.924]],
  },
});

map.addLayer(trackLine);
const onTrackStatusChange = (event) => {
  console.log(event.value.status, event.value.statusName);
};
const onTrackProgress = (event) => {
  console.log(event.value.process, event.value.point, event.value.distance);
};
trackLine.addEventListener('statuschange', onTrackStatusChange);
trackLine.addEventListener('progress', onTrackProgress);

function removeTrackLine() {
  trackLine.stop();
  trackLine.removeEventListener('progress', onTrackProgress);
  trackLine.removeEventListener('statuschange', onTrackStatusChange);
  map.removeLayer(trackLine);
}
```

首次加载时可视化实现是异步注入的；构造后立刻调用 `start()`、`pause()` 或 `resume()` 可能在实现尚未就绪时成为空操作。需要数据就绪后自动播放时使用 `autoStart: true`；在首次收到 `statuschange` 或 `progress` 后，再开放重播、暂停和继续操作。

## 事件或回调

- `PointLayer` 在 `enablePicked: true` 时派发 `click`、`dblclick`、`rightclick`、`mouseover`、`mousemove`、`mouseout`；事件含 `value`、`pixel`、`latLng` 与 `domEvent`。
- `ClusterLayer` 派发 `click`、`mouseover`、`mouseout` 与 `change`。鼠标事件的 `value` 含 `isCluster`、`pointCount`、`bbox` 与业务 `properties`；`change.value` 含 `singles`、`clusters`、`zoom`。
- `TrackLine` 派发 `statuschange` 与 `progress`。前者的 `value` 含 `status/statusName`，后者含 `process`、`elapsed`、`trace`、`distance`、`point` 与 `angle`。
- 所有业务监听都保存 handler 引用，并在移除图层前解绑。

## 其他扩展名称

JSAPI 4.0 还提供 `Marker3D`、`CanvasLayer`、`MapMask`、`PointCollection`、`Hotspot`、`XYZLayer`、`PixelLayer`、`ThreeLayer`、`VectorTileLayer`、`BaiduLayer`、`PanoramaCoverageLayer` 等名称。它们的接口形态差异较大，不能根据类名推测构造参数、事件或卸载方式。

对于已有更清晰 4.0 专页的常见需求，优先选择：

- 大量标准点线面：`PointIconLayer`、`PointShapeLayer`、`LineLayer`、`FillLayer`。
- 第三方栅格瓦片：`RasterTileLayer`、`WMSLayer`、`WMTSLayer`。
- DOM 内容：`DOMLayer` 或 `CustomOverlay`。
- 立体区域：`Prism`。

这里的“优先”仅表示接口更适合常见业务场景。

## 资源清理

1. 停止 TrackLine、timer、请求和业务数据流。
2. 用原 handler 解绑事件。
3. 对支持 `clearData()` 的图层先清业务数据。
4. 本页四类可视化图层均用 `map.removeLayer(layer)` 移除；其他扩展对象按各自的公开挂载方式清理。
5. 组件拥有 Map 时最后调用 `map.destroy()`。

## 常见错误

- 把所有扩展类都假定为 `map.addLayer/removeLayer` 生命周期。
- 根据名称猜测数据格式、事件名或清理方法。
