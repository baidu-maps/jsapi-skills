# 地图视角动画

## 何时读取

需要用关键帧平滑改变地图中心、缩放、倾斜或朝向，监听动画开始/结束/取消，并在组件卸载时安全取消动画和销毁地图时读取。

## 本页导航

[快速选择](#快速选择) · [最小示例](#最小可运行示例) · [核心 API](#核心-api) · [资源清理](#资源清理) · [常见错误](#常见错误)

## 快速选择

| 需求 | API | 调用重点 |
|---|---|---|
| 定义飞行动画 | `BMap.ViewAnimation` | 至少两帧；用 `percentage` 标出 0~1 时间位置 |
| 启动 | `map.startViewAnimation(animation)` | 内部按 delay 通过 setTimeout 启动，不依赖返回值 |
| 暂停/继续 | `pauseViewAnimation` / `continueViewAnimation` | 始终传同一个 ViewAnimation 实例，且同样要等动画真正启动之后 |
| 取消 | `cancelViewAnimation` | 只在 animationstart 调度完成后取消 |
| 生命周期事件 | ViewAnimation 事件 | 监听 `animationstart`、`animationend`、`animationcancel` |

## 最小可运行示例

```javascript
const map = new BMap.Map('map');
const start = new BMap.Point(116.391, 39.907);
const end = new BMap.Point(116.430, 39.935);
map.centerAndZoom(start, 14);

const animation = new BMap.ViewAnimation(
  [
    { center: start, zoom: 14, tilt: 0, heading: 0, percentage: 0 },
    { center: end, zoom: 18, tilt: 60, heading: 120, percentage: 1 },
  ],
  { duration: 4000, delay: 0, interation: 1 },
);

let readyToCancel = false;
let settled = false;
let cleanupRequested = false;
let cleaned = false;

function finishCleanup() {
  if (!readyToCancel || cleaned) {
    return;
  }
  cleaned = true;
  if (!settled) {
    map.cancelViewAnimation(animation);
  }
  animation.removeEventListener('animationstart', onAnimationStart);
  animation.removeEventListener('animationend', onAnimationEnd);
  animation.removeEventListener('animationcancel', onAnimationCancel);
  map.destroy();
}

function onAnimationStart() {
  // start 事件先于内部 Animation 创建；微任务后才可安全取消。
  Promise.resolve().then(() => {
    readyToCancel = true;
    if (cleanupRequested) {
      finishCleanup();
    }
  });
}

function onAnimationEnd() {
  settled = true;
}

function onAnimationCancel() {
  settled = true;
}

animation.addEventListener('animationstart', onAnimationStart);
animation.addEventListener('animationend', onAnimationEnd);
animation.addEventListener('animationcancel', onAnimationCancel);
map.startViewAnimation(animation);

function dispose() {
  cleanupRequested = true;
  finishCleanup();
}
```

该保护逻辑同时处理“启动后卸载”和“调用 start 后立即卸载”：后一种情况会等 `animationstart` 同步派发结束，再在微任务中取消并销毁 Map。动画实际帧率和视觉效果必须在带有效 AK 的 `v=4.0` 页面在线验证。

## 核心 API

`BMap.ViewAnimation` 的关键帧可包含 `center`、`zoom`、`tilt`、`heading` 和 `percentage`。启动时会用 Map 当前视角补齐缺失的四个视角字段，并在相邻关键帧间插值；正式代码仍应明确写首尾帧及 percentage，避免难以审查的隐式状态。

补齐动作是**原地写回你传入的关键帧对象**，不是拷贝。所以同一个关键帧数组被第二个 ViewAnimation 复用、或同一个动画被重新启动时，第一次补进去的视角值会留在数组里，不会再按当时的地图状态重新求值。需要「跟随当前视角」的语义就每次新建关键帧对象，不要复用同一份数组。

选项是 `delay`、`duration` 和拼写为 `interation` 的循环次数；无限循环使用字符串 `'INFINITE'`。不要自行改成 `iteration`，实现只读取 `interation`。

`transition` 接收进度并返回缓动后的进度，默认是线性。`BMap.Transitions` 提供预置缓动函数。

`interation: 'INFINITE'` 的动画永远不会派发 `animationend`：每轮结束只派发 `animationiterations` 然后重新开始。这类动画只能靠 `map.cancelViewAnimation(animation)` 结束，因此业务必须持有实例并在卸载时取消。

Map 用同一个动画实例执行 `startViewAnimation`、`pauseViewAnimation`、`continueViewAnimation` 和 `cancelViewAnimation`。取消实现会恢复动画启动前的 POI 显示状态、停止内部 Animation、清除 Map 的动画时间并派发 animationcancel。

`pauseViewAnimation`、`continueViewAnimation`、`cancelViewAnimation` 三个方法都直接操作动画实例内部的 Animation 对象，而这个对象是在 `animationstart` 派发**之后**才创建的。因此在动画真正启动前调用这三个方法都会抛 `TypeError`，不只是 cancel。主示例的 `readyToCancel` 标记同样适用于暂停与继续。

动画期间地图会临时关闭 POI 显示：启动时记下当前的 POI 开关并置为关闭，正常结束或取消时再恢复成记下的值。因此不要在动画进行中调 `setDisplayOptions({poi: true})`，那次修改会在动画收尾时被覆盖回启动前的值。

动画结束时不会再补渲染一帧末帧的精确值：收尾补帧在内部默认关闭，按进度插值的分支对进度 1 也不成立。所以 `animationend` 时地图停在最后一次脉冲的插值位置，`getZoom()` / `getTilt()` / `getHeading()` 可能与末帧有极小偏差。需要精确终态就在 `animationend` 回调里显式把视角设成末帧值。

短生命周期组件不要把业务等待时间写进 ViewAnimation 的 `delay`。保持 `delay: 0`，需要延后启动时由业务持有可清理的 timer；timer 触发后再进入主示例的 animationstart/cancel 生命周期：

```javascript
let startTimer = null;

function scheduleAnimation(delayMs) {
  if (startTimer !== null) window.clearTimeout(startTimer);
  startTimer = window.setTimeout(() => {
    startTimer = null;
    map.startViewAnimation(animation);
  }, delayMs);
}

function cancelScheduledStart() {
  if (startTimer === null) return;
  window.clearTimeout(startTimer);
  startTimer = null;
}
```

`startViewAnimation()` 的实现没有返回值。不要保存或清除其调用结果，也不要把它当作计时器 ID；取消动画使用 `map.cancelViewAnimation(animation)`。

当 ViewAnimation 自身配置非零 `delay` 时，`startViewAnimation` 创建的内部 setTimeout 没有公开句柄，`cancelViewAnimation` 也不能在 `_start` 前取消这段等待；立即卸载会留下迟到启动风险。因此建议固定 `delay: 0`，业务延迟使用可 `clearTimeout` 的外部 timer。

## 事件或回调

- ViewAnimation 公开 `animationstart`、`animationiterations`、`animationcancel` 和 `animationend`；实现分别在启动、每轮结束、取消和最终结束时派发。
- 监听器注册在 ViewAnimation，不是 Map；解绑要传注册时的同一函数引用。
- `animationstart` 在内部 Animation 构造之前同步派发；因此不要在该监听函数的当前调用栈直接 cancel，至少延迟到微任务。
- `animationend` 表示正常完成，`animationcancel` 表示主动取消；两者都可以把业务状态标记为 settled。
- 循环动画每轮结束派发 `animationiterations`；只有最后一轮才派发 `animationend`，`'INFINITE'` 则永远不派发 `animationend`。

## 资源清理

- Map 仍存活且动画正在运行：等待启动调度完成后调用 `map.cancelViewAnimation(animation)`，再移除 ViewAnimation 监听器。
- 动画已正常结束：无需再次 cancel，但仍应移除业务监听器。结束后再 cancel 不会报错，但会多派发一次 `animationcancel`，让只看事件判断状态的业务逻辑误判；主示例用 `settled` 标记避免这次多余调用。
- Map 与动画同寿命：按主示例先结束动画和监听，再调用 `map.destroy()`。
- 清理前先 `cancelScheduledStart()`；若业务 timer 已触发，则按主示例等待 animationstart 调度完成后 cancel。
- `startViewAnimation` 仍会用内部 setTimeout 调用 `_start`；主示例用 `delay: 0` 把不可控窗口压到单次任务调度，并把真正销毁延后到安全取消点。
- ViewAnimation 自身没有公开的 destroy 方法；生命周期由 cancel、监听器解绑和最终 Map.destroy 共同闭合。

## 常见错误

- 把 `interation` 写成直觉上的 `iteration`，导致循环配置不生效。
- 关键帧少于两帧、percentage 未按从 0 到 1 排列，或相邻 percentage 相同造成插值异常。
- 把 `startViewAnimation()` 的调用结果当作可清理的计时器 ID。
- 在调用 start 后同步 cancel；此时内部 animation 可能尚未创建。
- 把数秒业务延迟写进 ViewAnimation.delay，卸载时却没有可清理的 timer 句柄。
- 在 animationstart 监听函数内同步 cancel，仍早于内部 Animation 构造。
- 以为只有 cancel 需要等启动：pause、continue 在动画启动前调用同样抛 `TypeError`。
- 复用同一个关键帧数组创建第二个动画，却期待缺失字段按当时的地图视角重新补齐。
- 用 `'INFINITE'` 循环却等 `animationend` 收尾，或忘记在卸载时 cancel，导致 POI 一直处于关闭状态。
- 把 `animationend` 时的 `getZoom()`、`getTilt()` 当成与末帧完全相等。
- 给暂停、继续或取消方法传入另一个 ViewAnimation 实例。
- 只销毁 DOM，不取消活动动画、解绑监听和销毁 Map。
