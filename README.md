# Baidu Map JSAPI Skills

[中文文档](README_zh.md)


This repository provides AI assistant **Skills** for [Baidu Map JSAPI](https://lbsyun.baidu.com/index.php?title=jspopularGL). Use them with [Claude](https://claude.ai), [Cursor](https://cursor.com), or other clients that support skills so the AI can reference accurate API docs and examples when you work with Baidu Map.

## Included Skills

| Skill | Description |
|-------|-------------|
| **bmap-jsapi-gl** | Baidu Map JSAPI WebGL (BMapGL/BMap): map initialization, overlays (markers, polylines, polygons), events, layers, routing, geocoding, etc. For 2D and 2.5D map development. |
| **bmap-jsapi-three** | Baidu Map JSAPI Three (MapVThree): A Web-based 2D/3D integrated map visualization library built on three.js. Supports multi-source base map loading, 3D model loading, geo data visualization, natural environment rendering, measurement and editing, etc. For building professional 2D/3D integrated maps, WebGIS, digital twin and similar applications. |
| **jsapi-ui-kit** | Lightweight Baidu Map UI component library (@baidumap/jsapi-ui-kit). Provides PlaceSearch (keyword/nearby/bounds search) and PlaceDetail (POI details display) components for quick integration of standardized map UI. |

## Installation

Choose one of the following ways to install the skills.

### Option 1: npx skills add (recommended)

If you use a client that supports the `skills` CLI, add this repo’s skills in one command:

```bash
npx skills add baidu-maps/jsapi-skills
```

This will install all skills from this repository into your environment’s skills directory.

### Option 2: Manual installation

**Step 1: Get the repo**

Clone the repo:

```bash
git clone https://github.com/baidu-maps/jsapi-skills.git
cd jsapi-skills
```

Or download the `jsapi-skills.zip` asset from [Releases](https://github.com/baidu-maps/jsapi-skills/releases), then unzip:

```bash
unzip jsapi-skills.zip
```

**Step 2: Register the skills**

Link or copy the skill directories under `skills/` into your environment’s skills folder.

**Claude Desktop (local)**

- Skills directory is usually: `~/.claude/skills/`
- Register via symlink (recommended):
  ```bash
  ln -sfn "$(pwd)/skills/bmap-jsapi-gl" ~/.claude/skills/bmap-jsapi-gl
  ln -sfn "$(pwd)/skills/bmap-jsapi-three" ~/.claude/skills/bmap-jsapi-three
  ln -sfn "$(pwd)/skills/jsapi-ui-kit" ~/.claude/skills/jsapi-ui-kit
  ```
- Or copy the `skills/bmap-jsapi-gl`, `skills/bmap-jsapi-three`, and `skills/jsapi-ui-kit` folders into `~/.claude/skills/`.

**Cursor**

- Skills directory is usually: `~/.cursor/skills/`
- Register via symlink (recommended):
  ```bash
  ln -sfn "$(pwd)/skills/bmap-jsapi-gl" ~/.cursor/skills/bmap-jsapi-gl
  ln -sfn "$(pwd)/skills/bmap-jsapi-three" ~/.cursor/skills/bmap-jsapi-three
  ln -sfn "$(pwd)/skills/jsapi-ui-kit" ~/.cursor/skills/jsapi-ui-kit
  ```
- Or copy the `skills/bmap-jsapi-gl`, `skills/bmap-jsapi-three`, and `skills/jsapi-ui-kit` folders into `~/.cursor/skills/`.

## How to Use

When your questions mention “Baidu Map”, “BMapGL”, “jsapi-gl”, “MapVThree”, “jsapi-ui-kit”, or similar, the assistant will use this skill’s docs to give answers and code that match the Baidu Map JSAPI.

## Repo structure

```
.
├── skills/
│   ├── bmap-jsapi-gl/          # Baidu Map JSAPI WebGL skill
│   │   ├── SKILL.md            # Skill entry and index
│   │   └── references/         # API reference docs
│   ├── bmap-jsapi-three/       # MapV-Three 3D map skill
│   │   ├── SKILL.md            # Skill entry and index
│   │   └── references/         # API reference docs
│   └── jsapi-ui-kit/           # Baidu Map UI component library skill
│       ├── SKILL.md            # Skill entry and index
│       └── references/         # API reference docs
└── README.md
```

`SKILL.md` lists all reference files so the AI can load them as needed.

## License

If a skill declares a license (e.g. MIT), that applies to that skill; the repo may use the same license.
