# Marker 标注

## 何时读取

需要创建、更新、拖拽、监听、显示/隐藏或移除单个 `BMap.Marker`，以及给 Marker 配置 Icon、Label、InfoWindow 或一次性 PlaceDetail 时读取。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | API |
|---|---|
| 创建 Marker | `new BMap.Marker(point, options)` |
| 挂到地图 | 创建后由 Map 添加该覆盖物实例 |
| 更新位置、图标、偏移、标题、标签 | `setPosition`, `setIcon`, `setOffset`, `setTitle`, `setLabel` |
| 读取当前状态 | 对应 `getPosition`, `getIcon`, `getOffset`, `getTitle`, `getLabel` |
| 拖拽 | `enableDragging` / `disableDragging`，监听 `dragstart`、`dragging`、`dragend` |
| 显示与隐藏 | `show`, `hide`, `isVisible` |
| 层级、旋转 | `setZIndex`, `setRotation` |
| 锚点 | 构造 options 传 `anchor`（`setAnchor` 要等模块加载） |
| 是否参与批量清理 | `enableMassClear` / `disableMassClear` |
| 弹出普通气泡 | Marker click 后调用 `map.openInfoWindow(infoWindow, marker.getPosition())` |
| 地点详情 | 同一 `PlaceDetail` 只调用一次 `openPlaceDetail`，退出时调用 `dispose()` |
| 精确移除 | 保存实例并调用 `map.removeOverlay(marker)` |

## 最小可运行示例

页面先按 [项目接入](project-setup.md) 加载 `v=4.0`，并准备有明确宽高的 `#map` 容器：

```javascript
const map = new BMap.Map('map');
const start = new BMap.Point(116.404, 39.915);
const next = new BMap.Point(116.410, 39.920);

map.centerAndZoom(start, 15);

const svg = '<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 32 32"><path fill="#d93025" d="M16 1A11 11 0 0 0 5 12c0 8 11 19 11 19s11-11 11-19A11 11 0 0 0 16 1z"/><circle cx="16" cy="12" r="4" fill="white"/></svg>';
const icon = new BMap.Icon(
  'data:image/svg+xml;charset=UTF-8,' + encodeURIComponent(svg),
  new BMap.Size(32, 32),
  { anchor: new BMap.Size(16, 32) },
);
const label = new BMap.Label('可拖拽标注', {
  offset: new BMap.Size(18, -12),
});
const marker = new BMap.Marker(start, {
  icon,
  offset: new BMap.Size(0, 0),
  title: '示例地点',
  anchor: BMAP_ANCHOR_BOTTOM_CENTER,
  enableDragging: true,
  enableClicking: true,
  enableMassClear: true,
  raiseOnDrag: false,
  draggingCursor: 'move',
  rotation: 0,
});
const infoWindow = new BMap.InfoWindow('<strong>示例地点</strong>');

marker.setLabel(label);
map.addOverlay(marker);

const onClick = () => {
  map.openInfoWindow(infoWindow, marker.getPosition());
};
const onDragEnd = (event) => {
  console.log('拖拽结束', event.latLng);
};

marker.addEventListener('click', onClick);
marker.addEventListener('dragend', onDragEnd);

// 常见更新与读取。
marker.setIcon(icon);
console.log(marker.getIcon());
marker.setPosition(next);
console.log(marker.getPosition());
marker.setOffset(new BMap.Size(0, -4));
console.log(marker.getOffset());
marker.setTitle('更新后的标题');
console.log(marker.getTitle());
marker.setLabel(label);
console.log(marker.getLabel());
marker.setRotation(30);
console.log(marker.getRotation());
marker.setZIndex(100);
console.log(marker.getMap());

marker.disableDragging();
marker.enableDragging();
marker.disableMassClear();
marker.enableMassClear();
marker.hide();
console.log(marker.isVisible());
marker.show();

function dispose() {
  marker.removeEventListener('click', onClick);
  marker.removeEventListener('dragend', onDragEnd);
  marker.disableDragging();
  map.closeInfoWindow();
  map.removeOverlay(marker);
  map.destroy();
}
```

地图数据、图片加载和交互效果仍需用有效 AK 在线验证。

## 核心 API

### 构造与常用 Options

`new BMap.Marker(point, options)` 的公开常用 options 包括 `offset`、`icon`、`anchor`、`enableMassClear`、`enableDragging`、`enableClicking`、`raiseOnDrag`、`draggingCursor`、`rotation` 和 `title`。默认启用点击和 mass clear，默认关闭拖拽。

不传 `icon` 时 4.0 默认图标是 22×33 的标注图，偏移 `Size(11, 30)`、信息窗口偏移 `Size(10, 0)`。业务不要按默认尺寸反推布局，需要确定尺寸就显式传 `icon`。

`point` 也可以直接传 Point 类型的 GeoJSON Feature、Geometry 或 FeatureCollection；多几何时取第一个点，Feature 的 `properties` 会保存在 Marker 上。`setPosition(geojson)` 与 `updateByGeoJSON(geojson)` 使用同一套解析路径。

运行时还存在一个 `clickable`（默认 `true`），它和 `enableClicking` 作用范围不同：鼠标指针样式要求两者同时为真，而点击事件是否派发只看 `enableClicking`。控制可点击性统一用 `enableClicking`；只关闭 `clickable` 会去掉手型指针，但点击仍会触发。

### 图片 Icon 与矢量 Symbol

- 图片、Canvas 或雪碧图标注使用 `BMap.Icon`，这是 Marker 的正式图标类型。
- 不想维护多倍图时可用 SVG path 创建 `BMap.Symbol`。4.0 会把 Symbol 转成 Marker 可渲染的矢量图标。
- 需要内置语义图标时使用 `BMap.Icons.createIcon(BMap.Icons.MARKER_PIN, options)`；`color` 替换 SVG 的 `currentColor`，`size` 接受数字或 `BMap.Size`。

```javascript
const symbol = new BMap.Symbol('M 0 -10 L 9 8 L -9 8 Z', {
  scale: 1.5,
  fillColor: '#1677ff',
  fillOpacity: 0.9,
  strokeColor: '#ffffff',
  strokeWeight: 2,
});
const symbolMarker = new BMap.Marker(
  new BMap.Point(116.404, 39.915),
  { icon: symbol, title: '矢量标注' },
);
map.addOverlay(symbolMarker);

function removeSymbolMarker() {
  map.removeOverlay(symbolMarker);
}
```

Marker 的 `anchor` 默认值为空，表示不覆盖 Icon 自身的 anchor。需要稳定锚点时在构造 options 中显式传入，也不要假设未设置时 `getAnchor()` 一定非空。

`setAnchor` / `getAnchor` 只在异步标注模块加载后挂载，而 `new BMap.Marker(...)` 先返回同步外壳。因此构造后立刻调用 `marker.setAnchor(...)` 可能得到 `TypeError`。稳妥写法是在构造 options 中传 `anchor`；确需运行期修改时，等待标注模块完成首次绘制。

`setPosition`、`setIcon`、`setOffset`、`setTitle`、`setLabel`、`setRotation`、`setZIndex`、`show`/`hide` 等本页其余方法都在同步外壳上，构造后即可调用。

### 更新、查询和可见性

- 图标：`setIcon` / `getIcon`
- 位置：`setPosition` / `getPosition`
- 像素偏移：`setOffset` / `getOffset`
- 标题：`setTitle` / `getTitle`
- 文本标签：`setLabel` / `getLabel`
- 锚点：`setAnchor` / `getAnchor`（仅在标注模块加载完成后存在，见上一节）
- 旋转：`setRotation` / `getRotation`
- 拖拽：`enableDragging` / `disableDragging`
- 批量清理：`enableMassClear` / `disableMassClear`
- 层级与归属：`setZIndex` / `getMap`
- 继承的可见性：`show` / `hide` / `isVisible`

`setAnimation`、`setTop`、`setShadow` 在运行时存在，但语义和名字不完全一致，正式代码不要依赖：

- `setTop(top, addi)` 会写入内部置顶标记并立刻刷新 DOM 层级，属于内部层级调度的入口；需要稳定层级请用 `setZIndex`。
- `setAnimation(animation)` 依赖额外的动画模块异步加载，`raiseOnDrag: true` 内部就是靠它实现拖拽抬起/落下的；业务动效自行更新 icon 或 position 更可控。
- `setShadow` / `getShadow` 在 4.0 分支上只是把值存进内部配置（GL 分支直接 `console.warn`），渲染链上没有任何地方读取它，所以设置后**不会有视觉效果**。需要投影就把阴影画进图标图片里。

### InfoWindow 与 PlaceDetail

普通气泡由 Map 管理：保存 `InfoWindow` 实例，在 Marker 点击时用 Marker 的当前位置打开，退出时先关闭气泡，再解绑事件、移除 Marker。

PlaceDetail 只推荐一次性挂载：每次 `openPlaceDetail` 都会新增监听，而没有公开的解绑入口。不要反复把同一个 PlaceDetail 实例挂到同一个 Marker。

下面只展示首次挂载及已确认的可见资源清理路径：

```javascript
const detailMarker = new BMap.Marker(new BMap.Point(116.404, 39.915));
const detailPanel = document.createElement('div');
const placeDetail = new BMap.PlaceDetail(detailPanel);

placeDetail.setData('POI_UID');
map.addOverlay(detailMarker);
detailMarker.openPlaceDetail(placeDetail);

function removePlaceDetail() {
  placeDetail.dispose();
  map.removeOverlay(detailMarker);
}
```

`placeDetail.dispose()` 内部会先清空面板 DOM 再调用覆盖物上的 `closePlaceDetail()`，所以清理时只调 `dispose()` 就够；只想收起面板、保留实例时才单独调用 `detailMarker.closePlaceDetail()`。

## 事件或回调

Marker 公开事件包括：

| 类别 | 事件 |
|---|---|
| 鼠标 | `click`, `dblclick`, `rightclick`, `mousedown`, `mouseup`, `mouseover`, `mouseout` |
| 拖拽 | `dragstart`, `dragging`, `dragend` |
| 生命周期 | `remove` |

拖拽事件只有先启用拖拽才有业务意义。添加和移除监听器必须传入同一个函数引用；两个内容相同的匿名函数不是同一引用。

```javascript
const onDragging = (event) => {
  console.log(event.pixel, event.latLng);
};

marker.enableDragging();
marker.addEventListener('dragging', onDragging);

function stopWatchingDrag() {
  marker.removeEventListener('dragging', onDragging);
}
```

## 资源清理

组件退出时按顺序执行：

1. 使迟到的业务回调失效。
2. 用原 handler 解绑 Marker 监听器。
3. 关闭 InfoWindow；PlaceDetail 直接调用 `dispose()`，它会自行关闭面板。
4. 停止拖拽，再用 `map.removeOverlay(marker)` 精确移除同一实例。
5. 只有该组件拥有 Map 时才调用 `map.destroy()`。

若业务不希望 `map.clearOverlays()` 清掉 Marker，应提前 `disableMassClear()`；组件卸载仍应使用 `removeOverlay(marker)`，不要把批量清理当作所有权管理。

## 常见错误

- 只 `new BMap.Marker`，忘记 `map.addOverlay(marker)`。
- 解绑事件时重新写一个匿名函数，导致原监听器仍然存在。
- 开启拖拽却不保存 `dragstart`、`dragging`、`dragend` 的 handler 引用。
- 依赖 Marker 的默认 anchor，它实际为空而不是 `BMAP_ANCHOR_CENTER`。
- 构造后立刻调用 `marker.setAnchor(...)` 或 `marker.getAnchor()`：这两个方法要等标注模块加载完成才存在，同步调用会抛 `TypeError`。锚点请在构造 options 里传。
- 依赖 `setAnimation`、`setTop`、`setShadow` 的名称直觉；其中 `setShadow` 当前没有视觉效果。
- 用 `clickable: false` 去关闭点击：它只影响鼠标指针，事件派发只看 `enableClicking`。
- 按默认图标的像素尺寸反推布局：默认图标随 UI 版本不同。
- 反复对同一个 PlaceDetail 调用 `openPlaceDetail`，累积无法解绑的监听。
- 只删除 DOM 容器，不关闭气泡、不移除覆盖物、不销毁自己创建的 Map。
- 把静态语法检查当成 AK、网络、图标资源和地图交互已在线验证。
