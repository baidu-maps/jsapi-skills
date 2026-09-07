# 行政区边界与区域填色

## 何时读取

查询省市区县边界坐标、在地图上勾勒或填色行政区、做行政区点击交互时读取。POI 检索与地址解析在 [检索与地理编码](search-and-geocoding.md)，IP 城市定位在 [定位与坐标转换](geolocation-and-convertor.md)，给瓦片图层加行政区掩膜在 [瓦片与地图服务图层](tile-and-service-layers.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | 选择 | 说明 |
|---|---|---|
| 只是在地图上画/填色行政区 | `BMap.DistrictLayer` | 4.0 推荐入口，自带边界数据与交互事件 |
| 需要拿到原始边界坐标串自行处理 | `BMap.Boundary` + Polygon | 唯一能取回坐标串的入口，4.0 下构造会打印弃用告警 |
| 短生命周期路由组件里的行政区展示 | `BMap.Boundary` + Polygon + active token | DistrictLayer 有迟到挂载边界，见本页说明 |
| 给瓦片图层做行政区掩膜 | `BMap.Boundary` 取串 → `addBoundary` | 掩膜要的就是坐标字符串 |

4.0 下构造 `BMap.Boundary` 会打印弃用告警并推荐 `BMap.DistrictLayer`。这条告警是预期行为，不代表 Boundary 已经不可用：只是在地图上画行政区时优先 DistrictLayer，需要坐标串或需要可控卸载时仍然用 Boundary。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 10);

const districtLayer = new BMap.DistrictLayer({
  name: '北京市',
  kind: 0,
  autoViewport: true,
  strokeColor: '#1677ff',
  fillColor: '#1677ff',
  fillOpacity: 0.15,
  onComplete(geojson) {
    console.log('行政区要素数', geojson.features.length);
  },
});

const onDistrictClick = (event) => console.log('点击行政区', event);
districtLayer.addEventListener('click', onDistrictClick);
map.addLayer(districtLayer);

function dispose() {
  districtLayer.removeEventListener('click', onDistrictClick);
  map.removeLayer(districtLayer);
  map.destroy();
}
```

行政区名称能否解析、要素数量与坐标精度都由在线服务决定。`DistrictLayer` 存在异步数据请求，短生命周期限制见下文。

## 核心 API

### Boundary

`BMap.Boundary.get(name, callback)` 异步返回一个含 `boundaries` 数组的对象，数组元素是分号分隔的 `lng,lat` 坐标串。先显式解析成 Point 数组再构造 Polygon，不要把坐标串直接交给 Polygon。行政区是否有在线数据以实际回调为准，不作静态覆盖范围承诺；只画行政区时优先使用 DistrictLayer。

返回的环可能多于一个（飞地、多段边界），必须遍历整个 `boundaries`；点数少于 3 的环不构成多边形，跳过。回调之外没有可用结果，也没有公开的请求取消入口，卸载后用 active 标记忽略迟到回调。

### DistrictLayer

`BMap.DistrictLayer` 通过 `map.addLayer` / `map.removeLayer` 挂载，发布 `click`、`mouseover`、`mouseout` 事件，边界数据由图层自己请求。构造 options 可传 `onComplete(geojson)`，在数据解析并绘制后收到 BD09MC GeoJSON。

它的异步请求可能在 `removeLayer` 之后迟到并重新挂载内部图层。短生命周期的路由组件因此改用 Boundary + Polygon + active token；DistrictLayer 适合与 Map 同生命周期的行政区底图。

## 事件或回调

- Boundary：只有 `get` 的回调，先判 `result` 与 `result.boundaries` 非空再遍历。
- DistrictLayer：`click`、`mouseover`、`mouseout`；使用具名 handler 成对 `addEventListener`/`removeEventListener`。

## 资源清理

1. Boundary：让 active token 失效 → 逐个 `map.removeOverlay(polygon)` → 清空业务数组。Boundary 实例本身没有公开销毁方法。
2. DistrictLayer：解绑事件 → `map.removeLayer(districtLayer)`；因迟到挂载边界，把它与 Map 同生命周期。
3. 只有组件拥有 Map 时才调用 `map.destroy()`。

## 常见错误

- 把 `boundaries[0]` 当作唯一边界，漏掉多环行政区。
- 把分号坐标串直接传给 `BMap.Polygon` 构造。
- 把 Boundary 的弃用告警当成功能失效，或反过来在只需要画图时不用 DistrictLayer。
- 在短生命周期组件里使用 DistrictLayer，卸载后被迟到请求重新挂载。
- 丢弃 Boundary 生成的 Polygon 引用，之后无法精确移除。
- 把行政区名称直接交给瓦片图层的 `addBoundary()`，它不会自行查询边界。
