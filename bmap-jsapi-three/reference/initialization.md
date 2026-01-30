# MapV Three 初始化指南

## 安装

```bash
npm install @baidumap/mapv-three three
```

## 静态资源配置

### Webpack

```javascript
const CopyWebpackPlugin = require('copy-webpack-plugin');

module.exports = {
    plugins: [
        new CopyWebpackPlugin({
            patterns: [
                {from: 'node_modules/@baidumap/mapv-three/dist/assets', to: 'mapvthree/assets'},
            ],
        }),
    ]
};
```

### Vite

```javascript
import copy from 'rollup-plugin-copy';

export default {
    plugins: [
        copy({
            targets: [
                {src: 'node_modules/@baidumap/mapv-three/dist/assets', dest: 'public/mapvthree'},
            ],
            hook: 'buildStart',
        }),
    ]
};
```

### 全局变量

```html
<script>
    window.MAPV_BASE_URL = 'mapvthree/';
</script>
```

## 基础初始化

```javascript
import * as mapvthree from '@baidumap/mapv-three';

// 配置百度地图 AK（可选，使用百度底图时需要）
mapvthree.BaiduMapConfig.ak = '您的AK密钥';

// 创建引擎
const engine = new mapvthree.Engine(document.getElementById('map'), {
    map: {
        center: [116.404, 39.915],
        zoom: 15
    },
    rendering: {
        enableAnimationLoop: true
    }
});

// 添加底图
engine.add(new mapvthree.MapView({
    imageryProvider: new mapvthree.BingImageryTileProvider()
}));
```

## 构造函数配置

```javascript
new mapvthree.Engine(container, {
    map: {
        center: [116.404, 39.915],   // 初始中心点 [经度, 纬度]
        heading: 0,                   // 旋转角度（0-360）
        pitch: 45,                    // 俯仰角（0-90）
        range: 5000,                  // 视野距离（米）
        projection: 'EPSG:3857',      // 投影：'EPSG:3857' 或 'ecef'
        is3DControl: true             // 3D 控制器
    },
    rendering: {
        enableAnimationLoop: true,    // 启用动画循环
        pixelRatio: window.devicePixelRatio,
        features: {
            antialias: { enabled: true, method: 'smaa' },
            bloom: { enabled: false, strength: 0.1 },
            shadow: { enabled: false }
        }
    },
    controller: {
        enableRotate: true,           // 启用旋转
        enableZoom: true,             // 启用缩放
        enablePan: true,              // 启用平移
        enableTilt: true,             // 启用倾斜
        minimumZoomDistance: 1,       // 最小缩放距离
        maximumZoomDistance: Infinity // 最大缩放距离
    }
});
```

## 视野控制

### 基础方法

```javascript
// 设置/获取中心点
engine.map.setCenter([116.404, 39.915]);
engine.map.getCenter();  // [lng, lat]

// 设置/获取缩放级别
engine.map.setZoom(15);
engine.map.getZoom();

// 设置/获取旋转角度
engine.map.setHeading(45);
engine.map.getHeading();

// 设置/获取俯仰角
engine.map.setPitch(60);
engine.map.getPitch();

// 设置/获取视野距离
engine.map.setRange(5000);
engine.map.getRange();
```

### lookAt - 综合设置视野

```javascript
engine.map.lookAt([116.404, 39.915], {
    heading: 30,
    pitch: 60,
    range: 3000
});
```

### flyTo - 动画飞行

```javascript
engine.map.flyTo([116.404, 39.915], {
    heading: 0,
    pitch: 45,
    range: 2000,
    duration: 2000,  // 动画时长（毫秒）
    complete: () => console.log('飞行完成')
});
```

### setViewport - 根据点集设置视野

```javascript
// 自动调整视野以包含所有点
engine.map.setViewport([
    [116.38, 39.90],
    [116.42, 39.92],
    [116.40, 39.88]
]);
```

### 边界限制

```javascript
// 限制可拖动范围
engine.map.setBounds([
    [116.30, 39.85],  // 西南角
    [116.50, 39.95]   // 东北角
]);

// 限制缩放距离
engine.map.setMinRange(100);
engine.map.setMaxRange(50000);
```

## 添加图层

```javascript
// 添加可视化图层
const polygon = engine.add(new mapvthree.Polygon({
    color: 'red',
    extrude: true,
    extrudeValue: 100
}));
polygon.dataSource = await mapvthree.GeoJSONDataSource.fromURL('data.geojson');

// 添加 Three.js 对象
const position = engine.map.projectArrayCoordinate([116.404, 39.915, 0]);
const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(10, 10, 10),
    new THREE.MeshStandardMaterial({ color: 0xff0000 })
);
mesh.position.set(...position);
engine.add(mesh);

// 移除图层
engine.remove(polygon);
```

## 事件绑定

```javascript
// 图层事件
polygon.addEventListener('click', (e) => {
    console.log('点击了多边形', e.entity);
});

// 地图事件
engine.map.addEventListener('click', (e) => {
    console.log('点击了地图', e.point);
});
```

## 坐标转换

```javascript
// 地理坐标 → 投影坐标
const projected = engine.map.projectArrayCoordinate([116.404, 39.915, 0]);

// 投影坐标 → 地理坐标
const geographic = engine.map.unprojectArrayCoordinate(projected);
```

> 坐标系详细说明见 `common/coordinate-system.md`

## 底图提供者

```javascript
// Bing 卫星图
new mapvthree.BingImageryTileProvider()

// OpenStreetMap
new mapvthree.OSMImageryTileProvider()

// 天地图（需配置 TiandituConfig.tk）
new mapvthree.TiandituImageryTileProvider()

// XYZ 瓦片
new mapvthree.XYZImageryTileProvider({
    url: 'https://example.com/tiles/{z}/{x}/{y}.png'
})

// WMS 服务
new mapvthree.WMSImageryTileProvider({
    url: 'https://example.com/wms',
    layers: 'layer_name'
})

// WMTS 服务
new mapvthree.WMTSImageryTileProvider({
    url: 'https://example.com/wmts',
    layer: 'layer_name',
    style: 'default',
    tileMatrixSetID: 'EPSG:3857'
})
```

## 销毁

```javascript
// 销毁引擎，释放所有资源
engine.dispose();

// 注意：销毁后引擎不可再使用
```

## 百度地图适配器

兼容百度地图 JSAPI GL 的 API：

```javascript
import * as BMapGL from '@baidumap/mapv-three/adapters/bmap';

const map = new BMapGL.Map('map');
map.centerAndZoom(new BMapGL.Point(116.404, 39.915), 15);

// 添加标注
const marker = new BMapGL.Marker(new BMapGL.Point(116.404, 39.915));
map.addOverlay(marker);

// 添加多边形
const polygon = new BMapGL.Polygon([
    new BMapGL.Point(116.395, 39.910),
    new BMapGL.Point(116.405, 39.910),
    new BMapGL.Point(116.405, 39.920),
    new BMapGL.Point(116.395, 39.920)
], {
    strokeColor: 'blue',
    fillColor: 'red',
    fillOpacity: 0.4
});
map.addOverlay(polygon);
```

## 参考文档

- `common/coordinate-system.md` - 坐标系和投影
- `common/event-binding.md` - 事件绑定
- `datasource.md` - 数据源
- `editor.md` - 编辑器
- `measure.md` - 测量工具
