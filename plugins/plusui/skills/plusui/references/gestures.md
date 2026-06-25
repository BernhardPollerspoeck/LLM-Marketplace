# PlusUi Gesture Detectors Reference

Gesture detectors are **wrapper** controls: they take a single `UiElement` as a constructor argument, measure to the size of that content, and fire an `ICommand` when their specific gesture is detected. They are not decorators applied with `AddChild` — the content is always the constructor parameter.

All detectors inherit from `GestureDetector<T> : UiLayoutElement`, so they share the standard layout/visual properties (`Margin`, alignment, `Background`, `DesiredSize`, `Opacity`) plus `Bind*` counterparts for every `Set*`.

| Detector | Constructor | Command property | Parameter sent to `Execute` |
|----------|-------------|------------------|-----------------------------|
| `TapGestureDetector` | `new TapGestureDetector(content)` | `Command` | `CommandParameter` (optional `object`, default `null`) |
| `DoubleTapGestureDetector` | `new DoubleTapGestureDetector(content)` | `DoubleTapCommand` | `null` |
| `LongPressGestureDetector` | `new LongPressGestureDetector(content)` | `LongPressCommand` | `null` |
| `SwipeGestureDetector` | `new SwipeGestureDetector(content)` | `SwipeCommand` | `SwipeDirection` (the detected direction) |
| `PinchGestureDetector` | `new PinchGestureDetector(content)` | `PinchCommand` | `float` (scale factor) |

> Every detector checks `Command?.CanExecute(parameter)` before calling `Execute(parameter)`, so command availability is respected.
> `TapGestureDetector` is focusable and reports `AccessibilityRole.Button`.

---

## Shared properties

These are inherited from `UiLayoutElement` / `UiElement` and available on **all** detectors. Each `Set*` has a matching `Bind*`.

| Property | Type | Bind* | Default | Purpose |
|----------|------|:-----:|---------|---------|
| `Margin` | `Margin` | yes | `0` | Spacing around the detector |
| `HorizontalAlignment` | `HorizontalAlignment` | yes | `Stretch` | Horizontal placement within parent |
| `VerticalAlignment` | `VerticalAlignment` | yes | `Stretch` | Vertical placement within parent |
| `Background` | `Background` | yes | `null` | Fill for the detector area |
| `DesiredSize` | `Size` | yes | `null` (measured from content) | Explicit size override |
| `Opacity` | `double` | yes | `1.0` | Opacity (0–1) of detector and content |

---

## TapGestureDetector

Detects a single tap/click on the wrapped content. The only detector with a configurable command parameter.

| Property | Type | Bind* | Default | Purpose |
|----------|------|:-----:|---------|---------|
| `Command` | `ICommand` | yes | `null` | Executed on tap |
| `CommandParameter` | `object` | yes | `null` | Passed to `Command.Execute(...)` |

```csharp
new TapGestureDetector(
    new Label().SetText("Tap me!"))
    .SetCommand(vm.TapCommand);

// With a parameter
new TapGestureDetector(
    new Label().SetText("Select item"))
    .SetCommand(vm.SelectCommand)
    .SetCommandParameter(item.Id);
```

---

## DoubleTapGestureDetector

Detects two quick taps. Sends `null` to the command (no parameter).

| Property | Type | Bind* | Default | Purpose |
|----------|------|:-----:|---------|---------|
| `DoubleTapCommand` | `ICommand` | yes (`BindCommand`) | `null` | Executed on double tap |

```csharp
new DoubleTapGestureDetector(
    new Image().SetImageSource("photo.jpg"))
    .SetCommand(vm.ZoomCommand);
```

---

## LongPressGestureDetector

Detects a press-and-hold. Sends `null` to the command (no parameter).

| Property | Type | Bind* | Default | Purpose |
|----------|------|:-----:|---------|---------|
| `LongPressCommand` | `ICommand` | yes (`BindCommand`) | `null` | Executed when hold threshold is met |

```csharp
new LongPressGestureDetector(
    new Label().SetText("Hold for options"))
    .SetCommand(vm.ShowContextMenuCommand);
```

---

## SwipeGestureDetector

Detects directional swipes. Passes the detected `SwipeDirection` to the command, and can be filtered to only fire for chosen directions.

| Property | Type | Bind* | Default | Purpose |
|----------|------|:-----:|---------|---------|
| `SwipeCommand` | `ICommand` | yes (`BindCommand`) | `null` | Executed on swipe; receives `SwipeDirection` |
| `AllowedDirections` | `SwipeDirection` | yes | `SwipeDirection.All` | Which directions to detect |

`SwipeDirection` is a `[Flags]` enum: `None`, `Left`, `Right`, `Up`, `Down`, plus the combinations `Horizontal` (`Left | Right`), `Vertical` (`Up | Down`) and `All`. Combine with bitwise OR.

```csharp
new SwipeGestureDetector(
    new Border()
        .SetBackground(new SolidColorBackground(Colors.Blue))
        .SetDesiredSize(new Size(200, 100)))
    .SetCommand(vm.SwipeCommand)
    .SetAllowedDirections(SwipeDirection.Left | SwipeDirection.Right);
    // or: .SetAllowedDirections(SwipeDirection.Horizontal);
```

```csharp
// ViewModel — the command receives the direction
[RelayCommand]
private void Swipe(SwipeDirection direction)
{
    if (direction == SwipeDirection.Left) GoToNext();
    else if (direction == SwipeDirection.Right) GoToPrevious();
}
```

---

## PinchGestureDetector

Detects pinch-to-zoom. Passes a `float` scale factor to the command.

| Property | Type | Bind* | Default | Purpose |
|----------|------|:-----:|---------|---------|
| `PinchCommand` | `ICommand` | yes (`BindCommand`) | `null` | Executed on pinch; receives `float` scale |

```csharp
new PinchGestureDetector(
    new Image().SetImageSource("map.png"))
    .SetCommand(vm.PinchCommand);
```

```csharp
[RelayCommand]
private void Pinch(float scale) => ZoomLevel *= scale;
```

---

## Composing gestures

Nest detectors to recognise multiple gestures on the same content. Each detector wraps the next, and each fires its own command independently.

```csharp
new DoubleTapGestureDetector(
    new TapGestureDetector(
        new Image().SetImageSource("photo.jpg"))
        .SetCommand(vm.SelectCommand))
    .SetCommand(vm.ZoomCommand);
```

---

## Gotchas / Common LLM mistakes

- **Content is a constructor argument, not `AddChild`.** Use `new TapGestureDetector(label)`, never `new TapGestureDetector().AddChild(label)`.
- **Parameters vary by gesture.** Only `TapGestureDetector` has `SetCommandParameter`/`BindCommandParameter`. `Swipe` sends a `SwipeDirection`, `Pinch` sends a `float` scale, and `DoubleTap`/`LongPress` send `null`. Do not assume every gesture behaves like Tap.
- **Command must be an `ICommand`, not an `Action`/lambda.** Use `RelayCommand` (MVVM Community Toolkit) or another `System.Windows.Input.ICommand`.
- **`SwipeDirection` is a flag enum.** Combine directions with bitwise OR (`Left | Right`), not commas or addition. Convenience values `Horizontal`, `Vertical`, and `All` already cover common cases.
- **Default `AllowedDirections` is `All`.** A `SwipeGestureDetector` fires for every direction unless you restrict it with `SetAllowedDirections`.
- **Nested detectors fire independently.** A long-press on content wrapped by both `LongPress` and `Tap` triggers only the matching gesture's command — the hold does not also trigger the tap handler.
- **No automatic visual feedback.** Detectors render content as-is. For press/hover styling use a `Button`, or apply visual changes inside the command.
- **Hit testing is bounded by the content.** Gestures only fire inside the wrapped content's bounds; the empty `Margin` area does not respond.
- **`CanExecute` is honoured.** If the bound command's `CanExecute` returns `false`, the gesture is ignored — make sure the command can execute.
