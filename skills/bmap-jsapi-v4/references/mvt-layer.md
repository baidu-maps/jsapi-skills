# MVT 矢量瓦片图层

## 何时读取

需要加载 PBF 格式的矢量瓦片服务、给矢量瓦片写样式表达式、按要素做拾取高亮，或排查 MVT 网格与坐标对不上时读取。GeoJSON、DOM、行政区图层在 [数据图层](data-layers.md)；栅格/XYZ/WMS 服务在 [瓦片与地图服务图层](tile-and-service-layers.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | 做法 |
|---|---|
| 挂载与卸载 | `map.addLayer(mvtLayer)` / `map.removeLayer(mvtLayer)` |
| 按要素分层写样式 | `setStyle({ 图层名: { type, painter } })` |
| 拾取高亮 | `updateState('layerName_id', {...})` + 样式里的 `feature-state` 表达式 |
| 对接 Google/WebMercator 服务 | `gridModel: 1` + `transform` |
| 核心字段与事件载荷 | 本页“核心 API”与“事件或回调”章节 |

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 13);

const mvtLayer = new BMap.MVTLayer({
  tileUrlTemplate: 'https://example.com/mvt/[z]/[x]/[y].pbf',
  minZoom: 3,
  maxZoom: 18,
  gridModel: 1,
  transform: { source: 'EPSG3857', target: 'BD09MC' },
  idProperty: 'id',
});

mvtLayer.setStyle({
  roads: {
    type: 'polyline',
    painter: { strokeColor: '#1677ff', strokeWeight: 3 },
  },
  areas: {
    type: 'polygon',
    painter: {
      fillColor: [
        'case',
        ['boolean', ['feature-state', 'selected'], false],
        '#fa541c',
        '#69b1ff',
      ],
      fillOpacity: 0.45,
    },
  },
});

const onMVTClick = (event) => {
  const feature = event.value && event.value[0];
  mvtLayer.clearState();
  if (!feature) return;

  const stateKey = `${feature.layerName}_${feature.id}`;
  mvtLayer.updateState(stateKey, { selected: true }, false);
};
const onMVTMouseOut = () => mvtLayer.clearState();
mvtLayer.addEventListener('click', onMVTClick);
mvtLayer.addEventListener('mouseout', onMVTMouseOut);

map.addLayer(mvtLayer);

function removeMVTLayer() {
  mvtLayer.removeEventListener('click', onMVTClick);
  mvtLayer.removeEventListener('mouseout', onMVTMouseOut);
  mvtLayer.clearState();
  map.removeLayer(mvtLayer);
}

// 只有当前组件拥有 Map 时，销毁 Map 才能彻底释放 MVT 注册的监听。
function disposeOwnedMap() {
  removeMVTLayer();
  map.destroy();
}
```

## 核心 API

### 挂载与卸载

挂载与卸载统一走 `map.addLayer(mvtLayer)` / `map.removeLayer(mvtLayer)`：MVTLayer 带有 `isTileLayer` 标记，Map 的统一分发会识别它并转挂其内部 TileLayer。兼容入口 `addTileLayer`/`removeTileLayer` 在 4.0 会打印过时提示，新代码不要再用。

MVT 模板坐标使用 `[x]`、`[y]`、`[z]`、`[b]` 方括号占位；花括号写法不生效。

### 样式与状态

样式结构按 `{ type, painter }` 读取，颜色、线宽等字段放在 `painter` 中；平铺到 `point/polyline/polygon` 下不会被当前实现遍历。

状态键的字符串形式按 `layerName_id` 查询：源图层名为 `source-layer`、要素 id 为 `feature-id` 时必须传 `source-layer_feature-id`。实际项目从拾取结果的 `layerName` 与 `id` 组合出键，不要写死图层名。

上例的状态表达式按三个步骤求值：`feature-state` 读取 `selected`，`boolean` 在状态缺失时回落到 `false`，`case` 在选中色与默认色之间切换。`updateState` 写入的状态会进入样式求值上下文并触发重绘，不是只写状态、不改变视觉的伪高亮。

### 网格、坐标与业务 ID

| 数据服务 | `gridModel` | `transform` | 说明 |
|---|---:|---|---|
| 百度 Web 墨卡托网格，PBF 几何为 BD09MC | 省略或 `0` | 省略 | 默认 |
| 常见 Google/WebMercator `z/x/y` 网格，PBF 几何为 EPSG:3857 | `1` | `{ source: 'EPSG3857', target: 'BD09MC' }` | Google 网格 |
| Google 网格，但 PBF 几何已是 GCJ02 | `1` | `{ source: 'GCJ02', target: 'BD09MC' }` | `source` 必须与数据实际编码一致 |

`gridModel` 只能传数值，`0` 是百度网格、`1` 是 Google 网格，没有可用的静态枚举字段。`spanLevel` 是“地图 zoom 到请求瓦片 z”的偏移，默认 `0`，只有服务层级编号确实相差时才设置（`-1` 表示请求低一级瓦片）。百度网格不支持跨层级加载，传入的 `spanLevel` 会被强制改回 `0`，该参数只在 Google 网格下生效。

`idProperty` 必须等于 PBF `properties` 中真实且唯一（至少在各 source layer 内稳定唯一）的字段名。省略时解析器使用原始 feature id；配置后则直接读取 `properties[idProperty]`。字段不存在会得到 `undefined`，后续复合状态键无法稳定命中。`layers` 用于只解析指定 source layer；省略时不要假定服务只有一个图层，拾取后仍以实际 `feature.layerName` 组装状态键。

## 事件或回调

`click`、`dblclick`、`mousemove`、`mouseout`、`tilesloadstart`、`tilesloadend`。使用具名 handler 成对 `addEventListener`/`removeEventListener`。

真实服务还要在浏览器中验证 PBF 响应、CORS、坐标网格、图层名和样式字段。

## 资源清理

1. 同引用解绑业务事件。
2. `clearState()`。
3. `map.removeLayer(mvtLayer)`。

`map.removeLayer(mvtLayer)` 会清瓦片缓存、重置状态并解绑主要的鼠标/点击 handler，但每次挂载都会向 Map 追加无法解绑的内部监听，反复 add/remove 会持续累积。正式流程让一个 MVT 实例只挂载一次并与 Map 同生命周期；完整释放依赖拥有 Map 的组件最终调用 `map.destroy()`。

## 常见错误

- 新代码继续使用已 deprecated 的 `addTileLayer/removeTileLayer`，而不是统一的 `addLayer/removeLayer`。
- 把 `{x}/{y}/{z}` 风格占位符用在 MVT 模板上，实际只有方括号形式会被替换。
- 使用平铺样式字段，而不是当前实现读取的 `{ type, painter }` 结构。
- 状态键只写要素 id，漏掉 `layerName_` 前缀。
- 把 `gridModel: 1` 当成所有 MVT 服务的默认值，或让 `transform.source` 与 PBF 的真实坐标编码不一致。
- `idProperty` 指向不存在或不唯一的 properties 字段，导致 feature state 无法稳定命中。
- 认为复用同一实例就能安全反复 add/remove；每次挂载仍会累积无法解绑的 Map 监听。
