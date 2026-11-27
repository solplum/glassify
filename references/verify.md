# 玻璃化验收手册（机器 DOM 审计 + 三态视口截图）

仅靠肉眼看一张首屏截图，LLM 无法准确判别对比度、alpha 上限、嵌套 Backdrop Root 与下拉裁剪。Web 验收必须执行**「确定性 DOM 审计 + 三态视口截图」**双轨验证。

---

## 1. Web 服务启动约定

- 优先连接项目自带 dev server（`localhost:<port>`）。
- 无 dev server 的静态页面：在项目目录启 `python3 -m http.server <port>` 再访问（Playwright 等工具普遍拒绝 `file://` 协议）。
- **工具无关**：Playwright MCP、Chrome DevTools MCP、宿主内置浏览器或本地 Playwright CLI 均可，不要硬绑定单一工具。

---

## 2. 确定性 DOM 审计脚本（在页面上下文执行）

在目标页加载完毕后（双主题项目需在亮/暗态各跑一次），通过 `page.evaluate(...)` / `evaluate_script` 运行以下只读脚本，直接输出结构化核验报告：

```javascript
(() => {
  const vw = window.innerWidth;
  const vh = window.innerHeight;

  // 严格区分 3 参数不透明 rgb(r, g, b)（alpha=1）与 4 参数 rgba(r, g, b, a) / CSS4 slash 语法
  const parseAlpha = (colorStr) => {
    if (!colorStr) return 0;
    const s = colorStr.trim().toLowerCase();
    if (s === 'transparent' || s === 'none') return 0;
    // CSS4 slash 语法：rgb(r g b / a) / color(srgb r g b / a) / oklch(l c h / a)
    const slash = s.match(/\/\s*([\d.]+)(%?)\s*\)/);
    if (slash) {
      const v = parseFloat(slash[1]);
      return slash[2] ? v / 100 : v;
    }
    // 逗号分隔 4 参数：rgba(r, g, b, a) 或 rgb(r, g, b, a)
    const comma4 = s.match(/^rgba?\(\s*[\d.]+%?\s*,\s*[\d.]+%?\s*,\s*[\d.]+%?\s*,\s*([\d.]+)(%?)\s*\)$/);
    if (comma4) {
      const v = parseFloat(comma4[1]);
      return comma4[2] ? v / 100 : v;
    }
    // 3 参数不透明 rgb(r, g, b) / rgb(r g b) / 其他无 slash 色值
    return 1;
  };

  const extractGradientPeakAlpha = (bgImg) => {
    if (!bgImg || bgImg === 'none') return 0;
    const alphas = [];
    for (const m of bgImg.matchAll(/rgba?\([^)]+\)|(?:oklch|oklab|hsl|hsla|color)\([^)]+\)/gi)) {
      alphas.push(parseAlpha(m[0]));
    }
    return alphas.length ? Math.max(...alphas) : 0;
  };

  const relLum = (r, g, b) => {
    const f = (c) => {
      const s = c / 255;
      return s <= 0.03928 ? s / 12.92 : Math.pow((s + 0.055) / 1.055, 2.4);
    };
    return 0.2126 * f(r) + 0.7152 * f(g) + 0.0722 * f(b);
  };

  const parseRGB = (str) => {
    const m = str && str.match(/rgba?\(\s*([\d.]+)[,\s]+([\d.]+)[,\s]+([\d.]+)(?:\s*[,/]\s*([\d.]+)(%?))?\s*\)/i);
    if (!m) return null;
    const a = m[4] !== undefined ? (m[5] ? +m[4] / 100 : +m[4]) : 1;
    return { r: +m[1], g: +m[2], b: +m[3], a };
  };

  const all = [...document.querySelectorAll('*')];
  const glassEls = [];
  const nestedBackdropRoots = [];
  const overflowHiddenGlass = [];
  const solidViolations = [];
  const opaqueCardBgViolations = [];
  const darkAlphaWarnings = [];
  const opaqueShellBlockers = [];

  const isDark =
    document.documentElement.dataset.theme === 'dark' ||
    document.body.dataset.theme === 'dark' ||
    document.documentElement.classList.contains('dark') ||
    getComputedStyle(document.documentElement).colorScheme === 'dark';

  // 1. 检查不透明外壳是否挡住 fixed z-index:-1 光斑
  const hasFixedBackdrop = all.some((el) => {
    const cs = getComputedStyle(el);
    return cs.position === 'fixed' && parseInt(cs.zIndex, 10) < 0 && cs.backgroundImage !== 'none';
  });
  if (hasFixedBackdrop) {
    for (const sel of ['body', '#root', '#app', 'main']) {
      const el = document.querySelector(sel);
      if (!el) continue;
      const a = parseAlpha(getComputedStyle(el).backgroundColor);
      if (a > 0.9) opaqueShellBlockers.push({ selector: sel, bgColor: getComputedStyle(el).backgroundColor });
    }
  }

  // 2. 扫描所有带 backdrop-filter 的元素
  for (const el of all) {
    const cs = getComputedStyle(el);
    const bf = cs.backdropFilter && cs.backdropFilter !== 'none'
      ? cs.backdropFilter
      : cs.webkitBackdropFilter;
    if (!bf || bf === 'none') continue;

    const rect = el.getBoundingClientRect();
    if (rect.width === 0 || rect.height === 0) continue; // 跳过隐藏元素（如未打开的弹窗）
    const inViewport = rect.bottom > 0 && rect.top < vh && rect.right > 0 && rect.left < vw;
    const tag = el.tagName.toLowerCase();
    const label = `${tag}${el.id ? '#' + el.id : ''}${typeof el.className === 'string' && el.className.trim() ? '.' + el.className.trim().split(/\s+/).slice(0, 2).join('.') : ''}`;

    glassEls.push({ label, bf, inViewport, w: Math.round(rect.width), h: Math.round(rect.height) });

    // 误玻璃化实色交互元素检查
    if (['button', 'input', 'select', 'textarea'].includes(tag)) {
      solidViolations.push(label);
    }

    // 破坏性 overflow:hidden 检查（排除 scrim）
    if (cs.overflow === 'hidden' && !label.includes('scrim')) {
      overflowHiddenGlass.push(label);
    }

    // 旧实色 background-color 残留检查
    const bgColorAlpha = parseAlpha(cs.backgroundColor);
    if (bgColorAlpha > 0.6 && !label.includes('scrim')) {
      opaqueCardBgViolations.push({ label, backgroundColor: cs.backgroundColor });
    }

    // 暗色 alpha 上限检查（顶栏放宽至 0.55，普通面板 <= 0.35）
    if (isDark && !label.includes('scrim')) {
      const peakAlpha = Math.max(bgColorAlpha, extractGradientPeakAlpha(cs.backgroundImage));
      const isBar = ['header', 'nav', 'aside', 'footer'].includes(tag) || /bar|nav|header|sidebar/i.test(label);
      const limit = isBar ? 0.55 : 0.35;
      if (peakAlpha > limit) {
        darkAlphaWarnings.push({ label, peakAlpha, limit });
      }
    }

    // 嵌套 Backdrop Root 检查（忽略内部含独立不透明舞台的演示容器）
    let p = el.parentElement;
    while (p && p !== document.documentElement) {
      const pcs = getComputedStyle(p);
      const pbf = pcs.backdropFilter && pcs.backdropFilter !== 'none' ? pcs.backdropFilter : pcs.webkitBackdropFilter;
      if (pbf && pbf !== 'none') {
        nestedBackdropRoots.push({ child: label, ancestor: `${p.tagName.toLowerCase()}.${String(p.className || '').split(/\s+/)[0]}` });
        break;
      }
      if (parseAlpha(pcs.backgroundColor) > 0.95) break; // 中间有独立不透明画布则中断
      p = p.parentElement;
    }
  }

  // 3. 正文对比度抽样（沿祖先定位最近的不透明参考底色，避免误报暗色对照舞台/实色按钮）
  const rootBg = parseRGB(getComputedStyle(document.documentElement).backgroundColor) || (isDark ? { r: 15, g: 16, b: 24, a: 1 } : { r: 250, g: 247, b: 240, a: 1 });
  const nearestOpaqueBg = (node) => {
    let cur = node;
    while (cur && cur !== document.documentElement) {
      const bg = parseRGB(getComputedStyle(cur).backgroundColor);
      if (bg && bg.a >= 0.9) return bg;
      cur = cur.parentElement;
    }
    return rootBg;
  };
  const lowContrastTexts = [];
  for (const el of document.querySelectorAll('h1,h2,h3,p,td,th,span,a,li,label')) {
    const text = (el.textContent || '').trim();
    if (!text || el.children.length > 0) continue;
    const rect = el.getBoundingClientRect();
    if (rect.width === 0 || rect.height === 0) continue;
    const cs = getComputedStyle(el);
    if (cs.visibility === 'hidden' || parseFloat(cs.opacity) === 0) continue;
    const fg = parseRGB(cs.color);
    if (!fg || fg.a === 0) continue;
    const refBg = nearestOpaqueBg(el);
    const bgLum = relLum(refBg.r, refBg.g, refBg.b);
    const effR = fg.r * fg.a + refBg.r * (1 - fg.a);
    const effG = fg.g * fg.a + refBg.g * (1 - fg.a);
    const effB = fg.b * fg.a + refBg.b * (1 - fg.a);
    const fgLum = relLum(effR, effG, effB);
    const ratio = (Math.max(fgLum, bgLum) + 0.05) / (Math.min(fgLum, bgLum) + 0.05);
    const fontSize = parseFloat(cs.fontSize);
    const minRatio = fontSize >= 18.66 || (fontSize >= 14 && parseInt(cs.fontWeight, 10) >= 700) ? 3.0 : 4.5;
    if (ratio < minRatio) {
      lowContrastTexts.push({ text: text.slice(0, 24), ratio: +ratio.toFixed(2), minRatio });
    }
  }

  const activeInViewport = glassEls.filter((g) => g.inViewport && g.w * g.h > 10000).length;

  return {
    theme: isDark ? 'dark' : 'light',
    totalGlassEls: glassEls.length,
    activeLargeGlassInViewport: activeInViewport,
    gpuBudgetWarning: activeInViewport > 10,
    opaqueShellBlockers,
    nestedBackdropRoots,
    overflowHiddenGlass,
    solidViolations,
    opaqueCardBgViolations,
    darkAlphaWarnings,
    lowContrastSampleCount: lowContrastTexts.length,
    lowContrastSamples: lowContrastTexts.slice(0, 8),
  };
})();
```

**通过门槛**：`opaqueShellBlockers`、`nestedBackdropRoots`、`solidViolations`、`opaqueCardBgViolations`、`darkAlphaWarnings` 均为空；`overflowHiddenGlass` 为空（除非明确不需要溢出浮层）；`gpuBudgetWarning` 为 `false`；`lowContrastSampleCount` 为 `0`（或人工确认仅位于独立彩色实色徽章/对比示例内）。

---

## 3. 三态视口截图协议

**铁律：只拍视口截图，严禁 `fullPage: true`。**
原因：Chromium / Playwright 拼接长图时会把 `backdrop-filter` 采样钉死在第一屏，制造出「玻璃面板拦腰截断」的虚假伪影。

按项目具备的交互状态拍摄视口截图：
1. **默认首屏视口**（双主题项目必须**亮色 + 暗色各拍一张**）
   - 核验：光斑肉眼可见且大屏中间不空、暗色面板能透出光斑色相而非实色块、蚀刻边缘与双层阴影有厚度感。
2. **滚动后视口**（若页面有 sticky/fixed 顶栏或底栏）
   - 核验：滚动正文经过顶栏下方时，顶栏真实虚化下方内容且导航文字依然清晰。
3. **打开浮层/下拉态**（若页面有 Modal、Drawer、Dropdown 或 Popover）
   - 核验：卡片内下拉菜单/Tooltip 未被 `overflow: hidden` 截断；弹窗背后的 Scrim 压暗有效，底层文字不会刺穿弹窗正文的对比度。

---

## 4. Flutter 验收

1. 运行 `flutter analyze`，确保零 error / 零新增 warning。
2. 人工检查 `BoxShadow` 全部带 `blurStyle: BlurStyle.outer`，`ListView.builder` / `GridView.builder` 列表项未套 `BackdropFilter`。
3. 有模拟器或真机时执行 `flutter run` 并截图核对明暗透光与边缘效果。

---

## 5. 无浏览器 / 无设备时的降级静态自检

当运行环境无法启动浏览器截图或 Flutter 无可用设备时，**不得声称已通过视觉验收**，须执行静态代码检查并在交付消息中显式标注 **「未经视口验证」**（Web）或 **「未做截图验收」**（Flutter）：

- 静态核对清单：
  - [ ] 暗色面板 fill alpha `.12–.35`（顶栏 ≤ `.55`），无残留旧不透明 `bg-*` / `background-color`
  - [ ] Web 暗色 `blur ≥ 24px` + `backdrop-saturate ≥ 1.6`（绝非 `saturate-*`）；亮色 `blur ≥ 16px` + `1.5`
  - [ ] 手写 CSS 含 `-webkit-backdrop-filter`、`@supports not` 与 `prefers-reduced-transparency` 兜底
  - [ ] 玻璃容器无破坏性 `overflow: hidden` 与 `transform: translateZ(0)`，无嵌套 `backdrop-filter`
  - [ ] 外壳 `body` / `#root` 已透明化，所有 `var()` 带显式 fallback
  - [ ] Flutter 所有玻璃外投影带 `BlurStyle.outer`，列表项未套 `BackdropFilter`
- 将无法静态验证的视觉项（光斑肉眼可见、暗色透色相、滚动顶栏虚化、弹窗无裁剪）列成简短自查清单交付用户。
