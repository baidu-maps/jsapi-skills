# 项目接入

## 何时读取

首次接入 JSAPI 4.0、处理 AK、脚本加载顺序、容器尺寸、空白地图或单页应用卸载时读取。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

- 普通页面优先使用同步 `script`，并把业务脚本放在它之后。
- 一个页面只加载一次 JSAPI；需要多张地图时创建多个容器和多个 `BMap.Map` 实例。
- 加载地址固定 `v=4.0`，不要混入其他版本的参数。
- 4.0 加载完成后使用默认命名空间 `BMap`。
- 首屏不需要地图时，可用 `callback` 参数异步加载，见下文“事件或回调”。
- 存量项目升级时先把入口统一为 `v=4.0`，再逐项核对核心 API；不要把兼容回退当成新项目方案。

## 最小可运行示例

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>JSAPI 4.0 最小地图</title>
    <style>
      html,
      body,
      #map {
        width: 100%;
        height: 100%;
        margin: 0;
      }
    </style>
  </head>
  <body>
    <div id="map"></div>
    <script src="https://api.map.baidu.com/api?v=4.0&ak=YOUR_BAIDU_MAP_AK"></script>
    <script>
      const map = new BMap.Map('map');
      map.centerAndZoom(new BMap.Point(116.404, 39.915), 15);

      let disposed = false;

      function disposeMap() {
        if (disposed) return;
        disposed = true;
        window.removeEventListener('pagehide', onPageHide);
        map.destroy();
      }

      function onPageHide(event) {
        if (!event.persisted) disposeMap();
      }

      window.addEventListener('pagehide', onPageHide);
    </script>
  </body>
</html>
```

把 `YOUR_BAIDU_MAP_AK` 留作模板占位符；实际项目应由部署配置注入获授权的浏览器端 AK，不要把真实密钥写进技能、日志或回答。

## 核心 API

- `new BMap.Map(container, options?)`：`container` 可传元素 ID 或 `HTMLElement`；4.0 支持在 `options` 里直接给 `center` 和 `zoom`。
- `map.centerAndZoom(point, zoom)`：同时设置中心点和缩放级别。
- `map.setOptions({ enableWheelZoom: false })`：4.0 默认已开启滚轮缩放，需要关闭时再修改配置。
- `map.destroy()`：销毁地图并释放地图内部资源。

4.0 不强制调用 `centerAndZoom`：没有指定视野时，地图会以北京天安门（`116.404, 39.915`）、zoom 12 展示默认视野。所以漏调 `centerAndZoom` 的症状是“地图显示在北京”，不是“地图未初始化”或“必然空白”。正式项目仍应显式设置业务视野，不要依赖这个默认值。

页面容器必须在构造地图前存在，并具有非零宽高。

## 事件或回调

同步脚本后置时，后续脚本执行即表示 JSAPI 脚本已完成加载。

首屏不展示地图时可以异步加载：给入口地址加 `callback` 参数，脚本加载完成后该函数会被调用。

```html
<button id="open-map">打开地图</button>
<div id="map" style="width: 100%; height: 400px"></div>
<script>
  let map = null;
  let loading = false;

  window.initMap = function initMap() {
    loading = false;
    if (map) return;
    map = new BMap.Map('map');
    map.centerAndZoom(new BMap.Point(116.404, 39.915), 15);
  };

  const openButton = document.getElementById('open-map');
  const onOpenMap = () => {
    if (map || loading) return;
    if (window.BMap) return window.initMap();
    loading = true;
    const script = document.createElement('script');
    script.src =
      'https://api.map.baidu.com/api?v=4.0&ak=YOUR_BAIDU_MAP_AK&callback=initMap';
    script.onerror = () => {
      loading = false;
      console.error('JSAPI 4.0 加载失败');
    };
    document.body.appendChild(script);
  };
  openButton.addEventListener('click', onOpenMap);

  function dispose() {
    openButton.removeEventListener('click', onOpenMap);
    if (map) map.destroy();
    map = null;
  }
</script>
```

异步加载时 `initMap` 必须是全局函数，且要自己保证只插入一次脚本，否则会出现重复初始化。不要仅凭 `script` 的 `load` 事件就认为 `BMap` 已经可用——`callback` 才是明确的就绪信号。加载失败、超时和并发场景需要在真实部署里验证后再封装成项目基础设施。

SPA 建议在页面壳层只加载一次 JSAPI，路由组件只管理自己的 Map 实例。

地图交互事件统一按 [事件与生命周期](events-and-lifecycle.md) 绑定和解绑。`pagehide` 进入 BFCache 时 `event.persisted === true`，此时保留 Map 供后退恢复；非持久化离开才销毁。框架项目仍以组件卸载钩子主动销毁自己的 Map 为主。若产品选择在 persisted pagehide 时销毁，必须在 `pageshow` 中完整重建地图。AK 权限、网络和全局对象仍须浏览器在线验证。

## 资源清理

1. 先停止业务定时器、请求、动画和服务回调后续写入。
2. 移除自己注册的 DOM 与 BMap 事件监听。
3. 移除显式创建的控件、菜单、覆盖物和图层。
4. 最后调用 `map.destroy()`；不要在销毁后继续调用地图方法。

## 常见错误

- 地图容器没有高度，页面看起来像“加载成功但地图空白”。
- 业务脚本在 JSAPI 脚本之前执行，导致 `BMap is not defined`。
- 同页重复插入 JSAPI 脚本，形成初始化竞态。
- 误把默认北京视野当作业务初始化完成，或反过来把漏调 `centerAndZoom` 断言为地图空白。
- 异步加载时仅凭 `script` 的 `load` 事件就开始调用 `BMap`，而不用 `callback`。
- SPA 卸载只设置 active 标志，却不移除 pagehide 等 DOM 监听。
- 沿用其他版本示例的加载参数和命名空间。
- 只检查代码语法，不验证 AK、网络、WebGL 或在线底图。
