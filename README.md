# Komari Nexus

Komari Nexus 是一款面向 [Komari Monitor](https://komari-document.pages.dev/) 的极简状态监控主题。它使用 Vue 3、Tailwind CSS 4 与 shadcn-vue 的 CSS 变量约定构建，视觉上强调细边框、克制留白、等宽数字和状态优先的信息层级。

## 特性

- 极简仪表盘布局，适合服务器监控、副屏和节点墙
- Light / Dark / System 外观模式
- 节点分组筛选，使用 `nodeSelectedGroup` 本地存储字段
- Grid / Table 视图切换，使用 `nodeViewMode` 本地存储字段
- 响应式设计：移动端自动使用卡片视图，避免宽表格横向溢出
- 接入 Komari 公开接口与实时 WebSocket 数据
- 支持 Komari 1.0.5+ managed theme configuration
- 保留 Komari 主题必需的标题、描述占位和页脚声明

## 技术栈

- Vue 3 + `<script setup>`
- Vite
- TypeScript
- Tailwind CSS 4
- shadcn-vue CSS variables convention
- Lucide Vue icons

## 主题接口

主题会使用以下 Komari 公开接口：

- `GET /api/public`：站点公开设置和主题动态配置
- `GET /api/nodes`：节点基础信息
- `WebSocket /api/clients`：实时节点状态，连接后发送 `get` 获取快照

如果本地开发时接口不可用，页面会显示内置演示数据，方便预览主题效果。

## 动态配置

`komari-theme.json` 中声明了以下可管理配置：

| Key | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `nexus_title` | string | 空 | 顶部站点别名；留空时使用 Komari 站点名 |
| `nexus_density` | select | `comfortable` | 可选 `comfortable` / `compact`，用于控制节点墙密度 |
| `nexus_show_console` | switch | `true` | 是否显示底部 System Stream 面板 |

## 开发

安装依赖：

```bash
pnpm install
```

启动开发服务器：

```bash
pnpm dev
```

构建生产产物：

```bash
pnpm build
```

预览构建结果：

```bash
pnpm preview
```

## 打包主题

Komari 主题包需要在 zip 根目录包含 `komari-theme.json` 和 `dist/`：

```text
theme.zip
├── komari-theme.json
└── dist/
    ├── index.html
    └── assets/
```

构建后可以使用 Python 打包：

```bash
pnpm build
python3 - <<'PY'
from pathlib import Path
import zipfile

out = Path('komari-theme-nexus.zip')
out.unlink(missing_ok=True)

with zipfile.ZipFile(out, 'w', zipfile.ZIP_DEFLATED) as archive:
    archive.write('komari-theme.json', 'komari-theme.json')
    for path in Path('dist').rglob('*'):
        if path.is_file():
            archive.write(path, path.as_posix())

print(out.resolve())
PY
```

然后在 Komari 后台的主题管理中上传 `komari-theme-nexus.zip`。

## Komari 主题约定

本主题遵循 Komari 主题开发要求：

- `dist/index.html` 保留 `<title>Komari Monitor</title>`
- `dist/index.html` 保留 `<meta name="description" content="A simple server monitor tool.">`
- 页面保留 `Powered by Komari Monitor.` 页脚
- 不占用 `/admin` 和 `/terminal` 路由
- `vite.config.ts` 使用 `base: './'`，便于部署在 `/themes/Nexus/dist/` 下

## License

MIT
