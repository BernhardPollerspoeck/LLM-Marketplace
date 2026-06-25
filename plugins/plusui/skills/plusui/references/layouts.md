# PlusUi Layout & Container Controls Reference

Containers that arrange or wrap other elements. All derive from `UiLayoutElement` (multi-child or single-child) except `Separator` and `Scrollbar`, which are plain `UiElement` visuals. Layout controls are containers, not focusable controls (`IsFocusable == false`).

Every control uses the fluent builder API: construct, then chain `Set*` methods. Most visual/layout properties also expose a `Bind*` counterpart that wires the property to a ViewModel via an expression (see [binding-and-state.md](binding-and-state.md)). The tables below mark which properties have a `Bind*` form.

## Table of contents

- [Conventions](#conventions)
- [VStack — vertical stack](#vstack--vertical-stack)
- [HStack — horizontal stack](#hstack--horizontal-stack)
- [Grid — rows & columns](#grid--rows--columns)
- [UniformGrid — equal-size cells](#uniformgrid--equal-size-cells)
- [ScrollView — scrollable wrapper](#scrollview--scrollable-wrapper)
- [Scrollbar — standalone scroll control](#scrollbar--standalone-scroll-control)
- [Border — stroked single-child container](#border--stroked-single-child-container)
- [Separator — divider line](#separator--divider-line)
- [Common LLM mistakes](#common-llm-mistakes)

---

## Conventions

- **Children**: multi-child containers (`VStack`, `HStack`, `Grid`, `UniformGrid`) use `AddChild(...)`, or pass children to the `params UiElement[]` constructor. Only `UniformGrid` adds a bulk `AddChildren(params UiElement[])` method. Single-child containers (`Border`, `ScrollView`) hold exactly one element.
- **Spacing vs Margin**: `Spacing` adds gaps *between* children only (never before the first or after the last). `Margin` is outer spacing around the whole container; on a child it adds spacing for that child. Use `Margin` on children for padding-like inset.
- **Defaults from `PlusUiDefaults`**: `Spacing`, `Wrap`, alignments, stroke, and colors default from `PlusUiDefaults`, not zero/false. Set explicitly if you need a known value.
- **`Stretch` alignment** on a child fills the container's cross dimension. On stacks, multiple `Stretch` children share remaining space equally.

---

## VStack — vertical stack

Arranges children top-to-bottom with optional spacing. With `Wrap` it flows children into multiple **columns** when height is exceeded.

| Property | Type | Bind* | Notes |
|---|---|---|---|
| `Spacing` | `float` | yes | Gap between children. Default `PlusUiDefaults.StackSpacing`. |
| `Wrap` | `bool` | yes | Wrap into new column when height exceeded. Requires `SetDesiredHeight`. Default `PlusUiDefaults.StackWrap`. |
| `Margin` | `Margin` | yes | Outer spacing (inherited). |
| `HorizontalAlignment` | `HorizontalAlignment` | yes | Stack position in parent; child `Stretch` fills stack width. |
| `VerticalAlignment` | `VerticalAlignment` | yes | Stack position in parent (no effect on child order). |
| `DesiredSize` | `Size` | yes | Explicit width/height override (inherited). |
| `Background` | `IBackground` / `Color` | yes | Background fill/gradient (inherited). |
| `CornerRadius` | `float` | yes | Rounded background (inherited). |
| `FocusScope` | `FocusScopeMode` | yes | `None` / `Trap` / `TrapWithEscape` (inherited). |
| `AccessibilityLandmark` | `AccessibilityLandmark` | yes | ARIA landmark (inherited). |
| `Children` | `List<UiElement>` | no | Read via property; mutate with `AddChild`/`RemoveChild`/`ClearChildren`. |

```csharp
// Inline via constructor
new VStack(
    new Label().SetText("Title"),
    new Label().SetText("Subtitle"),
    new Button().SetText("Action")
);

// Fluent, with spacing
new VStack()
    .SetSpacing(8)
    .AddChild(new Label().SetText("First"))
    .AddChild(new Label().SetText("Second"))
    .AddChild(new Label().SetText("Third"));

// Wrapping into columns (height constraint required)
new VStack()
    .SetWrap(true)
    .SetSpacing(8)
    .SetDesiredHeight(150)
    .AddChild(new Label().SetText("1"))
    .AddChild(new Label().SetText("2"))
    .AddChild(new Label().SetText("3"))
    .AddChild(new Label().SetText("4"))
    .AddChild(new Label().SetText("5"))
    .AddChild(new Label().SetText("6"));
```

**Gotchas**
- `Wrap` does nothing without `SetDesiredHeight` to constrain height.
- `Wrap` flows into **columns** (vertical items overflowing left-to-right) — not the row wrapping of `HStack`.
- `VerticalAlignment.Stretch` on children divides remaining space equally among the stretching children only.
- `HorizontalAlignment.Stretch` on children resolves during Arrange, not Measure — don't rely on resolved width during the measure phase.
- The constructor signature is `params UiElement[]`; you cannot pass `Set*` config there. Configure with fluent methods after construction.

---

## HStack — horizontal stack

Arranges children left-to-right with optional spacing. With `Wrap` it flows children into new **rows** (like a WrapPanel) when width is exceeded.

| Property | Type | Bind* | Notes |
|---|---|---|---|
| `Spacing` | `float` | yes | Gap between children. Default `PlusUiDefaults.StackSpacing`. |
| `Wrap` | `bool` | yes | Wrap to next row when width exceeded. Default `PlusUiDefaults.StackWrap`. |
| `HorizontalAlignment` | `HorizontalAlignment` | yes | Stack position; child `Stretch` divides remaining width equally. |
| `VerticalAlignment` | `VerticalAlignment` | yes | Stack position; child `Stretch` resolves to row height, not full height. |
| `Margin` | `Margin` | yes | Outer spacing. |
| `Background` | `IBackground` | yes | Background fill. Default transparent. |
| `CornerRadius` | `float` | yes | Rounded corners. Default `0`. |
| `Opacity` | `float` | yes | `0.0`–`1.0`. Default `1.0`. |
| `IsVisible` | `bool` | yes | Whole-stack visibility. Default `true`. |
| `DesiredSize` | `Size` | yes | Explicit width/height override (inherited). `SetDesiredWidth`/`SetDesiredHeight` are helper methods that modify it; pair with `Wrap` to force wrapping. |
| `Children` | `List<UiElement>` | no | Mutate via `AddChild`/`RemoveChild`/`ClearChildren`. |

```csharp
// Label + input row
new HStack(
    new Label().SetText("Name:"),
    new Entry().SetPlaceholder("Enter name")
);

// Button bar
new HStack()
    .SetSpacing(8)
    .AddChild(new Button().SetText("Cancel"))
    .AddChild(new Button().SetText("Save"));

// Wrapping tag row (width constraint forces wrap)
new HStack()
    .SetWrap(true)
    .SetSpacing(8)
    .SetDesiredWidth(260)
    .AddChild(new Label().SetText("Tag1"))
    .AddChild(new Label().SetText("Tag2"))
    .AddChild(new Label().SetText("Tag3"));
```

**Gotchas**
- `Wrap` is **off** by default; without `SetWrap(true)` overflowing children clip/overflow rather than wrap.
- `Wrap` flows **downward to new rows**; each row's height is its tallest child.
- Child `HorizontalAlignment.Stretch` splits remaining width equally among the `Stretch` children (two `Stretch` → 50% each).
- Child `VerticalAlignment.Stretch` resolves to the **row height** (tallest non-stretching child), not the full available height — keeps vertical separators row-sized.
- `Margin` on the `HStack` affects the outer bounds; use `SetSpacing` for inter-child gaps.

---

## Grid — rows & columns

Arranges children in explicit rows/columns with absolute, auto (content-sized), or star (proportional) sizing. Children are placed by 0-based `row`/`column` and may span cells.

| Property | Type | Bind* | Notes |
|---|---|---|---|
| `Margin` | `Margin` | yes | Outer spacing; reduces layout area. Default `Margin(0)`. |
| `HorizontalAlignment` | `HorizontalAlignment` | yes | Default `Stretch`. |
| `VerticalAlignment` | `VerticalAlignment` | yes | Default `Stretch`. |
| `Background` | `IBackground` | yes | Background fill. |
| `CornerRadius` | `float` | yes | Default `0`. |
| `Opacity` | `float` | yes | Default `1.0`. |
| `IsVisible` | `bool` | yes | Default `true`. |
| `DesiredSize` | `Size` | yes | Explicit size constraint. |
| `FocusScope` | `FocusScopeMode` | yes | Default `None`. |
| `AccessibilityLandmark` | `AccessibilityLandmark` | yes | Default `None`. |
| `Children` | `List<UiElement>` | no | Add via `AddChild(element, row, column, rowSpan, columnSpan)`. Rows/columns are defined via `AddRow`/`AddColumn` (+ `AddBoundRow`/`AddBoundColumn`). |
| `Context` | `INotifyPropertyChanged?` | no | Propagated to bound rows/columns automatically. |

Row/column sizing modes: `Row.Absolute`/`Column.Absolute` (fixed px), `Row.Auto`/`Column.Auto` (content-sized), `Row.Star`/`Column.Star` (weight). Star is a **weight, not a percentage**: `Column.Star, 2` gets twice the space of `Column.Star, 1`.

**Runtime-sized tracks (`AddBoundColumn`/`AddBoundRow`).** For a track whose size changes at runtime (e.g. a drag-resizable sidebar), bind it. These take a `nameof` property name **+ a `Func<float>` getter** — not an `Expression` like other `Bind*` methods — and re-measure when that property raises `PropertyChanged`:

```csharp
new Grid()
    .AddBoundColumn(nameof(vm.SidebarWidth), Column.Absolute, () => vm.SidebarWidth)
    .AddColumn(Column.Star, 1)
    .AddRow(Row.Star, 1)
    .AddChild(BuildSidebar(), row: 0, column: 0)
    .AddChild(BuildContent(), row: 0, column: 1);
```

For the drag-to-resize handle that updates `SidebarWidth`, see [input-and-dragging.md](input-and-dragging.md).

```csharp
// Header row (fixed) + auto row, two proportional columns
new Grid()
    .AddRow(Row.Absolute, 50)
    .AddRow(Row.Auto)
    .AddColumn(Column.Star, 1)
    .AddColumn(Column.Star, 2)
    .AddChild(new Label().SetText("Top Left"), row: 0, column: 0)
    .AddChild(new Button().SetText("Top Right"), row: 0, column: 1);

// Two-column form: labels auto-sized, fields fill remaining width
new Grid()
    .AddColumn(Column.Auto)
    .AddColumn(Column.Star, 1)
    .AddRow(Row.Auto)
    .AddRow(Row.Auto)
    .AddChild(new Label().SetText("Name:"),  row: 0, column: 0)
    .AddChild(new Entry(),                    row: 0, column: 1)
    .AddChild(new Label().SetText("Email:"), row: 1, column: 0)
    .AddChild(new Entry(),                    row: 1, column: 1);

// Spanning + responsive star sizing
new Grid()
    .AddColumn(Column.Star, 1)
    .AddColumn(Column.Star, 2)
    .AddRow(Row.Auto)
    .AddChild(new Label().SetText("wide"), row: 0, column: 0, columnSpan: 2);
```

**Gotchas**
- Defaults to `Stretch` on both axes — set alignment explicitly if you don't want the grid to fill its parent.
- If you define no rows (or no columns), the grid auto-adds a single `Row.Auto` / `Column.Auto`. Easy to get surprising layouts by forgetting definitions.
- `AddChild` without `row`/`column` defaults to `(0, 0)`.
- Star is proportional weights — use `1`/`2`/`3`, not `0.5`/`1.0`.
- `rowSpan`/`columnSpan` are silently clamped to grid bounds.
- Define rows/columns first, then add children; re-chain to the grid rather than expecting `AddRow` to follow `AddChild` meaningfully.
- Use `SetDebug(true)` to visualize layout bounds with red borders during development.

---

## UniformGrid — equal-size cells

A grid where every cell is identical in size; children are placed left-to-right, top-to-bottom (row-major). Specify `Rows`, `Columns`, both, or neither (auto square-ish grid).

| Property | Type | Bind* | Notes |
|---|---|---|---|
| `Rows` | `int?` | yes | Row count; calculated from child count + `Columns` if null. |
| `Columns` | `int?` | yes | Column count; calculated from child count + `Rows` if null. |
| `DesiredSize` | `Size?` | yes | **Total** grid size; cell = total / grid dimensions. |
| `DesiredWidth` | `float?` | yes | Total grid width. |
| `DesiredHeight` | `float?` | yes | Total grid height. |
| `Margin` | `Margin` | yes | Outer spacing. Default `0`. |
| `HorizontalAlignment` | `HorizontalAlignment` | yes | Default `Stretch`. |
| `VerticalAlignment` | `VerticalAlignment` | yes | Default `Stretch`. |
| `Background` | `IBackground` | yes | Default none. |
| `CornerRadius` | `float` | yes | Default `0`. |

```csharp
// 4-column calculator keypad
new UniformGrid()
    .SetColumns(4)
    .SetDesiredSize(new Size(300, 400))
    .AddChildren(
        new Button().SetText("7"), new Button().SetText("8"), new Button().SetText("9"), new Button().SetText("/"),
        new Button().SetText("4"), new Button().SetText("5"), new Button().SetText("6"), new Button().SetText("*"),
        new Button().SetText("1"), new Button().SetText("2"), new Button().SetText("3"), new Button().SetText("-"),
        new Button().SetText("0"), new Button().SetText("."), new Button().SetText("="), new Button().SetText("+")
    );

// Bound dynamic cells
new UniformGrid()
    .SetRows(3)
    .SetColumns(3)
    .SetDesiredSize(new Size(300, 300))
    .SetBackground(new Color(40, 40, 40))
    .AddChildren(
        cells.Select(cell => new Button()
            .BindText(nameof(cell.Value), () => cell.Value)
            .SetCommand(cell.ClickCommand))
        .ToArray()
    );
```

**Gotchas**
- `SetDesiredSize` is the **total** grid size; individual cells are `total / gridDimensions`, not the cell size.
- With neither `Rows` nor `Columns` set, columns = `ceil(sqrt(childCount))` — square-ish, not guaranteed square.
- Hidden children (`IsVisible == false`) are skipped, so visible-child count drives auto dimensions.
- Per-cell child alignment is handled by each child within its cell; `UniformGrid` does not center children for you.
- `AddChildren` is the standard pattern; the `params` constructor exists but is less common.

---

## ScrollView — scrollable wrapper

Wraps a **single** element and scrolls content larger than the viewport, via mouse wheel and drag. Both directions enabled by default.

| Property | Type | Bind* | Notes |
|---|---|---|---|
| `CanScrollHorizontally` | `bool` | yes | Default `true`. |
| `CanScrollVertically` | `bool` | yes | Default `true`. |
| `ScrollFactor` | `float` | yes | Scroll-speed multiplier. Default `1.0`. |
| `HorizontalOffset` | `float` | yes | Current X position; auto-clamped. Default `0`. |
| `VerticalOffset` | `float` | yes | Current Y position; auto-clamped. Default `0`. |
| `DesiredHeight` | `float` | yes | Viewport height — set this to bound a vertical scroller. |
| `DesiredWidth` | `float` | yes | Viewport width — for horizontal scrolling. |
| `CornerRadius` | `float` | yes | Clips scrolled content to rounded corners. |
| `Background` | `IBackground` | yes | Viewport background. |
| `Margin` | `Margin` | yes | Outer spacing. |
| `HorizontalAlignment` | `HorizontalAlignment` | yes | Position in parent. |
| `VerticalAlignment` | `VerticalAlignment` | yes | Position in parent. |

```csharp
// Vertical list, fixed viewport height
new ScrollView(
    new VStack(
        Enumerable.Range(1, 100)
            .Select(i => new Label().SetText($"Item {i}"))
            .ToArray()
    )
)
.SetCanScrollHorizontally(false)
.SetDesiredHeight(200);

// Horizontal-only strip
new ScrollView(BuildColumns(30))
    .SetCanScrollVertically(false)
    .SetDesiredHeight(80);
```

**Gotchas**
- Accepts a **single** content element via the constructor — wrap multiple items in a `VStack`/`HStack` first. The content is not in `Children` (that list is empty for inspection); it is held internally.
- Both scroll directions are on by default — disable the unwanted one with `SetCanScrollHorizontally(false)` / `SetCanScrollVertically(false)`.
- Set `DesiredHeight`/`DesiredWidth` to define the viewport; without it the view expands to content instead of scrolling.
- Offsets clamp silently to valid bounds — scrolling past content is a no-op, not an error.
- `CornerRadius` clips content to the rounded rect; overflowing content is hidden.
- `AccessibilityRole` is fixed to `ScrollView`; default label is "Scrollable area" if unset.

---

## Scrollbar — standalone scroll control

A draggable scrollbar visual showing scroll position. Normally used internally by `ScrollView`/`ItemsList`; standalone use requires manual state management.

| Property | Type | Bind* | Notes |
|---|---|---|---|
| `Orientation` | `ScrollbarOrientation` | yes | `Vertical` (default) or `Horizontal`. |
| `Width` | `float` | yes | Thickness (perpendicular to scroll). Default `12`. |
| `MinThumbSize` | `float` | yes | Minimum thumb length px. Default `20`. |
| `ThumbColor` | `Color` | yes | Thumb normal color. |
| `ThumbHoverColor` | `Color` | yes | Thumb hover color. |
| `ThumbDragColor` | `Color` | yes | Thumb drag color. |
| `TrackColor` | `Color` | yes | Track background. Default semi-transparent. |
| `ThumbCornerRadius` | `float` | yes | Default `PlusUiDefaults.CornerRadius` (4). |
| `TrackCornerRadius` | `float` | yes | Default `PlusUiDefaults.CornerRadius` (4). |
| `AutoHide` | `bool` | yes | Fade out after inactivity. Default `false`. |
| `AutoHideDelay` | `int` | yes | ms before fade (only when `AutoHide`). Default `1000`. |
| `Value` | `float` | no | Read-only normalized position `0.0`–`1.0`. Set via `UpdateScrollState`/`HandleDrag`. |
| `ViewportRatio` | `float` | no | Read-only; viewport/content ratio (clamped 0.05–1.0). Drives thumb size. |
| `IsHovered` | `bool` | no | Read-only; set via `SetHovered`. |
| `IsDragging` | `bool` | no | Read-only; whether thumb is being dragged. |

```csharp
// Vertical scrollbar with custom colors
new Scrollbar()
    .SetOrientation(ScrollbarOrientation.Vertical)
    .SetWidth(10)
    .SetThumbColor(new Color(80, 80, 80))
    .SetThumbHoverColor(new Color(120, 120, 120))
    .SetTrackColor(new Color(30, 30, 30, 80))
    .SetThumbCornerRadius(4);

// Auto-hiding scrollbar with value callback
new Scrollbar()
    .SetAutoHide(true)
    .SetAutoHideDelay(2000)
    .SetOnValueChanged(offset => Console.WriteLine($"Scrolled to {offset}"));
```

**Gotchas**
- Not focusable (`IsFocusable == false`); skipped in tab order.
- `Value` is read-only externally — set it via `UpdateScrollState(...)` or `HandleDrag(...)`, never directly. Actual pixel offset is `Value * maxScrollOffset`.
- `Width` (thickness) and `MinThumbSize` (thumb length) are independent — don't confuse them.
- `SetHovered(bool)` must be driven by external mouse tracking; hover state is not auto-detected.
- The parent (`ScrollView`/`ItemsList`) must call `UpdateScrollState(...)` whenever content/viewport size changes, or thumb sizing breaks.
- Changing `Width` after layout requires `InvalidateMeasure()`.

---

## Border — stroked single-child container

Wraps exactly one child with a stroke (outline), optional background, and optional rounded corners. Ideal for cards and outlined panels.

| Property | Type | Bind* | Notes |
|---|---|---|---|
| `StrokeColor` | `Color` | yes | Outline color. |
| `StrokeThickness` | `float` | yes | Stroke width px; clamped `>= 0`. Default `PlusUiDefaults.StrokeThickness` (~1). |
| `StrokeType` | `StrokeType` | yes | `Solid` / `Dashed` (4px/2px) / `Dotted` (1px/1px). Default `PlusUiDefaults.StrokeType` (Solid). |
| `Background` | `IBackground` | yes | Inner fill (inherited). |
| `CornerRadius` | `float` | yes | Rounds stroke + background together (inherited). |
| `Margin` | `Margin` | yes | Outer spacing (inherited). |

```csharp
// Simple stroked box
new Border()
    .SetStrokeColor(SKColors.Red)
    .SetStrokeThickness(2)
    .AddChild(new Label().SetText("Red border").SetMargin(new Margin(12)));

// Dashed outline
new Border()
    .SetStrokeColor(PlusUiDefaults.BorderColor)
    .SetStrokeThickness(2)
    .SetStrokeType(StrokeType.Dashed)
    .AddChild(new Label().SetText("Dashed outline").SetMargin(new Margin(12)));

// Filled rounded card
new Border()
    .SetBackground(PlusUiDefaults.BackgroundControl)
    .SetCornerRadius(8)
    .SetStrokeColor(PlusUiDefaults.AccentPrimary)
    .SetStrokeThickness(2)
    .AddChild(new Label().SetText("Card style").SetMargin(new Margin(16)));
```

**Gotchas**
- Single child only — adding more works but only the first is arranged properly.
- Stroke adds to total size (`2 * StrokeThickness` across both sides); it is layout, not internal padding. Use child `Margin` for inset padding.
- Default stroke is 1px solid black — barely visible on dark themes. Always set `StrokeColor` + `StrokeThickness` explicitly.
- `Dashed`/`Dotted` only render when `StrokeThickness > 0`.
- `CornerRadius` rounds both stroke and background together — you cannot round only one.
- Stroke is drawn behind child content, so it never overlaps the child.

---

## Separator — divider line

A decorative line dividing sections, horizontal or vertical.

| Property | Type | Bind* | Notes |
|---|---|---|---|
| `Orientation` | `Orientation` | yes | `Horizontal` (default) or `Vertical`. Auto-sets alignment (see gotchas). |
| `Color` | `Color` | yes | Line color. Default `PlusUiDefaults.BorderColor` (light gray). |
| `Thickness` | `float` | yes | Line thickness px; clamped `>= 0`. Default `1`. |
| `Margin` | `Margin` | yes | Outer spacing (inherited). |
| `IsVisible` | `bool` | yes | Default `true`. |
| `Opacity` | `float` | yes | `0.0`–`1.0`. Default `1.0`. |

```csharp
// Default thin horizontal divider in a VStack
new Separator();

// Accent divider
new Separator().SetThickness(3).SetColor(PlusUiDefaults.AccentPrimary);

// Vertical divider in a toolbar HStack
new Separator().SetOrientation(Orientation.Vertical).SetMargin(new Margin(8, 4));
```

**Gotchas**
- Setting `Orientation` overrides alignment: `Horizontal` → `HorizontalAlignment.Stretch` (clears vertical); `Vertical` → `VerticalAlignment.Stretch` (clears horizontal). Set `Orientation` first, then any manual alignment — otherwise the alignment is reset.
- In unbounded containers (e.g. `ScrollView`), a vertical separator sizes to `Thickness` height, not full height — use `SetDesiredHeight` to force full height.
- Negative `Thickness` is silently clamped to `0` (invisible, no exception).
- Default light-gray color may be hard to see on light backgrounds; check contrast.
- Not focusable; `AccessibilityRole.None` — screen readers ignore it.

---

## Common LLM mistakes

- **Passing `Set*` config into a stack constructor.** `VStack`/`HStack` constructors are `params UiElement[]` only — children, not config. Configure with fluent methods afterward.
- **Expecting `Wrap` to work without a size constraint.** `VStack.Wrap` needs `SetDesiredHeight`; `HStack.Wrap` needs a width bound (`SetDesiredWidth` or a bounded parent). And `Wrap` is off by default.
- **Confusing wrap directions.** `VStack` wraps into columns (overflow flows right); `HStack` wraps into rows (overflow flows down).
- **Treating `Star` as a percentage.** Grid star sizing is weights — `Column.Star, 1` and `Column.Star, 2`, not `0.5`/`1.0`.
- **Forgetting to define Grid rows/columns**, then being surprised by the auto single `Auto` row/column; or omitting `row`/`column` on `AddChild` (defaults to 0,0).
- **Misreading `UniformGrid.SetDesiredSize`** as cell size — it is the total grid size; cells are `total / dimensions`.
- **Adding multiple children to `ScrollView` or `Border`.** Both are single-child wrappers — nest a `VStack`/`HStack` for multiple items. `ScrollView.Children` is empty by design.
- **Omitting the viewport size on `ScrollView`** — without `DesiredHeight`/`DesiredWidth` it expands instead of scrolling.
- **Setting `Scrollbar.Value` directly** — it is read-only; drive it via `UpdateScrollState`/`HandleDrag`, and remember pixel offset = `Value * maxScrollOffset`.
- **Relying on the default `Border` stroke** (1px black) being visible on dark themes — always set `StrokeColor` and `StrokeThickness`.
- **Setting `Separator` alignment before `Orientation`** — `Orientation` resets it. Set orientation first.
- **Using `Spacing` and `Margin` interchangeably.** `Spacing` is between children only; `Margin` is outer (or per-child) spacing.
