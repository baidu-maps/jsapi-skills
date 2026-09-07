# 坐标与几何

## 何时读取

创建经纬度点、像素位置、控件尺寸、矩形范围，做 Map/覆盖物像素换算或多点范围判断时读取。

## 快速选择

- 经纬度：`BMap.Point(lng, lat)`。
- 屏幕或容器像素：`BMap.Pixel(x, y)`。
- 宽高或偏移：`BMap.Size(width, height)`。
- 地理矩形：`BMap.Bounds(sw, ne)`，用 `extend` 扩展。
- 地图画布换算：`pointToPixel` / `pixelToPoint`。
- 自定义覆盖物容器换算：`pointToOverlayPixel` / `overlayPixelToPoint`。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const start = new BMap.Point(116.35, 39.88);
const end = new BMap.Point(116.47, 39.96);
const bounds = new BMap.Bounds(start, end);
const offset = new BMap.Size(12, 24);

map.centerAndZoom(bounds.getCenter(), 12);

const canvasPixel = map.pointToPixel(start);
const restoredPoint = map.pixelToPoint(canvasPixel);
const overlayPixel = map.pointToOverlayPixel(end);
const restoredOverlayPoint = map.overlayPixelToPoint(overlayPixel);

console.log({
  contains: bounds.containsPoint(restoredPoint),
  samePoint: end.equals(restoredOverlayPoint),
  offset,
});

map.destroy();
```

## 核心 API

| 类型 | 主要成员 | 用途 |
|---|---|---|
| `BMap.Point` | `lng`, `lat`, `equals` | 地理坐标 |
| `BMap.Pixel` | `x`, `y`, `equals` | 画布或容器像素 |
| `BMap.Size` | `width`, `height`, `equals` | 尺寸与偏移 |
| `BMap.Bounds` | `isEmpty`, `containsPoint`, `containsBounds`, `intersects`, `extend`, `getCenter` | 地理矩形计算 |

`equals` 在三个值对象上的判等口径不同，跨类型迁移判等逻辑时要留意：`Point.equals` 是**带容差**的，经纬度之差都小于 `1e-8` 就算相等，所以坐标换算、序列化往返产生的浮点末位误差不会让它返回 `false`；`Pixel.equals` 与 `Size.equals` 用的是严格 `===`，`100` 与 `100.0000001` 判为不等。三者传入 `null`/`undefined` 都返回 `false`，不会抛错。

构造函数只做类型归一，不做取值范围校验。`new BMap.Point(lng, lat)` 接受数字或数字字符串，无法解析时把该分量落成 `0`；它不会检查 `lng` 是否在 ±180、`lat` 是否在 ±90 之内，也不会告警，所以经纬度写反时得到的是一个「合法但位置错误」的点，只能靠业务自己校验。`new BMap.Size(w, h)` 对非数字入参走 `parseFloat`，解析失败会留下 `NaN`（没有落 0 兜底），用它做图标锚点会让覆盖物直接不可见。

复制值对象统一用构造函数重建，例如 `new BMap.Point(point.lng, point.lat)`。`Point`、`Pixel` 与 `Bounds` 提供 `clone()`，`Size` 没有，因此不要把 `clone()` 当成所有值对象的通用能力。

`BMap.Bounds` 允许无参创建空范围。`intersects`、`getCenter`、`getSouthWest`、`getNorthEast` 在空范围或不相交时可能返回 `null`；`toSpan()` 返回附加 `lng/lat` 别名的 `BMap.Size`。

处理空 Bounds 时先用 `isEmpty()` 建立业务分支；调用 `intersects`、`getCenter`、`getSouthWest`、`getNorthEast` 后仍按可能为空处理。

`containsPoint()` 没有空范围保护：空 Bounds 上调用会在读取 `sw.lng` 时抛错，传入假值时返回 `undefined` 而不是 `false`。相邻的 `equals` 与 `containsBounds` 都带 `isEmpty` 分支，只有它没有，所以这条判断要自己前置 `isEmpty()` 与入参检查。

`pointToPixel` 会考虑当前地图中心、缩放、旋转和倾斜；覆盖物容器换算是另一套 API，不要混用两类像素。

两套换算的坐标类型口径也不同，只在默认 BD09 地图上才看不出差别。`pointToOverlayPixel` 会先按地图的坐标类型把入参转成 BD09，`pointToPixel` 不做这一步；而 `overlayPixelToPoint` 的返回值固定是 BD09，不会转回地图坐标类型。所以地图坐标类型不是 BD09 时，`pointToPixel` / `pixelToPoint` 仍是可逆往返，`pointToOverlayPixel` / `overlayPixelToPoint` 却不是——上面示例里的 `samePoint` 只在默认坐标类型下为 `true`。自定义覆盖物的 `draw` 里统一用 `pointToOverlayPixel`，需要把容器像素还原成业务坐标时自己补一次坐标转换，见 [定位与坐标转换](geolocation-and-convertor.md)。

## 事件或回调

`Point`、`Pixel`、`Size`、`Bounds` 本身不派发事件。需要响应视野变化时，在 Map 上监听 `moveend`、`zoomend` 或 `resize`，再重新计算像素位置。

## 资源清理

几何值对象不需要销毁。它们若被长期监听器、覆盖物或缓存闭包引用，应在所属业务对象清理时解除引用；示例最后调用 `map.destroy()`。

## 常见错误

- 把 `Point` 参数顺序写成纬度、经度；构造顺序是 `lng, lat`。构造函数不校验取值范围，写反不会报错。
- 依赖 `clone()` 复制所有值对象；`Size` 没有该方法，应统一用构造函数重建。
- 把 `Point.equals` 的带容差判等（`1e-8`）与 `Pixel.equals` / `Size.equals` 的严格 `===` 当成同一套语义。
- 把 Map 画布像素传给只接受覆盖物容器像素的逻辑。
- 在非 BD09 坐标类型的地图上，把 `pointToOverlayPixel` / `overlayPixelToPoint` 当成可逆往返。
- 在地图移动、缩放、旋转后继续使用旧像素缓存。
- 用普通对象代替 API 明确要求的 `BMap.Point`、`BMap.Pixel` 或 `BMap.Size`。
- 忽略空 `Bounds`，直接使用可能为空的 `getCenter()` 结果。
- 在可能为空的 `Bounds` 上直接调用 `containsPoint()`，它没有空范围保护，会抛错而不是返回 `false`。
- 把 `toSpan()` 的返回对象当成 `Point`；它实际是 `Size`。
