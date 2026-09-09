# CoC 城墙编辑器 · Wall Crafter

一个纯前端、零依赖的 **Clash of Clans 阵型/城墙编辑器**。单文件 HTML + 原生 Canvas，打开即用，无需构建、无需安装。

A zero-dependency, single-file **Clash of Clans base / wall layout editor** built with vanilla Canvas. Just open it and start crafting — no build step, no install.

---

## ✨ Features / 功能

- **等距画布 Isometric canvas** — `44 × 44` 逻辑网格，45° 等距投影渲染；坐标判定发生在投影之前，所以连接逻辑与屏幕像素无关。
- **自动连接件 Auto wall joints** — 每格墙根据 4 个正交邻居（up / right / down / left，斜角不连接）自动挑选 7 种 sprite 之一，所见即所得。
- **21 级墙皮肤 Wall levels 1–21** — `21 × 7 = 147` 张 WebP 资产 + 村庄底图，切换等级即时生效。
- **四种编辑模式 Edit modes** — 绘制 draw / 文字 text / 图片 image / 模板 template，支持复制 copy、剪切 cut、粘贴 paste。
- **8 种预设地图 Map presets** — 边界 border、网格 grid、分形 fractal、希尔伯特 Hilbert、利萨如 Lissajous、同心环 rings、玫瑰线 rose、螺旋 spiral，一键铺满阵型骨架。
- **连接预览 Connection preview** — 预览画布与主画布共用同一套分类器 classifier，预览即是生产代码的直接诊断。
- **导出 Export** — 一键导出阵型 PNG。
- 缩放 / 重置视图 zoom & reset，全部在浏览器本地运行 100% client-side。

## 🚀 Usage / 使用

无需构建，直接打开：

```bash
# 方式一：本地直接打开
# Simply open index.html in your browser

# 方式二：起个静态服务（推荐，避免部分浏览器 file:// 限制）
python -m http.server 8000
# → http://localhost:8000
```

> GitHub Pages：仓库设置里把 Pages 指到根目录即可，零配置上线。

## 🧱 Wall connection spec / 连接件命名规范

每级墙 7 个资产，后缀表示几何形态与邻居规则：

| 后缀 Suffix | 几何 Geometry | 邻居规则 Neighbor rule |
| --- | --- | --- |
| `i` | 孤立墙 isolated | 无正交邻居 no orthogonal neighbor |
| `y` | Y/X 交叉 junction | 3–4 个正交邻居 |
| `v` | 拐角 corner (V) | 恰好 2 个相邻正交邻居 |
| `row` | 竖向延伸 vertical | 上下均有邻居 |
| `col` | 横向延伸 horizontal | 左右均有邻居 |
| `rowtail` | 竖向端头 vertical end | 恰好 1 个竖向邻居 |
| `coltail` | 横向端头 horizontal end | 恰好 1 个横向邻居 |

坐标模型：`(0, 0)` 位于菱形顶部，`(43, 0)` 位于右侧顶点；连接判定仅看 up / right / down / left 四方向，**斜角不连接**。

## 📁 Project structure / 目录结构

```text
coc wall crafter/
├── index.html          # 全部逻辑：UI + Canvas 渲染 + 分类器 (single-file app)
├── README.md
└── images/
    ├── base/           # 村庄底图 village background
    │   └── base4.webp
    └── walls/          # 21 级 × 7 连接件 = 147 张
        └── {1..21}/{level}{i|y|v|row|col|rowtail|coltail}.webp
```

## 🖼 Rendering pipeline / 渲染流程

1. 收集所有被占用的逻辑坐标 occupied cells。
2. 对每个 cell 检查 4 个正交邻居。
3. 按 neighbor rule 选出 `i / y / v / row / col / rowtail / coltail` 之一。
4. 将逻辑坐标投影到 45° 等距画布。
5. 绘制选中的 sprite（sprite 本身不旋转）。

## 📄 License / 许可

MIT。游戏内资产（WebP 图片）版权归 **Supercell** 所有，本项目仅作学习研究用途，与 Supercell 无关。
Game assets belong to Supercell; this project is for personal / educational use only, not affiliated with Supercell.
