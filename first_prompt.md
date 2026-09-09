# CoC Wall Crafter · 城墙编辑器 — 完整功能与实现说明书

> 本文档不是"产品构想"，而是对**当前已实现的 `index.html`** 的逐项还原说明，目的是让另一个 AI Agent 在**不看源码、只读本文档**的情况下，一次性重新生成功能等价的可交付 `index.html`。
>
> 文档结构：一、项目定位 → 二、整体 UI 结构与组件清单 → 三、四种输入模式详解 → 四、右侧参数面板详解 → 五、【难点 1】等距网格坐标系与城墙贴图对齐 → 六、【难点 2】7 种城墙连接性判断算法 → 七、资源文件约定 → 八、交互事件与状态机 → 九、非功能性要求。

---

## 一、项目定位

**CoC Wall Crafter** 是一个纯前端（单文件 HTML + CSS + JS，无后端、无构建工具）的《部落冲突》**44×44 等距村庄城墙布局编辑器**。当前实现是一个"编辑器/校准工具"产品形态，**不是**营销落地页——没有"Get Premium"、没有画廊/社区、没有多基地（家乡/建筑大师基地/部落都城）切换。核心价值是：

1. 把**文字 / 图片 / 参数化图案 / 手绘**四种来源统一转换成 44×44 网格上的城墙点集；
2. 用与游戏原生等距视角一致的方式，把这些城墙点渲染成叠加在村庄底图（base PNG）上的城墙贴图；
3. 每一段城墙根据其上下左右邻居，自动选用 7 种连接形态贴图中的一种，实现"墙段像游戏里一样首尾相连"的视觉效果。

---

## 二、整体 UI 结构与组件清单

页面为左右两栏布局（`.workspace` = `.stage`(主画布区，弹性宽) + `.rail`(右侧参数栏，固定宽，可滚动)）。

### 2.1 顶栏 `.topbar`

| 元素 | 类型 | id | 说明 |
| --- | --- | --- | --- |
| 品牌图标+标题 | 静态 | — | "Wall Crafter" + "COC" 徽标 tag + 副标题"部落冲突 · 城墙布局编辑器" |
| 状态指示 | 文本+圆点 | `statusText` | 显示当前模式状态，如"就绪""图片已加载 / 实时""绘制模式 / 就绪" |
| 清空布局 | 按钮 | `clearButton` | 清空图片/绘制数据，重置到图片模式空状态 |
| 导出阵型 PNG | 按钮（主色） | `exportButton` | `canvas.toDataURL('image/png')` 直接下载主画布当前渲染结果 |

### 2.2 主画布区 `.stage`

- **模式切换 Tab**（`.mode-tabs[role=tablist]`，4 个）：`文字` `图片` `模板` `绘制`，对应 `data-mode="text|image|test|draw"`。切换即调用 `selectMode(mode)`，联动右侧面板显隐（详见第八节状态机）。
- **模式提示文案**：`modeHint`，随模式变化提示一句话；旁边固定显示 `44 × 44 · 1936 格`。
- **主画布卡片 `.canvas-card`**：
  - `<canvas id="preview" width="1600" height="1200">`：实际渲染画布（2 倍于 800×600 的设计基准分辨率，保证导出清晰）。
  - HUD 悬浮层（画布左上角）：`wallCount`（当前墙段数）`/ statLimit`（上限，绘制模式下显示"—"）+ `densityLabel`（填充率百分比）。
  - 视口工具条 `[data-viewport-tools]`：`−`（缩小）`100%`（`[data-zoom-label]`，只读显示当前缩放）`+`（放大）`↺`（重置视图/居中）。缩放中心为画布几何中心，每次点击按 ×1.25/÷1.25 步进。
  - 空状态提示层 `emptyState`：仅在"图片"模式且未上传图片时显示，含"⌁"图标、"拖入阵型图开始"文案、"选择文件"按钮（`emptyUpload`，触发隐藏的 `fileInput`）。
- **画布底部说明行**：左侧 `imageNote`（如"暂无阵型图" / 图片尺寸+文件名），右侧固定提示"点击墙段可临时调暗，便于校准"。
- **连接参考面板 `connectionPanel`**（始终展示，不再局限于"模板"模式）：标题"连接参考"+ 副标题"7 种连接 · 对应 7 种墙片"。内含 6 个预览瓦片（`.connection-tile`），每个含一个小型 `<canvas data-connection="...">` + 名称 + 说明：

  | data-connection | 名称 | 说明 |
  | --- | --- | --- |
  | single | 单段 | 孤立墙段 |
  | col | 竖向直段 | 上下贯通 |
  | row | 横向直段 | 左右贯通 |
  | v | 拐角 | 上左相接 |
  | y | T 形 | 三向连接 |
  | cross | 十字 | 四向交汇 |

  > 注意：预览只展示 6 种可视形态，但真实贴图资源共有 **7 种**（见第六节），因为"竖向直段"在两端收尾处还需要 `coltail` / `rowtail` 两种"端头"贴图，这两种没有单独的预览瓦片，只在真实网格里、墙段只有单侧邻居时才会用到。

### 2.3 右侧参数栏 `.rail`（自上而下，5 个可折叠 `.panel`，标题带自动编号）

1. **"阵型来源" panel** — 依当前模式显示对应子面板（文字参数 / 图片上传区 / 模板选择区）。
2. **"阵型参数" panel** — 全局墙等级、墙段上限、显示开关（详见 4.2）。
3. **"采样区" panel（`roiPanel`）** — 仅文字/图片模式显示，ROI 预览画布（详见 4.3）。
4. **"变换" panel（`transformPanel`）** — 仅文字/图片模式显示，旋转/缩放/偏移滑杆（详见 4.4）。
5. **"网格校准" panel（`paramsPanel`）** — 菱形宽高、中心点 X/Y 四个滑杆；**当前默认永久隐藏**（`hidden = true`，代码保留但不对用户暴露，属于开发期校准工具，用于把 44×44 菱形网格与 base PNG 的村庄轮廓对齐，调好后固化为默认值即可，无需再暴露给终端用户）。

> 每次切换模式后会调用 `renumberPanels()`，按"当前可见的面板"重新从 1 开始编号标题前缀，保证可见面板序号连续无跳号。

---

## 三、四种输入模式详解

顶部 Tab 切换模式时：清空 `state.image`、清空临时调暗集合 `dimWalls`、按需重建 `state.demoWalls`、切换各面板 `hidden`、更新 `modeHint`、重绘。

### 3.1 文字模式（`text`，默认模式）

- **输入**：`textInput`（`<textarea rows=3>`，placeholder 示例文案，如部落战常见墙字"TH""WAR"等）。
- **字体**：`fontFamily`（下拉，通过浏览器 Local Font Access API `queryLocalFonts()` 探测系统已装字体，若不支持/被拒绝则回退到一组内置候选字体做 `probeAvailableFonts` 探测）+ `fontRescan`（🔍 按钮，重新扫描字体列表）+ `fontWeight`（字重/样式下拉，依所选字体动态生成 `populateFontStyles`）+ `textAlign`（左/居中(默认)/右）。
- **处理流程**：`renderTextRaster()` 用 canvas 2D 文本 API 把输入文字渲染成栅格图 → `textSampleWindow()` 计算实际取样窗口（正方形，居中裁剪/留白）→ `createTextWalls()` 按像素明暗生成墙段点集，赋 `rank = row + column` 用于图层排序。
- 任何影响栅格化的参数（字体、字重、对齐、RGB 权重、阈值等，凡在统一的 `input` 事件委托列表内的控件）变化时都会重新调用 `createTextWalls()` 并重绘。

### 3.2 图片模式（`image`）

- **上传区 `dropZone`**：`<label for="fileInput">` 支持拖放（文案"拖放阵型图" + "选择文件"按钮态）+ 隐藏的 `<input id="fileInput" type="file" accept="image/png,image/jpeg">`。
- **加载**：`loadImage(file)` 用 `URL.createObjectURL` 加载为 `Image` 对象，加载完成后隐藏空状态、更新 `imageNote` 为"宽×高 / 文件名"、状态文案改为"图片已加载 / 实时"。
- **识别参数**（详见 4.5）：RGB→灰度权重（R/G/B 三个滑杆，默认 0.299/0.587/0.114，语义等价于 ITU-R BT.601 亮度公式）、二值化阈值（0–255 滑杆，默认 128）、反相开关 `invert`。
- **像素采样**：源图先经过"变换"面板的旋转/缩放/偏移变换，绘制进一个 400×400 的离屏 `sourceCanvas`（背景保持 **透明**，不是填黑——这是刻意设计：透明区域永远不参与二值判断、永远不生成城墙，避免"图片四角被误判成墙"）。随后按 44×44 网格逐格采样中心像素点：
  - 若该像素 alpha < 8（近乎透明）→ 该格永远判定"无墙"，不受阈值/反相影响；
  - 否则计算灰度值 `gray = R×redWeight + G×greenWeight + B×blueWeight`；
  - 非反相：`gray < threshold` → 判定为墙；反相：`gray >= threshold` → 判定为墙。

### 3.3 模板模式（`test`）

用途：不依赖用户输入，用参数化算法生成 8 种预设"骨架图案"，再做等距变换/连接性校准验证。切换二级 Tab（`[data-map]`，8 个）：

| data-map | 名称 | 生成方式 |
| --- | --- | --- |
| border | 边界 | 44×44 外圈满墙（谓词式：`row/column` 处于边缘）|
| grid | 网格 | 规则网格线图案 |
| spiral | 螺旋 | 参数曲线（`buildParametric`），螺旋线采样 |
| fractal | 分形 | 递归/分形规则生成点集 |
| rings | 同心环 | 多层同心圆环谓词 |
| hilbert | 希尔伯特 | 希尔伯特曲线（`hilbertXY(order, distance)`，经典 d2xy 算法）|
| lissajous | 利萨如 | 利萨如曲线参数方程采样 |
| rose | 玫瑰线 | 玫瑰曲线（极坐标 r = cos(kθ)）参数方程采样 |

- **骨架→实体的"生长"算法**：每种模板先用 `buildFromPredicate` / `buildFromPoints` / `buildParametric` 生成一批"种子"墙格，再交给 `growFromSeeds(seeds, cap)`：
  1. `fieldDistance(seeds)` 用 **3-4 Chamfer 距离变换**（轴向权重 3、对角权重 4，整数运算避免浮点误差）对全网格算出每格到最近种子的近似欧氏距离场（双向扫描：先从左上到右下正向松弛，再从右下到左上反向松弛）；
  2. 按距离值分层，**整层取或不取**（保证图案在膨胀过程中始终对称完整，不会出现半层锯齿），逐层膨胀直到达到当前"墙段上限" `cap`，多余的层直接舍弃，返回 `{ walls, radius }`。
- 该模式下 ROI 采样区面板显示"测试模式不支持 ROI 预览"提示遮罩（`roiMask`），因为模板没有"源图像"概念。
- 变换面板 / 网格校准面板在该模式下依然隐藏（因为 `hasSource = mode==='text' || mode==='image'`，模板不算"有源"）。

### 3.4 绘制模式（`draw`）— 唯一支持手动编辑、多等级混用、复制粘贴的模式

- **进入绘制模式**：若从其他模式切入，`importWallsIntoDraw(fromMode, signature)` 会把当前来源模式渲染出的墙段"拍平"复制一份作为绘制模式初始内容（带一次性签名防重复导入），状态提示"绘制模式 / 已复制 N 段城墙"。
- **城墙等级选择**：`drawWallLevel`（下拉，1–21 级，默认 12），**新选的等级只影响之后新画的墙**，已放置的墙段各自"固化"等级（存于每个 wall 对象的 `.level` 字段），因此绘制模式支持同一布局里混用不同等级贴图。
- **工具条 `draw-toolbar`（4 个互斥工具按钮，`data-draw-tool`）**：
  - `drawBrushBtn` 画笔（默认激活）：左键拖动连续放置墙段（`pointerdown/pointermove` 期间持续判定，同一格不重复放置）。
  - `drawEraserBtn` 橡皮：与画笔共用同一拖动逻辑，但 `state.isErasing=true` 时改为删除已存在的墙段。
  - `drawSelectBtn` 选择：拖出矩形选区（`selectionStart`/`selectionEnd`，虚线橙色高亮框 + 半透明橙色填充），松开后选区内的墙段进入 `selectedWalls`。
  - `drawPanBtn` 平移：临时切到画布平移交互（不放墙/不选择）。
- **剪贴板工具条 `clipboardTools`**（仅当有选区/剪贴板内容时显示）：`drawCopyBtn` 复制、`drawCutBtn` 剪切（复制后同时删除原墙段）、`drawDeselectBtn` 取消选择。复制/剪切后进入"跟随鼠标的半透明预览"（`clipboardGhost=true`，0.8 透明度贴图跟随 `hoveredCell` 位置渲染），此时点击画布即在鼠标所在格"盖章"粘贴一次（可连续多次粘贴，按 Esc 取消跟随预览）。
- **清空绘制**：`drawClearBtn`（危险色按钮）清空 `state.drawWalls`。
- 悬停高亮：画笔/橡皮激活且未处于剪贴板跟随预览时，鼠标所在格用描边矩形高亮（画笔为黄绿色描边，橡皮为珊瑚色描边）。
- 该模式下：HUD 上限显示"—"（不受 325 等上限裁剪）；右侧"阵型参数"面板中的全局等级行与上限滑杆整体隐藏（因为每段墙有自己的等级、且不裁剪）。

---

## 四、右侧参数面板详解

### 4.1 "阵型来源"面板内的子控件（按模式互斥显示，见 3.1–3.3）

### 4.2 "阵型参数"面板（`paramsPanel` 内容，非"网格校准"那个同名易混淆面板——注意实现里这个可见面板标题也叫"阵型参数"，id 是 `#capacityControl` 所在的那个 section，务必在重写时保持两个不同 id 的面板不要混淆：可见的一个负责等级/上限/显示开关，隐藏的 `paramsPanel`(id) 负责菱形宽高与中心点校准）

- `wallLevelRow`：全局"城墙等级"下拉 `wallLevel`（1–21，默认 12），影响文字/图片/模板三种模式下所有墙段的贴图等级（绘制模式下隐藏此行，因为该模式每段墙自带等级）。
- `capacityControl`：滑杆 `capacity`（min=1, max=1936=44×44, 默认 325，对应 17 本上限），实时输出 `capacityOut`；生成的墙段按 `rank = row+column` 升序排序后截断到该上限（绘制模式不受此限制，该行连同 `wallLevelRow` 一并隐藏）。
- `showWalls` 开关：关闭后主画布切换为"调试网格视图"——不绘制城墙贴图和村庄背景细节层，而是绘制：44×44 灰色网格线（含每 6 格一个的行/列数字标注）+ 每个已占用格子的橙色菱形色块（用于直接核对"墙位坐标 ↔ 底图网格"是否对齐，是校准阶段的可视化工具）。

### 4.3 "采样区"面板（ROI，`roiPanel`）

- `<canvas id="roiPreview" width=440 height=440>` + 视口缩放工具（同主画布）+ 遮罩层 `roiMask`（无源数据/测试模式时覆盖提示文案）。
- 底部固定说明："透明区域不生成城墙"。
- **文字模式**下：ROI 直接展示"文字栅格化后的采样窗口"本身（棋盘格打底表示透明区，叠加 44×44 红色细网格线）。
- **图片模式**下：ROI 展示"应用了旋转/缩放/偏移变换 + 灰度化 + 二值化/反相"之后的最终判定结果图（黑白二值图，透明像素保持透明），同样叠加棋盘格底 + 红色网格线，并在标题旁显示"正常/反相 阈值=N"。
- 目的：让用户在动手前，用一张更直观的黑白/网格小图，预判每个格子到底会不会被识别成墙，而不必在主画布的等距斜视图里反复肉眼核对。

### 4.4 "变换"面板（`transformPanel`，仅文字/图片模式）

四个滑杆，作用于送入采样前的离屏 `sourceCanvas`（400×400 设计空间，`translate(200,200)` 到中心后再依次应用）：

| 控件 id | 范围 | 默认 | 作用 |
| --- | --- | --- | --- |
| `imageRotation` | −180° ~ 180° | −45° | 源图旋转角度 |
| `imageScale` | 0% ~ 300% | 100% | 源图缩放比例 |
| `shiftX` | −100 ~ 100（step .1） | 0.0 | 横向偏移（内部 ×10 换算为像素）|
| `shiftY` | −100 ~ 100（step .1） | 0.0 | 纵向偏移（内部 ×10 换算为像素）|

变换顺序（对应 `renderSourceImage()`）：`translate(200,200)` → `translate(shiftX×10, shiftY×10)` → `rotate(rotation)` → `scale(scale/100)` → 以图片自身作为正方形（边长取宽高最大值）居中绘制。

### 4.5 图片模式专属参数（`imageParams`，属于"变换"面板下方追加区块）

- `invert` 开关："反相识别"。
- `threshold` 滑杆：0–255，默认 128，"灰度阈值"。
- RGB 权重三行（`redWeight`/`greenWeight`/`blueWeight`，各 0–1，step .001，默认 .299/.587/.114），每行左侧一个色块字母图标（R/G/B）+ 滑杆 + 数值只读输出（保留 3 位小数）。

### 4.6 "网格校准"面板（`paramsPanel`，当前对用户隐藏，但逻辑必须完整实现并保留）

四个滑杆，联动 `projectionMetrics()` 与 `gridToCanvas()`（详见第五节公式）：

| 控件 id | 范围 | 默认 | 语义 |
| --- | --- | --- | --- |
| `projectionX` | 420–760 px | 657 | 44×44 菱形网格在 800×600 设计空间下的"宽度"（横向对角线长度）|
| `projectionY` | 300–580 px | 489 | 44×44 菱形网格的"高度"（纵向对角线长度）|
| `centerX` | 35%–65% | 50.4% | 网格中心点相对画布宽度的横向百分比位置 |
| `centerY` | 35%–65% | 48.8% | 网格中心点相对画布高度的纵向百分比位置 |

这四个值是把抽象的 44×44 逻辑网格"贴合"到具体村庄底图 PNG 的等距透视上的**唯一自由度**，调节它们直到"网格调试视图"（4.2 的 `showWalls` 关闭态）里的橙色菱形与底图的实际地皮边界完全重合即为校准完成。**校准完成后应把这四个值写死为默认值，面板保持隐藏**，避免终端用户误触导致错位。

---

## 五、【难点 1】等距网格坐标系 与 城墙贴图对齐

这是整个项目**最容易做错**的部分：村庄底图 `base PNG` 是美术画好的固定等距（isometric，45° 旋转 + 压扁）视角图片，而逻辑上的 44×44 城墙格子是笛卡尔网格，两者必须通过统一的坐标变换公式精确重合，否则城墙贴图会"漂移"或"错位压在草地/建筑上"。

### 5.1 设计基准与最终画布的关系

- **设计基准空间**：`backgroundSize = { width: 800, height: 600 }`。所有校准滑杆（`projectionX/Y`、`centerX/Y`）的含义都是相对这个 800×600 空间定义的。
- **最终画布**：`<canvas id="preview" width="1600" height="1200">`，正好是设计基准的 **2 倍**，用于导出更清晰的 PNG；但由于所有涉及像素的计算都用 `canvas.width` / `canvas.height` 直接参与公式（见下），2 倍关系是自动保持一致的，重写时**不要**引入额外的换算系数，直接用 `canvas.width` / `canvas.height` 参与计算即可。
- 底图绘制：`context.drawImage(backgroundImage, 0, 0, canvas.width, canvas.height)`，即背景图始终被拉伸铺满整个画布。

### 5.2 核心公式

```js
const gridSize = 44;

// 1) 投影尺度：把"设计基准下的菱形宽/高（像素）"换算成"实际画布下的菱形宽/高（像素）"
function projectionMetrics() {
  return {
    width:  canvas.width  * projectionX / backgroundSize.width,
    height: canvas.height * projectionY / backgroundSize.height
  };
}

// 2) 逻辑网格坐标 (column, row) → 画布像素坐标 (x, y)
function gridToCanvas(column, row) {
  const projection = projectionMetrics();
  const centerX = canvas.width  * centerX% / 100;
  const centerY = canvas.height * centerY% / 100;
  return {
    x: centerX + (column - row) * projection.width  / 2 / (gridSize - 1),
    y: centerY + (column + row - (gridSize - 1)) * projection.height / 2 / (gridSize - 1)
  };
}
```

这就是标准的**等距（dimetric/isometric）网格投影公式**：`(column - row)` 决定横向位置（沿菱形左右两条边方向），`(column + row - (gridSize-1))` 决定纵向位置（沿菱形上下两条边方向），除以 `2 / (gridSize-1)` 是把 0..43 的格子索引线性映射到菱形半宽/半高上，`-(gridSize-1)` 项使得网格整体以中心点为原点对称分布。

### 5.3 单个城墙贴图的绘制

```js
const WALL_W = 57.12, WALL_H = 39.84; // 单个墙段贴图在"最终画布"像素坐标系下的固定尺寸（对应 2× 分辨率）

function placeWall(sprite, column, row) {
  const { x, y } = gridToCanvas(column, row);
  context.drawImage(sprite, x - WALL_W / 2, y - WALL_H / 2, WALL_W, WALL_H);
}
```

- 贴图尺寸是**固定像素值**、不随缩放滑杆联动（校准滑杆只移动"格点位置"，不缩放"贴图本身大小"）——这是刻意简化：只要 44×44 网格间距校准对了，配合美术出的贴图本身留白比例，视觉上就能拼合成连续城墙。若重写时贴图源文件比例变化，需要相应调整这两个常量，但不要让它们跟着 `projectionX/Y` 联动缩放。
- **图层排序（避免等距视角下前后遮挡出错）**：所有墙段按 `rank = row + column` 升序排序后依次绘制（"从远到近"，`compareWallsByRank`：先比较 rank，再比较 row，再比较 column），保证画面中靠"下前方"（等距视角里视觉上更靠近观察者）的墙段贴图后画、能正确遮住后方墙段的底边。**排序只影响绘制顺序，不影响墙段是否被保留**（上限裁剪发生在生成阶段，不在排序阶段）。

### 5.4 调试网格视图（用于人工核对对齐效果）

当 `showWalls` 开关关闭时，改为绘制一套"占位可视化"而非真实贴图，专门用来核对坐标系是否与底图重合：

- 一个额外的**微调偏移量**（仅调试视图使用，不影响真实贴图绘制）：`CELL_SHIFT_X = -0.3`、`CELL_SHIFT_Y = 1.9`（单位是"半格宽/半格高" `dx`/`dy`）。这两个数值是实测得出的、让灰色网格线与橙色菱形格心与真实底图网格完全重合所需的整体平移量，重写时应作为常量原样保留。
- 灰色网格线：以半整数网格坐标（`k - 0.5, k = 0..44`）为格子边界，画 44+1 条纵向 + 44+1 条横向网格线，每格恰好是一个墙位的"势力范围"。
- 橙色菱形：每个已占用格子在其格心位置画一个与网格线同朝向、边长恰好填满一格的菱形（`fillStyle:'#f28b2e', globalAlpha:.92`），相邻菱形边对边无缝拼接。
- 行/列标号：沿网格左上边和左下边，每 6 格标一次数字（0,6,12,...），字体 `bold 14px 'DM Mono', monospace`，帮助人工核对具体是第几行/列错位。

### 5.5 反向映射：屏幕像素 → 最近的网格格子（供绘制模式取用）

```js
function screenToGrid(clientX, clientY) {
  // 1. 把鼠标客户端坐标换算到画布内部像素坐标（考虑 CSS 显示尺寸与画布真实像素尺寸的比例）
  // 2. 再反向撤销当前视口的 pan/zoom（先减去 panX/panY 再除以 zoom，围绕画布中心）
  // 3. 暴力遍历全部 44×44=1936 个格子，取 gridToCanvas(column,row) 距离鼠标点最近的一个
  // 4. 若最近距离超过 (半格宽 × 1.5) 判定为"未命中任何格子"，返回 null
}
```

> 44×44=1936 个格子的暴力最近邻在现代浏览器里性能完全够用（每次指针事件 < 2000 次距离计算），**不需要**引入空间索引或反解析析析析公式优化，重写时按原样暴力实现即可，避免过度设计引入新 bug。

---

## 六、【难点 2】7 种城墙连接性判断算法

### 6.1 资源与命名约定

每个城墙等级（1–21 级）各有 **7 张**贴图，命名规则：

```
./images/walls/{level}/{level}{style}.webp
```

`style` 取自固定数组，**顺序即优先级参考顺序**：

```js
const wallAssetNames = ['i', 'y', 'v', 'rowtail', 'coltail', 'row', 'col'];
```

| style | 含义 | 出现条件（见 6.2） |
| --- | --- | --- |
| `i` | 孤立单段 | 上下左右均无相邻墙 |
| `col` | 竖直贯通段 | 上、下都有相邻墙 |
| `row` | 水平贯通段 | 左、右都有相邻墙 |
| `v` | 直角拐角（上-左） | 上、左有相邻墙，且不满足 `y` 的条件 |
| `y` | T 形三/四向交汇 | 上、左都有相邻墙，且（下 或 右）至少一个也有 |
| `coltail` | 竖直方向的端头（只连上方） | 只有"上"有相邻墙（其余三向都没有）|
| `rowtail` | 水平方向的端头（只连左方） | 只有"左"有相邻墙，且没有"上"（其余同上排除后落到这条）|

底图村庄的城墙贴图统一取自 `./images/base/base4.webp`（单张，800×600 设计基准）。

### 6.2 精确判断逻辑（必须逐字保留，不要"优化"或"对称化"）

```js
function connectionAsset(walls, row, column) {
  const occupied = new Set(walls.map(w => `${w.column},${w.row}`));
  const has = (r, c) => occupied.has(`${c},${r}`);
  const up    = has(row - 1, column);
  const right = has(row,     column + 1);
  const down  = has(row + 1, column);
  const left  = has(row,     column - 1);

  if (up && left && (down || right)) return 'y';   // 三向及以上交汇（含真正的十字）
  if (up && down)                    return 'col';  // 竖直贯通
  if (left && right)                 return 'row';  // 水平贯通
  if (up && left)                    return 'v';    // 直角拐角
  if (up)                            return 'coltail';
  if (left)                          return 'rowtail';
  return 'i';                                        // 孤立段
}
```

**逐条判定顺序具有优先级、不可互换**：先检查"上+左同时存在"这一族（覆盖 T 形/十字/拐角三种情形），再检查"上下贯通"，再检查"左右贯通"，最后才是端头/孤立。

### 6.3 已知的实现特性（重写时应保持一致，除非产品明确要求修正）

- **该判定只显式覆盖了"上"与"左"两个方向的端头/拐角**：一个格子如果**只有右邻居**或**只有下邻居**，或**同时只有右+下邻居（没有上、左）**，四个 `if` 都不命中，会落到最后的 `return 'i'`，被当作"孤立段"渲染，即使它视觉上明明连着别的墙。这是当前实现里的一个**已知不对称限制**：算法默认"每一段的连接状态由它与左上方相邻墙的关系决定"，右/下方向的连接依赖"对方那一格"在计算自己时把"左/上"识别出来——也就是说，**一条水平/竖直直线内部的墙段能正确显示为 row/col，但线段最右端 / 最下端的"收尾"格子**，如果它右边或下边是空的、左边或上边有墙，会被正确识别（因为它检查的是"左"和"上"，收尾格子的"左有墙"或"上有墙"依然成立）；**真正会出问题的只有"这一段的唯一邻居在右边或下边"这种从右/下方向发起连接的孤立情况**（例如整条墙只画了两格且右边那格先手动放置、左边那格后放置时，右边那格会被误判成 `i` 而不是 `rowtail` 镜像）。重写时如果贴图资源本身没有再补充镜像的"仅右/仅下"端头贴图，就**不要**改动这个函数的逻辑，因为其余六种贴图很可能是按照"以上/左为基准"的美术方向绘制的，随意加对称分支反而会导致贴图朝向反了。
- **十字（4 向真正交汇）没有独立贴图**：`wallAssetNames` 里没有 `cross`，`connectionAsset` 永远不会返回 `'cross'` 这个值——四向交汇会被 `up && left && (down || right)` 命中并复用 `'y'` 贴图。连接参考面板里的"十字"预览瓦片（`data-connection="cross"`）只是**给用户看的示意图案**（用于展示"如果四向都连接会长什么样"），它内部渲染时依然是调用同一个 `connectionAsset` 函数，对预设的十字形 5 格图案逐格计算贴图，中心格会被判定为 `'y'` 而不是不存在的 `'cross'` 贴图。

### 6.4 每帧的完整调用链

```
draw()
 └─ state.walls（当前生效的墙段点集，已按来源模式截断/排序）
     └─ forEach wall → wallAssetFor(wall.row, wall.column)
          └─ connectionAsset(state.walls, row, column)   // 每次都用"当前全部墙段"重新计算，而非增量更新
          └─ 取 wallSprites[对应等级][对应style] 贴图
          └─ placeWall(sprite, column, row)               // 见 5.3
```

**连接性判断是"全量重算"而不是"增量维护"**：每次 `draw()` 都会拿当前完整的墙段集合重新跑一遍 `connectionAsset`，不维护缓存/增量更新逻辑。在 1936 格量级下性能完全可接受，重写时不需要为了"性能优化"引入增量脏检查，那样反而容易在多来源模式切换、复制粘贴等场景下产生状态不同步的 bug。

---

## 七、资源文件约定

```
/index.html                         ← 单文件交付物，内联全部 CSS/JS
/images/base/base4.webp             ← 村庄等距底图，设计基准 800×600
/images/walls/{level}/{level}{style}.webp   ← level = 1..21，style ∈ wallAssetNames（7 种）
                                        共 21 × 7 = 147 张贴图
```

若目标环境无法提供全部 147 张真实美术贴图，`draw()` 中已内置**优雅降级**：某个 `sprite` 尚未加载完成（`!sprite.complete || !sprite.naturalWidth`）时，退化为纯色矩形占位（`fillStyle:'#f46f54'`，尺寸同 `WALL_W×WALL_H`，居中于同一 `gridToCanvas` 坐标），保证在贴图缺失时页面依然可用、格位依然对齐，只是视觉上是色块而非贴图。重写时必须保留这个降级分支。

---

## 八、交互事件与状态机

### 8.1 全局状态对象 `state`（字段清单，重写需完整保留）

```js
{
  image: null,            // 当前上传/加载的 Image 对象（图片模式）
  limit: 325,             // 当前墙段上限（capacity 滑杆值）
  walls: [],              // 本帧最终参与渲染的墙段（每帧 draw() 内重新计算/截断/排序）
  demoWalls: null,        // 文字/模板模式生成的"全量"墙段（未截断），draw() 时按 limit 截断为 walls
  dimWalls: new Set(),    // 被"点击临时调暗"的墙段坐标集合（"col,row" 字符串），仅影响渲染透明度
  wallLevel: 12,          // 全局城墙等级（文字/图片/模板模式使用）
  mode: 'text',           // 'text' | 'image' | 'test' | 'draw'
  drawWalls: [],          // 绘制模式专属的墙段数组，每个元素自带 .level
  drawMode: 'draw',       // 兼容字段，实际交互工具见 interactionMode
  interactionMode: 'zoom',// 'zoom' | 'brush'，控制画布指针事件走"视口平移缩放"还是"绘制工具"分支
  selectedWalls: [],      // 选择工具框选出的墙段
  clipboard: null,        // 复制/剪切暂存的墙段（相对锚点的偏移坐标）
  clipboardGhost: false,  // 是否处于"跟随鼠标半透明预览、点击即粘贴"状态
  isErasing: false,       // 画笔/橡皮二选一
  selectionStart/selectionEnd/isSelecting, // 框选状态
  hoveredCell: null,      // 当前鼠标悬停的 {row, column}，驱动高亮/粘贴预览
  drawImportSignature: null, // 防止同一份来源数据重复导入绘制模式
  testMap: 'border',      // 模板模式当前选中的图案
  testGrowRadius: 0        // 模板生成后实际膨胀到的层数（供文案展示）
}
```

### 8.2 关键事件绑定一览

| 事件 | 目标 | 行为 |
| --- | --- | --- |
| `click` | 模式 Tab | `selectMode(mode)`，见 8.3 |
| `click` | 模板二级 Tab `[data-map]` | `selectTestMap(map)` → 重建 `demoWalls` |
| `change` | `fileInput` | `selectMode('image')` + `loadImage(file)` |
| `input`（统一委托） | threshold/imageScale/imageRotation/shiftX/shiftY/projectionX/projectionY/centerX/centerY/invert/showWalls/redWeight/greenWeight/blueWeight | 更新各自旁边的数值只读输出文本；若当前是文字模式，重新 `createTextWalls`；统一调用 `draw()` |
| `change` | `wallLevel` | 更新 `state.wallLevel`，重绘主画布 + 重绘连接参考预览 |
| `click` | `[data-draw-tool]` 四个按钮 | `setDrawTool(tool)`（互斥高亮 + 更新 `state.interactionMode`/`isErasing`）+ 重绘 |
| `pointerdown/pointermove/pointerup` | `canvas`（仅绘制模式且 `interactionMode==='brush'`） | 按当前工具（画笔/橡皮/选择）分支处理；剪贴板跟随预览态优先于其他分支 |
| `click` | `drawCopyBtn`/`drawCutBtn`/`drawDeselectBtn` | 复制=暂存选区墙段（偏移量相对选区左上角）；剪切=复制后从 `drawWalls` 中移除原墙段；取消=清空选区与剪贴板跟随态 |
| `click` | `drawClearBtn` | 清空 `state.drawWalls` |
| `click` | `clearButton` | 清空图片/demoWalls/drawWalls，切回图片模式的初始空状态 |
| `click` | `exportButton` | 生成 `<a download>` 触发 `canvas.toDataURL('image/png')` 下载，文件名 `coc-wall-layout.png` |
| `click` | `emptyUpload` | 触发隐藏的 `fileInput.click()` |
| `click`（画布） | 主画布 | 非绘制模式下，点击命中的墙格切换其 `dimWalls` 调暗状态（校准辅助） |
| 视口工具条 `click` | `[data-zoom-in/out/reset]`（主画布与 ROI 画布各一套，逻辑复用同一个 `setupViewport` 工厂函数） | 缩放 ×1.25/÷1.25，reset 归零 pan 并恢复默认 zoom |
| `Esc`（键盘，全局） | — | 取消 `clipboardGhost` 跟随粘贴预览态 |

### 8.3 `selectMode(mode)` 完整职责（必须一次性做完，避免面板状态与数据不同步）

1. 记录 `fromMode` 与其"来源签名"`wallSourceSignature(fromMode)`（须在清空 `state.image` 之前取值）。
2. 更新 `state.mode`、清空 `state.image`、清空 `dimWalls`。
3. Tab 高亮态切换（`classList.toggle('active', ...)`）。
4. 按新模式重建数据源：
   - `text` → `createTextWalls()`；
   - `test` → `createTestWalls()` + `syncTestMapTabs()` + `describeTestWalls()`（更新提示文案，含当前图案已膨胀到第几层）；
   - `image` → 清空 `demoWalls`，等待用户上传；
   - `draw` → 若非从绘制模式切入则 `importWallsIntoDraw()` 拍平导入一次，重置绘制工具为画笔。
5. 若离开绘制模式：把 `interactionMode` 复位为 `'zoom'`，清空悬停/选区/剪贴板跟随状态，鼠标样式复位为 `grab`。
6. 更新 `modeHint` 文案。
7. 按模式切换以下面板的 `hidden`：`dropZone`（仅 image）、`textParams`（仅 text）、`testParams`（仅 test）、`imageParams`（仅 image）、`drawPanel`（仅 draw）、`roiPanel`/`transformPanel`/`sourceParams`（`text`或`image`即"有源"时显示）、`wallLevelRow`/`capacityControl`（非 draw 时显示）、`connectionPanel`（**始终显示**，与模式无关）、`paramsPanel`（**始终隐藏**，网格校准面板不对用户暴露）。
8. `emptyState` 仅在 `image` 模式显示。
9. 触发一次 `draw()`。
10. 调用 `renumberPanels()` 重新给当前可见面板编号。

---

## 九、非功能性要求

| 项目 | 要求 |
| --- | --- |
| 技术范围 | 单个 `index.html`，内联全部 CSS/JS，原生 Canvas 2D，零第三方 JS 依赖（可用 Google Fonts CDN 引入网页字体，如 `DM Mono` + `Space Grotesk`） |
| 主画布分辨率 | 逻辑/设计基准 800×600，实际 `<canvas>` 元素 1600×1200（2× 供清晰导出），二者比例关系必须在改动任何尺寸相关常量时同步保持 |
| 兼容性 | 现代 Chrome / Firefox / Edge / Safari 最新版；Local Font Access API 属渐进增强，不支持时需有内置候选字体名单兜底，不能报错崩溃 |
| 性能 | 参数滑杆拖动时应逐帧实时重绘（无需额外节流/防抖，1936 格规模的暴力算法性能足够）；图片识别响应应在参数变化后的下一帧内完成 |
| 视觉风格 | 品牌配色见 CSS 变量：`--gold #f4c430`、`--gold-soft #fecc54`、`--gold-deep #d99e2b`、`--grass #84c44c`、`--wood #5b4636`（COC 主题强调色）+ 基础中性色 `--ink #1a1f24`、`--paper #f2f4ef`、`--panel #ffffff`、`--acid #d8f44e`、`--coral #f46f54`；圆角 `--radius 12px`/`--radius-sm 8px`；卡片投影 `--shadow 0 10px 28px rgba(38,43,41,.07)` |
| 交付物 | 一个可直接双击在浏览器打开运行、无需安装依赖的 `index.html`，以及配套的 `images/base/` 与 `images/walls/{1..21}/` 贴图目录（若无法提供全部真实美术资源，必须保证第七节所述的纯色占位降级分支生效，使页面在贴图缺失时依然可交付演示） |

---

## 十、给实现者的验收清单（Definition of Done）

- [ ] 44×44 网格坐标系严格按第五节公式实现，`showWalls` 关闭时的调试网格视图能清楚验证对齐效果。
- [ ] 7 种连接贴图 `i/y/v/rowtail/coltail/row/col` 的选取逻辑与 6.2 节代码逐字一致，包括其"上/左优先"的非对称已知特性。
- [ ] 文字、图片、模板、绘制四种模式均可独立完整跑通，且互相切换时面板显隐、数据导入（尤其是切入绘制模式时的拍平导入）行为符合第三、八节描述。
- [ ] 图片识别路径对"透明像素"的特殊处理（透明 = 永不判墙，不受阈值/反相影响）必须实现，这是与"简单二值化"最大的区别点。
- [ ] ROI 采样区、变换面板、（隐藏的）网格校准面板均按第四节实现，即使网格校准面板默认不可见也要在代码里完整保留其逻辑与滑杆元素。
- [ ] 导出按钮直接导出主画布当前帧为 PNG，文件名固定 `coc-wall-layout.png`。
- [ ] 贴图缺失时的纯色占位降级分支必须存在并生效。
