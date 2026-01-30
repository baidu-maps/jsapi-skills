# 百度地图 JSAPI Skills

[English Documentation](README.md)


本仓库专门存放**百度地图 JSAPI** 相关的 AI 助手 Skills，供 [Claude](https://claude.ai)、[Cursor](https://cursor.com) 等在使用百度地图相关 API 时加载，以提供准确的 API 说明与示例。

## 包含的 Skill

| Skill | 说明 |
|-------|------|
| **bmap-jsapi-gl** | 百度地图 JSAPI WebGL 版 (BMapGL/BMap)：地图初始化、覆盖物（标注/折线/多边形）、事件、图层、路线规划、地理编码等。适用于 2D 地图页面开发。 |
| **bmap-jsapi-three** | 使用 MapV-Three 构建专业的 3D 地图和 GIS 应用 - 基于 Z-up 坐标系的 3D 地图库，支持地图编辑、测量工具、要素绘制、数据管理等地理可视化功能。适用于创建地图编辑器、测量工具、空间数据可视化等 Web-GIS 应用。 |

## 如何使用

### 1. 克隆本仓库

```bash
git clone https://github.com/baidu-maps/jsapi-skills.git
cd jsapi-skills
```

### 2. 将 Skill 注册到你的 AI 助手

把 `bmap-jsapi-gl` 目录链接或复制到当前环境对应的 skills 目录，这样 AI 在对话时会自动读取这些文档。

**Claude Desktop（本地）**

- Skills 目录一般为：`~/.claude/skills/`
- 注册（软链，推荐）：
  ```bash
  ln -sfn "$(pwd)/bmap-jsapi-gl" ~/.claude/skills/bmap-jsapi-gl
  ```
- 或直接把 `bmap-jsapi-gl` 文件夹复制到 `~/.claude/skills/` 下。

**Cursor**

- 将上述目录软链或复制到 Cursor 使用的 skills 目录（具体路径以 Cursor 文档为准，通常为 `~/.cursor/skills/` 或项目内配置的 skills 路径）。

### 3. 在对话中使用

在支持 Skills 的客户端里，当你的问题涉及「百度地图」「BMapGL」「jsapi-gl」等时，助手会优先参考本仓库中对应 skill 的文档来回答，从而给出更贴合百度地图 JSAPI 的代码与用法。

## 仓库结构

```
.
├── bmap-jsapi-gl/          # 百度地图 JSAPI WebGL Skill
│   ├── SKILL.md            # Skill 入口与索引
│   └── references/         # API 参考文档
├── bmap-jsapi-three/       # MapV-Three 3D 地图 Skill
│   ├── SKILL.md            # Skill 入口与索引
│   └── references/         # API 参考文档
└── README.md
```

`SKILL.md` 中会列出其下所有参考文档，便于 AI 按需读取。

## 许可

Skill 内若声明了 license（如 MIT），以该声明为准；仓库整体可沿用相同许可。

