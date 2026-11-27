---
name: glassify
description: >-
  Retrofit existing frontend projects (CSS / Tailwind / Flutter) into glassmorphism (frosted/liquid glass). 将已有前端项目改造为玻璃拟态/毛玻璃风格。Triggers: glassify, glassmorphism, frosted glass, liquid glass, glass UI, 玻璃化, 玻璃拟态, 上玻璃, 毛玻璃, 磨砂. Only for retrofitting existing projects, not greenfield scaffolding.
---

# 玻璃化改造 — glassify

将已有前端项目改造为高完成度玻璃拟态（glassmorphism）风格。核心工作方式：**技术栈与不透明外壳审计 → 出方案一次确认 → Token 化按栈改造 → 机器 DOM 审计 + 多态视口截图验收 → 清单级可控回滚**。交互语言跟随用户。

## 触发

```
/glassify <项目路径> [颜色说明]
```

- **项目路径**：必填，绝对路径或当前工作目录下的相对路径
- **颜色说明**：可选（如「深邃蓝紫风」「暖陶土色」）；未指定时方案附 4 套颜色选择，一次确认即可开工
- **边界**：只改造已有项目，不做从零新建

## 核心光学与工程铁律（全栈通用）

下列任一违反 = 配方失败，必须返工，不要交付：

1. **三层光路必通（先剥不透明外壳，再铺光斑）**
   - 玻璃的本质是对背景信号做低通模糊。若 `body`、`#root`、布局外壳仍保留实色背景，会直接挡死底层光斑。改造前必须审计并透明化阻挡光路的外壳容器，再补光斑层（背景已有图片/渐变/光晕则保留）。
2. **暗色高透 + OKLCH 色度控制**
   - L1 主面板 fill alpha 严格落在 **`.12–.35`**（`.36` 起超标收紧，`≥ .60` 判失败重做）；L2 顶栏/侧栏可放宽至 **`.30–.55`**。
   - Web 暗色基线：`blur ≥ 24px` + `backdrop-filter: saturate ≥ 1.6`（Tailwind 必须用 `backdrop-saturate-*`，严禁误用 `saturate-*`）；亮色基线：`blur ≥ 16px` + `saturate ≥ 1.5`。
   - **高色度、控明度**：暗色光斑要肉眼艳丽，同时满足 WCAG AA（最差对比度 ≥ 4.5:1），须选用**低明度高色度**色相（OKLCH 建议 $L \le 0.50$、高 $C$，如深电紫/深青碧/深靛蓝），避免高明度浅青/浅白冲垮白字对比度；大屏补中区微光避免屏幕中央发空。
   - **光泽折进 `background`**，禁止用 `::before` 叠在文字上洗对比度。
   - **蚀刻边缘三件套**（内嵌高光 + 内嵌暗边 + 双层外投影）必齐；Flutter 侧以 `BlurStyle.outer` 双层外投影 + 顶部高光渐变近似。
3. **禁止破坏存量布局的副作用属性**
   - 玻璃容器**禁止加 `overflow: hidden`**（会截断存量卡片里的下拉菜单、Popover、Tooltip 与键盘 focus ring；光泽已折进 `background`，无需裁剪）。
   - 玻璃容器**禁止加 `transform: translateZ(0)`**（会覆盖原有 `translate(-50%, -50%)` 弹窗居中与动效，并徒增移动端合成层显存占用）。
   - **层叠上下文提示**：`backdrop-filter` 会强制创建独立 Stacking Context；若玻璃卡片内有非 Portal 渲染的下拉菜单/Popover，展开时须提升所在卡片的 `z-index`（低于顶栏、高于兄弟卡片），或将浮层挂至 Portal，否则越界部分会被后续同级玻璃卡片遮挡。
4. **4 级 Z 轴材质阶梯（严禁玻璃套玻璃）**
   - CSS `backdrop-filter` 会创建 **Backdrop Root**，父子同套会导致子级无法采样父级背后的光斑并翻倍 GPU 开销。按层级区分材质：

| Z 轴层级 | 适用元素 | 材质策略 |
|---------|---------|---------|
| **L0 玻璃内嵌子块** | 卡片内分组槽、统计小格、嵌套面板 | **禁 `backdrop-filter`**；用低透明槽位 + 细边框 |
| **L1 主容器** | 主卡片、核心面板 | 高透玻璃（暗色 alpha `.12–.35`） |
| **L2 滚动悬浮栏** | 顶栏、底栏、侧边栏 | 中透玻璃（暗色 alpha `.30–.55`），兼顾透光与滚字可读 |
| **L3 悬浮弹窗层** | Modal、Drawer、Dialog、Toast | 必须配套 **Scrim 暗化遮罩** 压低底层噪声，本体保持高透 |
| **实色交互元素** | 按钮、输入框、搜索框、标签/Badge、表格/列表行 | **保持实色**，可读性与可点击辨识度优先 |

5. **性能预算与无障碍兜底**
   - **Web 与 Flutter 均禁止给长列表 / Feed 流 / 密集网格的每一项套 `backdrop-filter`**；Web 同视口活跃大面积模糊层尽量控制在 ≤ 8–10 个。
   - Web 配方必须同时包含 `-webkit-backdrop-filter`（兼容旧版 Safari / iOS WebView）、`@supports not ((backdrop-filter: blur(1px)) or (-webkit-backdrop-filter: blur(1px)))` 以及 `@media (prefers-reduced-transparency: reduce)` 实色可读兜底。

## 流程

### 阶段 1：技术栈与结构审计

先识别技术栈并路由到对应配方文档（**只读取命中的栈文档，不要全量加载无关栈**）：

| 探测依据 | 判定与配方 |
|---------|-----------|
| `tailwind.config.*` / `@tailwind` / `@import "tailwindcss"` / `tailwindcss` 依赖 | **Tailwind（标注 v3 / v4，含 shadcn/ui 检查）** → 读 [references/tailwind.md](./references/tailwind.md) |
| `pubspec.yaml` + `lib/*.dart` | **Flutter（标注版本是否 ≥ 3.27）** → 读 [references/flutter.md](./references/flutter.md) |
| `.css` / `.scss` / HTML 内联 `<style>` / Vue·Svelte·Astro SFC / CSS Modules / AntD·MUI·Element Plus | **CSS / Token 覆写** → 读 [references/css.md](./references/css.md) |
| 同时匹配多个 | 取最外层/实际承载 UI 的栈；存疑则列出猜测请用户确认 |

同时完成 3 项结构审计（写入阶段 2 方案卡）：
1. **主题模式**：单亮 / 单暗 / **明暗双主题**（双主题项目必须用 Token 同时支持明暗两套，禁止覆盖破坏另一套主题）。
2. **不透明外壳清单**：定位 `body`、`#root`、Layout Wrapper 或 `Scaffold` 上会遮挡底层光斑的不透明背景。
3. **范围与分层**：盘点 L0 / L1 / L2 / L3 与实色元素；中大型仓库先沉淀共享 Token/语义类，再聚焦核心布局外壳与关键页面，避免失控扩散。

### 阶段 2：产出方案，一次确认

先查工作区状态（写入方案卡的「工作区检测」行）：
- **Git 仓库**：跑 `git status`，区分「干净」与「已有未提交改动（改用清单级逐文件回滚）」。
- **非 Git 目录**：明确提示无版本控制，询问用户是否先为待改文件留 `.bak` 备份或直接确认改造。

颜色方案处理：
- **触发时已给颜色说明** → 直接按该色板出方案。
- **未给颜色** → 不要单独发问，在方案卡中附以下 4 个选项让用户一次回复确认：
  1. **暖纸 Warm Paper** — 暖白底 + 黄蓝红光斑，亮色白玻璃
  2. **暗夜霓虹 Dark Neon** — 深夜底 + 低明高色紫青靛光晕，暗色高透玻璃
  3. **跟随项目现有主色（推荐双主题项目）** — 提取项目主/辅色衍生光斑，保留原有明暗主题架构
  4. **自定义** — 描述想要的色板与明暗（如「薄荷绿+深海蓝，冷调暗色」）

按用户当前语言展示方案卡：

```
📋 玻璃化方案
──────────────
技术栈：Tailwind v4（shadcn/ui）· 明暗双主题
色板：跟随项目主色衍生光斑（或待选 1 / 2 / 3 / 自定义）
Token 层：src/styles/glass.css（新增共享 Token 与 .glass-* 语义类）
外壳剥离：body / #root / MainLayout 移除实色背景，底层挂 .app-backdrop
分层改造：
  - L1 主卡片：DashboardCard, MetricPanel → .glass-card（移除旧 bg-*）
  - L2 悬浮栏：TopNav, Sidebar → .glass-bar
  - L3 弹窗层：ConfirmDialog → .glass-scrim + .glass-overlay
  - L0 内嵌槽：StatSubTile → .glass-slot（无 backdrop-filter）
维持实色：按钮、输入框、标签、表格行
工作区检测：干净（或：N 个未提交文件，改用清单级回滚 / 非 Git 目录）
──────────────
回复 y 开始改造（未选色板时附「1 / 2 / 3 / 自定义色板」一并回复）
```

**用户确认前，不写任何文件。**

### 阶段 3：按栈执行改造

动手前复核工作区闸口（Git 仓确认状态未变；未确认前零写入）。读取阶段 1 命中的 `references/<stack>.md`，严格遵守：
1. **Token 先行**：优先沉淀 1 份 `--glass-*` 变量 + L0/L1/L2/L3 语义类（或 Flutter `GlassSurface` 封装），禁止把几百字符的内联任意值类散弹式复制到几十个组件。
2. **光路剥离 + 清旧底色**：先透明化不透明外壳并挂载光斑层；给存量元素挂玻璃类时，**必须移除或覆盖原有不透明 `bg-*` / `background-color`**（避免 `background-color` 垫在渐变下方捂死玻璃）。
3. **记录变更清单**：维护本次**修改文件列表**与**新建文件列表**，供阶段 5 精准回滚。

### 阶段 4：机器审计 + 多态视口截图验收

读取 [references/verify.md](./references/verify.md) 执行验证：
- **Web 项目**：使用任意可用浏览器工具（Playwright / Chrome DevTools MCP / 内置浏览器）访问本地服务（静态页先启 `python3 -m http.server`，勿用 `file://`）：
  1. 执行 `references/verify.md` 中的 **DOM 确定性审计脚本**，机器校验 `backdrop-filter` 生效、无不透明外壳遮挡、无玻璃嵌套、无实色交互元素误伤、暗色 alpha 未超标、无 `overflow: hidden` 误加、抽样文字对比度达标；
  2. 拍摄**三态视口截图**（**严禁 `fullPage: true`**，避免 `backdrop-filter` 拼接钉死第一屏造成截断伪影）：① 默认首屏（双主题则明暗各一）；② 滚动后视口（验顶栏虚化正文）；③ 打开下拉框/弹窗（验无裁剪且 Scrim 下文字清晰）。
- **Flutter 项目**：执行 `flutter analyze` 检查编译与静态问题，有设备/模拟器时跑 `flutter run` 截图验收。
- **降级路径**（无浏览器或无 Flutter 设备）：按 `references/verify.md` 执行代码级静态自检，将视觉项整理为自查清单交用户，交付消息必须显式标注「未经视口验证」或「未做截图验收」。

### 阶段 5：可控回滚（如有需要）

基于阶段 3 记录的**修改文件列表**与**新建文件列表**执行精准恢复：
- **Git 仓库（无论原先干净与否）**：
  - 恢复修改文件：`git checkout -- <本次修改的文件列表>`
  - 清理新建文件：`rm <本次新建的文件列表>`（注意：单独 `git checkout -- .` 删不掉新建的 untracked 文件！）
- **非 Git 目录**：用 `.bak` 备份恢复，或按变更清单逐项撤销。

禁止在未核对工作区状态时执行无差别整体重置。

## 不支持的场景

- 纯文本 / 终端 CLI 项目
- 无 UI / 无样式的纯后端项目
- 非 Web / 非 Flutter 的原生桌面应用（Electron / Tauri 算 Web 项目）
- 不支持 `backdrop-filter` 的老旧浏览器（如 IE11）自动走 `@supports not` 实色卡片降级
