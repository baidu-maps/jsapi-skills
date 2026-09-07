# 批量可视化图层

## 何时读取

需要一次渲染大量点、线、面，并按要素状态更新样式、控制图层显隐/透明度/层级或处理拾取事件时读取。本页专门覆盖 `BMap.PointIconLayer`、`BMap.PointShapeLayer`、`BMap.LineLayer`、`BMap.FillLayer`；混合 GeoJSON 与 DOM 图层见 [数据图层](data-layers.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 数据 | 图层 | 常用样式 |
|---|---|---|
| 图片图标点 | `BMap.PointIconLayer` | `icon`、`sizes`、`anchors`、`rotation`、`opacity` |
| 圆/方/三角等几何点 | `BMap.PointShapeLayer` | `shapeType`、`size`、`color`、`strokeColor`、`strokeWeight` |
| LineString / MultiLineString | `BMap.LineLayer` | `strokeColor`、`strokeWeight`、`strokeOpacity`、`strokeStyle` |
| Polygon / MultiPolygon | `BMap.FillLayer` | `fillColor`、`fillOpacity`；描边默认开启，用 `strokeColor`/`strokeWeight` 配置 |

四者共享数据、状态、基础配置、样式、层级、显隐、透明度、缩放范围和事件 API；统一由 `map.addLayer/removeLayer` 管理。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 13);

const iconUrl = 'data:image/svg+xml;charset=UTF-8,' + encodeURIComponent(
  '<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24"><circle cx="12" cy="12" r="9" fill="#d93025" stroke="white" stroke-width="3"/></svg>',
);
const pointIconLayer = new BMap.PointIconLayer({
  crs: 'BD09LL',
  idKey: 'id',
  enablePicked: true,
  minZoom: 3,
  maxZoom: 20,
  style: { icon: iconUrl, sizes: [24, 24] },
});
const pointShapeLayer = new BMap.PointShapeLayer({
  crs: 'BD09LL',
  idKey: 'id',
  enablePicked: true,
  style: {
    shapeType: 0,
    size: 18,
    color: '#fa8c16',
    strokeColor: '#ffffff',
    strokeWeight: 2,
  },
});
const lineLayer = new BMap.LineLayer({
  crs: 'BD09LL',
  idKey: 'id',
  enablePicked: true,
  style: {
    strokeColor: ['case',
      ['boolean', ['feature-state', 'selected'], false],
      '#fa541c',
      '#1677ff',
    ],
    strokeWeight: ['case',
      ['boolean', ['feature-state', 'selected'], false],
      8,
      4,
    ],
    strokeOpacity: 0.9,
  },
});
const fillLayer = new BMap.FillLayer({
  crs: 'BD09LL',
  idKey: 'id',
  enablePicked: true,
  border: true,
  style: {
    fillColor: '#13a8a8',
    fillOpacity: 0.28,
    strokeColor: '#08979c',
    strokeWeight: 2,
  },
});

const onDataParsed = () => console.log('点图层数据解析完成');
const onPointClick = (event) => {
  if (!event.value || event.value.dataIndex === -1) return;
  console.log(event.latLng, event.value);
};
const onLineClick = (event) => {
  if (!event.value || event.value.dataIndex === -1) return;
  const id = event.value.dataItem
    && event.value.dataItem.properties.id;
  if (id === undefined || id === null) return;
  lineLayer.updateState(id, { selected: true }, false);
};
pointIconLayer.addEventListener('dataparsed', onDataParsed);
pointIconLayer.addEventListener('click', onPointClick);
lineLayer.addEventListener('click', onLineClick);

map.addLayer(pointIconLayer);
map.addLayer(pointShapeLayer);
map.addLayer(lineLayer);
map.addLayer(fillLayer);

pointIconLayer.setData({
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.404, 39.915] },
    properties: { id: 'icon-1', name: '图标点' },
  }],
});
pointShapeLayer.setData({
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    geometry: { type: 'Point', coordinates: [116.410, 39.918] },
    properties: { id: 'shape-1', name: '几何点' },
  }],
});
lineLayer.setData({
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    geometry: {
      type: 'LineString',
      coordinates: [[116.392, 39.906], [116.418, 39.924]],
    },
    properties: { id: 'line-1' },
  }],
});
fillLayer.setData({
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    geometry: {
      type: 'Polygon',
      coordinates: [[
        [116.397, 39.908],
        [116.413, 39.908],
        [116.410, 39.922],
        [116.397, 39.908],
      ]],
    },
    properties: { id: 'fill-1' },
  }],
});

console.log(pointIconLayer.getData());
console.log(lineLayer.getAllState());

function clearLineSelection() {
  lineLayer.clearState();
}

function removeLineSelection(id) {
  lineLayer.removeState(id);
}

pointIconLayer.setStyleOptions({ sizes: [28, 28] });
pointIconLayer.doOnceDraw();
console.log(pointIconLayer.getStyleOptions());

pointIconLayer.setVisible(false);
console.log(pointIconLayer.getVisible());
pointIconLayer.setVisible(true);
pointIconLayer.setOpacity(0.8);
console.log(pointIconLayer.getOpacity());
pointIconLayer.setMinZoom(5);
pointIconLayer.setMaxZoom(18);
console.log(pointIconLayer.getMinZoom(), pointIconLayer.getMaxZoom());
pointIconLayer.setZIndex(10);
console.log(pointIconLayer.getZIndex());

function dispose() {
  pointIconLayer.removeEventListener('dataparsed', onDataParsed);
  pointIconLayer.removeEventListener('click', onPointClick);
  lineLayer.removeEventListener('click', onLineClick);
  lineLayer.clearState();
  map.removeLayer(fillLayer);
  map.removeLayer(lineLayer);
  map.removeLayer(pointShapeLayer);
  map.removeLayer(pointIconLayer);
  map.destroy();
}
```

示例使用内嵌 data URL，不依赖外部图标站点；GeoJSON 坐标、WebGL 渲染、拾取效果与性能仍需在有效 AK 的 `v=4.0` 页面在线验证。

## 核心 API

### 数据与状态

四类图层都发布以下方法：

- 数据：`setData/getData`。
- 要素状态：`updateState`、`removeState`、`clearState`、`replaceAllState`、`getAllState`。
- 按数据索引隐藏：`addDelIndex`、`removeDelIndex`、`clearDelIndex`。
- 拾取结果：`getPickedItem(index, model)`；业务交互通常优先监听类型化事件。

构造时设置 `idKey`，状态 API 的 key 才能稳定对应到 feature properties；不传时默认取 `'id'`。点击拾取结果通过 `event.value.dataItem.properties.id` 取回同一个业务键，`updateState(id, { selected: true }, false)` 清掉旧状态并写入本次单选；样式中的 `feature-state → boolean → case` 随后把线切换为橙色并加粗。状态变化会重新解析数据；如果把 `strokeColor/strokeWeight` 改回固定值，状态仍会写入，但不会产生视觉变化。

拾取事件在“没有命中任何要素”时也会派发，此时 `event.value` 是 `{ dataIndex: -1, dataItem: undefined }`——它是真值，所以 `if (event.value)` 不能作为命中判断，必须显式检查 `event.value.dataIndex !== -1`。

### 样式与基础配置

`setStyleOptions/getStyleOptions` 修改各具体图层的 style，示例在更新样式后显式调用 `doOnceDraw()`。`setBaseOptions/getBaseOptions` 面向 `crs`、拾取等基础配置；`crs` 有下述副作用，跨坐标系业务不要用它做未验证的运行时切换。

`autoRender` 未显式设为 `false` 时，`setStyleOptions()` 自身已经重编样式并重解析数据，`doOnceDraw()` 只是保守兜底；另外 `getBaseOptions()` 会按全局 coordType 改写图层内部的 `crs`。正式代码保留“更新样式后显式重绘”的顺序，并把坐标系当成构造时固定项，运行时切换需要在线复核。

PointIconLayer 的尺寸有两套字段：`sizes: [宽, 高]` 与 `width`/`height`，由 `userSizes` 决定用哪套，默认 `true` 即只读 `sizes`（默认 `[32, 32]`）。只写 `width`/`height` 而不传 `userSizes: false` 时图标仍按 `sizes` 渲染。锚点字段是数组形式的 `anchors`，中心为 `[0, 0]`，取值范围 `[-1, 1]`。

FillLayer 的 `border` 默认已经是 `true`：构造时会用同一份 style 额外创建一个内部 LineLayer 来画描边，因此描边样式复用 `strokeColor`、`strokeWeight`、`strokeOpacity`、`strokeStyle`。`borderColor`/`borderWeight` 是这条描边线自身的外描边，默认 `borderWeight: 0` 即关闭。不需要描边时显式传 `border: false`。

### 图层状态与层级

- 显隐：`setVisible/getVisible`。
- 透明度：`setOpacity/getOpacity`，实现会限制到 0–1。
- 缩放范围：`setMinZoom/getMinZoom`、`setMaxZoom/getMaxZoom`。
- 层级：`setZIndex/getZIndex`，以及低频的 `setZIndexTop/setUpLevel/setDownLevel`。

层级调整实现会访问已关联的 Map 与图层管理器，因此先 `map.addLayer(layer)`，再调用 zIndex 调整方法；不要在未挂载实例上提前置顶或移动层级。

## 事件或回调

四类图层共享 `dataparsed`、`mousemove`、`click`、`dblclick`、`rightclick`。拾取事件提供 `pixel`、`latLng` 与 `value`；只有构造时启用 `enablePicked`，鼠标拾取事件才有业务意义。未命中要素时事件同样派发，靠 `event.value.dataIndex !== -1` 区分。

监听 `dataparsed` 时应先注册 handler，再调用 `setData`。卸载时使用原函数引用逐一解绑。

## 资源清理

1. 停止业务的数据流、timer、worker 和异步回调。
2. 用原 handler 解绑 `dataparsed` 与拾取事件。
3. 按所有权清理状态，并用保存的实例逐个调用 `map.removeLayer(layer)`。
4. 释放业务持有的大型 GeoJSON 引用；不要假定隐藏图层等于释放数据。
5. 只有组件拥有 Map 时才调用 `map.destroy()`。

`map.removeLayer(layer)` 会把图层从内部列表摘掉并触发它自身的销毁流程；业务持有的数据、监听与异步引用仍要自己解除。

## 常见错误

- 大批量数据仍逐条创建 Marker/Polygon，导致对象和事件监听过多。
- 没有设置 `idKey`，却假定 `updateState` 的 key 能稳定命中业务 ID。
- 未启用 `enablePicked` 就等待 click/mousemove 的要素拾取结果。
- 用 `if (event.value)` 判断是否命中要素；未命中时它也是真值，必须查 `dataIndex !== -1`。
- 给 PointIconLayer 只写 `width`/`height`，忘记 `userSizes` 默认 `true` 会让图标仍按 `sizes` 渲染。
- 在 `setData` 之后才监听 `dataparsed`，错过本次解析事件。
- 在 `map.addLayer` 前调用 `setZIndex`、`setZIndexTop`、`setUpLevel` 或 `setDownLevel`。
- 把 `setVisible(false)` 当成释放数据和 WebGL 资源。
- 运行时改变 `crs`，却没有验证 `getBaseOptions` 的全局坐标副作用。
