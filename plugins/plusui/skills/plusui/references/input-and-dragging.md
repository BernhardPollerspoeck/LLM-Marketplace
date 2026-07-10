# PlusUi Input, Pointer & Dragging Reference

Low-level pointer/mouse handling for controls that need **continuous** interaction —
drag-to-resize splitters, custom sliders/knobs, pan surfaces, hover effects. This is the
layer **below** the command-based gesture detectors (see [gestures.md](gestures.md)):
gesture detectors fire a one-shot `ICommand`; the interfaces here give you a live stream of
pixel deltas while the mouse is held.

> Use a **gesture detector** for discrete events (tap, swipe, long-press). Use the
> **interfaces here** when you need the drag delta every frame (resize, scrub, pan). Use the
> **global input bus** ([`IGlobalInputService`](#global-input-bus-iglobalinputservice--raw-keyboard--pointer-no-focushit-test))
> for raw app-wide keyboard/pointer streams independent of focus and hit-testing (games, shortcuts).

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
previously hovered one `false`) if it implements `IHoverableControl`. Independently, **any**
element can fire hover enter/leave `ICommand`s and set a mouse cursor — see
[Hover commands & cursor](#hover-commands--cursor--on-any-element) below.

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

## Hover commands & cursor — on ANY element

These two live on the `UiElement` base, so **every** control supports them through the fluent
API — no interface to implement, no custom control needed.

### Hover enter/leave commands

One-shot `ICommand`s fired when the pointer **enters** or **leaves** the element — the command
counterpart to the live `IHoverableControl.IsHovered` flag. Use these for side-effects (show a
detail panel, start an animation); use `IsHovered` for continuous visuals you read in `Render`.

```csharp
new Label().SetText("Hover me")
    .SetOnHoverEnterCommand(vm.ShowDetailsCommand)
    .BindOnHoverExitCommand(() => vm.HideDetailsCommand);
```

- `SetOnHoverEnterCommand(ICommand?)` / `BindOnHoverEnterCommand(...)`
- `SetOnHoverExitCommand(ICommand?)` / `BindOnHoverExitCommand(...)`
- Fired by `InputService` on the hover transition (same point as `IsHovered`/tooltips).
- Prefer `Bind*` from the VM (the command is owned by the VM) over an inline closure.
- **Not** fired on touch-only platforms (no pointer). The exit command is **not** fired on
  page-navigation teardown — a page change releases hover state without invoking it.

### Mouse cursor

The cursor shape an element shows while hovered. Driven by the same hover hit-test; when the
pointer leaves the element (or the page changes) the cursor resets to `Default`.

```csharp
new Button().SetText("Open").SetCursor(CursorType.Hand);
new ResizeHandle().SetCursor(CursorType.ResizeHorizontal);   // e.g. on a splitter
new Button().SetText("Brand").SetCursor(CursorType.PlusUi);  // self-drawn PlusUi logo cursor
```

- `SetCursor(CursorType)` / `BindCursor(...)` on any element.
- **All `CursorType` values** (every one renders on desktop — see resolution below):

  | Value | Shape |
  |---|---|
  | `Default` / `Arrow` | OS arrow |
  | `PlusUi` | the PlusUi brand mark (chevron+plus), hotspot at the chevron tip |
  | `Hand` | pointing hand (clickable) |
  | `Text` | I-beam (editable text) |
  | `Crosshair` | crosshair |
  | `Wait` | busy clock/spinner |
  | `Progress` | arrow + small busy badge (working in background) |
  | `NotAllowed` | circle with a slash |
  | `ResizeHorizontal` / `ResizeVertical` | ↔ / ↕ |
  | `ResizeAll` | 4-way move |
  | `ResizeNwse` / `ResizeNesw` | ↘↖ / ↙↗ diagonal resize |

- Backed by `IPlatformCursorService` — implemented on **desktop** (Silk.NET). Touch platforms
  have no implementation, so cursor requests are silently ignored.

**Resolution chain (desktop).** Each cursor resolves through a provider chain, first match wins:
1. **GLFW standard cursor** — loaded from the native OS cursor theme (used for the common
   shapes the OS provides: arrow, hand, text, crosshair, the basic resizes).
2. **Self-drawn backstop** — a Skia-rendered bitmap turned into a custom cursor. Guarantees
   **every** `CursorType` renders, even ones the OS/GLFW lacks (`Wait`/`Progress` on some
   backends, the diagonal/all resizes, and the brand `PlusUi` cursor which has no OS equivalent).

So you can use any value freely — there's no "unsupported, shows as arrow" gap. (`CursorType.PlusUi`
always goes straight to the self-drawn provider.)

## Global input bus (`IGlobalInputService`) — raw keyboard & pointer, no focus/hit-test

A framework-wide, subscribable stream of **raw** pointer and keyboard events, decoupled from
hit-testing and focus. Inject it anywhere (typically a ViewModel) to read input globally —
independent of which control is under the pointer or focused. This is the input counterpart to
the per-frame `GameCanvas` (see [display.md](display.md#gamecanvas)): a game draws in a
`GameCanvas` and reads input from here.

All input-bus types (`IGlobalInputService`, `PointerInputEvent`, `ScrollInputEvent`,
`KeyInputEvent`, `KeyModifiers`, `PlusKey`, `PointerButton`) live in the root namespace —
a single `using PlusUi.core;` covers everything, including in ViewModel assemblies.

Registered automatically as a **singleton** — just take `IGlobalInputService` in the constructor:

```csharp
public interface IGlobalInputService
{
    event Action<PointerInputEvent>? PointerMoved;   // Button = None
    event Action<PointerInputEvent>? PointerDown;
    event Action<PointerInputEvent>? PointerUp;
    event Action<ScrollInputEvent>?  Scrolled;
    event Action<KeyInputEvent>?     KeyDown;        // full PlusKey set
    event Action<KeyInputEvent>?     KeyUp;
}
```

### Event payloads (readonly structs)

| Struct | Members | Notes |
|---|---|---|
| `PointerInputEvent` | `Point Position`, `PointerButton Button`, `KeyModifiers Modifiers` | `Position` in **page coordinates** (same space as hit-testing). |
| `ScrollInputEvent` | `Point Position`, `float DeltaX`, `float DeltaY`, `KeyModifiers Modifiers` | Wheel/scroll deltas. |
| `KeyInputEvent` | `PlusKey Key`, `KeyModifiers Modifiers`, `bool IsRepeat` | Full unfiltered key set. |

- `PointerButton`: `None` (moves), `Left`, `Right`, `Middle`.
- `KeyModifiers` (`[Flags]`): `None`, `Shift`, `Ctrl`, `Alt`.

### The full `PlusKey` set

`KeyDown`/`KeyUp` carry the **full** key set — unlike the UI-focused path (Entry/focus
navigation), which only sees navigation/editing keys:

- Navigation/editing: `Backspace`, `Enter`, `Tab`, `Space`, `ShiftTab`, `Escape`,
  `ArrowUp/Down/Left/Right`, `Delete`, `Home`, `End`
- Letters `A`–`Z` (ASCII-aligned values), digits `D0`–`D9` (top row **and** numpad map here)
- Function keys `F1`–`F12`
- Modifiers as keys: `LeftShift`, `RightShift`, `LeftCtrl`, `RightCtrl`, `LeftAlt`, `RightAlt`
- `Unknown` for unmapped keys (not raised on the global bus)

### Pattern: WASD game input in a ViewModel

Track pressed keys in a set; read the set from the game loop. **Unsubscribe in `Dispose`** —
the service is an app-lifetime singleton, subscriptions keep the VM alive.

```csharp
public sealed partial class GameViewModel : ObservableObject, IDisposable
{
    private readonly IGlobalInputService _input;
    private readonly HashSet<PlusKey> _pressed = [];

    public GameViewModel(IGlobalInputService input)
    {
        _input = input;
        _input.KeyDown += OnKeyDown;
        _input.KeyUp += OnKeyUp;
        _input.PointerDown += OnPointerDown;
    }

    private void OnKeyDown(KeyInputEvent e) => _pressed.Add(e.Key);
    private void OnKeyUp(KeyInputEvent e) => _pressed.Remove(e.Key);
    private void OnPointerDown(PointerInputEvent e)
    { if (e.Button == PointerButton.Left) { /* fire */ } }

    public void Step(float dt)   // called from the GameCanvas draw loop
    {
        var dx = (_pressed.Contains(PlusKey.D) ? 1f : 0f)
               - (_pressed.Contains(PlusKey.A) ? 1f : 0f);
        // ...
    }

    public void Dispose()
    {
        _input.KeyDown -= OnKeyDown;
        _input.KeyUp -= OnKeyUp;
        _input.PointerDown -= OnPointerDown;
    }
}
```

### Global input gotchas

- **Additive, never exclusive.** Events are raised *in addition to* the normal UI pipeline —
  subscribing never suppresses clicks, focus, gestures, or text input. You cannot "handle"
  an event to stop the UI from seeing it.
- **Handlers run synchronously on the render/input thread.** Keep them fast; never throw.
  Do heavy work elsewhere.
- **Unsubscribe on teardown.** Singleton service ⇒ a forgotten `-=` leaks the subscriber
  (and a navigated-away page keeps reacting to input).
- **`IsRepeat` is currently always `false`** — track held keys yourself with a
  `HashSet<PlusKey>` (down = add, up = remove) instead of relying on auto-repeat.
- **`Modifiers` only reports `Shift` and `Ctrl` today.** The `Alt` flag exists but is not
  stamped; `LeftAlt`/`RightAlt` do arrive as `KeyDown`/`KeyUp` events, so track Alt yourself
  if needed.
- **Right-click arrives as a paired `PointerDown`+`PointerUp`** (platforms deliver it as one
  event). Real down/up tracking exists only for the **left** button; `Middle` is defined but
  not currently raised.
- **`PointerMoved` fires with `Button = None`** — it does not tell you whether a button is
  held. Track that from `PointerDown`/`PointerUp`.
- **Don't use this for normal UI.** Buttons, gestures, focus, text input all have their own
  APIs (gesture detectors, `Bind*Command`, `IKeyboardInputHandler`). The global bus is for
  games, canvases, shortcuts and diagnostics — not a replacement for the control pipeline.

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

A thin vertical handle the user drags to resize a neighbour. It implements `IDraggableControl`
for the drag stream and `IHoverableControl` for hover feedback, and reports each horizontal
delta through a callback.

> **Authoring custom controls OUTSIDE `PlusUi.core` (the common case).** Several base members
> are `protected internal` / `internal`, so they are **not** reachable from your own assembly —
> code that compiles inside the PlusUi repo will fail in a consumer project:
> - `IsFocusable` must be overridden as **`protected override`**, not `protected internal override`
>   (`internal` doesn't cross assembly boundaries).
> - **`VisualOffset` and the `HorizontalAlignment`/`VerticalAlignment` *property setters* are
>   not accessible.** Use the public fluent setters (`SetVerticalAlignment(...)`) instead of
>   assigning the property, and use the **public** `Position` / `ElementSize` (not `VisualOffset`)
>   when rendering.
>
> The cleanest fix is to **inherit a concrete control** (e.g. `Solid`) instead of raw `UiElement`:
> you inherit its rendering, `IsFocusable`, and `AccessibilityRole`, and never touch an internal
> member. That's the approach below.

```csharp
using PlusUi.core;
using PlusUi.core.Attributes;
using SkiaSharp;

[GenerateShadowMethods]
public partial class ResizeHandle : Solid, IDraggableControl, IHoverableControl
{
    // Framework-driven state — we only read these while rendering.
    public bool IsDragging { get; set; }
    public bool IsHovered { get; set; }

    private Action<float>? _onResize;
    private readonly SKColor _activeColor = new(90, 120, 200);

    // Solid ctor: (width, height, color). Fixed 6px width; null height = stretch to fill.
    public ResizeHandle() : base(6f, null, new Color(45, 45, 45)) { }

    public ResizeHandle SetOnResize(Action<float> onResize) { _onResize = onResize; return this; }

    public void HandleDrag(float deltaX, float deltaY) => _onResize?.Invoke(deltaX);

    public override void Render(SKCanvas canvas)
    {
        base.Render(canvas); // Solid draws the base strip

        if (IsDragging || IsHovered)   // accent grip line when active
        {
            using var grip = new SKPaint { Color = _activeColor, IsAntialias = true };
            // Use public Position/ElementSize — VisualOffset is internal to PlusUi.core.
            canvas.DrawRect(Position.X + ElementSize.Width / 2f - 1f, Position.Y,
                            2f, ElementSize.Height, grip);
        }
    }
}
```

> A raw `UiElement` subclass also works **inside `PlusUi.core`** (that's how `Slider`/`Separator`
> are written, with `protected internal override bool IsFocusable`, `MeasureInternal`, and
> `Position + VisualOffset` in `Render`). Use that form only for controls living in the framework
> assembly itself.

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

> This example lives directly in a **page** (`vm` is the page ViewModel), where the
> `PropertyChanged` → re-measure chain works automatically. If you extract the splitter layout
> into a `UserControl` with its own ViewModel, pass that VM to the base constructor
> (`: base(vm)`) — otherwise the drag updates the VM but the bound column never re-measures.
> See "UserControl with its own ViewModel" in [usercontrol.md](usercontrol.md).

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
- **Hover commands ≠ `IsHovered`.** `SetOnHoverEnterCommand`/`SetOnHoverExitCommand` are one-shot
  side-effects on any element; `IHoverableControl.IsHovered` is a live flag you read in `Render`.
  Don't implement `IHoverableControl` just to fire a command — use the hover commands.
- **`SetCursor` is desktop-only.** It needs `IPlatformCursorService` (Silk.NET on desktop); on
  touch platforms it's a no-op. Don't rely on the cursor for essential affordances.
- **Focused-key handling ≠ global keys.** `IKeyboardInputHandler` sees keys only while the
  control has focus and only the navigation/editing subset; `IGlobalInputService.KeyDown/KeyUp`
  sees **every** key regardless of focus. Games should use the global bus, not focus tricks.
