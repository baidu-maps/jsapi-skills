# 事件与生命周期

## 何时读取

为 Map、Marker、矢量覆盖物或普通图层绑定事件，排查重复触发、解绑失败、组件卸载泄漏或销毁顺序时读取。

## 快速选择

- Map 常见事件包括 `click`、`moveend`、`zoomend`、`tilesloaded`、`resize`。
- Marker 常见事件包括 `click`、`dragstart`、`dragging`、`dragend`。
- Polyline、Polygon、Circle、Rectangle 等图形覆盖物可监听 `click`、`mouseover`、`lineupdate`。
- 批量可视化图层按具体类确认 `dataparsed`、`mousemove`、`click` 等事件。
- `addEventListener` 与 `removeEventListener` 必须共享同一个函数对象。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const point = new BMap.Point(116.404, 39.915);

map.centerAndZoom(point, 15);

const onMapMoveEnd = () => {
  console.log('center', map.getCenter());
};

map.addEventListener('moveend', onMapMoveEnd);

function dispose() {
  map.removeEventListener('moveend', onMapMoveEnd);
  map.destroy();
}
```

## 核心 API

| 事件宿主 | 代表事件 |
|---|---|
| `BMap.Map` | `click`, `moveend`, `zoomend`, `tilesloaded`, `resize` |
| `BMap.Marker` | `click`, `dragstart`, `dragging`, `dragend` |
| 图形覆盖物 | `click`, `mouseover`, `lineupdate` |
| 批量可视化图层 | `dataparsed`, `mousemove`, `click`, `dblclick`, `rightclick`；以具体图层实现为准 |

`lineupdate` 不是编辑器专属事件，也不只在拖点时触发：`setPath`、`setPositionAt`、`setCenter`、`setRadius` 这类几何写入方法，以及 `show` / `hide` / `setZIndex` / 从地图移除，都会派发它。事件对象带 `overlay` 指回覆盖物，由显隐、层级和移除触发的那几次还带 `action`（`'show'`、`'hide'`、`'zIndex'`、`'remove'`）。因此不要把 `lineupdate` 当成「用户改了图形」的信号——自己调用 setter 也会触发，写回逻辑要防重入。

## 事件或回调

正确解绑：

```javascript
const onZoomEnd = () => console.log(map.getZoom());
map.addEventListener('zoomend', onZoomEnd);
map.removeEventListener('zoomend', onZoomEnd);
```

错误解绑：

```javascript
map.addEventListener('zoomend', () => console.log(map.getZoom()));
map.removeEventListener('zoomend', () => console.log(map.getZoom()));
```

第二段里的两个箭头函数是不同对象。解绑按函数身份匹配，内容相同并不等于能解绑。

## 资源清理

推荐顺序：停止新事件来源 → 解绑子对象事件 → 解绑 Map 事件 → 移除覆盖物/图层/控件 → 销毁对应服务或动画（若有明确 API）→ `map.destroy()`。

`map.destroy()` 会遍历并清空 Map 自身残留的监听器，所以漏掉一两个 Map 事件不会让监听器永久挂在已销毁的地图上。但它管不到三类东西：业务注册在覆盖物、控件、`document` / `window` 上的监听器；已经发出的检索、路线、定位等异步请求的回调；以及自己起的 `setInterval` / `requestAnimationFrame`。这些仍要按上面的顺序显式清理——把解绑写全的真正价值是让「销毁后迟到的回调」不再访问已释放对象，而不是替 `destroy()` 擦地。

没有 `destroy` 不等于这个类没有清理能力：部分类的清理入口叫 `dispose`、`remove`、`clearResults`，或者由 Map 侧的 `removeLayer` / `removeOverlay` 承担。各模块文档的“资源清理”给的是确实可用的方法；如果那里写明没有公开清理入口，就按“与 Map 同生命周期、由 `map.destroy()` 收尾”设计，不要自己发明 `destroy()` 调用。

## 常见错误

- 在渲染函数中反复绑定，未在下一次渲染或卸载时解绑。
- 用新匿名函数调用 `removeEventListener`。
- 假设 Map、Marker 与普通图层的同名事件一定具有相同字段。
- 先销毁 Map，再让子对象清理逻辑调用 Map。
- 把 `lineupdate` 当成「用户编辑了图形」，在回调里再调 `setPath` 造成自触发循环。
- 把回调执行当作网络数据、瓦片或样式一定成功。
