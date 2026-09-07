# 定位与坐标转换

## 何时读取

获取浏览器当前位置、按 IP 识别所在城市、把外部坐标系的点异步转换成百度坐标，或需要串联“定位 → 逆地理编码 → 后续服务”流程时读取。路线规划在 [路线与公交线检索](route-search.md)，Point/Pixel/Bounds 等本地几何计算在 [坐标与几何](coordinates-and-geometry.md)，地理编码在 [检索与地理编码](search-and-geocoding.md)。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | API | 调用重点 |
|---|---|---|
| 用户当前位置 | `BMap.Geolocation` | 回调中同时检查 `getStatus()` 与 result/point |
| IP 所在城市 | `BMap.LocalCity` | 成功字段来自在线服务，失败时回调空对象 |
| 坐标系批量转换 | `BMap.Convertor` | 只在 translate 异步回调内使用结果 |
| 定位按钮控件 | `BMap.GeolocationControl` | 见 [控件与右键菜单](controls-and-context-menu.md) |

这些都是一次性异步服务，4.0 的公开接口里没有请求取消入口。短生命周期组件用 active/generation token 丢弃迟到回调，token 只负责忽略结果，不会终止网络请求。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const fallbackCenter = new BMap.Point(116.404, 39.915);
map.centerAndZoom(fallbackCenter, 13);

const geolocation = new BMap.Geolocation();
let locationActive = true;
let locationGeneration = 0;
let locationMarker = null;

const requestGeneration = ++locationGeneration;
geolocation.getCurrentPosition((locationResult) => {
  if (!locationActive || requestGeneration !== locationGeneration) return;
  const status = geolocation.getStatus();
  const currentPoint = locationResult?.point;
  if (status !== BMAP_STATUS_SUCCESS || !currentPoint) {
    console.error('定位失败，状态码：', status);
    return;
  }
  map.centerAndZoom(currentPoint, 15);
  locationMarker = new BMap.Marker(currentPoint);
  map.addOverlay(locationMarker);
}, { timeout: 8_000 });

function dispose() {
  locationActive = false;
  locationGeneration += 1;
  if (locationMarker) map.removeOverlay(locationMarker);
  locationMarker = null;
  map.destroy();
}
```

定位权限与结果必须在带有效 AK 的 `v=4.0` 页面在线验证。

## 核心 API

### Geolocation

`BMap.Geolocation.getCurrentPosition` 异步执行，成功、拒绝、不可用或超时都会进入同一个回调。只有 `getStatus() === BMAP_STATUS_SUCCESS` 且 `result.point` 存在时才使用位置。浏览器定位依赖安全上下文（HTTPS）和用户授权（由浏览器判定，需在线验证）；失败时回调收到的实参是 `null`，所以「回调执行了」不代表定位成功。

构造选项共四个，默认值都已经是常用值：`timeout` 默认 `10000`（毫秒）、`maximumAge` 默认 `0`、`enableHighAccuracy` 默认 **`true`**、`SDKLocation` 默认 `false`。也就是说不需要为了「打开高精度」显式传 `enableHighAccuracy: true`。移动端还可以用 `enableSDKLocation()` / `disableSDKLocation()` 开关 SDK 辅助定位。`getCurrentPosition(successCallback, options)` 的第二个参数只会用 truthy 值覆盖构造配置；`false` 和 `0` 不会覆盖已有真值或非零值。需要关闭 SDK 辅助定位时调用 `disableSDKLocation()`，其他布尔关闭项应在构造实例时确定。

`timeout` 不是回调的最长等待时间：内部兜底定时器是 `timeout + 2000`，传 `timeout: 10000` 时最长约 12 秒才会拿到超时回调。业务侧的 loading 态要按这个上界设计。

定位是一条降级链：SDK 辅助定位 → 浏览器 HTML5 定位 → 高精度 IP 定位，全部失败才置 `BMAP_STATUS_UNKNOWN_LOCATION`。因此 `BMAP_STATUS_SUCCESS` 不等于拿到了设备级精度——结果可能来自 IP 定位。需要判断精度时读 `result.accuracy`（单位米，HTML5 分支会被截断到 1999）。

失败状态码的映射是固定的：权限被拒绝和非安全来源都落 `BMAP_STATUS_PERMISSION_DENIED`，位置不可用落 `BMAP_STATUS_UNKNOWN_LOCATION`，超时落 `BMAP_STATUS_TIMEOUT`。按码分支处理时以这三个为准。

`getStatus()` 不是纯读方法：只要该实例曾经成功定位过（内部还留着上一次的定位信息），后续调用会把状态改写成 `BMAP_STATUS_SUCCESS`。它只在**本次回调内部**有意义，不要在回调之外用它判断「最新一次请求成功了吗」。另外在 `otherSearch` 模块异步加载完成之前，`getStatus()` 恒定返回 `BMAP_STATUS_UNKNOWN_LOCATION`，这是同步壳的占位值，不是一次真实失败。

成功结果除了 `point` 还带 `accuracy`、`latitude`、`longitude`、`timestamp` 与 `address`（`country` / `province` / `city` / `city_code` / `district` / `street` / `street_number`），字段是否有值取决于线上返回，要逐个判空。`point` 会按当前地图的坐标类型转换后再交给回调，不恒为 BD09。

### LocalCity

`BMap.LocalCity.get(callback)` 通过 IP 服务返回城市信息；构造时提供 `renderOptions.map`，成功后会自动 `centerAndZoom` 到该城市。构造选项只读取 `renderOptions.map` 这一个字段，`panel`、`autoViewport` 传了不生效。失败时回调收到空对象，因此字段必须逐个判空，不要假定一定拿到 `center` 和 `name`。

成功结果是 `{center, level, name, code}`：`center` 是城市中心点，`name` 是城市名，`code` 是城市编码。

`level` 不是服务返回的「城市级别」，而是由当前地图推导出来的建议缩放级别：有 `map` 时取 `map.getZoom()`，没有 `map` 时以 `5` 为基准，再经过内部的最佳级别换算（结果被夹在 1..21）。所以在已有地图上调用时，`level` 往往等于当前级别，不要把它当成「城市应有的缩放级别」。

```javascript
const localCity = new BMap.LocalCity();
let cityActive = true;

localCity.get((cityResult) => {
  if (!cityActive || !cityResult?.center) return;
  console.log(cityResult.name, cityResult.code);
  map.centerAndZoom(cityResult.center, 11);
});

function disposeLocalCity() {
  cityActive = false;
}
```

IP 定位的精度与服务端策略有关，需在线验证；`Geolocation` 的「成功」也可能来自 IP 降级，两者都不能无条件当作设备级位置——要判断精度就读 `Geolocation` 结果里的 `accuracy`。

### Convertor

`BMap.Convertor.translate(points, from, to, callback)` 异步请求坐标转换；实现默认 `from = 1`、`to = 5`，成功时把服务结果转成 Point 数组。只在 callback 内检查 `result.status === 0` 和 `result.points` 后消费，不要在 translate 返回后同步读取。

`from` / `to` 用全局常量表达，不要写裸数字：

| 常量 | 值 | 坐标系 |
|---|---|---|
| `COORDINATES_WGS84` | 1 | WGS84 经纬度 |
| `COORDINATES_WGS84_MC` | 2 | WGS84 墨卡托 |
| `COORDINATES_GCJ02` | 3 | GCJ02 经纬度 |
| `COORDINATES_GCJ02_MC` | 4 | GCJ02 墨卡托 |
| `COORDINATES_BD09` | 5 | BD09 经纬度 |
| `COORDINATES_BD09_MC` | 6 | BD09 墨卡托 |
| `COORDINATES_MAPBAR` | 7 | 图吧坐标 |
| `COORDINATES_51` | 8 | 51 坐标 |

这组 `COORDINATES_*` 常量与 Map 的 `coordType` 常量（`BMAP_COORD_WGS84`、`BMAP_COORD_GCJ02`、`BMAP_COORD_BD09`、`BMAP_COORD_GCJ02MERCATOR`）是两套东西：前者是数字、只用于 Convertor，后者是字符串、只用于地图坐标类型，名字相近但不能互换。

一次转换的点数没有本地上限校验，但服务端仍可能限制，批量转换要自己分片。

失败时 `result` 本身可能是 `undefined`，即使有对象，`result.status` 也可能是 `undefined`。所以判定成功必须写全 `result && result.status === 0 && result.points`，不能只判 `status !== 0`。

```javascript
const convertor = new BMap.Convertor();
let convertActive = true;
let convertGeneration = 0;

function convertGCJ02(points) {
  if (points.length === 0) return;
  const current = ++convertGeneration;
  convertor.translate(points, COORDINATES_GCJ02, COORDINATES_BD09, (result) => {
    if (
      convertActive
      && current === convertGeneration
      && result
      && result.status === 0
      && result.points
    ) {
      console.log(result.points);
    }
  });
}

convertGCJ02([new BMap.Point(116.404, 39.915)]);

function disposeConvertor() {
  convertActive = false;
  convertGeneration += 1;
}
```

### 跨服务流程

定位 → 逆地理编码 → 驾车路线的跨服务流程必须在每一层检查失败，并用同一个 generation token 丢弃迟到回调：

```javascript
const map = new BMap.Map('map');
const fallbackCenter = new BMap.Point(116.404, 39.915);
const destination = new BMap.Point(116.431, 39.931);
map.centerAndZoom(fallbackCenter, 13);

const geolocation = new BMap.Geolocation();
const geocoder = new BMap.Geocoder();
let locationActive = true;
let locationGeneration = 0;
let activeRoute = null;

const requestGeneration = ++locationGeneration;
geolocation.getCurrentPosition((locationResult) => {
  if (!locationActive || requestGeneration !== locationGeneration) return;
  const status = geolocation.getStatus();
  const currentPoint = locationResult?.point;
  if (status !== BMAP_STATUS_SUCCESS || !currentPoint) {
    console.error('定位失败，状态码：', status);
    return;
  }

  map.centerAndZoom(currentPoint, 15);
  geocoder.getLocation(currentPoint, (addressResult) => {
    if (!locationActive || requestGeneration !== locationGeneration) return;
    if (!addressResult) {
      console.error('逆地理编码没有返回结果');
      return;
    }
    console.log(addressResult.addressComponents?.district ?? addressResult.address);

    activeRoute?.clearResults();
    const route = new BMap.DrivingRoute(currentPoint, {
      onSearchComplete(results) {
        if (!locationActive || requestGeneration !== locationGeneration) return;
        if (route.getStatus() !== BMAP_STATUS_SUCCESS || results.getNumPlans() === 0) {
          console.error('路线规划失败，状态码：', route.getStatus());
          return;
        }
        console.log(results.getPlan(0).getDistance(true));
      },
    });
    activeRoute = route;
    route.search(currentPoint, destination);
  });
}, { timeout: 8_000 });

function disposeLocatedRoute() {
  locationActive = false;
  locationGeneration += 1;
  activeRoute?.clearResults();
  map.destroy();
}
```

定位权限、逆地理编码与路线结果都需要真实页面在线验证；这些服务在 4.0 公开接口中都没有取消入口，因此 token 只负责忽略迟到结果，不会终止网络请求。

## 事件或回调

- Geolocation：在唯一完成回调里区分成功、权限拒绝、未知位置和超时；失败时实参是 `null`，不要只判断 result 非空。
- Geolocation：`getStatus()` 只在本次回调内读；它会因为「曾经成功过」被改写成成功。
- LocalCity：只有一个完成回调，先判 `result?.center` 再使用；`renderOptions.map` 模式下服务会自己改视野。
- Convertor：转换结果只在 translate callback 中有效，status 为 0 后还要检查 points。
- 新请求或组件卸载时使用 active/generation token 丢弃旧回调。

## 资源清理

- Geolocation、LocalCity 与 Convertor 没有可确认的公开取消方法，用 active token 忽略迟到回调。
- 定位成功后创建的 Marker、InfoWindow 等业务覆盖物由业务自己 `removeOverlay`。
- 跨服务流程中派生出的路线服务按 [路线与公交线检索](route-search.md) 的清理顺序处理。
- 只有组件拥有 Map 时才调用 `map.destroy()`。

## 常见错误

- 把浏览器权限拒绝、非安全上下文或超时当作成功定位；三者分别落 `BMAP_STATUS_PERMISSION_DENIED`、`BMAP_STATUS_PERMISSION_DENIED`、`BMAP_STATUS_TIMEOUT`。
- 为了「打开高精度」显式传 `enableHighAccuracy: true`：它本来就是默认值。
- 按 `timeout` 设计 loading 上限：真实兜底是 `timeout + 2000`。
- 在回调之外调用 `Geolocation.getStatus()` 判断最新一次请求：曾成功过的实例会返回成功。
- 把模块加载前 `getStatus()` 返回的 `BMAP_STATUS_UNKNOWN_LOCATION` 当成一次真实的定位失败。
- 把 `BMAP_STATUS_SUCCESS` 等同于设备级精度，忽略降级到 IP 定位的可能，也不读 `accuracy`。
- 把 LocalCity 的空对象回调当作成功结果，直接读 `center` 或 `name`。
- 把 LocalCity 的 `level` 当成服务给出的城市级别，它是由当前地图 zoom 推导的。
- 用 LocalCity 的城市级结果冒充设备精确位置。
- 只判断 `result.point` 非空而不读 `getStatus()`。
- 在 translate 下一行同步读取转换结果，或未检查 status/points。
- 给 `translate` 传裸数字 `3` / `5`，而不是 `COORDINATES_GCJ02` / `COORDINATES_BD09`。
- 把 `COORDINATES_*` 与 Map 的 `BMAP_COORD_*` 混用：一个是数字、一个是字符串，用途不同。
- 只判 `result.status !== 0` 就当成失败分支处理：失败时 `result` 与 `result.status` 都可能是 `undefined`。
- 用 `BMap.Convertor` 做纯本地几何换算：像素/经纬度换算是 Map 的同步方法，不需要发网络请求。
- 组件卸载后仍让迟到的定位回调调用已销毁的 Map。
