# 矢量覆盖物

## 何时读取

需要绘制、更新、编辑、监听或移除 `BMap.Polyline`、`BMap.Polygon`、`BMap.Rectangle`、`BMap.Circle` 时读取。本页覆盖路径、范围、样式、编辑事件和资源清理；3D 与自定义覆盖物见 [高级覆盖物](advanced-overlays.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 业务形状 | 类 | 核心数据 |
|---|---|---|
| 路径、轨迹、连线 | `BMap.Polyline` | Point 数组 |
| 区域、多环区域 | `BMap.Polygon` | Point 数组或多组 Point 数组 |
| 轴对齐矩形范围 | `BMap.Rectangle` | `BMap.Bounds` |
| 半径范围 | `BMap.Circle` | 圆心 Point + 米制半径 |

四者都由 `map.addOverlay/removeOverlay` 管理，支持样式、zIndex、mass clear、点击事件和编辑开关。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const center = new BMap.Point(116.404, 39.915);
map.centerAndZoom(center, 14);

const linePoints = [
  new BMap.Point(116.392, 39.906),
  center,
  new BMap.Point(116.418, 39.924),
];
const areaPoints = [
  new BMap.Point(116.397, 39.908),
  new BMap.Point(116.413, 39.908),
  new BMap.Point(116.410, 39.922),
];

const polyline = new BMap.Polyline(linePoints, {
  strokeColor: '#1677ff',
  strokeWeight: 4,
  strokeOpacity: 0.8,
  strokeStyle: 'solid',
  enableClicking: true,
  enableMassClear: true,
});
const polygon = new BMap.Polygon(areaPoints, {
  strokeColor: '#13a8a8',
  fillColor: '#13a8a8',
  fillOpacity: 0.25,
  enableClicking: true,
});
const rectangle = new BMap.Rectangle(
  new BMap.Bounds(
    new BMap.Point(116.390, 39.904),
    new BMap.Point(116.420, 39.926),
  ),
  { strokeColor: '#d93025', fillColor: '#d93025', fillOpacity: 0.1 },
);
const circle = new BMap.Circle(center, 800, {
  strokeColor: '#722ed1',
  fillColor: '#722ed1',
  fillOpacity: 0.16,
});

map.addOverlay(polyline);
map.addOverlay(polygon);
map.addOverlay(rectangle);
map.addOverlay(circle);

const onPolygonClick = (event) => {
  console.log('区域点击', event.latLng);
};
const onLineUpdate = (event) => {
  console.log('路径已变化', event.action);
};
polygon.addEventListener('click', onPolygonClick);
polyline.addEventListener('lineupdate', onLineUpdate);

// 路径、范围和圆参数更新。
polyline.setPath([linePoints[0], linePoints[2]]);
console.log(polyline.getPath(), polyline.getBounds());
polygon.setPath([...areaPoints, new BMap.Point(116.397, 39.908)]);
console.log(polygon.getPath(), polygon.getBounds());
rectangle.setBounds(new BMap.Bounds(
  new BMap.Point(116.394, 39.907),
  new BMap.Point(116.416, 39.923),
));
console.log(rectangle.getBounds());
circle.setCenter(new BMap.Point(116.406, 39.916));
circle.setRadius(1200);
console.log(circle.getCenter(), circle.getRadius(), circle.getBounds());

// 高频样式、层级、局部点位和编辑开关。
polyline.setStrokeColor('#0958d9');
polyline.setStrokeOpacity(0.65);
polyline.setStrokeWeight(6);
polyline.setStrokeStyle('dashed');
console.log(
  polyline.getStrokeColor(),
  polyline.getStrokeOpacity(),
  polyline.getStrokeWeight(),
  polyline.getStrokeStyle(),
);
polygon.setFillColor('#08979c');
polygon.setFillOpacity(0.35);
console.log(polygon.getFillColor(), polygon.getFillOpacity());
rectangle.setStrokeColor('#cf1322');
rectangle.setFillColor('#ffccc7');
circle.setStrokeColor('#531dab');
circle.setFillColor('#d3adf7');
circle.hide();
console.log(circle.isVisible());
circle.show();
polyline.setPositionAt(0, new BMap.Point(116.393, 39.907));
polygon.setPositionAt(1, new BMap.Point(116.414, 39.909));
polyline.setZIndex(20);
console.log(polyline.getMap());

polyline.enableEditing();
polygon.enableEditing();
rectangle.enableEditing();
circle.enableEditing();

function dispose() {
  polyline.disableEditing();
  polygon.disableEditing();
  rectangle.disableEditing();
  circle.disableEditing();
  polygon.removeEventListener('click', onPolygonClick);
  polyline.removeEventListener('lineupdate', onLineUpdate);
  map.removeOverlay(circle);
  map.removeOverlay(rectangle);
  map.removeOverlay(polygon);
  map.removeOverlay(polyline);
  map.destroy();
}
```

编辑控件、事件坐标和视觉样式仍需在有效 AK 的 `v=4.0` 页面在线验证。

## 核心 API

### 路径与几何

- Polyline：`setPath/getPath`、`getBounds`、`setPositionAt(index, point)`。
- Polygon：`setPath/getPath`、`getBounds`、`setPositionAt(index, point, deep?)`。`deep` 是环序号；单环 Polygon 可以省略，实现会自动按第一环处理。
- Rectangle：`setBounds/getBounds`。
- Circle：`setCenter/getCenter`、`setRadius/getRadius`、`getBounds`；半径单位为米。

Polyline 与 Polygon 的构造参数、`setPath()` 和 `updateByGeoJSON()` 都支持对应的 GeoJSON Feature、Geometry 或 FeatureCollection；多几何时取第一条线或第一个面，并保存 Feature 的 `properties`。需要保留全部 MultiLineString/MultiPolygon 分量时，不要依赖“取第一个”的快捷解析，先在业务侧拆分后分别创建覆盖物。

`getBounds()` 的返回坐标系在四类之间并不一致：Polyline 与 Rectangle 会按实例的 `coordType` 回转，Polygon 与 Circle 没有重写该方法，无论 `coordType` 传什么都只返回 BD09。使用非 BD09 坐标类型时，不要把 Polygon/Circle 的 `getBounds()` 结果直接与输入坐标比对。

### 样式与层级

Polyline 公开成对的 stroke color、opacity、weight、style setter/getter；Polygon、Rectangle、Circle 还公开 fill color 与 fill opacity。四者均有 `setZIndex`、`enableMassClear/disableMassClear` 与 `getMap`。

`strokeOpacity`、`fillOpacity` 使用 0–1；`strokeWeight` 使用像素。需要无填充面时给 fill color 传空字符串。四者还继承 `show/hide/isVisible`，临时显隐使用这组方法；组件卸载仍要精确移除实例。

不传 `strokeColor` / `fillColor` 时，4.0 的默认颜色跟随 UI 主题：描边取 CSS 变量 `--bmap-color-primary`（缺省 `#1677ff`），填充取 `--bmap-color-primary-bg`（缺省 `#eaf1ff`）。默认值在构造覆盖物时求值一次，之后再改 CSS 变量不会回溯已创建的实例；需要固定配色就显式传色值。换肤方式见 [UI 版本与主题](ui-version-and-theme.md)。

### 沿线方向箭头：优先用 strokeTexture

`Polyline` 的 `strokeTexture` 选项沿线重复铺贴图片，可用于方向箭头。它只在 WebGL 渲染模式下生效：`url` 必填，`width`、`height` 默认 16 像素，实现按 `strokeWeight` 与纹理宽高比计算重复步长。

```javascript
const directionalLine = new BMap.Polyline(
  [
    new BMap.Point(116.392, 39.906),
    new BMap.Point(116.418, 39.924),
  ],
  {
    strokeColor: '#1677ff',
    strokeWeight: 8,
    strokeTexture: {
      url: 'https://your-cdn.example.com/arrow.png',
      width: 16,
      height: 24,
    },
  },
);
map.addOverlay(directionalLine);

function removeDirectionalLine() {
  map.removeOverlay(directionalLine);
}
```

`strokeTexture` 必须在构造时传入：纹理在覆盖物挂载时生成，没有 `setStrokeTexture`。挂载后 `setStrokeColor`、`setStrokeOpacity`、`setStrokeStyle` 会重新生成纹理，但 `setStrokeWeight` 不走这条路径，改完线宽纹理步长不会跟着更新；需要变更线宽时重建 Polyline。渲染退化到 canvas 或 dom 时纹理不生效。实践建议（非 API 约束）：不要把方向提示做成唯一的信息载体。

`BMap.IconSequence` 只用于承接已有 3.0 折线箭头代码，帮助存量项目平滑升级；4.0 新代码不展开也不推荐该写法，统一使用 `strokeTexture`。

### 编辑

四类图形都发布 `enableEditing/disableEditing`。必须先 `map.addOverlay` 再 `enableEditing()`：顶点数不足 2 时该调用被**静默忽略**（不报错、也不会在后续补开），而 Rectangle 与 Circle 的顶点是在挂载时才生成的，所以对这两类在挂载前开启编辑等于没调。组件退出时先关闭编辑，再解绑事件并移除覆盖物。只读展示不要开启编辑，以免额外创建编辑顶点和交互状态。

## 事件或回调

四类图形共享高频事件：

| 类别 | 事件 |
|---|---|
| 鼠标 | `click`、`dblclick`、`rightclick`、`rightdblclick`、`mousedown`、`mouseup`、`mousemove`、`mouseover`、`mouseout` |
| 数据变化 | `lineupdate` |
| 编辑 | `editstart`、`editend`、`linevertexdragstart`、`linevertexdragging`、`linevertexdragend`、`linevertexdel` |
| 生命周期 | `remove` |

编辑事件只有开启编辑后才有业务意义。所有 handler 必须保留同一函数引用用于解绑。

## 资源清理

1. 停止业务动画、定时更新与异步回调。
2. 对已开启编辑的图形调用 `disableEditing()`。
3. 用原 handler 解绑每个图形事件。
4. 逐个调用 `map.removeOverlay(savedOverlay)`；共享 Map 不使用 `clearOverlays()` 代替所有权管理。
5. 只有组件拥有 Map 时才调用 `map.destroy()`。

## 常见错误

- 把 Circle 半径当成像素；公开语义是米。
- 创建图形后忘记 `map.addOverlay`。
- 在图形尚未挂载时开启编辑（Rectangle/Circle 尤其明显，调用被静默忽略），或卸载时不关闭编辑。
- 假定不传色值时描边是黑、填充是白：4.0 默认跟随主题色。
- 在非 BD09 坐标类型下把 Polygon/Circle 的 `getBounds()` 当成已回转到输入坐标系。
- 把 Polygon `setPositionAt` 的第三个参数当成别的语义：它是环序号 `deep`。
- 用新的匿名函数解绑 `click`、`lineupdate` 或编辑事件。
- 把临时 `hide()` 当成卸载；组件退出仍要 `map.removeOverlay`。
- 依赖 `map.clearOverlays()` 管理多个组件的覆盖物所有权。
- 挂载后调用 `setStrokeWeight` 却期待 `strokeTexture` 的重复步长同步更新。
- 多环 Polygon 传成了单层数组，或业务数据坐标系与全局 coordType 配置不一致。
