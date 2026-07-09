# Display & Feedback Controls

Read-only / non-interactive controls for showing status, progress, media, and data.
All inherit from `UiElement`, are configured purely through the fluent `Set*` / `Bind*` API,
and **none are focusable** (`IsFocusable = false`) — they never participate in keyboard
navigation. Configure with `new Control()...` and chain methods.

Controls covered: [ActivityIndicator](#activityindicator) · [ProgressBar](#progressbar) ·
[Image](#image) · [Solid](#solid) · [LineGraph](#linegraph) · [GameCanvas](#gamecanvas)

Conventions in the tables below:

- **Bind*** column: `yes` = a `Bind<Prop>(() => vm.X)` counterpart exists; `no` = `Set*` only.
- Colors are always `Color` objects (`new Color(r,g,b[,a])` or `Colors.Green`) — never strings.
- Sizes default to `Stretch` unless a `DesiredSize`/`DesiredWidth`/`DesiredHeight` is set.

---

## ActivityIndicator

Spinning 270° arc for **indeterminate** loading/activity state.

`new ActivityIndicator()`

| Property | Type | Default | Bind* | Notes |
|---|---|---|---|---|
| `IsRunning` | `bool` | `true` | yes | Animates only when `IsRunning` **and** `IsVisible` are both true. |
| `Color` | `Color` | `AccentPrimary` (iOS blue) | yes | Arc color. |
| `Speed` | `float` | `1.0f` | yes | Rotation multiplier; clamped to min `0.1f`. |
| `StrokeThickness` | `float` | `3.0f` | yes | Arc line thickness in px. |

```csharp
// Inline save spinner bound to a loading flag
new ActivityIndicator()
    .SetDesiredSize(new Size(20, 20))
    .SetColor(Colors.White)
    .BindIsRunning(() => vm.IsSaving)
    .BindIsVisible(() => vm.IsSaving);

// Prominent status spinner
new ActivityIndicator()
    .SetColor(PlusUiDefaults.AccentSuccess)
    .SetSpeed(1.5f)
    .SetStrokeThickness(4);
```

**Rules**

- Bind `IsRunning` and `IsVisible` to the same state so the spinner stops *and* hides together.
- Default size is **40x40** — always `SetDesiredSize(...)` for inline use (e.g. `16x16`).
- Use contrasting colors (white on dark overlays); wrap in a `Border` for overlay backgrounds.

**Gotchas**

- Default 40x40 is too large for most inline uses — size it explicitly.
- `Speed` silently clamps to `0.1f`; zero/negative values do not stop it.
- `SetIsVisible(false)` freezes the animation even if `IsRunning = true`.
- Renders a 270° arc (intentional, iOS-style), not a full circle.
- Accessibility is automatic: role `Spinner`, trait `Busy` while running, label `"Loading"`. Don't override without reason.
- Use the modern `BindIsRunning(() => vm.IsLoading)` form — the `nameof(...)`-prefixed signature in old docs is outdated.

---

## ProgressBar

Determinate progress as a filled, pill-shaped horizontal bar.

`new ProgressBar()`

| Property | Type | Default | Bind* | Notes |
|---|---|---|---|---|
| `Progress` | `float` | `0.0` | yes | **0.0–1.0** range; auto-clamped. |
| `ProgressColor` | `Color` | `AccentPrimary` | yes | Filled (foreground) portion. |
| `TrackColor` | `Color` | `TrackColor` (light gray) | yes | Unfilled (background) portion. |
| `DesiredHeight` | `float` | `8.0` | yes | Bar thickness. |
| `DesiredWidth` | `float` | stretches | yes | Set with non-Stretch alignment for fixed width. |
| `HorizontalAlignment` | enum | `Stretch` | yes | Inherited from `UiElement`. |
| `Margin` | `Margin` | `0` | yes | Inherited. |
| `IsVisible` | `bool` | `true` | yes | Inherited. |

```csharp
new ProgressBar().SetProgress(0.5f);   // 50%

// Color-coded reactive progress
new ProgressBar()
    .SetDesiredHeight(16)
    .BindProgress(() => vm.Progress)
    .BindProgressColor(() =>
        vm.Progress < 0.3f ? Colors.Red :
        vm.Progress < 0.7f ? Colors.Orange : Colors.Green);

// Fixed-width bar
new ProgressBar()
    .SetHorizontalAlignment(HorizontalAlignment.Left)
    .SetDesiredWidth(200)
    .SetProgress(0.8f);
```

**Rules**

- Use `SetDesiredHeight` to thicken the default 8px bar.
- Pair with a `Label` for a percentage readout; `AccessibilityValue` is auto-computed as a percent.
- `SetTrackColor(Colors.Transparent)` gives a track-less thin line effect.

**Gotchas**

- `Progress` is **0–1, not 0–100** — `0.5` for 50%, never `50`. Most common mistake.
- Renders pill-shaped (corner radius = height/2); `CornerRadius` does not override this.
- `TrackColor` (background) vs `ProgressColor` (fill) are easy to reverse.
- Default Stretch + default 8px height = a thin full-width bar; set height if you want it bolder.
- Colors require `Color` objects, not `"green"` strings.

---

## Image

Displays static and animated images (PNG/JPEG/WebP/BMP/ICO, GIF, SVG).

`new Image()`

| Property | Type | Default | Bind* | Notes |
|---|---|---|---|---|
| `ImageSource` | `string` | — | yes | Embedded (no prefix), `file:` local, or `http(s)://` URL; format auto-detected. |
| `Aspect` | `Aspect` | `AspectFit` | yes | `Fill` / `AspectFit` / `AspectFill`. |
| `TintColor` | `Color?` | `null` | yes | **SVG only**; ignored by raster images. |
| `DesiredSize` | `Size` | — | yes | Both dims set ⇒ aspect ignored, image distorted. |
| `DesiredWidth` | `float` | — | yes | Height derived from aspect ratio. |
| `DesiredHeight` | `float` | — | yes | Width derived from aspect ratio. |
| `CornerRadius` | `float` | `0` | yes | `size/2` for circular images. |
| `Background` | `IBackground`/`Color` | — | yes | Color or gradient behind image. |
| `Opacity` | `float` | `1.0` | yes | `0.0`–`1.0`. |
| `HorizontalAlignment` | enum | `Undefined` | yes | |
| `VerticalAlignment` | enum | `Undefined` | yes | |
| `Margin` | `Margin` | — | yes | |
| `IsVisible` | `bool` | `true` | yes | |
| `AccessibilityLabel` | `string?` | — | yes | Set for informational (non-decorative) images. |

```csharp
new Image().SetImageSource("plusui.png").SetDesiredHeight(120);

// Circular avatar from a URL
new Image()
    .SetImageSource("https://example.com/avatar.jpg")
    .SetAspect(Aspect.AspectFill)
    .SetDesiredSize(new Size(64, 64))
    .SetCornerRadius(32);

// Themed SVG icon
new Image()
    .SetImageSource("Assets/Icons/settings.svg")
    .SetTintColor(Colors.Blue)
    .SetDesiredSize(new Size(24, 24));
```

**Rules**

- Always `SetImageSource(...)` first.
- Set only `DesiredHeight` *or* `DesiredWidth` to keep aspect ratio; let the other auto-scale.
- Circular avatar: `AspectFill` + equal size + `SetCornerRadius(size/2)`.
- GIF/WebP animations play automatically when loaded.

**Gotchas**

- `TintColor` affects **SVG only** — raster images ignore it.
- Setting both `DesiredWidth` and `DesiredHeight` ignores aspect ratio and distorts.
- Animations cannot be paused/controlled programmatically.
- Web images load async; the element is blank until the download finishes.
- SVGs render at display size (max quality), not their intrinsic size.
- Source paths are **case-sensitive** on Linux/Mac for `file:` and embedded resources (`logo.png` ≠ `Logo.png`).

---

## Solid

A solid-color (or gradient) rectangle — spacers, dividers, colored blocks, cards, status dots.

`new Solid(float? width = null, float? height = null, Color? color = null)`

| Property | Type | Default | Bind* | Notes |
|---|---|---|---|---|
| `Background` | `IBackground` | `null` (transparent) | yes | `SetBackground(Color)` shorthand or `SetBackground(IBackground)` for gradients. |
| `CornerRadius` | `float` | `0` | yes | `size/2` for a circle. |
| `DesiredSize` | `Size` | `0,0` unless set in ctor | yes | Constructor width/height feed this. |
| `Opacity` | `float` | `1.0` | yes | |
| `HorizontalAlignment` | enum | `Stretch` | yes | |
| `VerticalAlignment` | enum | `Stretch` | yes | |
| `Margin` | `Margin` | `None` | yes | |
| `IsVisible` | `bool` | `true` | yes | |
| `ShadowColor` | `Color` | `Transparent` | yes | |
| `ShadowOffset` | `Point` | `0,0` | yes | |
| `ShadowBlur` | `float` | `0` | yes | |
| `ShadowSpread` | `float` | `0` | yes | |

```csharp
new Solid(60, 60, PlusUiDefaults.AccentError);           // colored block
new Solid(2, 24, PlusUiDefaults.BorderColor)             // vertical divider
    .SetMargin(new Margin(left: 8, right: 8));
new Solid().SetVerticalAlignment(VerticalAlignment.Stretch); // flexible spacer

// Card with shadow
new Solid(300, 200)
    .SetBackground(Colors.White)
    .SetCornerRadius(12)
    .SetShadowColor(new Color(0, 0, 0, 50))
    .SetShadowOffset(new Point(0, 4))
    .SetShadowBlur(8);

// Status dot bound to view model
new Solid(12, 12)
    .SetCornerRadius(6)
    .BindBackgroundColor(() =>
        vm.Status == "online" ? Colors.Green :
        vm.Status == "away"   ? Colors.Yellow : Colors.Gray);

// Gradient block
new Solid(300, 150)
    .SetBackground(new LinearGradient(Colors.Blue, Colors.Purple, 45))
    .SetCornerRadius(12);
```

**Rules**

- Fixed block: `new Solid(w, h, color)`. Horizontal divider: `new Solid(null, 1, color)`. Vertical: `new Solid(1, null, color)`.
- No-arg `new Solid()` in a `VStack`/`HStack` = flexible spacer.
- `BindBackgroundColor()` for data-bound colors; `BindBackground()` for data-bound `IBackground`.

**Gotchas**

- A bare `new Solid()` renders nothing — it has no color and stretches; set a color/background.
- Constructor takes a `Color`, not a `SolidColorBackground` (it wraps automatically).
- The null dimension stretches; the fixed dimension is the thickness. The constructor converts both `null` and `0` to the same value, so `new Solid(null, 1)` and `new Solid(0, 1)` behave identically — both stretch across that axis and render a 1px-tall divider.
- Defaults to Stretch in **both** axes; pass one of width/height for a single-dimension fixed size.
- Circle: `CornerRadius` must equal half the (square) size, e.g. `Solid(12,12)` + `SetCornerRadius(6)`.
- Purely decorative: `AccessibilityRole.None`, non-focusable.

---

## LineGraph

Plots a series of values as a line chart with optional fill, grid, and Y-axis labels.

`new LineGraph()`

| Property | Type | Default | Bind* | Notes |
|---|---|---|---|---|
| `DataPoints` | `IReadOnlyList<float>` | empty | yes | Needs **≥2** points to render. |
| `LineColor` | `Color` | `(100,200,255)` light blue | yes | |
| `LineThickness` | `float` | `2f` | **no** | |
| `FillColor` | `Color?` | `null` | yes | Area fill under the line; use alpha. |
| `MinValue` | `float?` | `null` (auto) | **no** | Fixed Y min; `Set*` only. |
| `MaxValue` | `float?` | `null` (auto) | **no** | Fixed Y max; `Set*` only. |
| `GridColor` | `Color?` | `null` (hidden) | **no** | 3 horizontal lines at 25/50/75%. |
| `ShowAxisLabels` | `bool` | `true` | **no** | Y-axis min/max labels on the left. |
| `LabelColor` | `Color` | `(150,150,150)` gray | **no** | |
| `ValueFormat` | `string` | `"F1"` | **no** | .NET numeric format for labels. |

```csharp
new LineGraph()
    .SetDataPoints(new float[] { 10, 20, 15, 25, 30, 22 })
    .SetDesiredHeight(160);

// Fixed scale, gridded, bound to live data
new LineGraph()
    .BindDataPoints(() => vm.TemperatureReadings)
    .SetLineColor(new Color(255, 100, 100))
    .SetFillColor(new Color(100, 200, 255, 80))
    .SetGridColor(new Color(200, 200, 200, 100))
    .SetMinValue(0f)
    .SetMaxValue(100f)
    .SetDesiredHeight(200);
```

**Rules**

- Provide ≥2 points; bind `DataPoints` (`List<float>`/`IReadOnlyList<float>`) for live updates.
- Use `SetMinValue`/`SetMaxValue` to lock the Y scale; omit for auto-scaling from data.
- Constrain with `SetDesiredHeight`/`SetDesiredSize` — it stretches by default.
- Semi-transparent `FillColor` (4th alpha arg) reads better than a solid fill.

**Gotchas**

- `MinValue`/`MaxValue` have **no Bind\*** — they cannot be data-bound.
- `<2` data points renders blank.
- If `MinValue == MaxValue` (or differ by `<0.0001f`), max is auto-bumped by `1.0f` to avoid divide-by-zero.
- `SetShowAxisLabels(false)` hides labels but still reserves **36px** for the Y-axis.
- Grid is always exactly 3 horizontal lines at fixed 25/50/75% — no vertical grid, no custom intervals.
- No X-axis / time labels; only Y-axis labels (auto-formatted via `ValueFormat`, no custom text).

---

## GameCanvas

Hands you a **raw Skia canvas once per frame** for imperative, immediate-mode rendering —
games, particle effects, continuous animations, custom visualizations. The opposite of the
declarative control tree: its `Render` runs **every frame** (~60 FPS on desktop), so it
redraws continuously without any invalidation/binding.

`new GameCanvas()`

| Property | Type | Default | Bind* | Notes |
|---|---|---|---|---|
| `OnDraw` | `Action<GameCanvasDrawContext>` | — | yes | Per-frame draw callback (`SetOnDraw`/`BindOnDraw`). |
| `DesiredWidth`/`DesiredHeight` | `float` | stretches | yes | Fills available space by default (Stretch both axes). |
| `Background`, `CornerRadius`, `Margin`, `IsVisible` | — | inherited | yes | Base `UiElement` members work as usual. |

The callback receives a `GameCanvasDrawContext` (readonly struct):

| Member | Type | Meaning |
|---|---|---|
| `Canvas` | `SKCanvas` | Draw target — already **translated and clipped**: (0,0) is the canvas' top-left, draw in local 0..Width / 0..Height. |
| `Size` | `Size` | Current drawable size. |
| `DeltaTime` | `TimeSpan` | Time since the previous frame (zero on the first frame). |
| `TotalTime` | `TimeSpan` | Time since the first rendered frame. |
| `FrameCount` | `long` | Zero-based frame index. |

```csharp
new GameCanvas()
    .SetBackground(new Color(20, 20, 28))
    .SetCornerRadius(8)
    .SetDesiredHeight(360)
    .SetOnDraw(ctx =>
    {
        vm.Step((float)ctx.DeltaTime.TotalSeconds, ctx.Size.Width, ctx.Size.Height);
        using var paint = new SKPaint { Color = SKColors.Lime, IsAntialias = true };
        ctx.Canvas.DrawCircle(vm.X, vm.Y, vm.Radius, paint);
    });
```

Alternative for an object-oriented loop: subclass and `override void Draw(GameCanvasDrawContext)`
(the base `Draw` invokes the `SetOnDraw` callback).

**Rules**

- **Input does NOT come from the canvas.** `GameCanvas` only draws; read keyboard/pointer from
  the injected `IGlobalInputService` (see [input-and-dragging.md](input-and-dragging.md)) —
  canvas and input are deliberately decoupled.
- Advance game state with `DeltaTime` (frame-rate independent), not a fixed per-frame step.
- Keep the state in the ViewModel; the draw callback should only step + render it.
- Constrain with `SetDesiredHeight`/`SetDesiredWidth` — it stretches both axes by default.

**Gotchas**

- Draw in **local coordinates** (0..`Size.Width`/`Height`) — the canvas is already translated
  to the element's position and clipped to its bounds. Don't add `Position`/offsets yourself.
- The first frame reports `DeltaTime`/`TotalTime` of **zero** (clock starts at first paint).
- `using`-dispose any `SKPaint`/`SKFont` you create in the callback — it runs every frame.
- The callback runs on the render thread every frame — keep it fast, no alloc-heavy work,
  never block or throw.
- There is no "start/stop" API — it draws as long as it's visible; `SetIsVisible(false)` stops
  rendering (and the delta clock keeps running, so the next visible frame has a large delta).
- Non-focusable, `AccessibilityRole.None` — it never takes keyboard focus; that's another
  reason input comes from `IGlobalInputService`.

---

## Common LLM mistakes

- Using `0–100` for `ProgressBar.Progress` instead of `0.0–1.0`.
- Passing color **strings** (`"green"`) — all color props need `Color`/`Colors.*`/`new Color(...)`.
- Forgetting `SetDesiredSize` on `ActivityIndicator` (defaults to a large 40x40).
- Expecting an `ActivityIndicator` to spin while hidden — it stops when `IsVisible = false`.
- Using `TintColor` on a PNG/JPEG (SVG only).
- Trying to bind `LineGraph.MinValue`/`MaxValue` or `LineGraph.LineThickness` (no `Bind*` exist).
- Leaving a bare `new Solid()` with no color (renders nothing) when a colored block was intended.
- Setting both `DesiredWidth` and `DesiredHeight` on an `Image` and expecting aspect ratio to be preserved (it distorts).
- Trying to make these controls focusable / keyboard-interactive — all are non-focusable by design.
- Trying to read input from `GameCanvas` (clicks, keys) — it only draws; input comes from `IGlobalInputService`.
- Driving a `GameCanvas` animation with bindings/invalidation — it already redraws every frame; just draw the current state.
