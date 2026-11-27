# CSS / HTML / Vue / Svelte / Token 组件库玻璃化配方

适用于：手写 CSS/SCSS、静态 HTML（含内联 `<style>`）、Vue/Svelte/Astro SFC、CSS Modules，以及支持全局 CSS Token 覆写的组件库（Ant Design / MUI / Element Plus / Chakra 等）。

---

## 1. 先打通光路（不透明外壳审计）

在插入光斑前，先检查 `html`、`body`、`#root`、`#app` 及最外层 Layout 容器：

1. 将**底色兜底**放到 `html`（`background-color: var(--glass-paper, #faf7f0)`）。
2. 将 `body`、`#root`、`#app`、外层页面容器的 `background` 改为 **`transparent`**（且避免给 `body` 设不必要的 `z-index` / `transform` 形成新层叠上下文）。
3. 将 `.app-backdrop` 挂在 `body` 顶层（`position: fixed; inset: 0; z-index: -1; pointer-events: none`）。这样 `.app-backdrop` 必然绘制在 `html` 底色之上、所有页面内容之下，不会被外壳挡死。

---

## 2. 共享 Token 层（同时兼容单主题与明暗双主题）

单主题项目只保留对应的一套变量；**双主题项目同时保留 `:root` 与 `[data-theme="dark"], .dark`**（按项目实际切换方式调整选择器），切勿破坏另一套主题。

> 所有 `var()` 均带显式 fallback，防止未定义变量引发整条 `background` IACVT 失效。
> 暗色光斑遵守 **OKLCH 低明度（$L \le 0.50$）、高色度** 原则，并在屏幕中区补一团低饱和深靛微光，防止 1440px+ 宽屏中央发空；`.app-backdrop::after` 叠 1.5% 微噪点消除 8-bit 暗色渐变色带。

```css
/* ============================================================
   glassify Token 层 —— 亮色白玻璃 / 暗色高透烟熏玻璃
   ============================================================ */
:root {
  --glass-paper: #faf7f0;
  --glass-text: #1c1914;
  --glass-text-2: rgba(28, 25, 20, .74);
  --glass-text-3: rgba(28, 25, 20, .70);

  /* 光斑主辅色（按色板替换，保持亮色明快 / 暗色低明高色） */
  --glass-blob-1: #ffe81a;
  --glass-blob-2: #0758f7;
  --glass-blob-3: #ff3044;

  /* L1 主面板 */
  --glass-bg:
    linear-gradient(180deg, rgba(255, 255, 255, .50), transparent 52%),
    linear-gradient(165deg, rgba(255, 255, 255, .62) 0%, rgba(255, 255, 255, .34) 100%);
  --glass-border: rgba(255, 255, 255, .75);
  --glass-radius: 18px;
  --glass-filter: blur(16px) saturate(1.5);
  --glass-shadow:
    inset 0 1px 0 rgba(255, 255, 255, .95),
    inset 0 0 0 1px rgba(255, 255, 255, .28),
    inset 0 -1px 0 rgba(17, 17, 17, .05),
    0 2px 5px rgba(17, 17, 17, .07),
    0 20px 44px -14px rgba(17, 17, 17, .24);

  /* L2 悬浮顶栏/侧栏 */
  --glass-bar-bg: linear-gradient(160deg, rgba(255, 255, 255, .68), rgba(255, 255, 255, .42));
  --glass-bar-border: rgba(255, 255, 255, .75);
  --glass-bar-filter: blur(16px) saturate(1.5);
  --glass-bar-shadow:
    inset 0 1px 0 rgba(255, 255, 255, .95),
    0 6px 22px rgba(17, 17, 17, .08);

  /* L0 玻璃内嵌子块（无 backdrop-filter） */
  --glass-slot-bg: rgba(255, 255, 255, .42);
  --glass-slot-border: rgba(28, 25, 20, .09);

  /* L3 弹窗遮罩 */
  --glass-scrim: rgba(20, 18, 14, .34);

  /* 无模糊/降透明兜底实色 */
  --glass-fallback-bg: rgba(255, 255, 255, .94);
  --glass-fallback-border: rgba(28, 25, 20, .12);
}

[data-theme="dark"],
.dark {
  --glass-paper: #0f1018;
  --glass-text: rgba(255, 255, 255, .94);
  --glass-text-2: rgba(255, 255, 255, .82);
  --glass-text-3: rgba(255, 255, 255, .68);

  /* OKLCH 低明高色光斑：艳丽透光且不击穿白字 AA 对比度 */
  --glass-blob-1: #6d4aff;
  --glass-blob-2: #06a8d0;
  --glass-blob-3: #4338ca;

  /* L1 暗色主面板：fill alpha 峰值 .32（严控 .12–.35） */
  --glass-bg:
    linear-gradient(180deg, rgba(255, 255, 255, .10), rgba(255, 255, 255, .03) 52%, transparent),
    linear-gradient(160deg, rgba(255, 255, 255, .14), rgba(20, 20, 28, .20) 46%, rgba(12, 12, 18, .32));
  --glass-border: rgba(255, 255, 255, .16);
  --glass-radius: 20px;
  --glass-filter: blur(24px) saturate(1.8);
  --glass-shadow:
    inset 0 1px 0 rgba(255, 255, 255, .30),
    inset 0 -1px 0 rgba(0, 0, 0, .30),
    inset 0 0 0 1px rgba(255, 255, 255, .04),
    inset 0 -24px 44px -26px rgba(0, 0, 0, .55),
    0 10px 26px rgba(0, 0, 0, .45),
    0 30px 64px -24px rgba(0, 0, 0, .60);

  /* L2 暗色顶栏/侧栏：alpha .30–.55 */
  --glass-bar-bg: linear-gradient(160deg, rgba(40, 40, 52, .52), rgba(16, 16, 22, .30));
  --glass-bar-border: rgba(255, 255, 255, .12);
  --glass-bar-filter: blur(24px) saturate(1.7);
  --glass-bar-shadow:
    inset 0 1px 0 rgba(255, 255, 255, .16),
    0 6px 22px rgba(0, 0, 0, .40);

  /* L0 暗色玻璃内嵌子块（无 backdrop-filter） */
  --glass-slot-bg: rgba(255, 255, 255, .06);
  --glass-slot-border: rgba(255, 255, 255, .11);

  /* L3 暗色弹窗遮罩 */
  --glass-scrim: rgba(6, 7, 12, .58);

  /* 无模糊/降透明兜底实色 */
  --glass-fallback-bg: rgba(20, 21, 30, .94);
  --glass-fallback-border: rgba(255, 255, 255, .14);
}
```

渐变里的中性暗色可替换为项目主色的极暗变体（如深咖、深夜蓝），**只换色相，不把 alpha 加回去**。

---

## 3. 背景光斑层（亮 / 暗 + 去色带微噪点）

```css
html {
  background-color: var(--glass-paper, #faf7f0);
}

.app-backdrop {
  position: fixed;
  inset: 0;
  z-index: -1;
  pointer-events: none;
  background:
    radial-gradient(720px 480px at 10% -6%,
      color-mix(in srgb, var(--glass-blob-1, #ffe81a) 78%, transparent), transparent 68%),
    radial-gradient(820px 560px at 96% 12%,
      color-mix(in srgb, var(--glass-blob-2, #0758f7) 28%, transparent), transparent 66%),
    radial-gradient(860px 600px at 34% 88%,
      color-mix(in srgb, var(--glass-blob-1, #ffe81a) 50%, transparent), transparent 70%),
    radial-gradient(720px 540px at 84% 84%,
      color-mix(in srgb, var(--glass-blob-3, #ff3044) 28%, transparent), transparent 70%),
    var(--glass-paper, #faf7f0);
}

[data-theme="dark"] .app-backdrop,
.dark .app-backdrop {
  background:
    radial-gradient(860px 580px at 10% -6%,
      color-mix(in srgb, var(--glass-blob-1, #6d4aff) 54%, transparent), transparent 70%),
    radial-gradient(920px 640px at 102% 10%,
      color-mix(in srgb, var(--glass-blob-2, #06a8d0) 34%, transparent), transparent 68%),
    radial-gradient(780px 540px at 50% 50%,
      color-mix(in srgb, var(--glass-blob-3, #4338ca) 32%, transparent), transparent 72%),
    radial-gradient(840px 600px at 80% 104%,
      color-mix(in srgb, var(--glass-blob-1, #6d4aff) 28%, transparent), transparent 70%),
    var(--glass-paper, #0f1018);
}

/* 可选：1.5% 微噪点消除暗色 8-bit 径向渐变色带（仅铺在背景层，绝不叠在卡片文字上） */
.app-backdrop::after {
  content: "";
  position: absolute;
  inset: 0;
  opacity: .018;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='120' height='120'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='.8' numOctaves='2' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
}
```

---

## 4. 4 级 Z 轴语义类（无 `overflow: hidden`，无 `transform: translateZ(0)`）

```css
/* L1 主容器 / 卡片 —— 不设 overflow:hidden 防止裁剪下拉菜单，不设 translateZ(0) 防止覆盖动效/居中 */
.glass-card {
  position: relative;
  border: 1px solid var(--glass-border, rgba(255, 255, 255, .75));
  border-radius: var(--glass-radius, 18px);
  background: var(--glass-bg, linear-gradient(165deg, rgba(255, 255, 255, .62), rgba(255, 255, 255, .34)));
  background-color: transparent;
  backdrop-filter: var(--glass-filter, blur(16px) saturate(1.5));
  -webkit-backdrop-filter: var(--glass-filter, blur(16px) saturate(1.5));
  box-shadow: var(--glass-shadow, inset 0 1px 0 rgba(255, 255, 255, .95), 0 20px 44px -14px rgba(17, 17, 17, .24));
}

/* 关键：backdrop-filter 会创建独立 Stacking Context；若卡片内有非 Portal 下拉/Popover，
   展开时须提升所在卡片 z-index（低于顶栏 z-index:20、高于同级卡片），否则越界部分会被下方玻璃卡盖住 */
.glass-card:has(details[open], [aria-expanded="true"]) {
  z-index: 15;
}

/* L2 顶栏 / 底栏 / 侧边栏 */
.glass-bar {
  background: var(--glass-bar-bg, linear-gradient(160deg, rgba(255, 255, 255, .68), rgba(255, 255, 255, .42)));
  background-color: transparent;
  border: 1px solid var(--glass-bar-border, rgba(255, 255, 255, .75));
  backdrop-filter: var(--glass-bar-filter, blur(16px) saturate(1.5));
  -webkit-backdrop-filter: var(--glass-bar-filter, blur(16px) saturate(1.5));
  box-shadow: var(--glass-bar-shadow, inset 0 1px 0 rgba(255, 255, 255, .95), 0 6px 22px rgba(17, 17, 17, .08));
}

/* L0 玻璃内嵌子块 —— 禁止再套 backdrop-filter（防 Backdrop Root 失效与糊屏） */
.glass-slot {
  border: 1px solid var(--glass-slot-border, rgba(28, 25, 20, .09));
  border-radius: 12px;
  background: var(--glass-slot-bg, rgba(255, 255, 255, .42));
}

/* L3 浮层弹窗 + Scrim 遮罩（关键：.glass-scrim 与 .glass-overlay 必须为兄弟节点，
   否则带 backdrop-filter 的遮罩会创建 Backdrop Root，阻断弹窗本体采样背景！） */
.glass-modal-root {
  position: fixed;
  inset: 0;
  z-index: 50;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}
.glass-modal-root[hidden] { display: none; }
.glass-scrim {
  position: absolute;
  inset: 0;
  background: var(--glass-scrim, rgba(20, 18, 14, .34));
  backdrop-filter: blur(4px);
  -webkit-backdrop-filter: blur(4px);
}
.glass-overlay {
  position: relative;
  z-index: 1;
  border: 1px solid var(--glass-border, rgba(255, 255, 255, .75));
  border-radius: var(--glass-radius, 18px);
  background: var(--glass-bg, linear-gradient(165deg, rgba(255, 255, 255, .62), rgba(255, 255, 255, .34)));
  background-color: transparent;
  backdrop-filter: var(--glass-filter, blur(16px) saturate(1.5));
  -webkit-backdrop-filter: var(--glass-filter, blur(16px) saturate(1.5));
  box-shadow: var(--glass-shadow, inset 0 1px 0 rgba(255, 255, 255, .95), 0 20px 44px -14px rgba(17, 17, 17, .24));
}
```

---

## 5. 必须附带的兼容与无障碍兜底

```css
/* 不支持 backdrop-filter 的浏览器降级为可读实色卡 */
@supports not ((backdrop-filter: blur(1px)) or (-webkit-backdrop-filter: blur(1px))) {
  .glass-card,
  .glass-bar,
  .glass-overlay {
    background: var(--glass-fallback-bg, rgba(255, 255, 255, .94));
    border-color: var(--glass-fallback-border, rgba(28, 25, 20, .12));
  }
}

/* 响应系统「降低透明度」无障碍设置 */
@media (prefers-reduced-transparency: reduce) {
  .glass-card,
  .glass-bar,
  .glass-overlay {
    background: var(--glass-fallback-bg, rgba(255, 255, 255, .94));
    border-color: var(--glass-fallback-border, rgba(28, 25, 20, .12));
    backdrop-filter: none;
    -webkit-backdrop-filter: none;
  }
}
```

---

## 6. Token 驱动组件库覆写提示（AntD / MUI / Element Plus）

遇到组件库项目时，优先在主题/容器层覆写，不要逐个硬改业务组件：

- **Ant Design**：用 `ConfigProvider` 将 `colorBgLayout` 设为 `transparent` 暴露光斑；再对 `.ant-card`、`.ant-modal-content`、`.ant-drawer-content` 注入 `.glass-card` / `.glass-overlay` 语义规则，同时保留 `.ant-btn`、`.ant-input`、`.ant-tag`、`.ant-table-row` 实色。
- **MUI**：在 `createTheme().components.MuiPaper.styleOverrides.root` 覆写表面材质，通过 `slotProps.backdrop` 配置 L3 Scrim，并保留 `MuiButton` / `MuiOutlinedInput` / `MuiChip` 实色。
- **Element Plus**：将页面底色变量透明化，对 `.el-card`、`.el-dialog`、`.el-drawer` 挂玻璃样式，注意避免给 `.el-table__row` 和 `.el-select` 下拉触发框上玻璃。
