# Flutter / Dart 玻璃化配方

---

## 1. Flutter 专属红线（动手前必读）

1. **`BoxShadow` 必须设 `blurStyle: BlurStyle.outer`（否则直接违反暗色铁律！）**
   - 与 CSS 不同，Flutter 的 `BoxShadow` 默认 `BlurStyle.normal` 会把整块阴影**实心画在透明容器内部正下方**。若外层 `Container` 带 `Colors.black.withValues(alpha: 0.45 / 0.60)` 阴影而内层是 `BackdropFilter`，模糊滤镜会把身后的黑影一起糊进玻璃里，瞬间把高透玻璃变成不透明黑块。
   - **正确结构**：外层 `DecoratedBox` 仅放 `borderRadius` + **`blurStyle: BlurStyle.outer`** 的双层外投影；中层 `ClipRRect` → `BackdropFilter`（带 `tileMode: TileMode.clamp` 防边缘黑晕）；内层 `DecoratedBox` 放渐变填充与 `border`（边框画在 `ClipRRect` 内部避免圆角细缝）。
2. **`withValues(alpha:)` 版本兼容**
   - Flutter ≥ 3.27 使用 `color.withValues(alpha: ...)`；更旧版本统一用 `color.withOpacity(...)`，全文件不要混用。
3. **饱和度与内嵌阴影的跨平台说明**
   - Flutter 无原生 inset `BoxShadow`：以 **`BlurStyle.outer` 双层外投影 + 顶部高光渐变** 近似厚度，即达标。
   - Web 侧 `saturate ≥ 1.6` 在 Flutter 中可用 `ImageFilter.compose(outer: ColorFilter.matrix(饱和度矩阵), inner: ImageFilter.blur(...))` 近似，或接受仅 `blur` 并在方案中向用户说明。
4. **性能红线（saveLayer 预算）**
   - `BackdropFilter` 触发离屏 `saveLayer`。**严禁在 `ListView.builder` / `GridView.builder` 的每个 item 上套 `BackdropFilter`**（必然滚成掉帧）；列表项改用 `GlassTier.slot`（无模糊的轻半透明槽位）。
   - Flutter ≥ 3.24 + Impeller 场景下，同屏多个共享背景的玻璃卡可用 `BackdropGroup` + `BackdropFilter.grouped` 合并采样降开销。
5. **剥离 `Scaffold` 不透明底色**
   - 将页面 `Scaffold(backgroundColor: Colors.transparent)` 放进 `Stack`，底层铺 `GlassBackdrop`；或直接将 `GlassBackdrop` 置于 `Scaffold.body` 的 `Stack` 最底层，确保玻璃背后真有光斑可模糊。

---

## 2. 完整背景光斑组件（`GlassBackdrop`）

按项目明暗自动切换亮色暖纸 / 暗色 OKLCH 低明高色光晕（也可传入自定义光斑色）：

```dart
import 'package:flutter/material.dart';

class GlassBackdrop extends StatelessWidget {
  const GlassBackdrop({
    super.key,
    this.isDark,
    this.blob1,
    this.blob2,
    this.blob3,
    this.paperColor,
  });

  final bool? isDark;
  final Color? blob1;
  final Color? blob2;
  final Color? blob3;
  final Color? paperColor;

  @override
  Widget build(BuildContext context) {
    final dark = isDark ?? Theme.of(context).brightness == Brightness.dark;
    final base = paperColor ?? (dark ? const Color(0xFF0F1018) : const Color(0xFFFAF7F0));
    final c1 = blob1 ?? (dark ? const Color(0xFF6D4AFF) : const Color(0xFFFFE81A));
    final c2 = blob2 ?? (dark ? const Color(0xFF06A8D0) : const Color(0xFF0758F7));
    final c3 = blob3 ?? (dark ? const Color(0xFF4338CA) : const Color(0xFFFF3044));

    return IgnorePointer(
      child: ColoredBox(
        color: base,
        child: Stack(
          fit: StackFit.expand,
          children: [
            _blob(const Alignment(-0.82, -1.05), 1.15, c1.withValues(alpha: dark ? 0.54 : 0.72)),
            _blob(const Alignment(0.95, -0.78), 1.20, c2.withValues(alpha: dark ? 0.34 : 0.24)),
            _blob(const Alignment(0.0, 0.04), 1.10, c3.withValues(alpha: dark ? 0.30 : 0.14)),
            _blob(const Alignment(0.65, 1.05), 1.10, c1.withValues(alpha: dark ? 0.28 : 0.44)),
          ],
        ),
      ),
    );
  }

  Widget _blob(Alignment center, double radius, Color color) {
    return DecoratedBox(
      decoration: BoxDecoration(
        gradient: RadialGradient(
          center: center,
          radius: radius,
          colors: [color, color.withValues(alpha: 0.0)],
          stops: const [0.0, 0.72],
        ),
      ),
    );
  }
}
```

---

## 3. 复用玻璃封装（4 级 Z 轴材质：`GlassSurface`）

将所有需要玻璃化的容器统一收敛到一个可复用 Widget，避免在页面里复制大段 `BackdropFilter` 样板：

```dart
import 'dart:ui';
import 'package:flutter/material.dart';

enum GlassTier {
  slot,    // L0：玻璃内嵌子块 / 列表项（无 BackdropFilter，省性能且不嵌套）
  panel,   // L1：主卡片 / 核心面板（暗色 alpha <= 0.32）
  bar,     // L2：顶栏 / 底栏 / 侧栏（暗色 alpha 0.30–0.52）
  overlay, // L3：弹窗 / Drawer（配合深色 barrierColor 使用）
}

class GlassSurface extends StatelessWidget {
  const GlassSurface({
    super.key,
    required this.child,
    this.tier = GlassTier.panel,
    this.borderRadius,
    this.padding,
    this.isDark,
  });

  final Widget child;
  final GlassTier tier;
  final BorderRadius? borderRadius;
  final EdgeInsetsGeometry? padding;
  final bool? isDark;

  @override
  Widget build(BuildContext context) {
    final dark = isDark ?? Theme.of(context).brightness == Brightness.dark;
    final cs = Theme.of(context).colorScheme;
    final radius = borderRadius ?? BorderRadius.circular(dark ? 20 : 18);

    // L0：玻璃内部子槽或长列表项 —— 绝不套 BackdropFilter
    if (tier == GlassTier.slot) {
      return Container(
        padding: padding,
        decoration: BoxDecoration(
          borderRadius: borderRadius ?? BorderRadius.circular(12),
          color: dark
              ? Colors.white.withValues(alpha: 0.06)
              : Colors.white.withValues(alpha: 0.42),
          border: Border.all(
            color: dark
                ? Colors.white.withValues(alpha: 0.11)
                : const Color(0xFF1C1914).withValues(alpha: 0.09),
          ),
        ),
        child: child,
      );
    }

    final sigma = dark ? 24.0 : 16.0;
    final isBar = tier == GlassTier.bar;

    final gradientColors = dark
        ? (isBar
            ? [
                const Color(0xFF282834).withValues(alpha: 0.52),
                const Color(0xFF101016).withValues(alpha: 0.30),
              ]
            : [
                Colors.white.withValues(alpha: 0.14),
                cs.surface.withValues(alpha: 0.24),
              ])
        : (isBar
            ? [
                Colors.white.withValues(alpha: 0.68),
                Colors.white.withValues(alpha: 0.42),
              ]
            : [
                Colors.white.withValues(alpha: 0.62),
                Colors.white.withValues(alpha: 0.34),
              ]);

    final borderColor = dark
        ? Colors.white.withValues(alpha: isBar ? 0.12 : 0.16)
        : Colors.white.withValues(alpha: 0.75);

    // 关键：BlurStyle.outer 确保阴影只画在容器外部，不会在玻璃背后垫出黑底
    final shadows = dark
        ? (isBar
            ? [
                BoxShadow(
                  color: Colors.black.withValues(alpha: 0.40),
                  blurRadius: 22,
                  offset: const Offset(0, 6),
                  blurStyle: BlurStyle.outer,
                ),
              ]
            : [
                BoxShadow(
                  color: Colors.black.withValues(alpha: 0.45),
                  blurRadius: 26,
                  offset: const Offset(0, 10),
                  blurStyle: BlurStyle.outer,
                ),
                BoxShadow(
                  color: Colors.black.withValues(alpha: 0.60),
                  blurRadius: 64,
                  offset: const Offset(0, 30),
                  blurStyle: BlurStyle.outer,
                ),
              ])
        : [
            BoxShadow(
              color: const Color(0xFF111111).withValues(alpha: 0.07),
              blurRadius: 5,
              offset: const Offset(0, 2),
              blurStyle: BlurStyle.outer,
            ),
            BoxShadow(
              color: const Color(0xFF111111).withValues(alpha: 0.22),
              blurRadius: 40,
              offset: const Offset(0, 18),
              blurStyle: BlurStyle.outer,
            ),
          ];

    return DecoratedBox(
      decoration: BoxDecoration(
        borderRadius: radius,
        boxShadow: shadows,
      ),
      child: ClipRRect(
        borderRadius: radius,
        child: BackdropFilter(
          filter: ImageFilter.blur(
            sigmaX: sigma,
            sigmaY: sigma,
            tileMode: TileMode.clamp,
          ),
          child: DecoratedBox(
            decoration: BoxDecoration(
              borderRadius: radius,
              border: Border.all(color: borderColor),
              gradient: LinearGradient(
                begin: Alignment.topLeft,
                end: Alignment.bottomRight,
                colors: gradientColors,
              ),
            ),
            child: padding != null ? Padding(padding: padding!, child: child) : child,
          ),
        ),
      ),
    );
  }
}
```

---

## 4. L3 弹窗与 Scrim 配对

在 `showDialog` / `showModalBottomSheet` 中用 `barrierColor` 承担 Scrim 压暗底层噪声，切勿把弹窗本体 alpha 拉高成实色：

```dart
showDialog<void>(
  context: context,
  barrierColor: Colors.black.withValues(alpha: 0.56),
  builder: (_) => Dialog(
    backgroundColor: Colors.transparent,
    elevation: 0,
    child: GlassSurface(
      tier: GlassTier.overlay,
      padding: const EdgeInsets.all(24),
      child: /* 弹窗内容（按钮与输入框维持实色） */,
    ),
  ),
);
```
