# 百度地图 JSAPI Skills

[English Documentation](README.md)


本仓库专门存放**百度地图 JSAPI** 相关的 AI 助手 Skills，供 [Claude](https://claude.ai)、[Cursor](https://cursor.com) 等在使用百度地图相关 API 时加载，以提供准确的 API 说明与示例。

## 包含的 Skill

| Skill | 说明 |
|-------|------|
| **bmap-jsapi-gl** | 百度地图 JSAPI WebGL 版 (BMapGL/BMap)：地图初始化、覆盖物（标注/折线/多边形）、事件、图层、路线规划、地理编码等。适用于 2D、2.5D 地图页面开发。 |
| **bmap-jsapi-three** | 百度地图 JSAPI Three 版 (MapVThree)：基于three.js的Web二三维一体化地图可视化库，支持多源底图加载、三维模型加载、地理数据可视化、自然环境渲染、量测编辑等功能。适用于构建专业的二三维一体化地图、WebGIS、数字孪生等应用。 |
| **jsapi-ui-kit** | 轻量级百度地图 UI 组件库 (@baidumap/jsapi-ui-kit)：提供 PlaceSearch（关键字/周边/范围检索）和 PlaceDetail（POI 详情展示）组件，快速集成标准化的地图 UI。 |

## 如何使用

### 1. 克隆本仓库

```bash
git clone https://github.com/baidu-maps/jsapi-skills.git
cd jsapi-skills
```

### 2. 将 Skill 注册到你的 AI 助手

把 `bmap-jsapi-gl`、`bmap-jsapi-three`、`jsapi-ui-kit` 目录链接或复制到当前环境对应的 skills 目录，这样 AI 在对话时会自动读取这些文档。

**Claude Desktop（本地）**

- Skills 目录一般为：`~/.claude/skills/`
- 注册（软链，推荐）：
  ```bash
  ln -sfn "$(pwd)/bmap-jsapi-gl" ~/.claude/skills/bmap-jsapi-gl
  ln -sfn "$(pwd)/bmap-jsapi-three" ~/.claude/skills/bmap-jsapi-three
  ln -sfn "$(pwd)/jsapi-ui-kit" ~/.claude/skills/jsapi-ui-kit
  ```
- 或直接把 `bmap-jsapi-gl`、`bmap-jsapi-three`、`jsapi-ui-kit` 文件夹复制到 `~/.claude/skills/` 下。

**Cursor**

- Skills 目录一般为：`~/.cursor/skills/`
- 注册（软链，推荐）：
  ```bash
  ln -sfn "$(pwd)/bmap-jsapi-gl" ~/.cursor/skills/bmap-jsapi-gl
  ln -sfn "$(pwd)/bmap-jsapi-three" ~/.cursor/skills/bmap-jsapi-three
  ln -sfn "$(pwd)/jsapi-ui-kit" ~/.cursor/skills/jsapi-ui-kit
  ```
- 或直接把 `bmap-jsapi-gl`、`bmap-jsapi-three`、`jsapi-ui-kit` 文件夹复制到 `~/.cursor/skills/` 下。

### 3. 在对话中使用

在支持 Skills 的客户端里，当你的问题涉及「百度地图」「BMapGL」「jsapi-gl」「MapVThree」「jsapi-ui-kit」等时，助手会优先参考本仓库中对应 skill 的文档来回答，从而给出更贴合百度地图 JSAPI 的代码与用法。

## 仓库结构

```
.
├── bmap-jsapi-gl/          # 百度地图 JSAPI WebGL Skill
│   ├── SKILL.md            # Skill 入口与索引
│   └── references/         # API 参考文档
├── bmap-jsapi-three/       # 百度地图 JSAPI Three Skill
│   ├── SKILL.md            # Skill 入口与索引
│   └── references/         # API 参考文档
├── jsapi-ui-kit/           # 百度地图 UI 组件库 Skill
│   ├── SKILL.md            # Skill 入口与索引
│   └── references/         # API 参考文档
└── README.md
```

`SKILL.md` 中会列出其下所有参考文档，便于 AI 按需读取。

## 许可

Skill 内若声明了 license（如 MIT），以该声明为准；仓库整体可沿用相同许可。

