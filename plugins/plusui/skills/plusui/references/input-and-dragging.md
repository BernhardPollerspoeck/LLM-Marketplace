# PlusUi Input, Pointer & Dragging Reference

Low-level pointer/mouse handling for controls that need **continuous** interaction —
drag-to-resize splitters, custom sliders/knobs, pan surfaces, hover effects. This is the
layer **below** the command-based gesture detectors (see [gestures.md](gestures.md)):
gesture detectors fire a one-shot `ICommand`; the interfaces here give you a live stream of
pixel deltas while the mouse is held.

> Use a **gesture detector** for discrete events (tap, swipe, long-press). Use the
> **interfaces here** when you need the drag delta every frame (resize, scrub, pan).

## How pointer events reach a control

The framework's `InputService` translates raw mouse events into method calls on the control
**under the pointer**, found via `HitTest`:

1. **Mouse down** → `HitTest(point)` finds the top-most element at that point. If it implements
   `IDraggableControl`, it becomes the active drag target and its `IsDragging` is set `true`.
   (A draggable target takes priority over a scrollable one.)
2. **Mouse move** (while pressed) → the active draggable's `HandleDrag(deltaX, deltaY)` is
   called with the movement **since the last move** — but only once movement exceeds **1px**
   (sub-pixel jitter is ignored).
3. **Mouse up** → `IsDragging` is set back to `false`. If any real dragging happened, the
   trailing click is **suppressed** (a drag does not also fire a click).

Hover is tracked continuously: the element under the pointer gets `IsHovered = true` (and the
previously hovered one `false`) if it implements `IHoverableControl`.

`HitTest` defaults (on `UiElement`) to "return `this` if the point is inside my bounds", so a
custom control is hittable across its whole area automatically. Override `HitTest` only for
non-rectangular hit areas.

## Interactive interfaces

All live in `PlusUi.core`. A control opts in by implementing the interface; the `InputService`
then drives it. `IDraggableControl` / `IScrollableControl` derive from the marker
`IInteractiveControl`.

| Interface | Members | When it's called |
|---|---|---|
| `IDraggableControl` | `void HandleDrag(float deltaX, float deltaY)`; `bool IsDragging { get; set; }` | Mouse held + moved over the control |
| `IScrollableControl` | `void HandleScroll(float deltaX, float deltaY)`; `bool IsScrolling { get; set; }` | Drag-scroll over the control |
| `IHoverableControl` | `bool IsHovered { get; set; }` | Pointer enters/leaves the control |
| `IFocusable` | `bool IsFocusable { get; }` (+ focus members) | Click focuses the control; tab order |
| `IKeyboardInputHandler` | key handlers | Control has focus and a key is pressed |

- **Delta sign differs between drag and scroll.** `HandleDrag` gets `location - lastPosition`
  (positive `deltaX` = moved right, positive `deltaY` = moved down). `HandleScroll` gets the
  **inverted** `lastPosition - location`. Don't copy the sign convention from one to the other.
- `IsDragging` / `IsScrolling` / `IsHovered` are **set by the framework** — you expose them as
  plain `{ get; set; }` and read them (e.g. for hover/drag visuals); you don't raise them yourself.

## Reference implementation: `Slider` (built-in `IDraggableControl`)

`Slider` is the canonical drag target — study it before writing your own:

```csharp
public partial class Slider : UiElement, IDraggableControl, IFocusable, IKeyboardInputHandler
{
    public bool IsDragging { get; set; }   // set by the framework

    public void HandleDrag(float deltaX, float deltaY)
    {
        var pixelRange = ElementSize.Width - 28;        // usable track minus thumb
        var valuePerPixel = (Maximum - Minimum) / pixelRange;
        Value = Math.Clamp(Value + deltaX * valuePerPixel, Minimum, Maximum);
        _onValueChanged?.Invoke(Value);                 // push to bound setter/callback
    }
}
```

## Custom drag control: a resize handle (splitter)

A thin vertical handle the user drags to resize a neighbour. It's a raw `UiElement`
(so `partial` + `[GenerateShadowMethods]`, per the custom-control rules in
[architecture.md](architecture.md)), implements `IDraggableControl` for the drag stream and
`IHoverableControl` for cursor-style feedback, and reports each horizontal delta through a
callback.

```csharp
using PlusUi.core;
using PlusUi.core.Attributes;
using SkiaSharp;

[GenerateShadowMethods]
public partial class ResizeHandle : UiElement, IDraggableControl, IHoverableControl
{
    protected internal override bool IsFocusable => false;
    public override AccessibilityRole AccessibilityRole => AccessibilityRole.None;

    // Framework-driven state — just storage we read while rendering.
    public bool IsDragging { get; set; }
    public bool IsHovered { get; set; }

    private float _width = 6f;
    private Action<float>? _onResize;
    private SKColor _baseColor = new(60, 60, 60);
    private SKColor _activeColor = new(90, 120, 200);

    public ResizeHandle SetHandleWidth(float w) { _width = w; InvalidateMeasure(); return this; }
    public ResizeHandle SetOnResize(Action<float> onResize) { _onResize = onResize; return this; }

    // Fixed thickness, stretch to fill the parent's height.
    public ResizeHandle() => VerticalAlignment = VerticalAlignment.Stretch;

    public override Size MeasureInternal(Size availableSize, bool dontStretch = false)
        => new(_width, dontStretch ? 0 : availableSize.Height);

    public void HandleDrag(float deltaX, float deltaY) => _onResize?.Invoke(deltaX);

    public override void Render(SKCanvas canvas)
    {
        base.Render(canvas);
        using var paint = new SKPaint
        {
            Color = (IsDragging || IsHovered) ? _activeColor : _baseColor,
            IsAntialias = true,
        };
        var x = Position.X + VisualOffset.X;
        var y = Position.Y + VisualOffset.Y;
        canvas.DrawRect(x, y, ElementSize.Width, ElementSize.Height, paint);
    }
}
```

The page/VM owns the actual width and clamps it; the handle only reports deltas:

```csharp
public partial class ShellViewModel : ObservableObject
{
    [ObservableProperty] private float _sidebarWidth = 260f;

    public void ResizeSidebar(float deltaX)
        => SidebarWidth = Math.Clamp(SidebarWidth + deltaX, 180f, 480f);
}
```

## Binding a `Grid` column/row size (drives the resize)

For a layout whose track size changes at runtime (the resized sidebar), use the **bound**
grid track APIs. Unlike most of PlusUi, these take a **`nameof` property name + a `Func<float>`
getter** — *not* an `Expression`. The grid re-measures when that property raises
`PropertyChanged`.

```csharp
public Grid AddBoundColumn(string propertyName, Column column, Func<float> sizeGetter)
public Grid AddBoundColumn(string propertyName, Func<float> sizeGetter)   // Column.Absolute
public Grid AddBoundRow(string propertyName, Row row, Func<float> sizeGetter)
public Grid AddBoundRow(string propertyName, Func<float> sizeGetter)      // Row.Absolute
```

## Full pattern: a resizable sidebar

Three columns — sidebar (bound width) · drag handle (fixed) · content (star). Dragging the
handle updates `SidebarWidth`, which the bound column observes and re-measures.

```csharp
protected override UiElement Build() => new Grid()
    .AddBoundColumn(nameof(vm.SidebarWidth), Column.Absolute, () => vm.SidebarWidth) // 0: sidebar
    .AddColumn(Column.Absolute, 6)                                                   // 1: handle
    .AddColumn(Column.Star, 1)                                                       // 2: content
    .AddRow(Row.Star, 1)
    .AddChild(BuildSidebar(), row: 0, column: 0)
    .AddChild(
        new ResizeHandle().SetOnResize(vm.ResizeSidebar),
        row: 0, column: 1)
    .AddChild(BuildContent(), row: 0, column: 2);
```

## Gotchas / Common LLM mistakes

- **Gesture detectors can't resize.** Tap/Swipe/Pinch fire a one-shot `ICommand` — they give
  you no per-frame delta. For drag-to-resize/scrub you must implement `IDraggableControl`.
- **`HandleDrag` deltas are incremental** (movement since the last move), not absolute pointer
  position. Accumulate them (`width += deltaX`), don't treat `deltaX` as the new size.
- **Drag vs scroll sign is inverted.** `HandleDrag` = `location - last`; `HandleScroll` =
  `last - location`. Copying one into the other inverts the direction.
- **Don't set `IsDragging`/`IsHovered`/`IsScrolling` yourself.** The `InputService` owns them;
  expose them as `{ get; set; }` and only read them (e.g. for visuals).
- **Sub-1px moves are dropped.** `HandleDrag` isn't called until movement exceeds 1px — don't
  rely on receiving every pixel.
- **A real drag suppresses the click.** If you also wrapped the control in a tap handler, the
  tap won't fire after a drag — that's intentional.
- **`AddBoundColumn`/`AddBoundRow` use `nameof` + a `Func<float>`**, not an `Expression` like
  other `Bind*` methods. The size property must raise `PropertyChanged` (`[ObservableProperty]`)
  for the grid to re-measure.
- **Clamp the size in the VM**, not in the handle — the handle should stay reusable and just
  report deltas.
- **Custom drag controls are still controls:** `partial` + `[GenerateShadowMethods]`, implement
  `IsFocusable` and `AccessibilityRole`, and override `MeasureInternal`/`Render`. See
  [architecture.md](architecture.md).
```
