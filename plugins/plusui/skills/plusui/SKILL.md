---
name: plusui
description: PlusUi cross-platform .NET UI framework reference — code-first fluent C# API, Skia rendering, MVVM. ALWAYS use this skill when writing or editing ANY PlusUi UI code — building pages, controls, layouts, the fluent Set*/Bind* API, styling/theming, navigation, data binding, custom UserControls, or MVVM with PlusUi. Also trigger when the user mentions PlusUi, a PlusUi control (VStack, HStack, Grid, Label, Entry, Button, ComboBox, ItemsList, DataGrid, etc.), or building a UI for desktop/android/ios/web-wasm/headless targets in PlusUi. PlusUi builds UI in C# code with a fluent/builder API — it is NOT XAML, NOT MAUI, NOT WPF — do not guess the API, always consult this skill first.
---

# PlusUi Reference Skill

PlusUi is a cross-platform .NET UI framework: Skia-rendered, fluent/builder API, MVVM.
UI is written **in C# code**, never XAML. Targets: desktop, android, ios, web/wasm, headless, h264.

Mental model: you compose a tree of control objects and configure each with chained
`Set*` methods. Every `Set*` has a matching `Bind*` for MVVM data binding. Controls
inherit from `UiElement` / `UiTextElement` / `UiLayoutElement`. Colors are `SKColor`.

**Never guess the API — check the reference files below.**

## When to read which file

| Task | Read |
|------|------|
| Do's & don'ts, MVVM/binding/layout/lifecycle best practices | [references/best-practices.md](references/best-practices.md) |
| First app, project setup, app/page lifecycle, platform targets | [references/getting-started.md](references/getting-started.md) |
| Framework structure, element base classes, render/layout pipeline | [references/architecture.md](references/architecture.md) |
| Data binding, `Set*`/`Bind*`, two-way binding, commands, ViewModels | [references/binding-and-state.md](references/binding-and-state.md) |
| Colors (`SKColor`), styles, themes, dark/light | [references/theming.md](references/theming.md) |
| Pages, routing, navigation between views | [references/navigation.md](references/navigation.md) |
| Layout containers: VStack, HStack, Grid, UniformGrid, ScrollView, Scrollbar, Border, Separator | [references/layouts.md](references/layouts.md) |
| Text controls: Label, RichTextLabel, Link, Entry, CodeEditor (syntax highlighting) | [references/text.md](references/text.md) |
| Input controls: Button, Checkbox, RadioButton, Toggle, Slider, ComboBox&lt;T&gt;, DatePicker, TimePicker | [references/input.md](references/input.md) |
| Collection controls: ItemsList, TreeView, DataGrid&lt;T&gt;, TabControl | [references/collections.md](references/collections.md) |
| Display controls: ActivityIndicator, ProgressBar, Image, Solid, LineGraph, GameCanvas (per-frame Skia draw loop) | [references/display.md](references/display.md) |
| Menus & overlays: Menu, ContextMenu, Toolbar, Tooltip, Popups (UiPopupElement) | [references/menus-overlays.md](references/menus-overlays.md) |
| Gesture detectors: Tap, DoubleTap, LongPress, Swipe, Pinch | [references/gestures.md](references/gestures.md) |
| Continuous pointer input: drag-to-resize, splitters, custom drag/hover controls, hover enter/leave commands, mouse cursor control, bound grid tracks; global raw input bus (IGlobalInputService: app-wide keyboard/pointer/scroll events, full PlusKey set, game input) | [references/input-and-dragging.md](references/input-and-dragging.md) |
| Building reusable custom controls via composition; UserControl with its own ViewModel (binding refresh, "VM changes but UI doesn't update") | [references/usercontrol.md](references/usercontrol.md) |

## Critical rules / differences

- **This is NOT XAML, MAUI, or WPF.** No `.xaml` files, no markup, no `DependencyProperty`,
  no `x:Bind`. UI is built entirely in C# with a fluent builder API. Patterns and control
  names from MAUI/WPF do not transfer — do not assume them.
- **Fluent builder.** Construct a control and chain configuration: each `Set*` returns the
  control so calls chain. Compose containers by passing children into the layout's constructor.
- **EVERY `Set*` has a matching `Bind*`.** Use `SetText("x")` for a static value and
  `BindText(...)` to bind to a ViewModel property. If you reach for binding, use `Bind*`,
  not manual event wiring.
- **Prefer built-in controls.** Use the controls in the reference table above before writing
  a custom one. Build a custom control only when no built-in fits.
- **Custom controls: `partial class` + `[GenerateShadowMethods]`.** Reusable controls are
  `UserControl`s defined by composition in a `Build` method, declared `partial` and annotated
  with `[GenerateShadowMethods]` so the fluent `Set*`/`Bind*` shadow methods are generated.
  See [references/usercontrol.md](references/usercontrol.md).
- **Base classes.** Controls inherit `UiElement` (or `UiTextElement` for text-bearing,
  `UiLayoutElement` for containers). Know the base class to find available members.
- **Colors are `SKColor`** (SkiaSharp), e.g. `SKColors.Red` or `new SKColor(0xFF112233)` —
  not WPF `Brush`/`Color` or hex strings.
