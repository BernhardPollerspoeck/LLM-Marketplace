# PlusUi Menus, Toolbars & Overlays Reference

Controls that surface commands and transient UI on top of page content: the application **Menu** bar, right-click **ContextMenu**, the **Toolbar** app bar, hover **Tooltip**s, and modal **Popups**. All are fluent (`Set*`/`Bind*`) and follow PlusUi's MVVM-with-`ICommand` model.

Three of these are not normal layout children. They are rendered by overlay services on top of the page:

- **Menu** dropdowns, **ContextMenu**, **Tooltip**, and the Toolbar **overflow menu** all require `IOverlayService` (registered by default). If the service is missing, opening silently no-ops.
- **Popups** are managed by `IPopupService` and shown imperatively from a ViewModel.

`MenuItem` is the shared data object for **Menu** and **ContextMenu** (it is a plain C# class, not a `UiElement`).

## Table of contents

- [Menu](#menu)
- [MenuItem (shared)](#menuitem-shared)
- [ContextMenu](#contextmenu)
- [Toolbar](#toolbar)
- [Tooltip](#tooltip)
- [Popups (UiPopupElement)](#popups-uipopupelement)
- [Common LLM mistakes](#common-llm-mistakes)

---

## Menu

Horizontal application menu bar. Top-level `MenuItem`s render as bar labels; clicking one opens a dropdown overlay built from that item's sub-items. `Menu : UiLayoutElement, IInputControl, IHoverableControl`.

| Property | Type | Default | Bind? | Notes |
|---|---|---|---|---|
| `HoverBackgroundColor` | `Color` | `BackgroundSecondary` | yes | Bar item hover fill. |
| `ActiveBackgroundColor` | `Color` | `BackgroundHover` | yes | Fill of the currently open bar item. |
| `TextColor` | `Color` | `TextPrimary` | yes | Bar label color. |
| `TextSize` | `float` | `FontSize` | yes | Bar label font size. |
| `Background` | `IBackground` | `BackgroundPrimary` | yes | Inherited; bar background. |
| `Margin` / `IsVisible` / `Opacity` | — | — | yes | Inherited from `UiElement`. |

Builder: `AddItem(MenuItem)` (top-level only). The bar holds `MenuItem`s; their sub-items render in the dropdown.

```csharp
new Menu()
    .AddItem(new MenuItem()
        .SetText("File")
        .AddItem(new MenuItem().SetText("New").SetShortcut("Ctrl+N").SetCommand(vm.NewCommand))
        .AddItem(new MenuItem().SetText("Open").SetShortcut("Ctrl+O").SetCommand(vm.OpenCommand))
        .AddSeparator()
        .AddItem(new MenuItem().SetText("Exit").SetCommand(vm.ExitCommand)))
    .AddItem(new MenuItem()
        .SetText("Edit")
        .AddItem(new MenuItem().SetText("Cut").SetShortcut("Ctrl+X").SetCommand(vm.CutCommand))
        .AddItem(new MenuItem().SetText("Copy").SetShortcut("Ctrl+C").SetCommand(vm.CopyCommand)));
```

Conditional/parameterized submenu:

```csharp
new MenuItem()
    .SetText("Recent Files")
    .BindIsEnabled(() => vm.HasRecentFiles)
    .AddItem(new MenuItem().SetText("Document1.txt").SetCommand(vm.OpenRecentCommand).SetCommandParameter("doc1.txt"))
    .AddItem(new MenuItem().SetText("Document2.txt").SetCommand(vm.OpenRecentCommand).SetCommandParameter("doc2.txt"))
    .AddSeparator()
    .AddItem(new MenuItem().SetText("Clear Recent").SetCommand(vm.ClearRecentCommand));
```

**Gotchas**
- `Menu` has **no** `AddSeparator()` — only `MenuItem` does. Separators appear only in dropdowns, never on the bar.
- A top-level `MenuItem` with no sub-items (`HasSubItems == false`) opens nothing when clicked.
- Hovering a different bar item auto-switches the open dropdown; clicking the already-open item closes it. Not configurable.
- Returns focus to the bar after a command runs; manage custom focus in your handler.

---

## MenuItem (shared)

Data object used by **Menu** and **ContextMenu** dropdowns. Not a `UiElement` — all state is exposed only through `Set*`/`Bind*` fluent methods (the backing properties are `internal`).

| Member | Type | Bind? | Notes |
|---|---|---|---|
| `SetText` | `string` | yes | Display label. |
| `SetIcon` | `string` | yes | Icon name — see gotcha (not rendered yet). |
| `SetShortcut` | `string` | yes | Decorative shortcut text only. |
| `SetCommand` | `ICommand` | yes | Action on click (respects `CanExecute`). |
| `SetCommandParameter` | `object` | yes | Passed to the command. |
| `SetIsEnabled` | `bool` | yes | Disabled items are grayed and skip clicks. |
| `SetIsChecked` | `bool` | yes | Renders a checkmark (no auto mutual-exclusion). |
| `AddItem(MenuItem)` | — | — | Nested submenu item. |
| `AddItem(MenuSeparator)` / `AddSeparator()` | — | — | Visual divider in the dropdown. |
| `HasSubItems` | `bool` (read) | — | `true` when `Items.Count > 0`. |

**Gotchas**
- `SetShortcut(...)` is **display-only text** — it does not bind keyboard input. Wire real hotkeys separately.
- `SetIcon(...)` is stored but **not drawn** currently (overlay rendering skips icons).
- `SetIsChecked(true)` shows a check but has no radio/toggle logic — manage state in your command.
- `MenuItem` instances are data, not UI; don't expect layout/measure on them directly.

---

## ContextMenu

Popup menu opened on **right-click** (desktop) or **long-press** (touch); dismissed by outside-click or **Escape**. `ContextMenu : UiElement`. Attached to any element via the `SetContextMenu` extension — never added as a child.

| Property | Type | Default | Bind? | Notes |
|---|---|---|---|---|
| `HoverBackgroundColor` | `Color` | `BackgroundHover` | yes | Item hover fill. |
| `TextColor` | `Color` | `TextPrimary` | yes | Item text color. |
| `Background` | `IBackground` | `BackgroundPrimary` | yes | Popup background (inherited). |
| `CornerRadius` | `float` | — | yes | Popup corners (inherited). |
| `Margin` / `Opacity` | — | — | yes | Inherited. |
| `IsOpen` | `bool` (read-only) | — | no | Framework-controlled; cannot be set. |

Attach by instance, by builder action, or via the `SetContextMenu*` styling extensions on the host element:

```csharp
new Border()
    .SetBackground(PlusUiDefaults.BackgroundControl)
    .SetCornerRadius(8)
    .AddChild(new Label().SetText("Right-click me"))
    .SetContextMenu(new ContextMenu()
        .AddItem(new MenuItem().SetText("Cut").SetShortcut("Ctrl+X").SetCommand(vm.CutCommand))
        .AddItem(new MenuItem().SetText("Copy").SetShortcut("Ctrl+C").SetCommand(vm.CopyCommand))
        .AddSeparator()
        .AddItem(new MenuItem().SetText("Delete").SetCommand(vm.DeleteCommand)));

// Builder-action overload (no intermediate variable):
new Button()
    .SetText("Click me")
    .SetContextMenu(m => m
        .SetHoverBackgroundColor(new Color(50, 80, 120))
        .SetTextColor(Colors.White)
        .AddItem(new MenuItem().SetText("Option 1"))
        .AddItem(new MenuItem().SetText("Option 2")));

// Host-level styling extensions:
element
    .SetContextMenuBackground(new Color(30, 30, 30))
    .SetContextMenuHoverBackgroundColor(new Color(50, 80, 120))
    .SetContextMenuTextColor(Colors.White)
    .SetContextMenu(new ContextMenu()
        .SetCornerRadius(4)
        .AddItem(new MenuItem().SetText("Edit"))
        .AddItem(new MenuItem().SetText("Delete")));
```

**Gotchas**
- `IsOpen` is read-only and `Open(Point)`/`Close()` are internal — you cannot show/hide it programmatically; only user gesture opens it.
- A `ContextMenu` with zero items does not display.
- The `SetContextMenu(Action<ContextMenu>)` action runs **immediately** during configuration, not at open time.
- `IsFocusable` is false — context menus stay out of the normal focus flow.

---

## Toolbar

Horizontal app bar with left/right item slots, a center title (or custom content), groupable icons, and responsive overflow collapse. `Toolbar : UiLayoutElement<Toolbar>`. **Has no default height or background — it is invisible until you set both.**

| Property | Type | Default | Bind? | Notes |
|---|---|---|---|---|
| `Title` | `string` | `null` | yes | Bar title. |
| `TitleFontSize` | `float` | `20` | yes | |
| `TitleColor` | `Color` | `TextPrimary` | yes | |
| `TitleAlignment` | `TitleAlignment` (`Center`/`Left`) | `Center` | yes | Affects overflow space split. |
| `ItemSpacing` | `float` | `Spacing` (~8) | yes | Gap between items/sections. |
| `OverflowBehavior` | `OverflowBehavior` (`None`/`CollapseToMenu`/`Scroll`) | `None` | yes | `None` lets items overlap/clip. `Scroll` is defined but not yet implemented in `Toolbar`. |
| `OverflowThreshold` | `float` | `600` | yes | Width at which collapse activates. |
| `OverflowIcon` | `string` | `"more_vert"` | yes | Overflow button icon. |
| `OverflowMenuBackground` | `Color` | `BackgroundPrimary` | yes | Dropdown background. |
| `OverflowMenuItemBackground` | `Color` | `BackgroundSecondary` | yes | |
| `OverflowMenuItemHoverBackground` | `Color` | `Color(70,70,70)` | yes | |
| `OverflowMenuItemTextColor` | `Color` | `TextPrimary` | yes | |
| `DesiredHeight` | `float` | `0` (unset) | yes | **Must set** — no default. |
| `Background` | `IBackground`/`Color` | Transparent | yes | **Must set** for visibility. |
| `Padding` | `Margin` | `0` | yes | Inherited. |

Item builders: `AddLeft(UiElement)`, `AddRight(UiElement)`, `AddLeftGroup(ToolbarIconGroup)`, `AddRightGroup(ToolbarIconGroup)`, `SetCenterContent(UiElement)`.

`ToolbarIconGroup`: `AddIcon(UiElement)`, `SetSeparator(bool)`, `SetSeparatorColor/Width/Margin`, `SetSpacing(float)`, `SetPriority(int)` — all with `Bind*` counterparts.

```csharp
// Basic app bar
new Toolbar()
    .SetTitle("My App")
    .SetDesiredHeight(56)
    .SetBackground(new Color(98, 0, 238))
    .AddLeft(new Button().SetIcon("menu").SetTextColor(Colors.White))
    .AddRight(new Button().SetIcon("search").SetTextColor(Colors.White));

// Grouped icons with a separator + responsive overflow
new Toolbar()
    .SetTitle("Document Editor")
    .SetDesiredHeight(48)
    .SetBackground(PlusUiDefaults.BackgroundControl)
    .AddLeftGroup(new ToolbarIconGroup()
        .AddIcon(new Button().SetIcon("undo"))
        .AddIcon(new Button().SetIcon("redo"))
        .SetSeparator(true)
        .SetPriority(10))          // higher priority collapses last
    .AddRight(new Button().SetText("Save"))
    .AddRight(new Button().SetText("Export"))
    .SetOverflowBehavior(OverflowBehavior.CollapseToMenu);

// Styled overflow dropdown (dark)
new Toolbar()
    .SetTitle("Actions")
    .SetDesiredHeight(56)
    .SetBackground(new Color(45, 45, 45))
    .AddRight(new Button().SetText("Action 1"))
    .AddRight(new Button().SetText("Action 2"))
    .SetOverflowBehavior(OverflowBehavior.CollapseToMenu)
    .SetOverflowMenuBackground(new Color(35, 35, 35))
    .SetOverflowMenuItemHoverBackground(new Color(65, 65, 65))
    .SetOverflowMenuItemTextColor(Colors.White);
```

**Gotchas**
- No default height/background → invisible. Always `SetDesiredHeight()` + `SetBackground()`.
- `SetCenterContent()` **replaces** the title; you can't show both.
- `OverflowBehavior` defaults to `None` (items overlap/clip). Set `CollapseToMenu` explicitly for responsive bars.
- `SetPriority` only applies to `ToolbarIconGroup`. Loose `AddLeft`/`AddRight` items are priority 0 and collapse in order.
- `OverflowBehavior.Scroll` exists in the enum but is **not implemented** in `Toolbar` — it behaves like `None`. Use `CollapseToMenu` for responsive bars.
- Overflowed items are wrapped/cloned into the dropdown — direct button-state mutations may not sync to the original.
- Use `SetIcon()` (not `SetText()`) for icon-only buttons for correct sizing. Overflow item clicks auto-close the menu.

---

## Tooltip

Hover popup with text or rich UI content. Not a control — an attachment (`TooltipAttachment`) added to any `UiElement` via `SetTooltip*` extensions; rendered by an internal `TooltipOverlay`.

| Property | Type | Default | Bind? | Notes |
|---|---|---|---|---|
| `Content` | `string` \| `UiElement` | `null` | yes | Required; null/empty → no tooltip. |
| `Placement` | `TooltipPlacement` | `Auto` | yes | `Auto`/`Top`/`Bottom`/`Left`/`Right`. |
| `ShowDelay` | `int` (ms) | `500` | yes | Negative clamped to 0. |
| `HideDelay` | `int` (ms) | `0` | yes | Negative clamped to 0. |

Extensions: `SetTooltip(string)`, `SetTooltip(UiElement)`, `SetTooltip(Action<TooltipAttachment>)`, plus `SetTooltipPlacement/ShowDelay/HideDelay` and `BindTooltipContent/Placement/ShowDelay/HideDelay`.

```csharp
// Simple text
new Button().SetText("Save").SetTooltip("Saves the current document");

// Configured placement
new Button().SetText("Delete")
    .SetTooltip(t => t
        .SetContent("This cannot be undone")
        .SetPlacement(TooltipPlacement.Bottom));

// Rich content
new Button().SetText("Info")
    .SetTooltip(new VStack()
        .SetSpacing(2)
        .AddChild(new Label().SetText("Keyboard shortcut").SetFontWeight(FontWeight.Bold))
        .AddChild(new Label().SetText("Ctrl+S").SetTextColor(PlusUiDefaults.TextSecondary)));
```

**Gotchas**
- `Content` must be non-null/non-empty or nothing shows.
- `Auto` does not guarantee a side — it resolves by available screen space (tries Top → Bottom → Right → Left), clamped 4px from window edges.
- Default `ShowDelay` is 500ms; set to 0 for instant display.
- The overlay is non-interactive (`HitTest` returns null) — tooltips can't be clicked/focused.
- Each tooltip owns the `UiElement` you pass — don't reuse one element across multiple tooltips.

---

## Popups (UiPopupElement)

Modal dialogs overlaying page content: confirmations, forms, alerts, detail views, with typed argument/result. Subclass `UiPopupElement<TArg>` (no result) or `UiPopupElement<TArg, TResult>`. Shown via injected `IPopupService`. `UiPopupElement : UiElement, IInputControl`.

| Member | Type | Bind? | Notes |
|---|---|---|---|
| `Argument` | `TArg?` | no | Input data; set before `Build()`, safe to read there. |
| `Result` | `TResult?` | no | Output; call `SetResult(...)` **before** `Close(true)`. |
| `ViewModel` | `INotifyPropertyChanged` | no | Passed to base ctor; auto-set as `Context` for the UI tree. |
| `Build()` | override | — | Returns the popup UI tree. |
| `Close(bool success)` | override | — | Calls `popupService.ClosePopup(success)`. |
| `Background` | `IBackground` | yes | Overlay backdrop (default semi-transparent black). |
| `HorizontalAlignment`/`VerticalAlignment` | enum | yes | Default `Center` (auto-centered). |
| `Margin`/`Opacity`/`IsVisible` | — | yes | Inherited. |

Configured at show-time via `IPopupConfiguration` (not properties): `CloseOnBackgroundClick` (default true), `CloseOnEscape` (default true), `BackgroundColor`.

Define a popup (arg-only):

```csharp
public class ConfirmPopup(ConfirmPopupViewModel vm) : UiPopupElement<string>(vm)
{
    protected override UiElement Build() =>
        new Border()
            .SetBackground(new SolidColorBackground(new Color(45, 45, 45)))
            .SetCornerRadius(16)
            .SetPadding(new Margin(24))
            .SetDesiredWidth(300)
            .AddChild(new VStack(
                new Label().SetText("Confirm").SetTextSize(20).SetFontWeight(FontWeight.Bold),
                new Label().BindText(nameof(vm.Message), () => vm.Message).SetMargin(new Margin(0, 16, 0, 24)),
                new HStack(
                    new Button().SetText("Cancel").SetOnClick(() => Close(false)),
                    new Button().SetText("OK").SetOnClick(() => Close(true))
                ).SetHorizontalAlignment(HorizontalAlignment.Right)));

    public override void Close(bool success)
    {
        var popupService = ServiceProviderService.ServiceProvider?.GetRequiredService<IPopupService>();
        popupService?.ClosePopup(success);
    }
}
```

Arg + result variant — set the result before closing:

```csharp
public class PropertyEditorPopup(PropertyEditorPopupViewModel vm)
    : UiPopupElement<PropertyDto, PropertyEditorResult>(vm)
{
    public override void Close(bool success)
    {
        if (success) SetResult(vm.GetResult());  // MUST precede base.Close(true)
        base.Close(success);
    }

    protected override UiElement Build() =>
        new VStack(
            new Label().SetText($"Edit {Argument?.Name ?? "Property"}"),
            new DataGrid<PropertyFieldViewModel>().SetItemsSource(vm.Fields),
            new HStack(
                new Button().SetText("Cancel").SetCommand(vm.CancelCommand),
                new Button().SetText("Save").SetCommand(vm.SaveCommand)))
        .SetBackground(new Color(40, 40, 40))
        .SetDesiredWidth(340)
        .SetCornerRadius(12);
}
```

Register (both popup + VM) and show:

```csharp
// ConfigureServices
services.AddTransient<ConfirmPopup>();
services.AddTransient<ConfirmPopupViewModel>();

// From a ViewModel holding IPopupService _popupService:
_popupService.ShowPopup<ConfirmPopup, string>(
    arg: "Are you sure?",
    onClosed: () => DeleteItem(),
    configure: config =>
    {
        config.CloseOnBackgroundClick = true;
        config.CloseOnEscape = true;
        config.BackgroundColor = new Color(0, 0, 0, 180);
    });

_popupService.ShowPopup<PropertyEditorPopup, PropertyDto, PropertyEditorResult>(
    property,
    onClosed: async result => await OnPropertyEditedAsync(property, result),
    configure: config => config.BackgroundColor = new Color(0, 0, 0, 180));
```

Lifecycle: override `Appearing()` (init on show) and `Disappearing()` (cleanup on close). Only one popup shows at a time — opening a new one closes the previous.

**Gotchas**
- `onClosed` fires only when `Close(true)`. `Close(false)` (Cancel) dismisses **without** the callback.
- `SetResult(...)` must run **before** `Close(true)` or the callback receives null.
- Register **both** the popup and its VM, or `ShowPopup` throws `InvalidOperationException`.
- Set `CloseOnBackgroundClick`/`CloseOnEscape`/`BackgroundColor` through the `configure` action, not as element properties.
- Don't call `BuildPopup()`/`SetArgument()` yourself — the service does. Only override `Build()` and `Close()`.
- Escape handling is automatic when `CloseOnEscape` is true; don't handle keys manually.

---

## Common LLM mistakes

- **Calling `Menu.AddSeparator()`** — doesn't exist. Only `MenuItem.AddSeparator()`; separators show only in dropdowns.
- **Expecting `SetShortcut(...)` to bind keys.** It is decorative text; wire real hotkeys separately.
- **Expecting `MenuItem.SetIcon(...)` to render.** Icons are stored but not drawn yet.
- **Trying to open/close a `ContextMenu` in code.** `IsOpen` is read-only; it opens only on right-click/long-press.
- **Forgetting Toolbar height/background.** Without `SetDesiredHeight()` + `SetBackground()` the bar is invisible.
- **Leaving `OverflowBehavior` at `None` and expecting responsive collapse.** Set `CollapseToMenu`.
- **Putting `SetPriority` on loose toolbar items.** Priority only affects `ToolbarIconGroup`.
- **Null/empty tooltip content** → no tooltip. Also: `Auto` placement is space-resolved, not guaranteed.
- **Popup `SetResult` after `Close(true)`** → null result. Set it first.
- **Forgetting to register the popup's ViewModel** (or the popup) in DI → `InvalidOperationException` at `ShowPopup`.
- **Expecting `onClosed` on Cancel.** Only `Close(true)` invokes it.
