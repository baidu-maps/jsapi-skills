# Baidu Map JSAPI Skills

[中文文档](README_zh.md)


This repository provides AI assistant **Skills** for [Baidu Map JSAPI](https://lbsyun.baidu.com/index.php?title=jspopularGL). Use them with [Claude](https://claude.ai), [Cursor](https://cursor.com), or other clients that support skills so the AI can reference accurate API docs and examples when you work with Baidu Map.

## Included Skills

| Skill | Description |
|-------|-------------|
| **bmap-jsapi-gl** | Baidu Map JSAPI WebGL (BMapGL/BMap): map init, overlays (markers, polylines, polygons), events, layers, routing, geocoding, etc. For 2D map development. |
| **bmap-jsapi-three** | Build professional 3D maps and GIS apps with MapV-Three - a Z-up coordinate based 3D map library. Supports map editing, measurement tools, feature drawing, data management and geo-visualization. For creating map editors, measurement tools, spatial data visualization and other Web-GIS applications. |

## How to Use

### 1. Clone the repo

```bash
git clone https://github.com/baidu-maps/jsapi-skills.git
cd jsapi-skills
```

### 2. Register the skill with your AI assistant

Link or copy the `bmap-jsapi-gl` directory into your environment’s skills folder so the AI can load its docs during conversations.

**Claude Desktop (local)**

- Skills directory is usually: `~/.claude/skills/`
- Register via symlink (recommended):
  ```bash
  ln -sfn "$(pwd)/bmap-jsapi-gl" ~/.claude/skills/bmap-jsapi-gl
  ```
- Or copy the `bmap-jsapi-gl` folder into `~/.claude/skills/`.

**Cursor**

- Symlink or copy the skill directory into Cursor’s skills path (see Cursor docs; often `~/.cursor/skills/` or a project-configured path).

### 3. Use it in chat

When your questions mention “Baidu Map”, “BMapGL”, “jsapi-gl”, or similar, the assistant will use this skill’s docs to give answers and code that match the Baidu Map JSAPI.

## Repo structure

```
.
├── bmap-jsapi-gl/          # Baidu Map JSAPI WebGL skill
│   ├── SKILL.md            # Skill entry and index
│   └── references/         # API reference docs
├── bmap-jsapi-three/       # MapV-Three 3D map skill
│   ├── SKILL.md            # Skill entry and index
│   └── references/         # API reference docs
└── README.md
```

`SKILL.md` lists all reference files so the AI can load them as needed.

## License

If a skill declares a license (e.g. MIT), that applies to that skill; the repo may use the same license.
