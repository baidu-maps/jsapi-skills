# RoutePlan 路径规划组件

路径规划 UI 组件，封装百度地图路径规划服务，提供统一的 API 和归一化的数据格式。

> **注意**: 目前仅开放**驾车**路径规划功能。公交、骑行、步行等后续会陆续开放。

## 构造函数

```typescript
new RoutePlan(container: string | HTMLElement, options: RoutePlanOptions)
```

**参数**:

| 参数      | 类型                    | 必填 | 说明                  |
| --------- | ----------------------- | ---- | --------------------- |
| container | `string \| HTMLElement` | 是   | 容器元素或 CSS 选择器 |
| options   | `RoutePlanOptions`      | 是   | 配置选项              |

**RoutePlanOptions**:

```typescript
interface RoutePlanOptions {
    /** 地图实例，必传 */
    map: BMap.Map | BMapGL.Map;
    /** 驾车配置 */
    drivingOptions?: DrivingOptions;
}

interface DrivingOptions {
    /** 驾车策略，默认 DEFAULT (0) */
    policy?: DrivingPolicy;
    /** 是否返回备选路线：0-返回一条推荐路线，1-返回1-3条路线供选择，默认 0 */
    alternatives?: 0 | 1;
}
```


## 导入方式

```typescript
import {
    RoutePlan,
    DrivingPolicy,
    // 以下暂未开放
    TransitPolicy,
    IntercityPolicy,
    TransitTypePolicy,
    LineType,
} from '@baidumap/jsapi-ui-kit';

// 类型导入
import type {
    RoutePlanOptions,
    RoutePlanType,
    RoutePlanSearchOptions,
    RoutePlanEventData,
    NormalizedRouteResult,
    NormalizedPlan,
    NormalizedPoint,
    NormalizedLocation,
    Segment,
    DriveSegment,
} from '@baidumap/jsapi-ui-kit';
```

## 驾车策略枚举

```typescript
enum DrivingPolicy {
    DEFAULT = 0,                        // 默认策略
    LEAST_DISTANCE = 2,                 // 距离最短（不考虑限行和路况，用于估价场景）
    AVOID_HIGHWAYS = 3,                 // 不走高速
    FIRST_HIGHWAYS = 4,                 // 高速优先
    AVOID_CONGESTION = 5,               // 躲避拥堵
    AVOID_PAY = 6,                      // 少收费
    HIGHWAYS_AVOID_CONGESTION = 7,      // 高速优先+躲避拥堵
    AVOID_HIGHWAYS_CONGESTION = 8,      // 不走高速+躲避拥堵
    AVOID_CONGESTION_PAY = 9,           // 躲避拥堵+少收费
    AVOID_HIGHWAYS_CONGESTION_PAY = 10, // 不走高速+躲避拥堵+少收费
    AVOID_HIGHWAYS_PAY = 11,            // 不走高速+少收费
    DISTANCE_PRIORITY = 12,             // 距离优先（考虑限行和路况，距离相对短但不一定稳定）
    TIME_PRIORITY = 13,                 // 时间优先
}
```


## 方法

### search(searchOptions)

发起路径规划搜索。

```typescript
search(searchOptions: RoutePlanSearchOptions): Promise<NormalizedRouteResult>
```

**RoutePlanSearchOptions**:

| 参数      | 类型              | 必填 | 说明                                   |
| --------- | ----------------- | ---- | -------------------------------------- |
| start     | `Point`           | 是   | 起点坐标                               |
| end       | `Point`           | 是   | 终点坐标                               |
| startName | `string`          | 否   | 起点名称（用于外部导航展示）           |
| endName   | `string`          | 否   | 终点名称（用于外部导航展示）           |
| startUid  | `string`          | 否   | 起点POI的UID，填写后将提升路线规划的准确性 |
| endUid    | `string`          | 否   | 终点POI的UID，填写后将提升路线规划的准确性 |
| waypoints | `Point[]`         | 否   | 途经点数组（驾车最多 10 个）           |

**示例**:

```javascript
// 基础用法
await routePlan.search({
    start: new BMapGL.Point(116.404, 39.915),
    end: new BMapGL.Point(116.305, 39.982),
});

// 带途经点
await routePlan.search({
    start: new BMapGL.Point(116.404, 39.915),
    end: new BMapGL.Point(116.305, 39.982),
    waypoints: [
        new BMapGL.Point(116.35, 39.95),
    ],
});

// 带UID提升准确性
await routePlan.search({
    start: new BMapGL.Point(116.404, 39.915),
    end: new BMapGL.Point(116.305, 39.982),
    startUid: 'xxx',
    endUid: 'yyy',
});

// 填写起终点名称
await routePlan.search({
    start: new BMapGL.Point(116.404, 39.915),
    startName: '北京西站',
    startUid: 'xxx',
    end: new BMapGL.Point(116.305, 39.982),
    endName: '西直门',
    endUid: 'yyy',
});
```

### clear()

清空搜索结果和地图覆盖物。

```typescript
clear(): void
```

### getCurrentType()

获取当前路径规划类型。

```typescript
getCurrentType(): RoutePlanType
```

**返回值**: `'driving'` | `'transit'` | `'riding'` | `'walking'`

### getLastResult()

获取上次搜索的归一化结果。

```typescript
getLastResult(): NormalizedRouteResult | null
```

### destroy()

销毁组件，清理 DOM、事件和地图覆盖物。

```typescript
destroy(): void
```


### on(event, handler)

注册事件监听。

```typescript
on(event: string, handler: Function): this
```


### off(event, handler?)

移除事件监听。

```typescript
off(event: string, handler?: Function): this
```


## 事件

| 事件名      | 触发时机               | 回调参数                                    |
| ----------- | ---------------------- | ------------------------------------------- |
| `result`    | 搜索成功               | `RoutePlanEventData`                        |
| `error`     | 搜索失败               | `Error`                                     |
| `typechange`| 路径类型切换           | `{ type: RoutePlanType }`                   |
| `planselect`| 方案选中（展开详情）   | `{ type, planIndex, plan }`                 |
| `clear`     | 清空结果               | 无                                          |
| `navclick`  | 点击"开始导航"按钮     | `{ type, result, planIndex, plan }`         |

**示例**:

```javascript
// 监听搜索结果
routePlan.on('result', (data) => {
    console.log('起点:', data.start.title);
    console.log('终点:', data.end.title);
    console.log('方案数量:', data.plans.length);
});

// 监听错误
routePlan.on('error', (err) => {
    console.error('路径规划失败:', err.message);
});

// 监听方案切换
routePlan.on('planselect', (data) => {
    console.log('选中方案:', data.planIndex);
    console.log('距离:', data.plan.distanceText);
    console.log('时长:', data.plan.durationText);
});

// 监听导航调起
routePlan.on('navclick', (data) => {
    // 可在此处理跳转百度地图 App 进行导航
    console.log('调起导航', data);
});
```


## 类型定义

### RoutePlanEventData

```typescript
interface RoutePlanEventData {
    /** 路径规划类型 */
    type: RoutePlanType;
    /** 起点信息 */
    start: NormalizedPoint;
    /** 终点信息 */
    end: NormalizedPoint;
    /** 方案数组 */
    plans: NormalizedPlan[];
}
```

### NormalizedRouteResult

```typescript
interface NormalizedRouteResult {
    /** 路径规划类型 */
    routeType: RoutePlanType;
    /** 起点信息 */
    start: NormalizedPoint;
    /** 终点信息 */
    end: NormalizedPoint;
    /** 方案数组 */
    plans: NormalizedPlan[];
}
```

### NormalizedPlan

```typescript
interface NormalizedPlan {
    /** 总距离（米） */
    distance: number;
    /** 距离文本 */
    distanceText: string;
    /** 总时长（秒） */
    duration: number;
    /** 时长文本 */
    durationText: string;
    /** 路段数组 */
    segments: Segment[];

    // 驾车特有字段
    /** 过路费（元） */
    toll?: number;
    /** 收费路段距离（米） */
    tollDistance?: number;
    /** 红绿灯个数 */
    trafficLights?: number;
    /** 路况/方案特点摘要，如「一路畅通|时间少」 */
    tag?: string;
    /** 途经点数组 */
    waypoints?: string[];
    /** 完整路径坐标点 */
    path?: NormalizedLocation[];
}
```

### NormalizedPoint

```typescript
interface NormalizedPoint {
    /** 地点名称 */
    title: string;
    /** 坐标位置 */
    location: NormalizedLocation;
    /** 所属城市 */
    city?: string;
    /** UID */
    uid?: string;
}
```

### NormalizedLocation

```typescript
interface NormalizedLocation {
    /** 经度 */
    lng: number;
    /** 纬度 */
    lat: number;
}
```

### DriveSegment

```typescript
interface DriveSegment extends BaseSegment {
    type: 'drive';
    /** 路段描述 */
    description: string;
    /** 起点位置 */
    location: NormalizedLocation;
    /** 道路名称 */
    roadName?: string;
    /** 行驶时长（秒） */
    duration?: number;
}
```


## 使用示例

### 基础用法

```javascript
import { RoutePlan, DrivingPolicy } from '@baidumap/jsapi-ui-kit';

// 初始化地图
const map = new BMapGL.Map('map');
map.centerAndZoom(new BMapGL.Point(116.404, 39.915), 12);

// 创建路径规划组件
const routePlan = new RoutePlan('#route-panel', {
    map: map,
    drivingOptions: {
        policy: DrivingPolicy.AVOID_CONGESTION,
        alternatives: 1,  // 返回 1-3 条备选路线
    },
});

// 发起搜索
await routePlan.search({
    start: new BMapGL.Point(116.404, 39.915),
    end: new BMapGL.Point(116.305, 39.982),
});
```

### 配合地点搜索组件使用

```javascript
import { RoutePlan, PlaceAutocomplete, PlaceSearch } from '@baidumap/jsapi-ui-kit';

const map = new BMapGL.Map('map');
map.centerAndZoom(new BMapGL.Point(116.404, 39.915), 12);

// 起点输入框
const startInput = new PlaceAutocomplete('#start-input', { map });
// 终点输入框
const endInput = new PlaceAutocomplete('#end-input', { map });

// 路径规划组件
const routePlan = new RoutePlan('#result-panel', { map });

// 存储选中的起终点
let startPoint, endPoint;

startInput.on('select', (poi) => {
    startPoint = poi.point;
    trySearch();
});

endInput.on('select', (poi) => {
    endPoint = poi.point;
    trySearch();
});

function trySearch() {
    if (startPoint && endPoint) {
        routePlan.search({
            start: startPoint,
            end: endPoint,
        });
    }
}
```

### 自定义地图绘制

```javascript
const routePlan = new RoutePlan('#panel', { map });

const overlays = [];

function clearOverlays() {
    overlays.forEach(o => map.removeOverlay(o));
    overlays.length = 0;
}

function drawRoute(plan) {
    if (plan.path && plan.path.length > 0) {
        const points = plan.path.map(pt => new BMapGL.Point(pt.lng, pt.lat));
        const polyline = new BMapGL.Polyline(points, {
            strokeColor: '#22c55e',
            strokeWeight: 8,
            strokeOpacity: 0.8,
        });
        map.addOverlay(polyline);
        overlays.push(polyline);
    }
}

function addMarker(location, label, color) {
    const point = new BMapGL.Point(location.lng, location.lat);
    const marker = new BMapGL.Marker(point);
    map.addOverlay(marker);
    overlays.push(marker);
}

routePlan.on('result', (data) => {
    clearOverlays();
    const plan = data.plans[0];
    if (plan) {
        drawRoute(plan);
        addMarker(data.start.location, '起', '#0f9977');
        addMarker(data.end.location, '终', '#e53935');
    }
});

routePlan.on('planselect', (data) => {
    clearOverlays();
    drawRoute(data.plan);
    const result = routePlan.getLastResult();
    if (result) {
        addMarker(result.start.location, '起', '#0f9977');
        addMarker(result.end.location, '终', '#e53935');
    }
});
```

## 注意事项

1. **坐标点必填**: start 和 end 参数必须是坐标点（Point），不支持地点名称
2. **途经点限制**: 驾车路径规划最多支持 10 个途经点（起终点除外）
3. **组件销毁**: 组件销毁时会自动清理所有事件监听器和地图覆盖物
4. **地图依赖**: 必须先加载百度地图 JS API（BMap 或 BMapGL）
