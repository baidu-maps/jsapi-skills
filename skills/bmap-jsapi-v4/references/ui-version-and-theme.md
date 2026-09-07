# UI 主题

## 何时读取

需要调整 4.0 控件、信息窗口、检索或路线面板的整体配色，或者实现深色模式时读取。底图要素样式使用 `map.setMapStyle()`，不属于本页主题系统。

## 快速选择

| 需求 | 做法 |
|---|---|
| 切换内置深色/浅色 | `map.setTheme('dark')` / `map.setTheme('light')` |
| 覆盖部分变量 | `map.setTheme('light', variables)` |
| 注册复用主题 | `map.registerTheme(name, variables)` 后 `map.setTheme(name)` |
| 读取当前主题 | `map.getTheme()` |

使用 `v=4.0` 时不需要额外设置 UI 版本。

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
map.centerAndZoom(new BMap.Point(116.404, 39.915), 12);

const navigation = new BMap.NavigationControl();
map.addControl(navigation);

map.registerTheme('brand', {
  '--bmap-color-primary': '#7c3aed',
  '--bmap-color-primary-bg': '#f3e8ff',
  '--bmap-border-radius': '10px',
});
map.setTheme('brand');

console.log(map.getTheme());

function dispose() {
  map.removeControl(navigation);
  map.setTheme('light');
  map.destroy();
}
```

## 核心 API

`map.setTheme(theme, customVariables?)` 应用主题。内置主题名是 `'light'` 和 `'dark'`；未注册的名称会回落到 `'light'`。第二个参数会覆盖所选主题的同名变量。

`map.registerTheme(name, variables)` 注册自定义主题；`map.getTheme()` 返回当前主题名。主题与 `setMapStyle()` 相互独立：前者控制页面 UI，后者控制底图要素。

主题变量写入 `document.documentElement`，属于页面级共享状态。同一页面创建多张地图时，最后一次主题调用会影响全部使用这些变量的地图 UI。

## 常用变量

| 变量 | 作用 |
|---|---|
| `--bmap-color-primary` | 主色，也参与覆盖物默认描边 |
| `--bmap-color-primary-bg` | 主色浅底，也参与覆盖物默认填充 |
| `--bmap-color-bg-base` | 控件与面板底色 |
| `--bmap-color-text-base` | 主文本色 |
| `--bmap-color-text-weak` | 次要文本色 |
| `--bmap-color-border` | 边框色 |
| `--bmap-color-fill` | 次级填充色 |
| `--bmap-border-radius` | 圆角 |
| `--bmap-font-size` | 基础字号 |
| `--bmap-box-shadow` | 浮层阴影 |

覆盖物默认颜色通常在构造时读取主题变量；之后切换主题不能保证已经创建的覆盖物全部回溯更新。业务要求固定颜色时，在覆盖物 options 中显式设置。

## 事件与验证

主题 API 没有完成事件。基础样式表和浏览器样式计算可能存在时序差异；不要把调用返回当作视觉效果已经验证。最终配色需要在有效 AK 的 4.0 页面实际检查。

## 资源清理

- 移除本组件创建的控件、覆盖物和面板。
- 如果组件负责页面主题，卸载前恢复约定的默认主题。
- `map.destroy()` 不会自动清除写在根元素上的主题变量。

## 常见错误

- 把 `setTheme()` 当成底图样式 API。
- 同页多地图分别设置不同主题，忽略主题变量是页面级共享的。
- 依赖默认 Marker 尺寸或默认覆盖物颜色做像素级布局。
- 认为切换主题必然回溯更新所有已创建覆盖物。
