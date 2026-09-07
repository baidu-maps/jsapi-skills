# GeoJSON 与 DOM 图层

## 何时读取

需要混合 GeoJSON、把 GeoJSON 转成普通覆盖物、批量 DOM 或底图文字标签时读取。大量同类点线面见 [批量可视化图层](visualization-layers.md)，矢量瓦片见 [MVT 图层](mvt-layer.md)，行政区边界与填色见 [行政区边界与区域填色](administrative-district.md)，低频能力见 [扩展 API](runtime-extended-apis.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | 选择 |
|---|---|
| 混合 Point/LineString/Polygon 的 GeoJSON | `BMap.GeoJSONLayer` |
| GeoJSON 转普通 Marker/Polyline/Polygon | `BMap.GeoJSONParse`，业务持有结果并逐个移除 |
| 一组业务 DOM | `BMap.DOMLayer`，只建议与 Map 同生命周期 |
| 行政区 | 进入 [行政区边界与区域填色](administrative-district.md) |
| 大批量同类点线面 | 四种具体批量图层，进入可视化专页 |
| 底图文字标签 | `map.addMapLabels/removeMapLabels`，持有返回 uid |

JSAPI 4.0 提供可继承的 `BMap.NormalLayer` 图层基类，但不提供 `BMap.FeatureLayer`。常规业务优先直接使用 PointIconLayer、PointShapeLayer、LineLayer、FillLayer；只有实现自定义普通图层时才继承 NormalLayer。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const center = new BMap.Point(116.404, 39.915);
map.centerAndZoom(center, 13);

const geoJSONLayer = new BMap.GeoJSONLayer('business-data', {
  reference: 'BD09LL',
  minZoom: 5,
  maxZoom: 19,
  level: -50,
  markerStyle: (properties) => ({ title: String(properties.name || '') }),
  polylineStyle: { strokeColor: '#1677ff', strokeWeight: 4 },
  polygonStyle: { strokeColor: '#13a8a8', fillColor: '#13a8a8', fillOpacity: 0.25 },
});

const onFeatureClick = (event) => {
  const picked = geoJSONLayer.pickOverlays(event);
  console.log(event.latLng, event.features, picked);
};
geoJSONLayer.addEventListener('click', onFeatureClick);
map.addLayer(geoJSONLayer);

geoJSONLayer.setData({
  type: 'FeatureCollection',
  features: [
    {
      type: 'Feature',
      geometry: { type: 'Point', coordinates: [116.404, 39.915] },
      properties: { id: 'poi-1', name: '示例点' },
    },
    {
      type: 'Feature',
      geometry: {
        type: 'LineString',
        coordinates: [[116.395, 39.910], [116.415, 39.920]],
      },
      properties: { id: 'line-1' },
    },
  ],
});

console.log(geoJSONLayer.getData());
geoJSONLayer.setLevel(-40);
console.log(geoJSONLayer.getLevel());
geoJSONLayer.setVisible(false);
console.log(geoJSONLayer.getVisible());
geoJSONLayer.setVisible(true);
geoJSONLayer.resetStyle();

function dispose() {
  geoJSONLayer.removeEventListener('click', onFeatureClick);
  geoJSONLayer.clearData();
  map.removeLayer(geoJSONLayer);
  geoJSONLayer.destroy();
  map.destroy();
}
```

`getData()` 返回解析生成的普通 Overlay 数组，不是原始 GeoJSON。`clearData()` 先从 Map 移除这些覆盖物并清空集合；组件退出时先清数据，再走统一 `removeLayer`，最后释放自己拥有的 Map。

## 核心 API

### GeoJSONLayer

常用 options 是 `dataSource`、`reference`、`markerStyle`、`polylineStyle`、`polygonStyle`、`minZoom`、`maxZoom`、`level` 和 `visible`。样式可直接传 options，也可按 feature properties 返回 options。

常用方法：

- 数据：`setData/getData/clearData`。
- 样式与命中：`resetStyle/pickOverlays`。
- 层级：`setLevel/getLevel`；层级值原样透传给每个解析出的覆盖物的 `setZIndex`，未设置时默认 `-99`。
- 显隐：`setVisible/getVisible`。
- 生命周期：`destroy` 与 Map 的 `addLayer/removeLayer`。

`map.removeLayer(geoJSONLayer)` 已经摘掉覆盖物、解绑监听并清空图层持有的 Map 引用，之后再调用 `destroy()` 不起作用。要真正清空 `getData()` 集合，得在 `removeLayer` 之前调用 `clearData()`，或者先 `destroy()` 再 `removeLayer()`。

GeoJSONLayer 的 click 事件对象赋值 `latLng`、`pixel` 和 `features`，没有 `point`。业务代码读取 `event.latLng`。

### GeoJSONParse

`BMap.GeoJSONParse` 明确分派 Point、LineString、Polygon、MultiPoint、MultiLineString、MultiPolygon，并构造 Marker、Polyline、Polygon 等普通覆盖物。业务必须持有返回数组并逐个 `map.removeOverlay`。

`reference` 转换与覆盖物构造链存在二次坐标转换冲突；GeometryCollection 不受 `readGeometry` 支持。使用 GeoJSONParse 时优先在全局 BD09 下处理受支持的几何类型。

```javascript
if (map.getCoordType() !== BMAP_COORD_BD09) {
  throw new Error('GeoJSONParse 示例要求全局坐标类型为 BD09');
}

const parser = new BMap.GeoJSONParse({ reference: 'WGS84' });
const parsedOverlays = parser.readFeaturesFromObject({
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: { name: '示例点' },
  }],
});
for (const overlay of parsedOverlays) map.addOverlay(overlay);

function removeParsedData() {
  for (const overlay of parsedOverlays) map.removeOverlay(overlay);
  parsedOverlays.length = 0;
}
```

### Map Labels

`map.addMapLabels(labels)` 返回实际 uid；精确移除时把保存的 uid 原样传给 `removeMapLabels`。`clearLabels()` 会清空当前 Map 的全部自定义底图标签，不适合共享 Map 的模块所有权管理。

```javascript
const mapLabelUids = map.addMapLabels([{
  text: '业务标签',
  position: new BMap.Point(116.404, 39.915),
}]);

function removeBusinessLabels() {
  map.removeMapLabels(mapLabelUids);
}
```

### DOMLayer

DOMLayer 发布 `setData`、`show/hide`、`getCustomOverlays`、`removeOverlay`、`removeAllOverlays`、`setStyleOptions` 和 `addEventListener`。但它没有对应的 `removeEventListener`，注册的业务事件解绑不掉，因此只建议用在与整张 Map 同生命周期的页面壳层，不要在短生命周期路由组件里给它注册事件。

```javascript
const domLayer = new BMap.DOMLayer(
  (properties) => {
    const element = document.createElement('div');
    element.textContent = String(properties.name || '业务点');
    element.style.cssText = 'padding:4px 8px;background:#fff;border-radius:4px;';
    return element;
  },
  { minZoom: 8, maxZoom: 18, anchors: [0.5, 1] },
);
domLayer.setData({
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: { name: '门店 A' },
  }],
});
map.addLayer(domLayer);

domLayer.hide();
domLayer.show();
domLayer.setStyleOptions({ minZoom: 9, maxZoom: 17 });
const currentDOMOverlays = domLayer.getCustomOverlays();
if (currentDOMOverlays[0]) domLayer.removeOverlay(currentDOMOverlays[0]);

function removeDOMLayerWithMap() {
  domLayer.removeAllOverlays();
  map.removeLayer(domLayer);
}
```

`setData(null)` 只清空数据引用，不会移除已经渲染出来的 overlays；清空时必须显式调用 `removeAllOverlays()`。

## 事件或回调

- GeoJSONLayer：`click`、`mousemove`、`mouseout`，使用具名 handler 成对解绑。
- DOMLayer：同样发布 click/mouseover/mouseout，但没有公开 removeEventListener，短生命周期模块不注册。

## 资源清理

1. GeoJSONLayer：解绑事件 → `clearData()` → `map.removeLayer()` → `destroy()`。
2. GeoJSONParse：持有返回覆盖物，逐个 `map.removeOverlay()`。
3. Map labels：持有 add 返回的 uid，用 `removeMapLabels(uidList)`。
4. DOMLayer：先 `removeAllOverlays()` 再 `removeLayer()`；由于事件无法解绑，让它与最终销毁的 Map 同生命周期。
5. 只有组件拥有 Map 时才调用 `map.destroy()`。

## 常见错误

- 把 GeoJSONLayer `getData()` 当作原始 GeoJSON；它返回 Overlay 数组。
- 把 GeometryCollection 直接交给 GeoJSONParse 正式流程。
- 丢弃 GeoJSONParse 返回数组，导致无法精确移除覆盖物。
- 用 `clearLabels()` 清理某个模块，误删其他模块标签。
- 对 DOMLayer 只调用 `setData(null)`，旧 DOM 覆盖物仍在。
- 在短生命周期组件注册 DOMLayer 事件。
- 读取 GeoJSONLayer click 的 `event.point`，它不会被赋值。
