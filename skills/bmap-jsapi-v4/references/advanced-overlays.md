# 高级与自定义覆盖物

## 何时读取

需要 `BMap.CustomOverlay`、`BMap.GroundOverlay`、`BMap.GroundPoint`、`BMap.Prism`、`BMap.BezierCurve` 或一次性 PlaceDetail 时读取。高频基础能力已拆分到 [Marker](marker.md)、[Label 与 InfoWindow](label-and-info-window.md) 和 [矢量覆盖物](vector-overlays.md)。

基础覆盖物、事件、几何与 Map 所有权规则仍是本页示例的依赖。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | API |
|---|---|
| 业务 DOM 覆盖物 | `BMap.CustomOverlay` |
| 地理范围贴图 | `BMap.GroundOverlay` |
| 地面图片点 | `BMap.GroundPoint` |
| 立体区域 | `BMap.Prism` |
| 贝塞尔路径 | `BMap.BezierCurve` |
| Marker 地点详情面板 | `BMap.PlaceDetail`，同一实例不要反复 `openPlaceDetail` |

普通 Overlay 由 `map.addOverlay/removeOverlay` 管理，PlaceDetail 由 Marker 开关并由面板 `dispose()` 清理可见资源。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 14);

const imageUrl = 'data:image/svg+xml;charset=UTF-8,' + encodeURIComponent(
  '<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64"><rect width="64" height="64" fill="#1677ff" fill-opacity=".35"/></svg>',
);
const groundOverlay = new BMap.GroundOverlay(
  new BMap.Bounds(
    new BMap.Point(116.395, 39.908),
    new BMap.Point(116.413, 39.922),
  ),
  { url: imageUrl, opacity: 0.7 },
);
const groundPoint = new BMap.GroundPoint(
  new BMap.Point(116.404, 39.915),
  { url: imageUrl, size: new BMap.Size(32, 32), scale: 1 },
);
const prism = new BMap.Prism(
  [
    new BMap.Point(116.399, 39.910),
    new BMap.Point(116.409, 39.910),
    new BMap.Point(116.406, 39.920),
  ],
  300,
  { topFillColor: '#1677ff', sideFillColor: '#69b1ff' },
);
const bezierCurve = new BMap.BezierCurve(
  [new BMap.Point(116.392, 39.906), new BMap.Point(116.418, 39.924)],
  [[new BMap.Point(116.405, 39.936)]],
  { strokeColor: '#722ed1', strokeWeight: 4 },
);

map.addOverlay(groundOverlay);
map.addOverlay(groundPoint);
map.addOverlay(prism);
map.addOverlay(bezierCurve);

groundOverlay.setOpacity(0.5);
groundPoint.setRotation(30).setScale(1.2, true);
prism.setAltitude(500);
bezierCurve.setControlPoints([[new BMap.Point(116.408, 39.938)]]);

function dispose() {
  map.removeOverlay(bezierCurve);
  map.removeOverlay(prism);
  map.removeOverlay(groundPoint);
  map.removeOverlay(groundOverlay);
  map.destroy();
}
```

示例使用 data URL 避免外部图片依赖；地面贴图、3D 高度和交互仍需在有效 AK 的 `v=4.0` 页面在线验证。

## 核心 API

### CustomOverlay

`BMap.CustomOverlay` 由业务 DOM 工厂创建元素。`setPoint(point, noReCreate?)` 第二参数默认 false，会重建 DOM；纯位移传 true。`setProperties` 也会重建 DOM，因此业务必须先释放旧 DOM 上的 listener、timer、observer 和第三方实例。

```javascript
let customElement = null;
const onCustomClick = () => console.log('custom overlay clicked');

const customOverlay = new BMap.CustomOverlay(
  () => {
    const button = document.createElement('button');
    button.type = 'button';
    button.textContent = '门店 A';
    button.addEventListener('click', onCustomClick);
    customElement = button;
    return button;
  },
  {
    point: new BMap.Point(116.404, 39.915),
    anchors: [0.5, 1],
    properties: { id: 'store-a' },
  },
);

function releaseCustomElement() {
  customElement?.removeEventListener('click', onCustomClick);
  customElement = null;
}

map.addOverlay(customOverlay);
customOverlay.setPoint(new BMap.Point(116.408, 39.918), true);
releaseCustomElement();
customOverlay.setProperties({ id: 'store-a', selected: true });

function removeCustomOverlay() {
  releaseCustomElement();
  map.removeOverlay(customOverlay);
}
```

`CustomOverlay` 的 `enableMassClear: false` 当前不会生效，实例仍会参与批量清理。共享 Map 不要靠这个配置隔离所有权，固定精确移除自己保存的实例。

`useTranslate: true` 与 `autoFollowHeadingChanged: true` 组合会注册匿名 `heading_changed` 监听；`synUpdate: true` 会单独注册匿名 `ondraw` 监听（不依赖 `useTranslate`，单独打开即生效）。这两处监听都没有公开的解绑入口，基础 remove 只清 DOM。反复挂载/卸载的组件不要启用这些配置；确需使用时让 CustomOverlay 与 Map 同生命周期。

### Ground、Prism 与 Bezier

- GroundOverlay：按 Bounds 构造，常用 `setBounds`、`setImage`、`setOpacity`。
- GroundPoint：按 Point 和完整图片 options 构造，常用 `setSize`、`setScale`、`setRotation`。
- Prism：路径 + 高度，常用 `setPath`、`setAltitude` 及顶面/侧面样式。顶面与侧面填充色的默认值在 4.0 下跟随主题色 `--bmap-color-primary-bg`（缺省 `#eaf1ff`），顶面不透明度默认 0.6、侧面默认 0.8；需要固定配色就显式传 `topFillColor` / `sideFillColor`。
- BezierCurve：端点路径 + 控制点数组，常用 `setPath`、`setControlPoints` 与线样式。

`CustomOverlay` 与 `GroundPoint` 构造时都会直接读取 options；`CustomOverlay` 还会拒绝缺少 point 的输入。正式代码显式传完整 options 与 point。

### PlaceDetail

`BMap.PlaceDetail` 通过 Marker 的 `openPlaceDetail(panel, opts?)` 挂载，用法与清理路径见 [Marker](marker.md)。

每次调用 `openPlaceDetail` 都会给面板新增一个 `open` 监听，且没有对应的解绑入口。视觉上重复挂载不会叠加面板（`open` 回调第一句就是 `closePlaceDetail()`），但监听会持续累积，所以同一个 PlaceDetail 实例只挂载一次，位置变化通过面板自身的数据更新处理。

关闭与释放是两个归属不同的方法：`closePlaceDetail()` 在覆盖物（Marker 等）上，`dispose()` 在面板上。`placeDetail.dispose()` 内部会先清空面板 DOM 再调用 `overlay.closePlaceDetail()`，因此清理时直接调 `dispose()` 即可；只想收起面板、保留实例时才调用 `marker.closePlaceDetail()`。

## 事件或回调

GroundOverlay、GroundPoint、Prism、BezierCurve 都发布鼠标事件（`click`、`dblclick`、`rightclick`、`rightdblclick`、`mousedown`、`mouseup`、`mouseover`、`mouseout`、`mousemove`）、`remove` 与 `lineupdate`。Prism 与 BezierCurve 不实现编辑与顶点事件，也没有 `enableEditing`；需要可编辑图形请用 [矢量覆盖物](vector-overlays.md) 里的四类。

CustomOverlay 的业务 DOM 事件由业务自己成对注册和解绑。

## 资源清理

1. 停止异步更新、timer、observer 和动画。
2. 释放 CustomOverlay 当前 DOM 上的业务资源。
3. PlaceDetail 直接调用 `placeDetail.dispose()`，它会自行关闭已打开的面板。
4. 用保存的实例逐个 `map.removeOverlay`。
5. 只有组件拥有 Map 时才调用 `map.destroy()`。

## 常见错误

- 省略 CustomOverlay point 或 GroundPoint 的完整 options，依赖冲突签名。
- `setProperties` 重建 DOM 前不解绑旧元素资源。
- 依赖 CustomOverlay 的 `enableMassClear: false` 隔离共享 Map 所有权。
- 在短生命周期组件启用匿名 heading 跟随或 `synUpdate` 组合。
- 把 `closePlaceDetail()` 当成 PlaceDetail 上的方法：它在覆盖物上，`dispose()` 才在面板上，而且 `dispose()` 已经包含关闭动作。
- 反复对同一个 PlaceDetail 实例调用 `openPlaceDetail`，累积不会解绑的 `open` 监听。
- 期望 Prism 或 BezierCurve 支持顶点编辑：这两类不发布编辑事件，也没有 `enableEditing`。
- 假定 Prism 顶面/侧面默认是白色：4.0 下默认跟随主题色。
- 只删除业务 DOM，不移除覆盖物和销毁自己拥有的 Map。
