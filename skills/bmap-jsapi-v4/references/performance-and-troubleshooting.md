# 性能与排障

## 何时读取

出现地图空白、AK 或加载版本异常、容器尺寸为零、坐标偏移、CORS/瓦片错误、类型声明未生效、事件重复或资源释放不完整时读取；大量对象造成交互卡顿时也从本页按可观察信号缩小范围。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 可观察症状 | 首先读取 |
|---|---|
| `BMap is not defined`、地图空白、AK/脚本加载 | [项目接入](project-setup.md) |
| Point 偏移、像素位置错误 | [坐标与几何](coordinates-and-geometry.md)、[定位与坐标转换](geolocation-and-convertor.md) |
| XYZ/TMS 404、上下颠倒、CORS | [瓦片与服务图层](tile-and-service-layers.md) |
| `Cannot find namespace 'BMap'` | [TypeScript 集成](typescript-integration.md) |
| 回调多次、解绑失败、卸载后仍执行 | [事件与生命周期](events-and-lifecycle.md) |
| 大量点线面或 GeoJSON 卡顿 | 标准批量图层见 [批量可视化图层](visualization-layers.md)，聚合/扩展图层见 [扩展 API](runtime-extended-apis.md)，通用 GeoJSON 见 [GeoJSON 与 DOM 图层](data-layers.md) |
| 动画、服务、全景或 Map 未释放 | 对应业务 reference 的“资源清理”章节 |

排障只记录事实：页面元素尺寸、实际 script URL、网络请求/响应、控制台错误、运行时命名空间、当前坐标类型和实际瓦片 URL。不要先改多个参数再猜原因。

## 最小可运行示例

```javascript
const container = document.getElementById('map');
if (!(container instanceof HTMLElement)) {
  throw new Error('缺少 #map 容器');
}

const initialRect = container.getBoundingClientRect();
if (initialRect.width === 0 || initialRect.height === 0) {
  throw new Error('#map 容器尺寸必须非零');
}

const map = new BMap.Map(container);
const center = new BMap.Point(116.404, 39.915);
map.centerAndZoom(center, 15);
let disposed = false;
let reportFrame = null;

function reportMapState() {
  reportFrame = null;
  if (disposed) return;
  const size = map.getSize();
  console.table({
    width: size.width,
    height: size.height,
    zoom: map.getZoom(),
    coordType: map.getCoordType(),
  });
}

const onWindowResize = () => {
  map.checkResize();
  if (reportFrame === null) {
    reportFrame = requestAnimationFrame(reportMapState);
  }
};

window.addEventListener('resize', onWindowResize);
reportMapState();

function dispose() {
  if (disposed) return;
  disposed = true;
  window.removeEventListener('resize', onWindowResize);
  if (reportFrame !== null) {
    cancelAnimationFrame(reportFrame);
    reportFrame = null;
  }
  map.destroy();
}
```

该示例提供容器、视野、尺寸和坐标类型的只读诊断，并闭合 DOM 监听与 Map 生命周期；它不证明 AK、底图、WebGL、CORS 或在线数据可用。

## 核心 API

### 1. 地图空白

- 只读检查：依次检查 `typeof BMap`、实际 script URL、控制台首个异常、`#map` 的 bounding rect、Map 的 `getSize()`；再确认是否显式调用过 `centerAndZoom`，但不要把漏调直接当成空白根因。
- 判定信号：BMap 不存在是加载问题；容器或 Map 宽高为 0 是布局问题；4.0 即使未显式设视野也会落到北京/zoom 12。两者正常但仍无底图才继续查 AK、网络或渲染。
- 最小修复：一次只修一个根因；先让 loader 与容器成立，并显式设置业务视野，再进入在线请求排障。

### 2. AK/鉴权

- 只读检查：在 Network 中定位 JSAPI loader、底图或服务请求，记录实际请求地址、HTTP 状态和响应；同时记录控制台中的鉴权提示。
- 判定信号：脚本下载失败与 AK 拒绝不是同一个问题；静态类型检查也不能判断 AK 是否有效。
- 最小修复：确认浏览器端 AK 已授权当前域名和所用服务；模板仍只保留 `YOUR_BAIDU_MAP_AK`，不要把真实 AK 复制进日志、技能或回答。

### 3. 容器尺寸

- 只读检查：比较 `getBoundingClientRect()`、`map.getSize()` 和元素计算样式；特别检查弹窗、Tab、折叠面板初始化时是否为 `display:none`。
- 判定信号：Map 构造时读取 `container.clientWidth/clientHeight`，零尺寸会成为初始地图尺寸。
- 最小修复：容器可见且非零后再构造 Map；若现有 Map 在隐藏期间改变尺寸，显示后调用 4.0 公共方法 `map.checkResize()`。

### 4. 加载版本

- 只读检查：查看浏览器最终请求的 script URL 和运行时使用的命名空间，而不是只看配置模板。
- 判定信号：入口必须是 `v=4.0`，业务命名空间统一为 `BMap`；出现其他版本参数或混用命名空间时，先修正加载入口再排查业务代码。
- 最小修复：统一 loader 和业务代码的命名空间，并清理重复注入的 script。

### 5. 坐标偏移

- 只读检查：先记录数据源声明的 WGS84/GCJ02/BD09/墨卡托类型，再读取 `map.getCoordType()`；选择一个已知点同时比较原始值和渲染位置。
- 判定信号：所有点以近似固定方向偏移通常是坐标系解释不一致，不是经纬度随机误差；单点完全失真还应检查 lng/lat 顺序。
- 若地图的坐标类型不是默认的 BD09，还要看偏移是否只出现在自定义覆盖物上：`pointToOverlayPixel` 会先把入参按地图坐标类型转成 BD09，`pointToPixel` 不做这一步，而 `overlayPixelToPoint` 返回的是 BD09、不转回地图坐标类型。也就是说非 BD09 地图上「覆盖物容器换算」不是可逆往返，两套像素 API 的入参口径也不一致。
- 最小修复：统一输入坐标契约；确需转换时使用 Convertor 的异步结果并检查 status/points，不要手工加常量偏移。

### 6. CORS/网络

- 只读检查：在 Network 中确认请求是否发出、是否重定向、响应是否被浏览器标为 CORS、超时或混合内容；保存失败 URL 和响应头。
- 判定信号：这一类是**浏览器侧**的拦截结果，不是 JSAPI 的返回值——请求 URL 正确但被浏览器拦下，是数据源/代理的跨域配置问题，不是 Map 构造器或静态声明问题。JSAPI 没有对应的错误事件可监听，只能从 Network 面板取证。
- 最小修复：在瓦片/数据服务端配置允许的 Origin，或通过受控同源代理转发；不要用关闭浏览器安全策略作为交付方案。

### 7. 瓦片层级/Y 方向

- 只读检查：从 Network 复制一个实际 `{z}/{x}/{y}` URL，和服务商同级样例比较 z、x、y、投影、范围及是否 TMS 翻转。
- 判定信号：整层 404 常见于层级或 URL 模板错误；同级影像上下颠倒通常是 XYZ/TMS Y 方向不一致；只有部分层级空白时确认地图 zoom 位于 minZoom..maxZoom，并比较请求 z 是否等于 `floor(mapZoom) + spanLevel`。
- `spanLevel` 同时抬高了层级闸门：请求级别要落在 `minZoom + spanLevel` 到 `maxZoom + spanLevel` 之间才会发出，否则直接返回空并清缓存。只改 `spanLevel` 不改 `minZoom` / `maxZoom`，表现就是「某几级整层空白但没有 404」。另外栅格/XYZ 图层算的是 `floor(mapZoom) + spanLevel`，MVT 图层算的是 `floor(mapZoom + spanLevel)`；`spanLevel` 传整数时两者一致，传小数会让两类图层落到不同级别，所以只用整数。
- 最小修复：RasterTileLayer 的 XYZ 使用 `{y}`，TMS 使用 `{-y}`/`{reverseY}`，并显式校准 projection、minZoom、maxZoom、spanLevel。`startLevel` 在当前实现中不存在，不能把它当作 4.0 正式排障项。

### 8. TypeScript 集成

`Cannot find namespace 'BMap'`、声明包安装、`tsconfig`、EventMap 类型和 Vue 集成见 [TypeScript 集成](typescript-integration.md)。

### 9. 事件重复

- 只读检查：给注册和清理点计数，记录 handler 身份与组件 mount/unmount 次数；不要只在回调内部打印。
- 判定信号：一次交互触发 N 次且 N 随重挂载增加，通常是重复注册或 remove 时函数引用不同。
- 最小修复：使用具名/稳定函数引用，每次 add 对应一次 remove；异步服务再用 generation/active token 阻止迟到回调写入已卸载组件。

### 10. 内存/资源释放

- 只读检查：卸载前后比较 DOM、监听数、定时器、活动网络回调和 WebGL context；用重复挂载/卸载而不是单次快照发现增长。
- 判定信号：容器已移除但事件、动画、全景或图层仍回调，说明只删 DOM、没有关闭业务对象生命周期。
- 最小修复：先让异步回调失效，再停动画/服务渲染，解绑事件，移除控件/覆盖物/图层，销毁全景，最后 `map.destroy()`。

### 性能定位

先用 Performance/Memory 记录基线，分开测数据解析、网络、主线程事件和绘制。若大量独立覆盖物/监听器随数据量增长，再评估 GeoJSONLayer 或 PointIcon/PointShape/Line/Fill 等具体批量图层；不要在没有测量时承诺某个阈值或固定帧率。

## 事件或回调

- 排障监听器也必须使用稳定引用并在 dispose 中移除；临时日志代码同样会造成事件重复。
- 在线服务没有公开取消方法时，用 active/generation token 隔离迟到回调，并在对应业务 reference 确认清理边界。
- 先记录首个错误和首次失败请求；后续连锁异常通常不是根因。
- `requestAnimationFrame`、setTimeout、ResizeObserver 和自建 Worker 都属于业务资源，Map.destroy 不会自动替业务清理。

## 资源清理

统一按以下顺序收口：

1. generation/active token 让迟到回调失效。
2. 停止自建 timer、requestAnimationFrame、observer、Worker 和视角动画。
3. 解绑 DOM、Map、Overlay、Layer、ViewAnimation、Panorama 的业务监听器。
4. 清服务自动渲染结果，移除 InfoWindow、覆盖物、控件、菜单和图层。
5. 清 PanoramaLabel 并 destroy 独立 Panorama。
6. 最后调用 `map.destroy()`，断开业务保存的 Map 引用。

每种资源的精确方法以对应 reference 的“资源清理”为准；没有明确公开 API 的场景就按“当前没有公开方法”处理，不要发明 destroy/cancel。

## 常见错误

- 同时修改 AK、版本、容器和坐标，导致无法知道哪一项修复生效。
- 只截图“地图空白”，不提供实际 script URL、容器尺寸和首个网络错误。
- 用 tsc 通过证明 AK、CORS、WebGL 或在线服务已经成功。
- 用匿名函数 removeEventListener，误以为内容相同就能解绑。
- 把 TMS 的 Y 翻转、瓦片层级偏移和投影问题混成一个猜测。
- 只删除地图 DOM，不按子资源到 Map 的顺序完成资源释放。
- 没有采样数据就宣称某覆盖物数量一定安全或一定卡顿。
