# UserControl & Composition

Reusable, composable custom controls in PlusUi. A `UserControl` defines its UI by *composing* existing elements inside a `Build()` method, rather than rendering directly. Use it to package a chunk of UI (a card, an avatar, a labeled field) into a single reusable type with its own configurable properties.

> For custom SkiaSharp canvas drawing, use `RawUserControl` instead — `UserControl` is for composing built-in elements, not for low-level rendering.

## UserControl

**Purpose:** Create reusable, composable custom controls by defining UI through composition of other elements in a `Build` method.

**Base class:** `UiElement<UserControl>` (abstract). Marked `[GenerateGenericWrapper]`.

### Members

| Member | Type | Bind* | Notes |
|---|---|---|---|
| `Build()` | `protected abstract UiElement` | no | Override to return the composed UI tree. Called once during `BuildContent()`; result is cached in `_content`. |
| `IsFocusable` | `bool` (override) | no | Hardcoded `false` — cannot be overridden. Focus flows through child elements. |
| `AccessibilityRole` | `AccessibilityRole` (override) | no | Hardcoded `AccessibilityRole.Container` — cannot be overridden. |

`UserControl` exposes **no `Set*`/`Bind*` properties of its own** for configuration. You add your own plain CLR properties in the subclass; the consumer sets them via object initializer or direct assignment. Inherited fluent methods (`SetMargin`, `SetBackground`, `SetCornerRadius`, alignment, etc.) apply to the *container*, not automatically to children inside `Build()`.

### Examples

Primary-constructor card composing a `Border` + `VStack`:

```csharp
public class StatCard(string title, string value) : UserControl
{
    protected override UiElement Build() =>
        new Border()
            .SetBackground(PlusUiDefaults.BackgroundControl)
            .SetCornerRadius(8)
            .AddChild(new VStack()
                .SetSpacing(4)
                .SetMargin(new Margin(16))
                .AddChild(new Label().SetText(value).SetTextSize(24).SetFontWeight(FontWeight.Bold))
                .AddChild(new Label().SetText(title).SetTextColor(PlusUiDefaults.TextSecondary).SetTextSize(12)));
}

// Usage
new StatCard("Revenue", "$12.4k")
```

Configurable public properties driving `Build()`:

```csharp
public class UserCard : UserControl
{
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";

    protected override UiElement Build() =>
        new VStack(
            new Label().SetText(Name).SetFontWeight(FontWeight.Bold).SetTextSize(18),
            new Label().SetText(Email).SetTextSize(14).SetTextColor(Colors.Gray).SetMargin(new Margin(0, 4, 0, 0))
        ).SetMargin(new Margin(12));
}

// Usage
new UserCard { Name = "John Doe", Email = "john@example.com" }
```

Parameterized via properties with defaults:

```csharp
public class Avatar : UserControl
{
    public string ImageUrl { get; set; } = "";
    public float Size { get; set; } = 48;

    protected override UiElement Build() =>
        new Image()
            .SetImageSource(ImageUrl)
            .SetDesiredSize(new Size(Size, Size))
            .SetAspect(Aspect.AspectFill)
            .SetCornerRadius(Size / 2);
}

// Usage
new Avatar { ImageUrl = "user.jpg", Size = 64 }
```

### Patterns

- Define configurable public CLR properties on the subclass; they drive the `Build()` output and make the control reusable with different data.
- Use **primary constructors** for required initialization: `public class MyCard(string title) : UserControl { }`.
- Return a **single** composed `UiElement` tree from `Build()` — typically a layout (`VStack`, `HStack`, `Border`) wrapping the children. There is no `AddChild()` on `UserControl` itself; wrap multiple top-level children in a stack.
- Handle missing/null data by returning `new NullElement()` to avoid layout issues.
- Combine with `TapGestureDetector` (or other gesture detectors) inside `Build()` to make interactive components.
- Prefer composing small, focused controls over deep inheritance hierarchies.
- Data binding flows automatically: `UpdateBindings()` on the `UserControl` forwards to the composed content (`_content.UpdateBindings(...)`).

### Gotchas

- **Abstract** — you must subclass; you cannot instantiate `UserControl` directly.
- **`Build()` runs once.** It is called during `BuildContent()` and cached in `_content`; it is **not** re-invoked when your properties change. If a property that affects layout changes after construction, call `InvalidateMeasure()` (or recreate the control) to refresh.
- `UserControl` has **no built-in `Set*`/`Bind*` configuration properties** (no `SetTitle`, etc.). All configuration is via the public properties you define in the subclass.
- `IsFocusable` (`false`) and `AccessibilityRole` (`Container`) are **hardcoded and not overridable** — focus and accessibility flow through the child elements you compose.
- Inherited fluent methods (`SetMargin`, `SetBackground`, `SetCornerRadius`, …) style the `UserControl` container, **not** the children — apply child styling inside `Build()`.
- Not a generic container: there is **no `AddChild()`** on `UserControl`. Return one tree; wrap multiple children in `VStack`/`HStack`.
- Do **not** confuse with `RawUserControl` (SkiaSharp canvas rendering). `UserControl` composes built-in elements only.

## Common LLM mistakes

- Calling `new UserControl()` directly — it is abstract; always subclass.
- Adding fluent `SetX`/`BindX` methods to a `UserControl` subclass for config — use plain auto-properties instead; the consumer sets them via initializer.
- Expecting `Build()` to re-run on property change — it does not; trigger `InvalidateMeasure()` or recreate.
- Calling `.AddChild(...)` on the `UserControl` — wrap children in a layout returned from `Build()`.
- Returning multiple elements from `Build()` — it returns exactly one `UiElement`.
- Applying `SetMargin`/`SetBackground` to the control expecting it to style inner children — it styles the container only.
