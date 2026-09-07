# 全景、全景服务与标签

## 何时读取

需要在独立容器展示街景、按坐标查询全景数据或创建全景标签时读取。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | API | 调用重点 |
|---|---|---|
| 独立全景容器 | `BMap.Panorama` | 容器必须已有尺寸；先设置位置，位置就绪后再防御性读取 |
| 查询附近或指定 ID 的全景 | `BMap.PanoramaService` | 回调结果可能是 `null`，坐标查询默认半径 50 米 |
| 全景文字标签对象 | `BMap.PanoramaLabel` | 构造时传 Point；可更新位置、内容与高度 |
| 结束独立全景 | `destroy` | 业务自建的独立 Panorama 必须自己调用 |

## 最小可运行示例

```javascript
const position = new BMap.Point(116.403850, 39.913795);
const panorama = new BMap.Panorama('panorama', {
  navigationControl: true,
  linksControl: true,
});

panorama.setPosition(position);

function dispose() {
  panorama.destroy();
}
```

页面需要预先提供有明确宽高的 `#panorama` 容器。全景覆盖范围、影像加载和标签实际可见性必须在带有效 AK 的 `v=4.0` 页面在线验证。

## 核心 API

`BMap.Panorama` 接受元素 ID 或 HTMLElement，options 在构造时合并进全景配置。

`getPosition()` 在全景数据尚未就绪时返回 `null`。构造后立即读取不安全；确认数据就绪后再调用并判空。

`setId`、`setPov`、`addOverlay`、`removeOverlay`、`clearOverlays` 与 `destroy` 都是可正常调用的公开方法。

`BMap.PanoramaService.getPanoramaById(id, callback)` 与 `getPanoramaByLocation(point, callback)`/`getPanoramaByLocation(point, radius, callback)` 都是异步调用。省略半径时按 50 米处理；没有数据时回调收到 `null`，使用前必须判空。网络失败是否一定会形成回调需要在线验证，业务应自行设置超时边界。

```javascript
const service = new BMap.PanoramaService();
let generation = 0;

function showNearest(point) {
  const current = ++generation;
  service.getPanoramaByLocation(point, 100, (data) => {
    if (current !== generation || data === null) {
      return;
    }
    console.log('附近全景 ID', data.id);
  });
}

function ignorePendingLookup() {
  generation += 1;
}
```

`BMap.PanoramaLabel` 接受文本和位置/高度/距离显示配置，支持更新位置、内容与高度：

```javascript
const panorama = new BMap.Panorama('panorama-label-demo');
const panoramaPosition = new BMap.Point(116.403850, 39.913795);
panorama.setPosition(panoramaPosition);
const label = new BMap.PanoramaLabel('当前位置附近', {
  position: panoramaPosition,
  altitude: 2,
  displayDistance: true,
});

const onLabelClick = () => {
  label.setContent('已选择当前位置');
};
label.addEventListener('click', onLabelClick);
panorama.addOverlay(label);

label.setContent('新的标签文本');
label.setAltitude(3);
const labelPosition = label.getPosition();
console.log(labelPosition);

function disposePanoramaLabel() {
  label.removeEventListener('click', onLabelClick);
  panorama.removeOverlay(label);
  panorama.destroy();
}
```

## 事件或回调

- 不要依赖 `position_changed` 回调参数中的位置字段；需要位置时在回调里调用 `getPosition()` 并判空。
- 注册 `position_changed` 时内部会包一层新函数，所以用原 handler 调用 `removeEventListener` 解绑不掉。长期复用 Panorama 时避免注册该事件，只用能同引用解绑的事件；整体退出由 `panorama.destroy()` 收尾。
- PanoramaService 回调可能迟到，且没有请求取消 API。用 generation/active token 忽略过期结果。

## 资源清理

- 独立 Panorama：先让异步业务回调失效，再调用 `destroy()`。
- PanoramaLabel：先用同一 handler 解绑业务事件，再把同一标签实例传给 `panorama.removeOverlay(label)`；最后销毁独立 Panorama。
- 由 Map 内部创建的全景随 `map.destroy()` 一起销毁；业务自己 `new` 出来的独立 Panorama 必须自己销毁。
- PanoramaService 没有可确认的公开 destroy/cancel 方法；它不持有主示例的渲染容器，迟到结果用 token 隔离。

## 常见错误

- 全景容器没有固定宽高，误以为无影像是服务失败。
- 把普通 Marker/Label 传给 `panorama.addOverlay`，而不是 PanoramaLabel。
- 忽略 PanoramaService 的 `null` 结果，直接读取 data.id。
- 依赖 `position_changed` 参数携带完整位置，而不主动调用 `getPosition()`。
- 用原 handler 尝试解绑 `position_changed`，误以为包装注册仍保留了函数身份。
- 只从 DOM 删除容器，不调用 `destroy` 结束全景实例。
- 把有效 AK、覆盖范围或在线影像响应当作静态验证能够保证的结果。
