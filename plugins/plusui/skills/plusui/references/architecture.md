# PlusUi Architecture & Mental Model

The single most important reference for writing correct PlusUi code. PlusUi is a Skia-rendered, code-only (no XAML) cross-platform UI framework. UIs are built as **fluent C# trees** of `UiElement`s with a **builder pattern**, data-bound to an `INotifyPropertyChanged` view model.

## Contents
- [The UiElement hierarchy](#the-uielement-hierarchy)
- [The fluent builder pattern](#the-fluent-builder-pattern)
- [IRON RULE: every Set* has a matching Bind*](#iron-rule-every-set-has-a-matching-bind)
- [Source generators: GenerateShadowMethods & partial classes](#source-generators-generateshadowmethods--partial-classes)
- [Common inherited properties](#common-inherited-properties)
- [Measure / Arrange / sizing](#measure--arrange--sizing)
- [Abstract members: IsFocusable & AccessibilityRole](#abstract-members-isfocusable--accessibilityrole)
- [Minimal custom control (correct)](#minimal-custom-control-correct)
- [Common LLM mistakes / Gotchas](#common-llm-mistakes--gotchas)

---

## The UiElement hierarchy

All visual things derive from `UiElement`. The five base classes you must know:

| Class | Derives from | Role | Key abstract/override |
|-------|--------------|------|-----------------------|
| `UiElement` | `IDisposable` | Root of everything. Layout, rendering, binding, styling, accessibility, input, shadow, focus ring. `abstract partial`. | declares `IsFocusable`, `AccessibilityRole` |
| `UiTextElement` | `UiElement` | Base for text-displaying controls (e.g. `Label`, `Entry`). Adds font/paint mgmt, wrapping, truncation. `abstract partial`. | `Label` sets `AccessibilityRole.Label` |
| `UiLayoutElement` | `UiElement` | Base for containers (`Grid`, `VStack`, …). Holds `Children`, propagates `Context`, focus scope, landmarks. `IsFocusable => false`, `AccessibilityRole => Container`. | also has generic `UiLayoutElement<T>` for typed fluent returns of `AddChild`/etc. |
| `UiPageElement` | `UiLayoutElement<UiPageElement>` | A screen. Holds a `ViewModel` and builds a single content **tree** via `protected abstract UiElement Build()`. Navigation lifecycle hooks. | implement `Build()` |
| `UiPopupElement` | `UiElement`, `IInputControl` | Modal overlay. `protected abstract UiElement Build()`, `abstract void Close(bool success)`. `AccessibilityRole => Dialog`. Generic `UiPopupElement<TArg>` / `<TArg,TResult>` add typed args/results. | implement `Build()`, `Close()` |

Notes that trip people up:
- `UiPageElement` and `UiPopupElement` are **not** normal containers. They render an internal `_tree` (returned by `Build()`), **not** their `Children` list. Build the page UI by returning the root element from `Build()`.
- `Context` (the bound `INotifyPropertyChanged`) flows down: a layout element pushes its `Context` to all children; a page sets `_tree.Context = ViewModel`.
- `NullElement` is the empty placeholder used before a tree is built.

Property storage convention everywhere: a property with an **`internal` setter** + public fluent **`Set*`** + **`Bind*`** methods. Public consumers never assign properties directly; they chain `Set*`/`Bind*`.

---

## The fluent builder pattern

Every `Set*`/`Bind*` returns the element so calls chain. Controls are constructed with `new`, configured by chaining, and composed by passing children into containers.

```csharp
protected override UiElement Build() =>
    new VStack(
        new Label()
            .SetText("Welcome")
            .SetTextSize(24)
            .SetFontWeight(FontWeight.Bold)
            .SetTextColor(Colors.White)
            .SetMargin(new Margin(0, 0, 0, 8)),
        new Button()
            .SetText("Save")
            .SetCornerRadius(6)
            .BindText(() => ViewModel.SaveCaption))
    .SetHorizontalAlignment(HorizontalAlignment.Center)
    .SetMargin(new Margin(16));
```

- Children are usually passed via container constructors or `AddChild(...)`.
- `Colors.White`, `Colors.Red`, … are PlusUi `Color`s. `Color` implicitly converts **to** `SKColor` (not the reverse), so pass PlusUi `Color`/`Colors.*` or `new Color(r,g,b)` — not `new SKColor(...)` — into `Set*` methods.

---

## IRON RULE: every Set* has a matching Bind*

For **every** public `Set<Prop>(value)` there MUST be a `Bind<Prop>(...)`. This is enforced by convention across the codebase and by the source generator for inherited props.

- `Set*` takes a **literal value**: `SetText("Hi")`, `SetTextColor(Colors.Red)`.
- `Bind*` takes an **`Expression<Func<TProperty>>`** — a lambda that reads the value off the view model: `BindText(() => ViewModel.Name)`.

```csharp
new Label().BindText(() => ViewModel.UserName);          // string
new Label().BindTextColor(() => ViewModel.StatusColor);  // Color
new Entry().BindIsVisible(() => ViewModel.IsEditing);    // bool
```

How `Bind*` works internally — **this pattern is only available INSIDE `PlusUi.core`** (framework code):

```csharp
// FRAMEWORK-INTERNAL pattern. Does NOT compile in app assemblies:
// UiElement.ExpressionPathService is private protected (CS0122).
public MyControl BindFoo(Expression<Func<Foo>> propertyExpression)
{
    var path = ExpressionPathService.GetPropertyPath(propertyExpression); // string[] property path
    var getter = propertyExpression.Compile();                            // Func<Foo>
    RegisterPathBinding(path, () => Foo = getter());                      // re-runs on PropertyChanged
    return this;
}
```

`RegisterPathBinding` subscribes to the view model's `INotifyPropertyChanged` along the extracted path; whenever any segment raises `PropertyChanged`, the setter re-runs (and the property setter typically calls `InvalidateMeasure()`).

### Bindable state on custom controls OUTSIDE PlusUi.core

Do **not** try to replicate the expression pattern in app code. It is inaccessible by design (`ExpressionPathService` is `private protected`), and runtime `Expression.Compile()` is the wrong approach for AOT targets anyway. What works instead, in order of preference:

1. **Composition.** Model bindable state with built-in controls and their existing `Bind*` methods — e.g. a selection highlight as an overlay element with `BindIsVisible(() => vm.IsSelected)` instead of a custom `IsSelected` property on your control.
2. **String-based binding via `RegisterBinding`** (`protected`, available to your subclass; AOT-safe, no expressions). This is the same consumer-facing style the framework itself uses for `AddBoundColumn`/`AddBoundRow` — a `nameof` property name plus a `Func<T>` getter:

```csharp
public Badge BindCount(string propertyName, Func<int> getter)
{
    RegisterBinding(propertyName, () => SetCount(getter()));  // seeds immediately,
    return this;                                              // re-runs on PropertyChanged(propertyName)
}

// Usage
new Badge().BindCount(nameof(vm.UnreadCount), () => vm.UnreadCount)
```

`RegisterBinding` hooks into the page's string-based `UpdateBindings(propertyName)` channel: the update action runs once at registration (initial value) and again whenever the **page ViewModel** raises `PropertyChanged` with that name. The usual gotcha applies: properties on a non-page VM only refresh if the page VM forwards their `PropertyChanged` (see [usercontrol.md](usercontrol.md)).

`UiTextElement.BindText` additionally offers a **generic formatter** overload (note: generic methods are NOT shadow-generated, see below):

```csharp
new Label().BindText(() => ViewModel.Total, t => $"${t:N2}");  // BindText<T>(expr, formatter)
```

---

## Source generators: GenerateShadowMethods & partial classes

Two generators matter (attributes live in `PlusUi.core.Attributes` and `PlusUi.core.UiPropGen`):

| Attribute | Put it on | What it generates |
|-----------|-----------|-------------------|
| `UiPropGen*` (e.g. `[UiPropGenMargin]`, `[UiPropGenBackground]`) | base classes only (already applied to `UiElement`/`UiTextElement`) | the backing property + `Set*` + `Bind*` methods for that property |
| `[GenerateGenericWrapper]` | base classes only (`UiElement`) | generic wrapper plumbing — **you never use this on your controls** |
| `[GenerateShadowMethods]` | **every concrete control you write** | "shadow" overrides of inherited fluent methods so they return YOUR type |

### Why `[GenerateShadowMethods]` is mandatory

Inherited fluent methods declare their return type as the base (e.g. `UiTextElement SetMargin(...)`). Without shadowing, `new Label().SetMargin(...)` would return `UiTextElement`, so you could no longer chain `Label`-specific methods. The generator emits, for each inherited public fluent method that returns a base type:

```csharp
public new Label SetMargin(Margin margin)   // covariant return via 'new'
{
    base.SetMargin(margin);
    return this;
}
```

Rules the generator follows (and their consequences for you):
- It walks the inheritance chain and shadows public, non-static, non-`[Obsolete]` methods returning a base type.
- **Generic methods are skipped** (e.g. `BindText<T>(expr, formatter)` is not shadowed — calling it mid-chain returns `UiTextElement`; call it last, or recast).
- Methods you define yourself on the control are not duplicated.

### Hard requirements for any control

1. `public partial class` — the class **must** be `partial` (generated code is a second partial part). Non-partial = compile error.
2. `[GenerateShadowMethods]` on the class.
3. Implement the two abstract members (`IsFocusable`, `AccessibilityRole`).
4. **Never redefine** an inherited property (`Background`, `CornerRadius`, `Margin`, …). Use the inherited `Set*`; initialize in the constructor if needed.

---

## Common inherited properties

These come from `UiElement` (via `UiPropGen*`) and are available on **every** element through `Set*`/`Bind*`:

| Property | Type | Fluent setters | Notes |
|----------|------|----------------|-------|
| `Margin` | `Margin` | `SetMargin` / `BindMargin` | space outside. `new Margin(uniform)`, `new Margin(h, v)`, `new Margin(l,t,r,b)` |
| `HorizontalAlignment` | enum | `SetHorizontalAlignment` / `Bind…` | `Left`/`Center`/`Right`/`Stretch` |
| `VerticalAlignment` | enum | `SetVerticalAlignment` / `Bind…` | `Top`/`Center`/`Bottom`/`Stretch` |
| `Background` | `IBackground?` | `SetBackground(Color)` / `SetBackground(IBackground?)` / `BindBackground` / `BindBackgroundColor` | `SolidColorBackground`, gradients, etc. |
| `CornerRadius` | `float` | `SetCornerRadius` / `BindCornerRadius` | applies to background, shadow, focus ring |
| `DesiredSize` | `Size?` | `SetDesiredSize` / `SetDesiredWidth` / `SetDesiredHeight` (+ `Bind*`) | **this is how you set explicit width/height** — there is no `SetWidth`/`SetHeight`. `-1` per axis = unset |
| `IsVisible` | `bool` | `SetIsVisible` / `BindIsVisible` | |
| `Opacity` | `float` | `SetOpacity` / `BindOpacity` | `0..1`; `<1` renders via a layer |
| `VisualOffset` | `Point` | `SetVisualOffset` / `Bind…` | render-time translation (animations/transitions) |
| Shadow: `ShadowColor`/`ShadowOffset`/`ShadowBlur`/`ShadowSpread` | | `Set*`/`Bind*` | shadow only renders when `ShadowColor.Alpha > 0` and `ShadowBlur > 0` |
| Focus ring: `FocusRingColor`/`Width`/`Offset`, `FocusedBackground`, `FocusedBorderColor` | | `Set*`/`Bind*` | |
| Accessibility: `AccessibilityLabel`/`Hint`/`Value`, `IsAccessibilityElement` | | `Set*`/`Bind*` | + `AccessibilityTraits` via `Set/Add/Remove/Bind` |
| `Debug` | `bool` | `SetDebug` | draws margin/bounds/padding overlays |

`Padding` is **not** on `UiElement` — it exists only on controls that own inner content (`Button`, `Entry`, `ComboBox`, `Toolbar`, pickers, …) via `[UiPropGenPadding]`. Do not assume `SetPadding` exists on an arbitrary element.

`UiTextElement` adds: `Text`, `TextSize` (`SetTextSize` — **not** `SetFontSize`), `TextColor`, `FontFamily`, `FontWeight`, `FontStyle`, `HorizontalTextAlignment`, `TextWrapping`, `MaxLines`, `TextTruncation` — each with `Set*`/`Bind*`.

---

## Measure / Arrange / sizing

Classic two-pass layout. The framework drives it; you override the `*Internal` hooks.

| Public (framework calls) | Override this | Purpose |
|--------------------------|---------------|---------|
| `Measure(Size availableSize, …)` | `MeasureInternal(Size availableSize, …)` | compute & return the element's desired `Size`; framework stores it in `ElementSize` |
| `Arrange(Rect bounds)` | `ArrangeInternal(Rect bounds)` | return the element's top-left `Point`; framework stores it in `Position` |

Flow & rules:
- **Measure** clamps to `availableSize`, applies `DesiredSize` (explicit W/H), `Margin`, `Stretch`, and minimum touch target. Base `MeasureInternal` returns ~zero — you almost always override it for a leaf control. Containers measure their children inside it.
- **Arrange** default (`ArrangeInternal`) positions using `HorizontalAlignment`/`VerticalAlignment` + `Margin`. `Stretch` expands `ElementSize` to fill `bounds` minus margin (unless an explicit `DesiredSize` is set). Containers position children inside their override.
- `ElementSize` and `Position` are set by the framework — read them in `Render`, don't set them.
- **Invalidation:** changing a layout-affecting property calls `InvalidateMeasure()`, which dirties this node **and all ancestors to the root** (`NeedsMeasure`/`NeedsArrange`). Containers also propagate to children. Use `InvalidateMeasure()` after mutating a custom layout property; `InvalidateArrange()` for position-only changes.
- **Rendering:** override `Render(SKCanvas)`. Call `base.Render(canvas)` first — it paints background (with `CornerRadius`), shadow, opacity layer, focus ring, and (for containers) children. Then draw your content using `Position`, `ElementSize`, and `VisualOffset` (always add `VisualOffset` to coordinates).

---

## Abstract members: IsFocusable & AccessibilityRole

`UiElement` declares two abstract members every concrete control must implement:

```csharp
protected internal abstract bool IsFocusable { get; }   // can it take keyboard focus?
public abstract AccessibilityRole AccessibilityRole { get; }  // semantic role
```

- `IsFocusable => false` for static/visual controls; `true` for interactive ones (then also implement `IFocusable` to actually receive focus, and override `OnFocus`/`OnBlur` as needed).
- `AccessibilityRole` examples: `Label`, `Button`, `Image`, `Container`, `Dialog`, `None`. `UiLayoutElement` already returns `Container`; `UiPopupElement` returns `Dialog` — only override if your control differs.

---

## Minimal custom control (correct)

Always prefer an existing control (`Grid`, `VStack`, `Label`, `Button`, `TabControl`, …) before writing one. When you genuinely need a new control:

```csharp
using System;
using System.Linq.Expressions;
using PlusUi.core;
using PlusUi.core.Attributes;
using SkiaSharp;

namespace MyApp.Controls;

[GenerateShadowMethods]                       // REQUIRED: generates covariant fluent shadows
public partial class Badge : UiElement        // REQUIRED: partial
{
    // ---- abstract members ----
    protected internal override bool IsFocusable => false;
    public override AccessibilityRole AccessibilityRole => AccessibilityRole.Image;

    public Badge()
    {
        SetCornerRadius(8);                    // use INHERITED fluent setters; do not redefine props
        SetBackground(Colors.Red);
    }

    // ---- control-specific property: Set* + Bind* (IRON RULE) ----
    internal int Count
    {
        get => field;
        set
        {
            if (field == value) return;
            field = value;
            InvalidateMeasure();               // re-layout when it changes
        }
    }

    public Badge SetCount(int count)
    {
        Count = count;
        return this;
    }

    // App-side Bind*: string-based (AddBoundColumn style). The expression-based
    // pattern used by built-in controls only compiles inside PlusUi.core.
    public Badge BindCount(string propertyName, Func<int> getter)
    {
        RegisterBinding(propertyName, () => Count = getter());
        return this;
    }

    // ---- layout: return desired size ----
    public override Size MeasureInternal(Size availableSize, bool dontStretch = false)
    {
        var d = Math.Min(16f, Math.Min(availableSize.Width, availableSize.Height));
        return new Size(d, d);
    }

    // ---- rendering: base paints bg/shadow/cornerradius; draw on top ----
    public override void Render(SKCanvas canvas)
    {
        base.Render(canvas);
        if (!IsVisible) return;

        var cx = Position.X + VisualOffset.X + ElementSize.Width / 2;
        var cy = Position.Y + VisualOffset.Y + ElementSize.Height / 2;
        using var paint = new SKPaint { Color = Colors.White, IsAntialias = true };
        canvas.DrawText(Count.ToString(), cx, cy, SKTextAlign.Center, new SKFont { Size = 10 }, paint);
    }
}
```

Usage chains correctly because of `[GenerateShadowMethods]`:

```csharp
new Badge().SetCount(3).SetMargin(new Margin(4)).BindCount(() => ViewModel.UnreadCount);
//          ^Badge      ^returns Badge (shadowed)  ^returns Badge
```

For a text control, derive from `UiTextElement` instead and reuse `Text`, `Font`, `Paint`, `WrapText`, `ApplyTruncation` (see `Label` as the canonical example).

---

## Common LLM mistakes / Gotchas

- **Forgetting `partial`** or **`[GenerateShadowMethods]`** → compile errors, or fluent chains that break after the first inherited call (return `UiTextElement`/`UiElement` instead of your type).
- **Redefining inherited properties** (`Background`, `CornerRadius`, `Margin`, `Padding`) → hides the generated members and breaks styling/binding. Use the inherited `Set*`; initialize in the constructor.
- **Adding a `Set*` without a matching `Bind*`** → violates the IRON RULE. Always add both.
- **Wrong `Bind*` argument on built-in controls**: they take `Expression<Func<T>>` (`() => ViewModel.X`), **not** a `nameof(...)` string and **not** a precomputed value. The older `Bind(nameof(x), () => x)` two-arg form is gone. (Only *your own* app-side `Bind*` methods and `AddBoundColumn`/`AddBoundRow` use the `nameof` + getter style.)
- **Copying the expression-based `Bind*` internals into app code** → `ExpressionPathService` is `private protected` (CS0122) and `Expression.Compile()` hurts AOT. Use composition or the string-based `RegisterBinding` pattern (see "Bindable state on custom controls OUTSIDE PlusUi.core").
- **Assuming `SetWidth`/`SetHeight` exist** → use `SetDesiredWidth`/`SetDesiredHeight`/`SetDesiredSize` (`DesiredSize`). `-1` on an axis means "unset".
- **`SetFontSize`** does not exist → it's **`SetTextSize`** (the `Label` XML-doc example is stale).
- **Assuming `SetPadding` is universal** → `Padding` lives only on content-owning controls, not base `UiElement`.
- **Passing `new SKColor(...)`** to `SetBackground`/`SetTextColor` → pass a PlusUi `Color` (`Colors.X` or `new Color(r,g,b)`); only `Color → SKColor` converts implicitly.
- **Building page UI in `Children`** → a `UiPageElement`/`UiPopupElement` renders the tree returned from `Build()`, not its `Children` collection.
- **Setting `ElementSize`/`Position` directly** → these are framework-owned outputs of Measure/Arrange. Override `MeasureInternal`/`ArrangeInternal` instead, and forgetting `InvalidateMeasure()` after a layout-affecting change leaves stale layout.
- **Forgetting `base.Render(canvas)`** in a custom `Render`, or omitting `VisualOffset` from draw coordinates → missing background/shadow or mis-positioned content during transitions.
- **Calling the generic `BindText<T>(expr, formatter)` mid-chain** → generic methods aren't shadowed; it returns `UiTextElement`. Call it last.
