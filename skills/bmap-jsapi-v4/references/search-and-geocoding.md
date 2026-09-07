# 检索与地理编码

## 何时读取

实现关键词/周边/范围检索、输入建议、地址与坐标互转，并需要正确处理异步结果、状态码、自动渲染和卸载时读取。行政区边界在 [行政区边界与区域填色](administrative-district.md)，浏览器定位与 IP 城市在 [定位与坐标转换](geolocation-and-convertor.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | API | 结果边界 |
|---|---|---|
| POI 关键词、周边或范围检索 | `BMap.LocalSearch` | 在 `onSearchComplete` 中先读 `getStatus()` |
| 输入框建议 | `BMap.Autocomplete` | 正式流程必须提供可解析的 input；实例与页面壳层保持同生命周期 |
| 地址 → Point | `BMap.Geocoder.getPoint` | 调用前校验非空；已处理的无结果响应可回调 `null` |
| Point → 地址信息 | `BMap.Geocoder.getLocation` | 失败或参数非法时回调 `null` |

`renderOptions.map` 会让 LocalSearch 自动创建标注并可调整视野；不传 map 时只消费结果回调，由业务自行渲染。短生命周期组件还必须把构造参数 `location` 设为 Point 或城市字符串，而不是共享 Map：BaseSearch 会为 Map location 注册匿名 `language_change` 监听，没有公开的解绑或销毁入口。active/generation token 只能阻止业务回调，不能解绑该监听或阻止 JSAPI 自动渲染器处理迟到响应。自动渲染模式只用于 Map 与服务同生命周期且请求不重叠直到完成的页面。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const center = new BMap.Point(116.404, 39.915);
map.centerAndZoom(center, 13);

let active = true;
let generation = 0;
let activeSearch = null;
const poiMarkers = [];

function clearPoiMarkers() {
  for (const marker of poiMarkers) map.removeOverlay(marker);
  poiMarkers.length = 0;
}

function searchPois(keyword) {
  const normalizedKeyword = keyword.trim();
  if (!normalizedKeyword) {
    generation += 1;
    activeSearch?.clearResults();
    activeSearch = null;
    clearPoiMarkers();
    return;
  }
  const current = ++generation;
  activeSearch?.clearResults();

  const search = new BMap.LocalSearch(center, {
    onSearchComplete(results) {
      if (
        !active
        || current !== generation
        || search.getStatus() !== BMAP_STATUS_SUCCESS
      ) {
        return;
      }
      const result = Array.isArray(results) ? results[0] : results;
      clearPoiMarkers();
      for (let index = 0; index < result.getCurrentNumPois(); index += 1) {
        const poi = result.getPoi(index);
        if (!poi) continue;
        const marker = new BMap.Marker(poi.point);
        poiMarkers.push(marker);
        map.addOverlay(marker);
      }
      if (poiMarkers.length > 0) {
        map.setViewport(poiMarkers.map((marker) => marker.getPosition()));
      }
    }
  });
  activeSearch = search;
  search.search(normalizedKeyword);
}

searchPois('咖啡');

function dispose() {
  active = false;
  generation += 1;
  activeSearch?.clearResults();
  activeSearch = null;
  clearPoiMarkers();
  map.destroy();
}
```

`onSearchComplete` 的状态、条目数量、名称与坐标都由在线服务返回，必须对状态和空结果做防御处理。

### 周边、范围与分页检索

同一个 LocalSearch 实例一次只推进一种检索流程；`gotoPage(page)` 复用最近一次关键词/周边/范围查询，因此应在首批结果完成后由分页 UI 调用。下面使用城市字符串作为 location，避免给短生命周期实例注册共享 Map 的匿名语言监听：

```javascript
let listActive = true;
const listSearch = new BMap.LocalSearch('北京市', {
  onSearchComplete(results) {
    if (!listActive || listSearch.getStatus() !== BMAP_STATUS_SUCCESS) return;
    const result = Array.isArray(results) ? results[0] : results;
    const rows = [];
    for (let index = 0; index < result.getCurrentNumPois(); index += 1) {
      const poi = result.getPoi(index);
      if (poi) rows.push({ title: poi.title, point: poi.point });
    }
    console.log(rows);
  },
});

listSearch.setPageCapacity(20);

function searchNearby(keyword, center, radiusMeters = 2000) {
  listSearch.clearResults();
  listSearch.searchNearby(keyword, center, radiusMeters);
}

function searchInBounds(keyword, southWest, northEast) {
  listSearch.clearResults();
  listSearch.searchInBounds(keyword, new BMap.Bounds(southWest, northEast));
}

function gotoPage(pageIndex) {
  listSearch.gotoPage(pageIndex);
}

function disposeListSearch() {
  listActive = false;
  listSearch.clearResults();
}
```

周边半径单位为米；范围检索使用 `BMap.Bounds`。`setPageCapacity` 在发起检索前设置；当前 4.0 实现只接受 `0 <= pageIndex < result.getNumPages()`，所以 `gotoPage` 使用从 `0` 开始的页索引。

输入建议 → 地址编码 → 自定义 Icon Marker → InfoWindow 的组合流程需要页面先提供 `<input id="search-input" />` 与 `#map`：

```javascript
const searchInputCandidate = document.getElementById('search-input');
if (!(searchInputCandidate instanceof HTMLInputElement)) {
  throw new Error('未找到输入框 #search-input');
}
const searchInput = searchInputCandidate;

const map = new BMap.Map('map');
const center = new BMap.Point(116.404, 39.915);
map.centerAndZoom(center, 13);

const markerIconUrl = 'data:image/svg+xml;charset=UTF-8,' + encodeURIComponent(
  '<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 32 32"><path fill="#d93025" d="M16 1a11 11 0 0 0-11 11c0 8 11 19 11 19s11-11 11-19A11 11 0 0 0 16 1z"/><circle cx="16" cy="12" r="4" fill="white"/></svg>',
);
const icon = new BMap.Icon(
  markerIconUrl,
  new BMap.Size(32, 32),
  { anchor: new BMap.Size(16, 32) },
);
const marker = new BMap.Marker(center, { icon });
const infoWindow = new BMap.InfoWindow('已选择的位置');
map.addOverlay(marker);

let autocompletePageActive = true;
let generation = 0;
let resolvedPoint = null;
const geocoder = new BMap.Geocoder();

const onMarkerClick = () => {
  if (resolvedPoint) map.openInfoWindow(infoWindow, resolvedPoint);
};
marker.addEventListener('click', onMarkerClick);

const autocomplete = new BMap.Autocomplete({
  input: searchInput,
  location: center,
  onConfirm(item) {
    if (!autocompletePageActive) return;
    const value = item.value;
    const address = [
      value.province,
      value.city,
      value.district,
      value.street,
      value.business,
    ].filter(Boolean).join('');
    if (!address.trim()) return;

    const current = ++generation;
    geocoder.getPoint(address, (point) => {
      if (!autocompletePageActive || current !== generation) return;
      const targetPoint = point ?? value.location ?? null;
      if (targetPoint === null) return;
      resolvedPoint = targetPoint;
      marker.setPosition(targetPoint);
      map.centerAndZoom(targetPoint, 16);
      map.openInfoWindow(infoWindow, targetPoint);
    }, value.city);
  },
});

function destroyAutocompletePage() {
  if (!autocompletePageActive) return;
  autocompletePageActive = false;
  generation += 1;
  marker.removeEventListener('click', onMarkerClick);
  map.closeInfoWindow();
  map.removeOverlay(marker);
  window.removeEventListener('pagehide', onAutocompletePageHide);
  map.destroy();
}

const onAutocompletePageHide = (event) => {
  if (!event.persisted) destroyAutocompletePage();
};
window.addEventListener('pagehide', onAutocompletePageHide);
```

Autocomplete 与页面壳层同生命周期。上面的页面终止函数只在非 BFCache 的 `pagehide` 中让业务 Geocoder 回调失效，并清理业务持有的 Map、Marker、InfoWindow 与 DOM 监听；进入 BFCache 时保留原实例供后退恢复。它不调用 Autocomplete.dispose。`dispose()` 会删除实例的非函数字段，但并发 JSONP 回调仍可能读取 `_options.input`；哪个输入事件对应最后一个在途回调没有可靠的一一映射，公开取消全部在途建议请求的 API 在当前实现中不存在。因此复用该页面级实例，真正离开页面时由浏览器销毁 Autocomplete 上下文。

## 核心 API

`BMap.LocalSearch` 的 location 可以是 Map、Point 或城市字符串；提供 `search`、`searchNearby`、`searchInBounds`、分页、结果读取和 `clearResults`。单关键词回调得到 LocalResult，多关键词得到数组；状态在回调之前写入，所以回调里先比较 `getStatus()` 与 `BMAP_STATUS_SUCCESS`。设置 `renderOptions.map` 后会自动创建标注并按配置调整视野，`clearResults` 移除它自己持有的覆盖物和面板内容，但不取消在途请求，也不阻止迟到响应重新触发自动渲染。构造时把共享 Map 传给 location 还会留下无法解绑的语言监听；短生命周期实例传 Point 或城市字符串，用纯回调模式。

`BMap.Autocomplete` 的正式流程必须绑定有效 `input`；4.0 新代码优先使用 options 的 `onSearchComplete`、`onConfirm`、`onHighlight`。只有 `input` 能解析成 DOM 元素时才会更新结果并触发 `onSearchComplete`，所谓“无 input 的纯数据 search”实际不生效。`dispose()` 不是请求取消 API，只在确认最近一次请求已完成后调用。`onConfirm` 拿到的 `item.value.streetNumber` 在 4.0 恒为空字符串，拼地址时不必带；服务返回的 `value.location` 会被包装成 `BMap.Point`，可作为 Geocoder 正向解析失败时的兜底。

`BMap.Geocoder.getPoint(address, callback, city?)` 做正向解析，已识别的无结果响应可回调 null；`getLocation(point, callback, options?)` 做逆向解析，返回地址、地址组件与附近 POI，失败时回调 null。所有返回值都必须在回调内判空后使用。

正向解析并不保证一定会回调：实际遇到空白地址或无法识别的响应时可能直接返回。调用前先校验 `address.trim()`，等待状态用 generation/active token 加业务超时兜底，不要假定每次调用都会完成回调。

## 事件或回调

- LocalSearch：用 `onSearchComplete` 或 `setSearchCompleteCallback`，先读 `getStatus()`；自动渲染时还可用 `onMarkersSet` 等回调。
- Autocomplete：优先使用 options 回调；`onConfirm` 的参数是选中条目，`onHighlight` 依次收到当前与之前条目。
- Geocoder：结果只在异步回调内可用，不要在调用下一行同步读取。
- 所有服务回调都用 `active`/generation token 防止组件卸载或新请求发出后处理旧结果；允许卸载/重叠请求的服务不要传 `renderOptions.map`。

## 资源清理

- LocalSearch：可卸载组件使用纯回调模式；先让业务回调失效，调用 `clearResults()` 清结果，再移除业务持有的 Marker/InfoWindow。自动渲染模式只用于 Map 一直存活、且请求不重叠的场景。
- LocalSearch：短生命周期实例用 Point/城市字符串作为 location；Map location 只用于与 Map 同生命周期的复用实例。
- Autocomplete：与页面壳层同生命周期；不要把 `onSearchComplete` 当作全部并发 JSONP 已完成的证明，也不要提供组件级 dispose 方案。
- Geocoder 的公开取消/销毁方法在当前实现中不存在；卸载后用 active 标记忽略迟到回调。
- 最后调用 `map.destroy()`；不要把服务清理等同于 Map 销毁。

## 常见错误

- 在 `onSearchComplete` 中不检查 `getStatus()` 就读取第一个 POI。
- 多关键词检索仍把 results 固定当作单个 LocalResult。
- 设置 `renderOptions.map` 后又为相同结果手工创建一套 Marker，导致重复渲染和所有权混乱。
- 组件卸载只销毁 Map，不清 LocalSearch 结果、业务覆盖物或 Autocomplete。
- 把 Autocomplete 的 `input` 当作选择器语法；公开类型只承诺元素或元素 ID。
- 假设地址一定能解析成功且一定返回固定内容。
- 用 active token 包裹自动渲染 LocalSearch，就误以为迟到响应不会再修改 Map。
- 短生命周期 LocalSearch 把共享 Map 作为 location，误以为纯回调模式不会注册 Map 监听。
- 输入后立即调用 Autocomplete.dispose，导致迟到 JSONP 回调读取已删除的 `_options`。
