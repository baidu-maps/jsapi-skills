# TypeScript 集成

## 何时读取

项目使用 TypeScript、需要安装 JSAPI 4.0 类型声明、解决 `Cannot find namespace 'BMap'`，或为类型包暂未覆盖的运行时成员补充 `.d.ts` 时读取。

## 本页导航

[官方类型包](#官方类型包) · [引入声明](#引入声明) · [加载运行时](#加载运行时) · [补充暂缺声明](#补充暂缺声明) · [排障](#排障)

## 官方类型包

- npm：<https://www.npmjs.com/package/@baidumap/jsapi-v4-types>
- GitHub：<https://github.com/baidu-maps/jsapi-v4-types>

安装：

```bash
npm install -D @baidumap/jsapi-v4-types
```

该包是纯 `.d.ts` 声明包，`package.json` 的入口是 `"types": "index.d.ts"`，不包含 JSAPI 运行时代码，也不提供可在浏览器执行的模块导出。

## 引入声明

未设置 `compilerOptions.types` 时，TypeScript 通常会自动发现已安装的声明包。项目已经限定 `types` 时，将包名追加到原数组：

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2020", "DOM"],
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true,
    "types": ["@baidumap/jsapi-v4-types"]
  }
}
```

当前官方 `4.0.3` 发布包自身仍有少量未解析的声明引用，因此示例暂时开启 `skipLibCheck`；它只跳过依赖声明文件的内部检查，业务源码仍受 `strict` 约束。升级类型包后应重新关闭验证。

不要写下面这种具名导入：

```typescript
// 错误：类型包不是运行时模块。
import { Map } from '@baidumap/jsapi-v4-types';
```

## 加载运行时

类型包不会创建全局 `BMap`。页面仍要在业务入口之前加载 JSAPI 4.0：

```html
<div id="map"></div>
<script src="https://api.map.baidu.com/api?v=4.0&ak=YOUR_BAIDU_MAP_AK"></script>
<script type="module" src="/src/main.ts"></script>
```

业务代码直接使用全局命名空间：

```typescript
const map = new BMap.Map('map');
const center = new BMap.Point(116.404, 39.915);
map.centerAndZoom(center, 15);

const onClick = (event: BMap.MapEventMap['click']) => {
  console.log(event.point);
};

map.addEventListener('click', onClick);

export function disposeMap(): void {
  map.removeEventListener('click', onClick);
  map.destroy();
}
```

## 补充暂缺声明

类型包尚未收录项目实际使用的 4.0 运行时成员时，把最小补充集中放在项目自己的 `.d.ts` 文件中；升级类型包后重新核对并删除已被官方覆盖的声明。

例如 4.0 提供 `BMap.Icons`，但当前类型包尚未声明，可建立 `src/types/bmap-v4-runtime.d.ts`：

```typescript
export {};

declare global {
  namespace BMap {
    const Icons: {
      readonly MARKER_PIN: string;
      createIcon(svg: string, options?: {
        color?: string;
        size?: number | Size;
        anchor?: Size;
        imageOffset?: Size;
      }): Icon;
    };
  }
}
```

只声明业务实际使用的字段，不复制整套实现，也不要用 `any` 掩盖问题。

## 排障

出现 `Cannot find namespace 'BMap'` 时依次检查：

1. `npm ls @baidumap/jsapi-v4-types` 是否能找到包。
2. `tsconfig.json` 是否用 `types` 排除了该包。
3. `include` 是否包含业务源码和自定义 `.d.ts`。
4. `tsc --traceResolution` 实际解析到了哪个 `index.d.ts`。

编译通过但浏览器报 `BMap is not defined`，说明运行时脚本未加载成功或业务入口执行得更早；这不是类型声明问题。

## 常见错误

- 把 `@baidumap/jsapi-v4-types` 当作运行时模块导入。
- 安装类型包后省略 JSAPI 4.0 `<script>`。
- 覆盖原有 `compilerOptions.types` 条目。
- 把类型包是否包含某个名称当成浏览器运行时是否可用的唯一依据。
- 为内部未导出类编写声明并在业务代码中使用。
