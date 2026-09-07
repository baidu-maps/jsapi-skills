# 路线与公交线检索

## 何时读取

规划驾车、步行、骑行或公交路线，查询具体公交线路与站点，并需要区分自动渲染与纯回调、判断状态码、设计清理所有权时读取。浏览器定位与坐标转换在 [定位与坐标转换](geolocation-and-convertor.md)，POI 检索与地理编码在 [检索与地理编码](search-and-geocoding.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | API | 调用重点 |
|---|---|---|
| 驾车路线与途经点 | `BMap.DrivingRoute` | 起终点用 Point/LocalResultPoi；`search` options 可传 waypoints |
| 步行路线 | `BMap.WalkingRoute` | Point/LocalResultPoi 最稳妥 |
| 骑行路线 | `BMap.RidingRoute` | Point/LocalResultPoi 最稳妥 |
| 市内/跨城公交方案 | `BMap.TransitRoute` | 起终点必须传 Point 或 LocalResultPoi，见本页说明 |
| 具体公交线与站点 | `BMap.BusLineSearch` | 先 getBusList，再把同实例结果项传给 getBusLine；`clearResults` 需判存在 |

路线构造时的 location 可传 Map、Point 或城市字符串（城市字符串依赖在线城市解析，需实际页面验证）；只有 `renderOptions.map` 才授权服务自动创建路线与标注。不设置 `renderOptions.map` 是纯回调模式，但若 location 仍传共享 Map，服务基类会注册匿名 `language_change` 监听，没有公开的解绑或销毁入口。可卸载组件必须同时使用纯回调模式和 Point/城市字符串 location；Map location 只用于与 Map 同生命周期的复用实例。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const start = new BMap.Point(116.391, 39.910);
const waypoint = new BMap.Point(116.410, 39.920);
const end = new BMap.Point(116.431, 39.931);
map.centerAndZoom(start, 13);

let active = true;
let generation = 0;
let routePolyline = null;
const drivingRoute = new BMap.DrivingRoute(start, {
  policy: BMAP_DRIVING_POLICY_AVOID_CONGESTION,
  onSearchComplete(results) {
    if (
      !active
      || requestGeneration !== generation
      || drivingRoute.getStatus() !== BMAP_STATUS_SUCCESS
      || results.getNumPlans() === 0
    ) {
      return;
    }
    const plan = results.getPlan(0);
    const path = [];
    for (let index = 0; index < plan.getNumRoutes(); index += 1) {
      const drivingSegment = plan.getRoute(index);
      path.push(...drivingSegment.getPath());
      console.log('本段导航步骤数', drivingSegment.getNumSteps());
    }
    if (routePolyline) map.removeOverlay(routePolyline);
    routePolyline = path.length >= 2
      ? new BMap.Polyline(path, { strokeColor: '#1677ff', strokeWeight: 6 })
      : null;
    if (routePolyline) {
      map.addOverlay(routePolyline);
      map.setViewport(path);
    }
    console.log(plan.getDistance(true));
  },
});

const requestGeneration = ++generation;
drivingRoute.search(start, end, { waypoints: [waypoint] });

function dispose() {
  active = false;
  generation += 1;
  drivingRoute.clearResults();
  if (routePolyline) map.removeOverlay(routePolyline);
  routePolyline = null;
  map.destroy();
}
```

路线状态、方案数、距离和自动绘制结果必须在带有效 AK 的 `v=4.0` 页面在线验证。

## 核心 API

`BMap.DrivingRoute` 支持 Point 或 LocalResultPoi 起终点，search options 的 `waypoints` 为 Point 数组。驾车成功路径上状态在回调之前写入。设置 `renderOptions.map` 后会绘制路线并按 `autoViewport` 调整视野，`clearResults` 关闭信息窗口、移除该路线服务持有的覆盖物并清理面板，但不取消在途请求，也不阻止迟到响应重新触发自动渲染。纯回调模式省略 `renderOptions.map`，读取 result 后由业务决定如何展示。

`clearResults()` 只在异步路线模块加载完成后才有实际动作：同步壳上它是空实现，构造后立刻调用既不清面板也不移覆盖物，且不会报错。它同时会把内部状态清空，所以 **`clearResults()` 之后 `getStatus()` 返回 `undefined`**，不能再用它判断上一次检索的结果。

迟到响应不是「叠加一层」：服务收到原始数据后的第一步就是 `clearResults()`，也就是先清掉当前已渲染的路线再重绘。自动渲染模式下用 active token 包住业务代码并不能阻止这次清空 + 重绘。

驾车还有两个与版本相关的选项：`enableTraffic` 的默认值在 4.0 下是 `true`（渲染路况彩色线），显式传布尔值才会覆盖；`alternatives` 只在 4.0 下会拼进请求。

`BMap.WalkingRoute`、`BMap.RidingRoute` 与 `BMap.TransitRoute` 均公开 search、getResults、getStatus、clearResults 和呈现回调。步行/骑行与驾车共用同一套状态与清理链；公交路线有自己的结果解析和覆盖物移除实现。

`renderOptions.panel` 的 DOM 是 JSAPI 内部实现，业务代码不要依赖内部 class 名或节点结构做二次加工；需要稳定的自定义样式时使用纯回调模式自行渲染。

路线实例公开 `setPolylineStyle(style)`，会增量合并到 `renderOptions.polylineStyle`，并原地刷新已经绘制的 Polyline，无需重新检索。`polylineStyle` 支持公共基础字段，以及 `transit`、`walking`、`driving`、`riding`、`highlight`、`jumpLine`、`decorate` 分桶；TruckRoute 与 DrivingRouteLine 使用 LineLayer，应改配 `renderOptions.lineLayerStyle`。

驾车、步行结果按 plan → route → step 遍历并读取步骤描述；公交结果按 plan → line 遍历并读取线路标题与上下车站，适合渲染到自定义面板。

下面把其余路线与公交线入口拆成独立函数；业务只调用需要的函数，不要一次触发所有在线请求：

```javascript
const startPoint = new BMap.Point(116.391, 39.910);
const endPoint = new BMap.Point(116.431, 39.931);
let serviceActive = true;
let walkingGeneration = 0;
let ridingGeneration = 0;
let transitGeneration = 0;
let busLineGeneration = 0;
let activeWalkingRoute = null;
let activeRidingRoute = null;
let activeTransitRoute = null;
let activeBusLineSearch = null;

function searchWalking() {
  const current = ++walkingGeneration;
  activeWalkingRoute?.clearResults();
  const route = new BMap.WalkingRoute(startPoint, {
    onSearchComplete(results) {
      if (
        serviceActive
        && current === walkingGeneration
        && route.getStatus() === BMAP_STATUS_SUCCESS
        && results.getNumPlans() > 0
      ) {
        const plan = results.getPlan(0);
        for (let routeIndex = 0; routeIndex < plan.getNumRoutes(); routeIndex += 1) {
          const walkingSegment = plan.getRoute(routeIndex);
          for (let stepIndex = 0; stepIndex < walkingSegment.getNumSteps(); stepIndex += 1) {
            console.log(walkingSegment.getStep(stepIndex).getDescription(false));
          }
        }
      }
    },
  });
  activeWalkingRoute = route;
  route.search(startPoint, endPoint);
}

function searchRiding() {
  const current = ++ridingGeneration;
  activeRidingRoute?.clearResults();
  const route = new BMap.RidingRoute(startPoint, {
    onSearchComplete() {
      if (
        serviceActive
        && current === ridingGeneration
        && route.getStatus() === BMAP_STATUS_SUCCESS
      ) {
        console.log(route.getResults());
      }
    },
  });
  activeRidingRoute = route;
  route.search(startPoint, endPoint);
}

function searchTransit() {
  const current = ++transitGeneration;
  activeTransitRoute?.clearResults();
  const route = new BMap.TransitRoute(startPoint, {
    policy: BMAP_TRANSIT_POLICY_LEAST_TRANSFER,
    onSearchComplete(results) {
      if (
        serviceActive
        && current === transitGeneration
        && route.getStatus() === BMAP_STATUS_SUCCESS
        && results.getNumPlans() > 0
      ) {
        const plan = results.getPlan(0);
        console.log(plan.getDescription(false), plan.getDuration(true));
        for (let lineIndex = 0; lineIndex < plan.getNumLines(); lineIndex += 1) {
          const line = plan.getLine(lineIndex);
          console.log(line.getTitle(), line.getGetOnStop(), line.getGetOffStop());
        }
      }
    },
  });
  activeTransitRoute = route;
  route.search(startPoint, endPoint);
}

function clearBusLineSearch() {
  const service = activeBusLineSearch;
  activeBusLineSearch = null;
  if (service && typeof service.clearResults === 'function') {
    service.clearResults();
  }
}

function searchBusLine(keyword) {
  const normalizedKeyword = keyword.trim();
  const current = ++busLineGeneration;
  clearBusLineSearch();
  if (!normalizedKeyword) return;
  const service = new BMap.BusLineSearch('北京市', {
    onGetBusListComplete(result) {
      if (
        serviceActive
        && current === busLineGeneration
        && service.getStatus() === BMAP_STATUS_SUCCESS
        && result.getNumBusList() > 0
      ) {
        service.getBusLine(result.getBusListItem(0));
      }
    },
    onGetBusLineComplete(busLine) {
      if (
        serviceActive
        && current === busLineGeneration
        && service.getStatus() === BMAP_STATUS_SUCCESS
      ) {
        const polyline = busLine.getPolyline();
        const path = typeof polyline?.getPath === 'function' ? busLine.getPath() : [];
        const stations = [];
        for (let index = 0; index < busLine.getNumBusStations(); index += 1) {
          const station = busLine.getBusStation(index);
          stations.push({ name: station.name, position: station.position });
        }
        console.log(busLine.name, busLine.startTime, busLine.endTime, path, stations);
      }
    },
  });
  activeBusLineSearch = service;
  service.getBusList(normalizedKeyword);
}

function clearOptionalServices() {
  serviceActive = false;
  walkingGeneration += 1;
  ridingGeneration += 1;
  transitGeneration += 1;
  busLineGeneration += 1;
  activeWalkingRoute?.clearResults();
  activeRidingRoute?.clearResults();
  activeTransitRoute?.clearResults();
  clearBusLineSearch();
  activeWalkingRoute = null;
  activeRidingRoute = null;
  activeTransitRoute = null;
}
```

`TransitRoute.search` 和 `DrivingRoute.search` 的起终点传字符串时会得到 `INVALID_REQUEST`。步行与骑行会把字符串当关键字连同城市发出，结果依赖在线解析。四类路线统一传 Point 或 LocalResultPoi 才是可预期的写法。

公交检索的第三个参数（`opts.start_uid` / `opts.end_uid`）只有在 `transitSearch` 模块加载完成后才会被读取；模块加载前发起的检索会静默丢掉第三个参数，只保留起终点。

`TransitRoute.setPageCapacity(cp)` 的有效区间是 1..100，越界或非法值回落到 100。

`BMap.BusLineSearch` 是两阶段服务：`getBusList(keyword)` 返回 `void`，随后通过 `onGetBusListComplete` 或 `setGetBusListCompleteCallback` 收到 BusListResult，再把该结果中的 BusListItem 交给同一个实例的 `getBusLine(item)`。传入的 item 必须来自同一实例的上一次 `getBusList` 结果；校验失败是**静默返回**——不触发 `onGetBusLineComplete`，也不写状态码，`getStatus()` 感知不到，表现就是「回调再也不来了」。自动渲染时 `clearResults` 会清结果、面板和覆盖物。

`clearResults` 在 BusLineSearch 上的可用性和路线服务不同：同步壳的原型链里根本没有这个方法，只有 `buslineSearch` 模块加载完成后才存在。构造后立刻调用会抛 `TypeError`，所以清理代码要先判存在：

```javascript
function clearBusLineSearch(service) {
  if (service && typeof service.clearResults === 'function') {
    service.clearResults();
  }
}
```

`BusLine.getPath()` 内部直接访问折线对象，没有拿到线路几何数据时该对象是**空对象**（不是 `null`，判真值过不去），调用 `getPath()` 即抛 `TypeError`。读路径前先确认 `busLine.getPolyline()` 上真的有 `getPath` 方法。

注意 BusLineSearch 清理覆盖物时会调用 `map.clearOverlays()`，批量清除地图上所有可清除覆盖物，而不只是公交线自身。共享地图上优先用不设置 `renderOptions.map` 的纯回调模式；若采用自动渲染，必须把这个全局副作用纳入所有权设计。

## 事件或回调

- 四类路线：`onSearchComplete` 中先读实例 `getStatus()`，成功后再读方案数和 plan。
- `onSearchComplete` 不是无条件必达：设置了 `renderOptions.map` 后，如果起点或终点不明确，服务会接管为「地址候选选择」流程并直接返回，这一轮不触发 `onSearchComplete`。需要唯一出口就用纯回调模式。
- BusLineSearch：列表回调中先检查状态与数量，再调用 getBusLine；线路回调再次检查状态。`getBusLine` 的入参校验不通过时是静默返回，回调不会来。
- 新请求或组件卸载时使用 active/generation token 丢弃旧回调；这些服务在 4.0 公开接口中没有取消入口。

## 资源清理

- 自动渲染的 Driving/Walking/Riding/TransitRoute：仅用于 Map 保持存活且请求不重叠直到完成的页面；`clearResults()` 不视为请求取消。
- 纯回调路线：移除业务自己创建的 Polyline、Marker、InfoWindow 和监听器。
- BusLineSearch：纯回调模式清业务对象；自动渲染模式调用 clearResults 前先确认允许其 `clearOverlays()` 的全局副作用，并且先判断该方法是否已存在。
- 使用 `renderOptions.map` 时路线服务会向地图注册内部监听（除语言切换外还有 POI 选中监听），而没有路线级的公开解绑或销毁 API；Map 需要继续存活时，短生命周期功能优先纯回调模式。
- 即使不设置 renderOptions.map，构造 location 传共享 Map 也会注册匿名语言监听；该监听没有公开解绑路径，短生命周期实例应改传 Point 或城市字符串。
- 纯回调模式需要持有业务创建的 Polyline；先让 generation 失效并移除 Polyline，最后调用 `map.destroy()`。

## 常见错误

- 把构造参数 location=map 误认为会自动绘制，漏写 `renderOptions.map`；或在短生命周期纯回调服务中仍传共享 Map，留下匿名语言监听。
- 同时启用自动渲染并手工画同一条路线，导致重复覆盖物。
- 用 active token 包裹自动渲染路线，就误以为迟到响应不会再修改 Map：迟到响应会先清空已有渲染再重绘。
- 构造后立刻调用 `clearResults()`，以为已经清干净：异步模块加载完成前它是空实现。
- 在 `clearResults()` 之后用 `getStatus()` 判断上一次检索结果，此时它已是 `undefined`。
- 对 `BusLineSearch` 直接调用 `clearResults()`：该方法在模块加载完成前不存在，会抛 `TypeError`。
- 不检查 `getStatus()` 和 `getNumPlans()` 就读取 `getPlan(0)`。
- 给驾车或公交传字符串起终点，实际会得到 `INVALID_REQUEST`。
- 直接调用 `BusLine.getPath()`，没有几何数据时会抛 `TypeError`。
- 依赖 `renderOptions.panel` 的内部 DOM 结构或 class 名做二次加工。
- 给 TruckRoute 或 DrivingRouteLine 调 `setPolylineStyle()`，忽略它们使用 LineLayer 渲染。
- 从一个 BusLineSearch 实例拿 BusListItem，却交给另一个实例 getBusLine：校验失败是静默返回，没有任何错误信号。
- 在共享地图上调用自动渲染 BusLineSearch.clearResults，误删其他组件覆盖物。
- 只清路线结果而不销毁最终 Map，或 Map 长期存活时反复创建自动渲染路线服务。
