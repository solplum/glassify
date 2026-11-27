# Tailwind CSS（v3 / v4）与 shadcn/ui 玻璃化配方

---

## 1. 版本与关键坑位速查（动手前必核）

| 项目特征 | 版本 | 关键语法差异 |
|---------|------|-------------|
| `@import "tailwindcss";` / `@theme` | **Tailwind v4** | 用 `@utility glass-card` 定义语义类；支持任意整数比例（如 `backdrop-saturate-180`、`border-white/16`） |
| `tailwind.config.{js,ts,mjs}` / `@tailwind base;` | **Tailwind v3** | 用 `@layer components { .glass-card { ... } }`；透明度默认步进 5（须写 `border-white/[.16]`）；`backdrop-saturate` 仅有 `0/50/100/150/200`（须写 `backdrop-saturate-[1.8]`） |

### 三个必避硬坑

1. **必须用 `backdrop-saturate-*`，严禁误写成 `saturate-*`**
   - `saturate-150` / `saturate-[1.8]` 编译为 `filter: saturate(...)`，会把卡片内的文字、头像、按钮全拉爆饱和度，且不会增强玻璃背后光斑。正确写法是 **`backdrop-saturate-150`** / **`backdrop-saturate-[1.8]`**。
2. **改造存量元素时，必须清除原有 `bg-*`（或显式加 `bg-transparent`）**
   - 存量组件常带 `bg-white` / `bg-zinc-900` / `bg-card`（输出 `background-color`）。如果只追加 `bg-[linear-gradient(...)]`，它仅输出 `background-image`，原有不透明 `background-color` 仍垫在底下，玻璃直接变实色块。
3. **禁止在 `bg-[radial-gradient(...)]` 末尾拼色值**
   - `bg-[radial-gradient(...),var(--paper)]` 会被 Tailwind 编译为 `background-image: radial-gradient(...), var(--paper);`，CSS 规范禁止 `background-image` 尾随 `<color>`，整条规则会被浏览器丢弃！
   - 正确写法：把底色和渐变拆成两个类 `bg-[var(--glass-paper,#faf7f0)] bg-[radial-gradient(...)]`，或用 CSS 语义类 `.app-backdrop`。

---

## 2. 推荐架构：Token + 语义类（禁止 450 字符内联到处粘贴）

无论项目有 5 个还是 50 个卡片，**先在全局 CSS（如 `globals.css` / `index.css`）定义 `--glass-*` 变量与 L0–L3 语义类**，再在 JSX 中移除旧 `bg-*` 并挂单类名（如 `<div className="glass-card p-6">`）。这样天然兼容双主题（`.dark`），且全局调参只改一处。

Token 变量（`:root` / `.dark`）与 `.app-backdrop`、`.glass-scrim`、`@supports not`、`@media (prefers-reduced-transparency: reduce)` 直接复用 [css.md](./css.md) 第 2–5 节；在 Tailwind 中注册方式如下：

### Tailwind v4（`globals.css`）

```css
@import "tailwindcss";

@theme inline {
  --blur-glass: 16px;     /* 亮色面板 */
  --blur-glass-lg: 24px;  /* 暗色面板 / 顶栏 */
}

/* 导入或粘贴 css.md 中的 :root / .dark --glass-* 变量与 .app-backdrop */

@utility glass-card {
  position: relative;
  border: 1px solid var(--glass-border, rgba(255, 255, 255, .75));
  border-radius: var(--glass-radius, 18px);
  background: var(--glass-bg, linear-gradient(165deg, rgba(255, 255, 255, .62), rgba(255, 255, 255, .34)));
  background-color: transparent;
  backdrop-filter: var(--glass-filter, blur(16px) saturate(1.5));
  -webkit-backdrop-filter: var(--glass-filter, blur(16px) saturate(1.5));
  box-shadow: var(--glass-shadow, inset 0 1px 0 rgba(255, 255, 255, .95), 0 20px 44px -14px rgba(17, 17, 17, .24));
}

@utility glass-bar {
  background: var(--glass-bar-bg, linear-gradient(160deg, rgba(255, 255, 255, .68), rgba(255, 255, 255, .42)));
  background-color: transparent;
  border: 1px solid var(--glass-bar-border, rgba(255, 255, 255, .75));
  backdrop-filter: var(--glass-bar-filter, blur(16px) saturate(1.5));
  -webkit-backdrop-filter: var(--glass-bar-filter, blur(16px) saturate(1.5));
  box-shadow: var(--glass-bar-shadow, inset 0 1px 0 rgba(255, 255, 255, .95), 0 6px 22px rgba(17, 17, 17, .08));
}

@utility glass-slot {
  border: 1px solid var(--glass-slot-border, rgba(28, 25, 20, .09));
  border-radius: 12px;
  background: var(--glass-slot-bg, rgba(255, 255, 255, .42));
}

@utility glass-overlay {
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

### Tailwind v3（`tailwind.config.*` + `globals.css`）

在 `tailwind.config` 补模糊档位（如需工具类配合）：

```js
// tailwind.config.js
export default {
  theme: {
    extend: {
      backdropBlur: {
        glass: '16px',
        'glass-lg': '24px',
      },
    },
  },
};
```

在 `globals.css` 用 `@layer components` 声明 `.glass-card` / `.glass-bar` / `.glass-slot` / `.glass-overlay`（样式体与上面 `@utility` 完全一致，并附加 `@supports not` 与 `prefers-reduced-transparency` 兜底）。

---

## 3. 零散单处工具类写法（仅限极少数不便加全局 CSS 的场景）

确保先删除原元素上的不透明 `bg-*` 类；**不加 `overflow-hidden`，不加 `[transform:translateZ(0)]`**；下面采用 v3/v4 均合法的括号刻度写法：

```jsx
// 亮色 L1 玻璃卡
<div className="relative bg-transparent border border-white/75 rounded-[18px] bg-[linear-gradient(180deg,rgba(255,255,255,.50),transparent_52%),linear-gradient(165deg,rgba(255,255,255,.62),rgba(255,255,255,.34))] backdrop-blur-[16px] backdrop-saturate-150 shadow-[inset_0_1px_0_rgba(255,255,255,.95),inset_0_0_0_1px_rgba(255,255,255,.28),inset_0_-1px_0_rgba(17,17,17,.05),0_2px_5px_rgba(17,17,17,.07),0_20px_44px_-14px_rgba(17,17,17,.24)]">

// 暗色 L1 玻璃卡（alpha 峰值 .32，严禁 bg-[rgba(30,30,50,.85)]）
<div className="relative bg-transparent border border-white/[.16] rounded-[20px] bg-[linear-gradient(180deg,rgba(255,255,255,.10),rgba(255,255,255,.03)_52%,transparent),linear-gradient(160deg,rgba(255,255,255,.14),rgba(20,20,28,.20)_46%,rgba(12,12,18,.32))] backdrop-blur-[24px] backdrop-saturate-[1.8] shadow-[inset_0_1px_0_rgba(255,255,255,.30),inset_0_-1px_0_rgba(0,0,0,.30),inset_0_0_0_1px_rgba(255,255,255,.04),inset_0_-24px_44px_-26px_rgba(0,0,0,.55),0_10px_26px_rgba(0,0,0,.45),0_30px_64px_-24px_rgba(0,0,0,.60)]">
```

**合法的 Tailwind 背景光斑层写法**（底色类与渐变类分开，且所有变量带 fallback）：

```jsx
{/* 固定背景光斑层 —— 需同步将 body / 布局外壳上的 bg-background / bg-white / bg-zinc-950 改为 bg-transparent */}
<div
  aria-hidden="true"
  className="fixed inset-0 -z-10 pointer-events-none bg-[var(--glass-paper,#faf7f0)] bg-[radial-gradient(720px_480px_at_10%_-6%,color-mix(in_srgb,var(--glass-blob-1,#ffe81a)_74%,transparent)_0%,transparent_68%),radial-gradient(820px_560px_at_96%_12%,color-mix(in_srgb,var(--glass-blob-2,#0758f7)_24%,transparent)_0%,transparent_66%),radial-gradient(860px_600px_at_50%_52%,color-mix(in_srgb,var(--glass-blob-1,#ffe81a)_30%,transparent)_0%,transparent_72%)] dark:bg-[var(--glass-paper,#0f1018)] dark:bg-[radial-gradient(860px_580px_at_10%_-6%,color-mix(in_srgb,var(--glass-blob-1,#6d4aff)_54%,transparent)_0%,transparent_70%),radial-gradient(920px_640px_at_102%_10%,color-mix(in_srgb,var(--glass-blob-2,#06a8d0)_34%,transparent)_0%,transparent_68%),radial-gradient(780px_540px_at_50%_50%,color-mix(in_srgb,var(--glass-blob-3,#4338ca)_32%,transparent)_0%,transparent_72%)]"
/>
```

---

## 4. shadcn/ui / Radix 项目改造要点

1. **剥离根外壳**：将 `app/layout.tsx` 或根容器上的 `bg-background` 移到 `html`，`body` 与主布局壳改 `bg-transparent`，在根部挂载 `.app-backdrop`。
2. **统一改原语组件（`components/ui/*`）**：
   - `card.tsx`（`Card`）：去掉 `bg-card`，挂 `glass-card`；卡片内部的子指标块改用 `glass-slot`（不嵌套 `backdrop-filter`）。
   - `dialog.tsx` / `sheet.tsx`：`DialogOverlay` 用 `glass-scrim`，`DialogContent` 去 `bg-background` 挂 `glass-overlay`。
   - `popover.tsx` / `dropdown-menu.tsx`：`Content` 挂 `glass-overlay`（浮层独立 Portal，不与父级卡片构成嵌套 Backdrop Root）。
   - `button.tsx`、`input.tsx`、`textarea.tsx`、`badge.tsx`、`table.tsx`（行）：**保持实色不动**。
3. 切勿在 `Card` 上追加 `overflow-hidden`，否则内嵌的 `Select` / `DropdownMenu` / `Tooltip` 与 `focus-visible:ring` 会被截断。
