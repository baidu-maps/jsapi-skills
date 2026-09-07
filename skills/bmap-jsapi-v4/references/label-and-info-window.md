# Label 与 InfoWindow

## 何时读取

需要独立文本标注、Marker 文本标签、点击对象弹出信息窗口、动态更新气泡内容或完整处理监听与卸载时读取。本页覆盖 `BMap.Label` 和 `BMap.InfoWindow` 的高频开发闭环；Marker 自身 API 见 [Marker 标注](marker.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | API |
|---|---|
| 地图上的常驻文本 | `new BMap.Label(content, options)`，再由 Map 添加 |
| Marker 旁的文本 | 创建 Label 后交给 Marker 关联 |
| 临时信息气泡 | `new BMap.InfoWindow(content, options)`，再由 Map 在目标 Point 打开 |
| 更新 Label | `setContent`、`setPosition`、`setOffset`、`setStyles`、`setTitle` |
| 更新 InfoWindow | `setContent`、`setTitle`、`setWidth`、`setHeight`；气泡已打开时它们会自动重绘 |
| 检查气泡状态 | `isOpen()`；只在已打开时读取 `getPosition()` |
| 关闭与移除 | Map 关闭当前 InfoWindow；独立 Label 由 Map 精确移除 |

## 最小可运行示例

页面先按 [项目接入](project-setup.md) 加载 `v=4.0`，并准备有明确宽高的 `#map` 容器：

```javascript
const map = new BMap.Map('map');
const start = new BMap.Point(116.404, 39.915);
const next = new BMap.Point(116.410, 39.920);
map.centerAndZoom(start, 15);

const label = new BMap.Label('点击查看详情', {
  position: start,
  offset: new BMap.Size(12, -16),
  enableClicking: true,
  enableMassClear: true,
});
label.setStyles({
  padding: '6px 10px',
  color: '#1677ff',
  backgroundColor: '#ffffff',
  border: '1px solid #1677ff',
  borderRadius: '4px',
});
label.setAnchor(BMAP_ANCHOR_BOTTOM_CENTER);
// 层级用 setZIndex，不要写 styles.zIndex。
label.setZIndex(100);
map.addOverlay(label);
// 挂载时重建 DOM 不会重放 opacity，必须挂载之后再设置。
label.setOpacity(0.95);

const content = document.createElement('div');
content.textContent = '这里可以放业务 DOM';
const infoWindow = new BMap.InfoWindow(content, {
  title: '示例地点',
  width: 280,
  height: 100,
  offset: new BMap.Size(0, -8),
  enableAutoPan: true,
  enableCloseOnClick: true,
});

const onInfoOpen = () => {
  console.log('气泡已打开', infoWindow.isOpen());
  if (infoWindow.isOpen()) console.log(infoWindow.getPosition());
};
const onInfoClose = () => {
  console.log('气泡已关闭');
};
const onLabelClick = () => {
  map.openInfoWindow(infoWindow, label.getPosition());
};

infoWindow.addEventListener('open', onInfoOpen);
infoWindow.addEventListener('close', onInfoClose);
label.addEventListener('click', onLabelClick);

// 高频动态更新与读取。
label.setContent('更新后的文本');
console.log(label.getContent());
label.setPosition(next);
console.log(label.getPosition());
label.setOffset(new BMap.Size(14, -18));
console.log(label.getOffset());
label.setTitle('更新后的标题');
console.log(label.getTitle(), label.getAnchor(), label.getMap());
label.disableMassClear();
label.enableMassClear();

infoWindow.setContent('<strong>更新后的气泡内容</strong>');
console.log(infoWindow.getContent());
infoWindow.setTitle('更新后的气泡标题');
console.log(infoWindow.getTitle());
infoWindow.setWidth(320);
infoWindow.setHeight(120);
infoWindow.disableAutoPan();
infoWindow.enableAutoPan();
infoWindow.disableCloseOnClick();
infoWindow.enableCloseOnClick();
// 仅在直接改动气泡内容 DOM 后才需要；气泡未打开时该调用直接返回。
infoWindow.redraw();

function dispose() {
  label.removeEventListener('click', onLabelClick);
  infoWindow.removeEventListener('open', onInfoOpen);
  infoWindow.removeEventListener('close', onInfoClose);
  map.closeInfoWindow();
  map.removeOverlay(label);
  map.destroy();
}
```

真实 AK、地图资源、自动平移和浏览器交互仍需在有效 `v=4.0` 页面在线验证。

## 核心 API

### Label

`BMap.Label` 的常用 options 是 `position`、`offset`、`enableClicking`、`enableMassClear`、`anchor`、`width` 与 `styles`。层级统一用 `setZIndex()`，不要写 `styles.zIndex`：它虽然有默认值 `'80'` 且会被写进 DOM，但每次 `draw()` 结尾都会用实例层级重新覆盖 DOM 的 `z-index`，而实例层级的初始值是 `0`，所以 `styles.zIndex`（以及后续通过 `setStyles` 传入的 `zIndex`）**最终都不生效**。`title` 与 `coordType` 在 4.0 运行时可用，但不在已发布的 LabelOptions 中；构造时传 `title` 与挂载前调用 `setTitle()` 效果相同，挂载后更新标题才必须用 `setTitle()`。独立 Label 必须有位置并由 Map 添加；作为 Marker 标签时由 Marker 管理关联位置。

`setOpacity()` 会同时写入配置和当前 DOM，但挂载时重建 DOM 的重放逻辑只会重新应用 `styles`，不会读取已保存的 opacity，因此在 `map.addOverlay(label)` 之前调用等于白设，必须挂载后再设置。`setZIndex()` 与 `setAnchor()` 在挂载前调用是安全的：前者写实例层级并在每次绘制时生效，后者写进配置并在挂载绘制时应用。

一个 Label 实例不能同时兼任独立覆盖物和 Marker 标签。`marker.setLabel(label)` 会把 Label 绑定到 Marker，而 `Label.setPoint()` 内部有「未绑定 Marker」的前置判断，所以绑定之后 `setPosition()` 会静默失效，位置只跟着 Marker 走。两种用途各建一个实例。

高频方法按职责分组：

- 内容与外观：`setContent/getContent`、`setStyles`、`setOpacity`。
- 位置与偏移：`setPosition/getPosition`、`setOffset/getOffset`、`setAnchor/getAnchor`。
- 标题与层级：`setTitle/getTitle`、`setZIndex`、`getMap`。
- 批量清理：`enableMassClear/disableMassClear`。

### InfoWindow

`BMap.InfoWindow` 接受字符串或 HTMLElement 内容。常用 options 包括 `title`、`width`、`height`、`offset`、`enableAutoPan`、`enableCloseOnClick`、`enableMaximize` 与 `maxContent`。宽度非 0 时实现会限制到 220–730 像素，高度非 0 时限制到 60–650 像素；传 0 表示自适应。

InfoWindow 不作为普通 Overlay 调用 `map.addOverlay`；打开和关闭由 Map 完成。打开状态用 `isOpen()` 判断。`setContent`、`setTitle`、`setWidth`、`setHeight` 在气泡已打开时会自行触发重绘，不需要再手工调用 `redraw()`；只有直接改动业务自己持有的内容 DOM（气泡尺寸随之变化）时才需要 `redraw()`。`redraw()` 在气泡未打开时直接返回，不能用它替代 `openInfoWindow`。

InfoWindow 尚未打开或没有关联 overlay 时，`getPosition()` 可能拿不到位置。只在 `isOpen()` 为 true 之后读取，并仍然做空值防御。

`getOffset()` 在运行时可用。`setContent` 的异步实现存在内部第二参数 `notRedraw`，但公开同步入口只接收内容；业务代码不要依赖该内部参数跳过重绘。

需要最大化能力时，先设置最大化内容并启用，再在窗口已打开后执行；还原后可关闭该能力：

```javascript
infoWindow.setMaxContent('<div>更完整的详情内容</div>');
infoWindow.enableMaximize();
if (infoWindow.isOpen()) infoWindow.maximize();
if (infoWindow.isOpen()) infoWindow.restore();
infoWindow.disableMaximize();
```

最大化后的窗口上限是 730×650 像素；业务不要按固定像素做最大化态的内部排版。

## 事件或回调

Label 的公开事件包括 `click`、`dblclick`、`rightclick`、`mousedown`、`mouseup`、`mouseover`、`mouseout` 和 `remove`。InfoWindow 的公开事件包括 `open`、`close`、`clickclose`、`maximize`、`restore` 和 `resize`。

监听和解绑必须使用同一个函数引用：

```javascript
const onResize = () => console.log('InfoWindow 尺寸已变化');
infoWindow.addEventListener('resize', onResize);

function stopWatchingResize() {
  infoWindow.removeEventListener('resize', onResize);
}
```

## 资源清理

1. 停止更新 Label/InfoWindow 的 timer、observer 和异步回调。
2. 用原 handler 解绑 Label 与 InfoWindow 的全部业务监听器。
3. 调用 `map.closeInfoWindow()`。
4. 独立 Label 用保存的实例调用 `map.removeOverlay(label)`；Marker 标签随 Marker 所有权处理。
5. 只有组件拥有 Map 时才调用 `map.destroy()`。

## 常见错误

- 给独立 Label 设置了 position，却忘记 `map.addOverlay(label)`。
- 把 InfoWindow 当成 Overlay 调用 `map.addOverlay(infoWindow)`。
- 直接改动气泡内的业务 DOM 使尺寸变化后忘记 `redraw()`，或反过来在气泡未打开时调用 `redraw()` 却期待它把气泡显示出来。
- 在挂载前调用 `label.setOpacity()`，误以为透明度会在 `map.addOverlay` 时被应用。
- 用 `styles.zIndex` 或 `setStyles({zIndex})` 控制 Label 层级：每次绘制都会被实例层级覆盖，只有 `setZIndex()` 有效。
- 让同一个 Label 既做独立覆盖物又做 Marker 标签：绑定 Marker 后 `setPosition()` 会静默失效。
- 依赖 `setContent` 的内部第二参数跳过重绘。
- 在 InfoWindow 打开前把 `getPosition()` 当作必定非空。
- 解绑时重新创建匿名函数，导致旧监听器仍存在。
- 只移除 Label DOM，不关闭 InfoWindow、不移除覆盖物。
- 只检查代码语法，不验证 AK、网络和实际气泡交互。
