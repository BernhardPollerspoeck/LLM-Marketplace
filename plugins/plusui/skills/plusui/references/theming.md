# PlusUi Styling & Theming Reference

How styles are defined, applied, and switched in PlusUi. All APIs verified against `PlusUi.core/Styling/*` and `PlusUi.core/Structures/*`.

## Table of contents
- [Mental model](#mental-model)
- [Defining a global style (IApplicationStyle)](#defining-a-global-style-iapplicationstyle)
- [Default styling (PlusUiDefaults)](#default-styling-plusuidefaults)
- [Theme switching (Default / Light / Dark)](#theme-switching-default--light--dark)
- [Colors](#colors)
- [Backgrounds](#backgrounds)
- [Per-page styles](#per-page-styles)
- [Per-control (inline) styling & opting out](#per-control-inline-styling--opting-out)
- [Text, border, shadow properties](#text-border-shadow-properties)
- [Complete adaptive theme example](#complete-adaptive-theme-example)
- [Common LLM mistakes / gotchas](#common-llm-mistakes--gotchas)

---

## Mental model

There are three layers of styling, applied in this order (later overrides earlier):

| Layer | Where | Scope |
|-------|-------|-------|
| Built-in defaults | `PlusUiDefaults` (baked into control property defaults) | Every control, always |
| Global style | `IApplicationStyle.ConfigureStyle(Style)` via `builder.StylePlusUi<T>()` | Whole app, by control type |
| Page style | `UiPageElement.ConfigurePageStyles(Style)` override | One page, by control type |
| Inline | Fluent `Set*` calls in `Build()` | One control instance |

Within the global/page layers, theme actions layer too: the `Default` theme actions are applied first, then the current theme (`Light`/`Dark`) actions are applied on top. See [theme switching](#theme-switching-default--light--dark).

Styles are matched by `IsAssignableFrom`: a style registered for a base type also applies to derived controls (e.g. `AddStyle<UiTextElement>` affects `Label`, `Entry`, etc.).

---

## Defining a global style (IApplicationStyle)

Implement `IApplicationStyle` and register it. `ConfigureStyle` runs once at startup (`InitializePlusUi`).

```csharp
using PlusUi.core;

public class MyAppStyle : IApplicationStyle
{
    public void ConfigureStyle(Style style)
    {
        style.AddStyle<Label>(label => label
            .SetTextColor(Colors.White)
            .SetTextSize(16));

        style.AddStyle<Button>(button => button
            .SetTextColor(Colors.White)
            .SetBackground(new Color(0, 122, 255))
            .SetHoverBackground(new SolidColorBackground(new Color(0, 100, 200)))
            .SetCornerRadius(8)
            .SetPadding(new Margin(16, 8)));

        style.AddStyle<Entry>(entry => entry
            .SetTextColor(Colors.White)
            .SetPlaceholderColor(new Color(150, 150, 150))
            .SetBackground(new Color(45, 45, 45))
            .SetCornerRadius(4)
            .SetPadding(new Margin(12, 8)));

        // Page background (UiPageElement is the base type of every page)
        style.AddStyle<UiPageElement>(page => page
            .SetBackground(new Color(30, 30, 30)));
    }
}
```

Register during app configuration:

```csharp
builder.StylePlusUi<MyAppStyle>();   // AddSingleton<IApplicationStyle, MyAppStyle>
```

`Style.AddStyle<TElement>(Action<TElement>)` returns the `Style` for chaining. `TElement` must derive from `UiElement`.

---

## Default styling (PlusUiDefaults)

PlusUi ships a coherent **dark theme out of the box** — you do not need any style class to get sensible defaults. Defaults live in the static `PlusUiDefaults` class and are baked into each control's property initializers (e.g. `Button` sets corner radius and hover background in its constructor; `UiPageElement` sets `BackgroundPage`).

Reuse `PlusUiDefaults` constants/colors in your own styles for consistency:

| Member | Value | Use |
|--------|-------|-----|
| `BackgroundPage` | `(30,30,30)` | Page/window background (default for all pages) |
| `BackgroundPrimary` | `(45,45,45)` | Panels, cards |
| `BackgroundControl` | `(60,60,60)` | Buttons, inputs |
| `BackgroundHover` | `(75,75,75)` | Hover state |
| `BackgroundInput` | `(50,50,50)` | Entry fields |
| `BackgroundSelected` / `AccentPrimary` | `(0,120,215)` | Selection / accent |
| `TextPrimary` | `Colors.White` | Primary text |
| `TextSecondary` | `(180,180,180)` | De-emphasized text |
| `TextPlaceholder` | `(128,128,128)` | Placeholder text |
| `BorderColor` | `(80,80,80)` | Borders/separators |
| `AccentSuccess` / `AccentError` / `AccentWarning` | green / red / orange | Status colors |
| `FontSize` / `FontSizeLarge` / `FontSizeSmall` | `14` / `18` / `12` | Typography |
| `CornerRadius` | `4` | Standard rounding |
| `Spacing` / `SpacingSmall` / `SpacingLarge` | `8` / `4` / `16` | Layout spacing |

```csharp
style.AddStyle<Button>(b => b
    .SetBackground(PlusUiDefaults.BackgroundControl)
    .SetHoverBackground(new SolidColorBackground(PlusUiDefaults.BackgroundHover))
    .SetCornerRadius(PlusUiDefaults.CornerRadius));
```

> There is **no `DefaultStyle` class**. Built-in defaults come from `PlusUiDefaults`, not from a style you register.

---

## Theme switching (Default / Light / Dark)

The `Theme` enum has exactly three values: `Default`, `Light`, `Dark`. Register theme-specific overrides by passing a `Theme` (or theme string) to `AddStyle`:

```csharp
public void ConfigureStyle(Style style)
{
    // Base look — applies in EVERY theme
    style.AddStyle<Label>(l => l.SetTextSize(16));
    style.AddStyle<UiPageElement>(p => p.SetBackground(new Color(30, 30, 30)));

    // Light theme overrides (layered on top of Default)
    style.AddStyle<UiPageElement>(Theme.Light, p => p
        .SetBackground(new Color(250, 250, 250)));
    style.AddStyle<Label>(Theme.Light, l => l
        .SetTextColor(Colors.Black));

    // Dark theme overrides
    style.AddStyle<UiPageElement>(Theme.Dark, p => p
        .SetBackground(new Color(18, 18, 18)));
    style.AddStyle<Label>(Theme.Dark, l => l
        .SetTextColor(Colors.White));
}
```

How application works (`Style.ApplyStyle`): the `Default` theme's actions are **always** applied first, then — if the current theme is not `Default` — the current theme's actions are applied on top. So put shared properties under `Default` and only the differences under `Light`/`Dark`.

Switch theme at runtime via `IThemeService` (resolve from DI). `SetTheme` immediately re-applies styles to the current page:

```csharp
public class SettingsViewModel(IThemeService themeService)
{
    public void UseDark() => themeService.SetTheme(Theme.Dark);
    public void UseLight() => themeService.SetTheme(Theme.Light);
    // string overload also exists: themeService.SetTheme("Dark");
}
```

`IThemeService` members: `string CurrentTheme { get; }`, `bool SetTheme(Theme theme)`, `bool SetTheme(string theme)`. Default current theme is `"Default"`.

---

## Colors

`Color` is an immutable ARGB `readonly struct`: `Color(byte r, byte g, byte b, byte a = 255)`.

```csharp
var rgb   = new Color(255, 128, 0);          // opaque orange
var rgba  = new Color(255, 255, 255, 128);   // 50% white
var faded = Colors.Blue.WithAlpha(80);       // copy with new alpha
var c     = Color.FromRgb(34, 49, 220);      // factory helpers
var c2    = Color.FromArgb(255, 34, 49, 220);
```

- `Colors` is a static palette of named colors (`Colors.White`, `Colors.Black`, `Colors.Red`, `Colors.Blue`, `Colors.LightGray`, `Colors.Transparent`, etc.).
- `Color` implicitly converts to SkiaSharp's `SKColor` (used internally for rendering). You normally pass `Color`, not `SKColor`, to the public fluent API.

> There is **no hex/int constructor** (`new Color(0x3498db)` does not exist). Build colors from byte channels or `Colors.*`.

---

## Backgrounds

All backgrounds implement `IBackground`. `SetBackground` accepts either a `Color` (convenience overload) or an `IBackground`.

| Type | Constructor | Notes |
|------|-------------|-------|
| `SolidColorBackground` | `SolidColorBackground(Color color)` | Implicitly converts from `Color` |
| `LinearGradient` | `LinearGradient(Color startColor, Color endColor, float angle = 0)` | Angle in degrees: 0 = L→R, 90 = T→B |
| `MultiStopGradient` | `MultiStopGradient(float angle, params GradientStop[] stops)` | Needs ≥ 2 stops; also takes `IEnumerable<GradientStop>` |
| `RadialGradient` | `RadialGradient(Color centerColor, Color edgeColor, Point? center = null)` | Center point in relative 0–1 coords; default `(0.5, 0.5)` |

`GradientStop` is a record: `GradientStop(Color Color, float Position)` where `Position` is 0–1.

```csharp
// Solid (two equivalent forms; Color overload is simplest)
.SetBackground(new Color(45, 45, 45))
.SetBackground(new SolidColorBackground(new Color(45, 45, 45)))

// Linear gradient at 45 degrees
.SetBackground(new LinearGradient(Colors.Blue, Colors.Purple, 45))

// Multi-stop gradient, top-to-bottom
.SetBackground(new MultiStopGradient(
    90,
    new GradientStop(Colors.Red, 0f),
    new GradientStop(Colors.Yellow, 0.5f),
    new GradientStop(Colors.Green, 1f)))

// Radial gradient
.SetBackground(new RadialGradient(Colors.White, Colors.Black))
```

Related background setters: `SetHoverBackground(IBackground?)`, `SetFocusedBackground(...)`, `SetHighContrastBackground(...)`. Each has a matching `Bind*` for data binding.

---

## Per-page styles

Override `ConfigurePageStyles` on a page to add styles scoped to that page only. They are layered on top of the global style and follow the same theme rules.

```csharp
public class SettingsPage(SettingsPageViewModel vm) : UiPageElement(vm)
{
    protected override void ConfigurePageStyles(Style pageStyle)
    {
        // Only affects Buttons on this page
        pageStyle.AddStyle<Button>(b => b
            .SetBackground(PlusUiDefaults.AccentSuccess));

        // Page-scoped dark override
        pageStyle.AddStyle<Button>(Theme.Dark, b => b
            .SetBackground(new Color(30, 120, 60)));
    }

    protected override UiElement Build() => /* ... */;
}
```

Page styles are rebuilt and re-applied whenever the page is built (`BuildPage`).

---

## Per-control (inline) styling & opting out

Inline `Set*` calls in `Build()` override style-layer values for that one instance:

```csharp
new Label()
    .SetText("Title")
    .SetTextColor(Colors.White)
    .SetTextSize(24);
```

Opt a control out of **all** global/page styling with `IgnoreStyling()`:

```csharp
new Label()
    .SetText("Unstyled")
    .IgnoreStyling();   // styles in ConfigureStyle/ConfigurePageStyles are skipped
```

---

## Text, border, shadow properties

Text setters (on `UiTextElement` and text-bearing controls). Every `Set*` has a matching `Bind*`.

| Method | Signature |
|--------|-----------|
| `SetTextColor` | `(Color value)` |
| `SetTextSize` | `(float value)` |
| `SetFontFamily` | `(string? value)` |
| `SetPlaceholderColor` | `(Color value)` — inputs |
| `SetCornerRadius` | `(float value)` — single radius for all corners |
| `SetPadding` | `(Margin padding)` |

```csharp
new Label().SetTextColor(Colors.White).SetTextSize(18).SetFontFamily("MyFont");
```

`Border` control (use for outlined containers):

```csharp
new Border()
    .SetStrokeColor(Colors.Red)        // (Color)
    .SetStrokeThickness(2f)            // (float)
    .SetStrokeType(StrokeType.Solid)   // Solid | Dashed | Dotted
    .SetCornerRadius(10)
    .AddChild(content);
```

Shadows are set via **separate** property methods (there is no `Shadow` object / `SetShadow`):

```csharp
element
    .SetShadowColor(new Color(0, 0, 0, 80))
    .SetShadowOffset(new Point(0, 4))
    .SetShadowBlur(8f)
    .SetShadowSpread(0f);
```

---

## Complete adaptive theme example

```csharp
public class MyAppTheme : IApplicationStyle
{
    // Shared palette
    private static readonly Color Primary   = new(0, 122, 255);
    private static readonly Color Secondary = new(88, 86, 214);

    public void ConfigureStyle(Style style)
    {
        // --- Default theme: shared structure ---
        style.AddStyle<Label>(l => l.SetTextSize(16));
        style.AddStyle<Button>(b => b
            .SetTextColor(Colors.White)
            .SetBackground(Primary)
            .SetHoverBackground(new SolidColorBackground(Secondary))
            .SetCornerRadius(8)
            .SetPadding(new Margin(16, 10)));
        style.AddStyle<Entry>(e => e
            .SetCornerRadius(6)
            .SetPadding(new Margin(12, 8)));

        // --- Light overrides ---
        style.AddStyle<UiPageElement>(Theme.Light, p => p.SetBackground(new Color(250, 250, 250)));
        style.AddStyle<Label>(Theme.Light, l => l.SetTextColor(Colors.Black));
        style.AddStyle<Entry>(Theme.Light, e => e
            .SetTextColor(Colors.Black)
            .SetPlaceholderColor(new Color(120, 120, 120))
            .SetBackground(new Color(235, 235, 235)));

        // --- Dark overrides ---
        style.AddStyle<UiPageElement>(Theme.Dark, p => p.SetBackground(new Color(18, 18, 18)));
        style.AddStyle<Label>(Theme.Dark, l => l.SetTextColor(Colors.White));
        style.AddStyle<Entry>(Theme.Dark, e => e
            .SetTextColor(Colors.White)
            .SetPlaceholderColor(new Color(150, 150, 150))
            .SetBackground(new Color(40, 40, 40)));
    }
}

// Registration + runtime switch
builder.StylePlusUi<MyAppTheme>();
// later: themeService.SetTheme(Theme.Dark);
```

---

## Common LLM mistakes / gotchas

- **No `DefaultStyle` class.** `builder.StylePlusUi<DefaultStyle>()` will not compile. Defaults come from `PlusUiDefaults` automatically; register your own `IApplicationStyle` instead.
- **`Color` has no hex/int constructor.** Use `new Color(r, g, b[, a])`, `Color.FromRgb/FromArgb`, `WithAlpha`, or `Colors.*`. Do not write `new Color(0x3498db)`.
- **`SetCornerRadius` takes one `float` only** — there is no four-corner overload. All corners share the radius.
- **No `SetShadow(...)` / `Shadow` type.** Set shadows with `SetShadowColor`, `SetShadowOffset(Point)`, `SetShadowBlur`, `SetShadowSpread`.
- **`SetHoverBackground` takes `IBackground` only — not `Color`.** Wrap in `new SolidColorBackground(color)`. (C# does not chain the implicit `Color`→`SolidColorBackground` conversion to reach `IBackground`.) However, `SetFocusedBackground` and `SetHighContrastBackground` both have `Color` overloads. `SetBackground` also has a `Color` overload.
- **Default-theme actions always run.** Properties under `Theme.Light`/`Theme.Dark` layer *on top* of `Default`; they don't replace it. Put shared values under `Default`, only differences under the named themes.
- **Styles match by base type.** `AddStyle<UiTextElement>` (or `<UiElement>`) also styles derived controls like `Label`/`Entry`. Register against the most specific type you want to target.
- **Pages default to dark `(30,30,30)`.** Override `UiPageElement` background per theme (especially for Light) or pages stay dark.
- **`ConfigureStyle` runs once at startup.** Do not put per-frame/dynamic logic there; use theme keys for variants and `IThemeService.SetTheme` to switch at runtime.
- **Use `Color` and `Colors` in public fluent calls**, not `SKColor`/`SKColors`. `SKColor` is internal rendering detail (a few legacy control overloads accept it, but prefer `Color`).
- **Opt out with `IgnoreStyling()`** when a control must keep its inline look regardless of global/page styles.
