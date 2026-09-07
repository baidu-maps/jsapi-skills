# 控件与右键菜单

## 何时读取

添加内置控件、选择定位控件、实现自定义控件或配置右键菜单时读取。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

- 常规缩放/平移：`BMap.NavigationControl` 或轻量 `BMap.ZoomControl`。
- 三维导航：`BMap.NavigationControl3D`。
- 比例尺：`BMap.ScaleControl`。
- 地图类型：`BMap.MapTypeControl`。
- 浏览器定位：正式 v4 代码用公开的 `BMap.GeolocationControl`。
- 城市、鹰眼、版权、全景：`BMap.CityListControl`、`BMap.OverviewMapControl`、`BMap.CopyrightControl`、`BMap.PanoramaControl`。
- 右键操作：`BMap.ContextMenu` + `BMap.MenuItem`。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 15);

const navigation = new BMap.NavigationControl();
const scale = new BMap.ScaleControl();
const geolocation = new BMap.GeolocationControl();
map.addControl(navigation);
map.addControl(scale);
map.addControl(geolocation);

const onLocationSuccess = (event) => {
  console.log(event.point, event.addressComponent);
};
const onLocationError = (event) => {
  console.error('定位失败', event.code);
};
geolocation.addEventListener('locationSuccess', onLocationSuccess);
geolocation.addEventListener('locationError', onLocationError);

function dispose() {
  geolocation.stopLocationTrace();
  geolocation.removeEventListener('locationError', onLocationError);
  geolocation.removeEventListener('locationSuccess', onLocationSuccess);
  map.removeControl(geolocation);
  map.removeControl(scale);
  map.removeControl(navigation);
  map.destroy();
}
```

这个可随时卸载的控件示例不主动调用 `geolocation.location()`：一次性定位发出后没有可确认的取消入口。需要立即发起定位时，使用 [定位与坐标转换](geolocation-and-convertor.md) 中带 active/generation token 的 Geolocation service；若坚持由控件发起，则让控件与页面同生命周期，并等定位完成后再移除。

## 核心 API

4.0 可直接使用这些内置控件：

- `BMap.NavigationControl`
- `BMap.NavigationControl3D`
- `BMap.ScaleControl`
- `BMap.ZoomControl`
- `BMap.MapTypeControl`
- `BMap.GeolocationControl`
- `BMap.CityListControl`
- `BMap.OverviewMapControl`
- `BMap.CopyrightControl`

`BMap.PanoramaControl` 也是公开控件，但它不是普通 `Control` 子类，而是由全景模块提供的工厂函数，依赖全景资源加载。使用时同样保存实例并成对挂载、移除：`const control = new BMap.PanoramaControl(); map.addControl(control); map.removeControl(control);`。

普通 `Control` 实例可用 `setAnchor/getAnchor`、`setOffset/getOffset` 调整停靠位置，用 `show/hide/isVisible` 管理可见性。`CopyrightControl` 还提供 `addCopyright`、`removeCopyright`、`getCopyright` 与 `getCopyrightCollection` 管理版权项。

控件停靠只支持四角。`BMAP_ANCHOR_TOP_LEFT`(0)、`BMAP_ANCHOR_TOP_RIGHT`(1)、`BMAP_ANCHOR_BOTTOM_LEFT`(2)、`BMAP_ANCHOR_BOTTOM_RIGHT`(3) 有效；`BMAP_ANCHOR_TOP_CENTER`、`MIDDLE_LEFT`、`CENTER`、`MIDDLE_RIGHT`、`BOTTOM_CENTER` 这几个常量虽然存在，传给控件会被静默回落到该控件的 `defaultAnchor`，不会报错也不会生效。`offset` 缺省时同样回落到 `defaultOffset`。

常见桌面地图可按需组合轻量缩放、地图类型和鹰眼控件；实例必须保存，组件卸载时按相反顺序移除：

```javascript
const zoomControl = new BMap.ZoomControl();
const mapTypeControl = new BMap.MapTypeControl();
const overviewControl = new BMap.OverviewMapControl({ isOpen: true });

map.addControl(zoomControl);
map.addControl(mapTypeControl);
map.addControl(overviewControl);

function removeCommonControls() {
  map.removeControl(overviewControl);
  map.removeControl(mapTypeControl);
  map.removeControl(zoomControl);
}
```

Map 通过 `addControl` / `removeControl` 管理同一个控件实例；一个实例只添加一次。

自定义控件继承 `BMap.Control`，实现 `initialize(map)`，把返回的 DOM 加入地图容器；DOM 事件由业务自己保存和解绑，`removeControl` 只负责控件容器生命周期。下面的按钮接收业务回调，可与 [瓦片与地图服务图层](tile-and-service-layers.md) 组合成 WMS/路况切换器：

```javascript
class ActionControl extends BMap.Control {
  defaultAnchor = BMAP_ANCHOR_BOTTOM_RIGHT;
  defaultOffset = new BMap.Size(16, 16);

  constructor(action) {
    super();
    this.action = action;
    this.button = null;
    this.onClick = () => this.action();
  }

  initialize(map) {
    const button = document.createElement('button');
    button.type = 'button';
    button.textContent = '切换图层';
    button.addEventListener('click', this.onClick);
    map.getContainer().appendChild(button);
    this.button = button;
    return button;
  }

  dispose() {
    const button = this.button;
    if (!button) return;
    button.removeEventListener('click', this.onClick);
    this.button = null;
  }
}

const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 15);

let enabled = false;
const actionControl = new ActionControl(() => {
  enabled = !enabled;
  console.log('图层开关', enabled);
});
map.addControl(actionControl);

function disposeCustomControl() {
  actionControl.dispose();
  map.removeControl(actionControl);
  map.destroy();
}
```

右键菜单通过 `addContextMenu` / `removeContextMenu` 挂载，菜单项通过 `addItem` / `removeItem` 管理。`ContextMenu` 自身也有 `open`、`close` 事件。

`addItem(item, insertIndex)`、`addSeparator(insertIndex)` 与 `removeSeparator(index)` 可以指定位置。`ContextMenu` 构造函数接受 `{container, cursor, marker}`；普通地图右键菜单通常无需传这些内部上下文选项。

`MenuItem(text, callback, opts)` 的回调收到 `(point, pixel, overlay)`：`point` 是右键位置，`pixel` 是当次像素坐标，`overlay` 是命中的覆盖物，没有命中时为空。`text` 为空或 `callback` 不是函数时，构造会静默返回一个不可用的菜单项。

4.0 下菜单外观由样式表控制：容器 class 是 `BMap_contextMenu`，菜单项是 `BMap_cmItem`（禁用态追加 `BMap_cmItem_disabled`），首末项额外带 `BMap_cmFstItem` / `BMap_cmLstItem`，分隔线是 `BMap_cmDivider`。需要改配色或间距时覆写这些 class，或直接用 `map.setTheme` 换肤，见 [UI 版本与主题](ui-version-and-theme.md)。

`removeContextMenu` 只移除菜单 DOM 并清空菜单持有的 Map 引用，ContextMenu 初始化时向 document 与 Map 注册的监听没有公开的解绑入口。因此长寿命 Map 只创建并复用一个 ContextMenu，不要在路由组件里反复构造、挂载和移除；业务自己添加的菜单事件和 MenuItem 仍要成对清理。

定位控件统一写 `BMap.GeolocationControl`。4.0 同时保留同实现的 `BMap.LocationControl` 名称，但新代码无需混用。Logo 由 Map 自动管理，业务通常不需要自行构造 `LogoControl`。

`BMap.GeolocationControl` 公开 `location/startLocation/stopLocationTrace` 与 `locationSuccess/locationError` 事件。跟踪模式基于 `watchPosition`，只有 `stopLocationTrace` 会清除它；控件自身的 `remove()` 不会顺带停止跟踪。因此清理顺序是：停止跟踪 → 解绑具名定位事件 → `map.removeControl(control)`。

一次性 `getCurrentPosition` 已发出后，GeolocationControl 的公开声明和当前实现中没有提供取消方法。短生命周期组件若必须忽略迟到结果，优先使用 [定位与坐标转换](geolocation-and-convertor.md) 的 Geolocation service + active token；不要把移除控件描述成取消了一次性定位。

## 事件或回调

右键菜单监听器同样保存引用：

```javascript
const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 15);

const menu = new BMap.ContextMenu();
const menuItem = new BMap.MenuItem('设为地图中心', (point) => {
  map.centerAndZoom(point, 15);
});
menu.addItem(menuItem);
menuItem.setText('以此处为地图中心');

const onOpen = (event) => {
  if (event.pixel) {
    const point = map.pixelToPoint(event.pixel);
    console.log(point);
  }
};
menu.addEventListener('open', onOpen);
map.addContextMenu(menu);

function disableContextMenuBusiness() {
  menu.removeEventListener('open', onOpen);
  menu.removeItem(menuItem);
}
```

这个 ContextMenu 在页面壳层创建，与 document 和长寿命 Map 同生命周期；`disableContextMenuBusiness()` 只清理业务注册的事件和菜单项，不是 ContextMenu 的完整销毁——它向 document 注册的监听没有公开解绑入口，`map.destroy()` 也只释放 Map 自身资源。

ContextMenu 的 `open`/`close` 事件对象只赋值 `pixel`。需要经纬度时调用 `map.pixelToPoint(event.pixel)`。

定位结果及其状态处理转到 [定位与坐标转换](geolocation-and-convertor.md)，不要仅用点击定位按钮作为成功依据。

## 资源清理

ContextMenu 只在页面壳层创建一次并复用；业务功能关闭时解绑自己的菜单事件并移除菜单项，`removeContextMenu` 与 `map.destroy()` 都不覆盖它向 document 注册的监听。定位控件先 `stopLocationTrace` 并解绑定位事件；其他控件用创建时保存的实例逐一 `removeControl`，最后销毁 Map。

## 常见错误

- `addControl(new Control())` 后没有保存实例，卸载时无法精确移除。
- 同一控件实例重复添加。
- 同时混用 `BMap.LocationControl` 与 `BMap.GeolocationControl`；新代码统一写后者。
- 自定义控件只移除 DOM，却没有解绑业务添加的 click 等 DOM 事件。
- 给 `MenuItem` 传了空文案或非函数回调，构造静默失败，表现为「菜单项凭空少了一条」。
- 传 `BMAP_ANCHOR_TOP_CENTER` 等非四角常量定位控件，被静默回落到 `defaultAnchor`。
- 移除右键菜单但遗漏其自定义事件或菜单项持有的业务闭包。
- 在长寿命 Map 上反复创建/移除 ContextMenu，并误以为 `removeContextMenu` 已解绑它自己注册的监听。
- 直接读取 ContextMenu 的 `event.point`，它不会被赋值。
- 先移除 GeolocationControl 再停止 watch，导致浏览器定位继续运行。
